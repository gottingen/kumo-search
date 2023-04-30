common runtime executor
===

# 概念

执行器是TF的核心中的核心了，前面做了这么多的准备工作，最后要在这里集大成了，想想还有点小激动。不过笔者在这里先打个预防针，执行器的概念多、结构复杂，想要透彻理解并不容易，为了保持文章的易读性，我们也是尽量对细枝末节做了舍弃，以求反应执行器的核心本质，但无奈执行器涉及到的内容实在太多，因此本篇的篇幅可能会有点长，大家做好准备。
执行器的概念虽然复杂，但宏观上的理解却很简答，给定一张待执行的图，给定它的输入，让它按照计划执行，获得输出就好了。如果读者对之前我们这个系列的内容有所了解，对于执行器的执行过程，应该能有一个大概的印像了。为了计算图能够执行，TF设计了op的概念，设计了实际执行op的kernel，构建了能够表达计算内容的node和graph，对内存、设备给出了专门的管理类，这些结合在一起，为计算图的执行提供了最全面的支持。但具体的执行中，还有非常多的细节需要处理，接下来我们将分两节进行介绍，第一节介绍executor.h头文件，它给出了执行器提供的对外接口，第二部分介绍executor.cc源文件，它给出了执行器的执行原理。

# API

```c++
Graph* graph = ...;//构建图
Executor* executor;
NewSimpleExecutor(my_device, graph, &executor);//生成执行器
Rendezvous* rendezvous = NewNaiveRendezvous();//构建通信通道
rendezvous->Send("input", some_input_tensor);//提供输入
executor->Run({ExecutorOpts, rendezvous, nullptr});
rendezvous->Recv("output",&output_tensor);//获得输出
```

过程非常的简单易懂，TF通过抽象给我们提供了易用的外部API，但这种易用性是以底层复杂的内部结构作为支持的，接下来我们就看一下，对外API方面TF都做了哪些工作

## Executor

首先，当然是执行器本身。执行器本身提供的接口很简单，如下：

```c++
class Executor {
  public:
    typedef std::function<void(const Status&)> DoneCallback;
    virtual void RunAsync(const Args& args, DoneCallback done) = 0;
    
    //RunAsync()函数的同步版本
    Status Run(const Args& args){
        Status ret;
        Notification n;
        RunAsync(args, [&ret, &n](const Status& s) {
            ret = s;
            n.Notify();
        });
        n.WaitForNotification();
        return ret;
    }
};

```

执行器本质上应当是异步执行的，这个我们可以理解，因为图计算是一个非常复杂且漫长的过程，异步计算效率更高。但同时执行器也提供了异步计算的同步包装，让用户可以用同步的方式来执行。
执行器接口的简洁性，与执行器的复杂功能之间形成了巨大的反差，以至于我们不得不怀疑，执行器内部是不是隐藏了什么结构，果然，我们发现执行函数的第一个参数是Args，下面来看下它的结构：

```c++
struct Args {
    int64 step_id = 0;
    Rendezvous* rendezvous = nullptr;
    StepStatsCollector* stats_collector = nullptr;
    FunctionCallFrame* call_frame = nullptr;
    CancellationManager* cancellation_manager = nullptr;
    SessionState* session_state = nullptr;
    TensorStore* tensor_store = nullptr;
    ScopedStepContainer* step_container = nullptr;
    
    //如果为真，在设备上调用Sync函数
    bool sync_on_finish = false;
    
    typedef std::function<void()> Closure;
    typedef std::function<void(Closure)> Runner;
    Runner runner = nullptr;
    
    //每当一个节点完成执行的时候，都会调用这个回调函数
    typedef std::function<Status(const string& node_name, const int output_slot, const Tensor* tensor, const bool is_ref, OpKernelContext* ctx)> NodeOutputsCallback;
};
```
关于其中的参数，我们给出一些说明：
* step_id是一个进程级别的唯一标识符，用来标识执行的步骤。当一个步骤运行了一个需要在多个设备上执行的op时，这些不同设备上的执行器将会收到相同的step_id。step_id是被用来追踪一个步骤中用到的资源的。
* RunAsync()函数使用rendezvous，作为与计算图之间沟通输入和输出的机制；
* RunAsync()调用stats_collector来收集统计信息。这允许我们能根据需求收集统计和traces信息。
* 如果执行器被用来执行一个函数，那么RunAsync()可以使用call_frame，用来在调用者和被调用者之间传递参数和返回值。
* RunAsync()可以使用cancellation_manager来注册一些，在计算图执行被取消后的回调函数。
* RunAsync()将执行的闭包分配给runner，通常来说，runner背后都有一个线程池支持。

## NewLocalExecutor

TF教我们怎样生成一个本地的执行器，它需要用到下面这个函数：
```c++
::tensorflow::Status NewLocalExecutor(const LocalExecutorParams& params, const Graph* graph, Executor** executor);
```
这里面又出现了一个，我们未曾见过的结构，LocalExecutorParams，顾名思义，它是我们生成本地执行器需要的参数，这个类的结构如下：

```c++
struct LocalExecutorParams {
    Device* device;
    FunctionLibraryRuntime* function_library = nullptr;
    std::function<Status<const NodeDef&, OpKernel**)> create_kernel;
    std::function<void(OpKernel*)> delete_kernel;
    Executor::Args::NodeOutputsCallback node_outputs_cb;
};
```
它包含了设备、函数库、kernel构造和删除过程、节点执行完毕的回调函数，后面我们将会看到，在函数的实现里面，是怎样利用这些信息构建执行器的。

## ExecutorBarrier

在实际的应用中，我们可能需要用到不止一个执行器。为了使多个执行器能并行运行，我们需要对这些同时执行的执行器进行管理和统筹，于是就产生了ExecutorBarrier类。如下：

```c++
class ExecutorBarrier {
  public:
    typedef std::function<void(const Status&)> StatusCallback;
    
    //为num个不同的执行器进行统筹和管理，r是一个共享的数据传输通道，如果任意一个执行器失败，rendezvous仅会崩溃一次。等最后一个执行器执行完毕时，会调用done，并且ExecutorBarrier对象会被删除掉
    ExecutorBarrier(size_t num, Rendezvous* r, StatusCallback done);
    
    //返回一个执行器在执行完毕之后必须调用的函数闭包，执行器会使用它们结束时的状态作为执行闭包的参数
    StatusCallback Get() {
        return std::bind(&ExecutorBarrier::WhenDone, this, std::placeholders::_1);
    }
  private:
    Rendezvous* rendez_ = nullptr;
    StatusCallback done_cb_ = nullptr;
    
    mutable mutex mu_;
    int pending_ GUARDED_BY(mu_) = 0;//还剩几个执行器没执行完
    Status status_ GUARDED_BY(mu_);
    
    void WhenDone(const Status& s){
        //...
    }
};
```
# 内部类

## 数据结构

在图构建的时候，为了方便操作，提供更多的功能呢，我们把很多结构设计的比较复杂，比如`graph`, `node`等，但在执行的时候，一则这些复杂的结构我们不一定用得上，二则它们的存在也会影响执行效率，因此TF就设计了很多对之前复杂结构的简化，比如我们这一节将要介绍的`EdgeInfo`和`NodeItem`，以及下一节将要介绍的`GraphView`。

首先我们来看下`EdgeInfo`：

```c++
struct EdgeInfo {
    int dst_id;
    int output_slot:31;
    bool is_last:1;
    int input_slot;
};
```
显然，它表示的是计算图中的边，包含了目的节点（dst_id），目的节点的端口号（output_slot），源节点的端口号（input_slot），之所以没有包含源节点，我们猜测是因为这个结构体就是被包含在源节点内部的。
另外，is_last表示，这条边对应的是不是目的节点的最后一个端口。
最后，int output_slot:31这个结构，表示接下来的这四个字节（int）共32个bit，output_slot仅占其中的31个，而接下来的这个bool is_last:1则占了最后一个bit位，这种定义方式是c++11之后才有的，可以更高效的利用存储空间。
接下来我们看一下`NodeItem`这个结构，它表示计算图中的一个节点：
```c++
struct NodeItem {
    const Node * node = nullptr;//表示一个计算图中的节点
    
    OpKernel* kernel = nullptr;//这个节点对应的OpKernel
    
    bool kernel_is_expensize:1;
    bool kernel_is_async:1;
    bool is_merge:1;
    bool is_enter:1;
    bool is_exit:1;
    bool is_exit:1;
    bool is_control_trigger:1;
    bool is_sink:1;
    bool is_enter_exit_or_next_iter:1;
    
    int num_inputs;
    int num_outputs;
    
    int input_start = 0;//输入的起始索引
    
    size_t num_output_edges;//输出边的数量
    
    PendingCounts::Handle pending_id;
    
    const EdgeInfo* output_edge_list() const { return output_edge_base(); }
    
    const EdgeInfo& output_edge(int i);
    
    DataType input_type(int i);
    DataType output_type(int i);
    
    const AllocatorAttributes* output_attrs();
  
  private:
    char* var();
    EdgeInfo output_edge_base();
    AllocatorAttributes* output_attr_base();
    uint8* input_type_base();
    uint8* output_type_base();
}
```
NodeItem提供了对于计算图节点的静态信息的非常详细的描述。

## GraphView
为了执行的效率，执行器对一些基础结构进行了简化，剔除了不必要的信息，例如，对于计算图来说，由于在执行过程中，不需要对图结构进行更改，因此原来的Graph类中很多修改图的接口都没用了，所以TF提供了一个不可改变的视图，用来使图的执行更加高效。
下面我们来看下这个类的接口和数据：
```c++
class GraphView {
  public:
    GraphView(): space_(nullptr) {}
    void Initialize(const Graph* g);//GraphView初始化
    Status SetAllocAttrs(const Graph* g, const Device* device);
    NodeItem* node(size_t id) const;//返回指定的节点信息
  private:
    char* InitializeNode(char* ptr, const Node* n);//初始化节点信息
    size_t NodeItemBytes(const Node* n);
    
    int32 num_nodes_ = 0;
    uint32* node_offsets_ = nullptr;//节点的偏置，node_offsets_[id]保存了节点id在space_中的偏移量
    char* space_;//保存了指向NodeItem对象的存储地址的指针
};
```
所以，从数据上来说就很清楚了，`GraphView`之所以是`Graph`的一个不可改变的视图，是因为它分配了一块内存空间，然后把图中所有节点的信息（NodeItem）都依次存入这个空间中，并提供了对空间中信息进行检索的接口，但是，没有提供对这些信息进行修改的接口，所以，我们仍然能够访问到Graph中的任何静态信息，但是无法对其进行修改。

## ExecutorImpl
Executor类只是一个基类，真正的执行器实现，需要看它的子类，TF提供了一个实现类ExecutorImpl，它的结构仍然比较简单：

```c++
class ExecutorImpl : public Executor {
  public:
    ExecutorImpl(const LocalExecutorParams& p, const Graph* g) : params_(p), graph_(g), gview_(){
        CHECK(p.create_kernel != nullptr);
        CHECK(p.delete_kernel != nullptr);
    }
    ~ExecutorImpl() override {
        for(int i=0;i<graph_->num_node_ids();i++){
            NodeItem* item = gview_.node(i);
            if(item != nullptr){
                params_.delete_kernel(item->kernel);
            }
        }
        for(auto fiter : frame_info_){
            delete fiter.second;
        }
        delete graph_;
    }
    
    Status Initialize();
    
    //处理当前图中的每一个节点，尝试分析出它们在分配内存时的内存分配属性
    Status SetAllocAttrs();
    
    void RunAsync(const Args& args, DoneCallback done) override;

  private:
    //构建控制流信息
    static Status BuildControlFlowInfo(const Graph* graph, ControlFlowInfo* cf_info);
    
    //初始化待执行计数信息
    void InitializePending(const Graph* graph, const ControlFlowInfo& cf_info);
    
    //确认每一个FrameInfo都已准备好
    FrameInfo* EnsureFrameInfo(const string& fname){
        auto slot = &frame_info_[fname];
        if(*slot == nullptr){
            *slot = new FrameInfo;
        }
        return *slot;
    }
    
    //被当前的对象拥有
    LocalExecutorParams params_;
    const Graph* graph_;
    GraphView gview_;
    
    //对于params_的缓存
    bool device_record_tensor_accesses_ = false;
    
    //没有任何输入边的根节点，它们应当组成初始预备队列
    std::vector<const Node*> root_nodes_;
    
    //从帧名称到帧信息的映射
    gtl::FlatMap<string, FrameInfo*> frame_info_;
};
```
为了说明细节，我们特意给出了部分函数的实现方式，对于其中的重点进行如下说明：

* 关于析构函数，它一共做了三件事情，第一，利用GraphView找到每个node包含的OpKernel，并且将它删除，第二，将所有的帧信息删除，第三，将GraphView对象删除。
* 当前执行器实际拥有的对象有三个，一是LocalExecutorParams执行器生成时的参数，二是Graph*，对应图的指针，注意执行器仅拥有这个指针，并不拥有这张图，第三，GraphView，这是执行器完全拥有的结构。
* 看到root_nodes_这个变量，应该会给我们一些启发，图的执行过程，是从一些不需要输入的根节点出发的，根据节点之间的依赖关系依次执行，这个过程会用到队列的数据结构，一旦一个队列中某个节点的前驱节点都准备好了，这个节点就可以被执行了。
* frame_info_是一个帧映射，图执行过程中的帧信息主要是为了控制结构准备的，控制结构的加入使得TF真正从一个高效的计算引擎升级为一个类编程语言，关于它的说明将在下文中给出。


另外，这个类中也包含了我们之前没有见过的两种结构，ControlFlowInfo和FrameInfo，下面依次介绍它们的结构：

```c++
struct ControlFlowInfo {
    gtl::FlatSet<string> unique_frame_names;
    std::vector<string> frame_names;
};
struct FrameInfo {
    //帧的输入数量
    int input_count;
    
    //帧的各节点输入张量数量的总和
    int total_inputs;
    
    //决定了在我们最终创建的pending_counts数据结构中，接下来将要被分配内存的位置
    PendingCounts::Layout pending_counts_layout;
    
    //每个帧都包含了它自己的PendingCounts信息，只为了当前帧中的节点
    PendingCounts* pending_counts;
    
    //帧中的节点，仅在调试时使用
    std::vector<const Node*>* nodes;
};
```
ControlFlowInfo只包含了帧的名称，只不过提供了set和vector两种方式，set是为了更方便的查找某个帧的名称是否被包含在内。而FrameInfo则包含了帧的详细信息，主要是输入数量，以及未完成的节点计数等信息。
接下来是一些函数的具体实现，本来不应该纠结与细节，但这些内容对于理解执行器相关类的执行原理非常重要，因此这里给出直观解释，并不详解代码。感兴趣的读者可以去阅读源码。

```c++
//GraphView类相关

//对于其包含的每个NodeItem，调用其析构函数，并且删除相关指针对应的内存
GraphView::~GraphView();

//计算某个Node对应的NodeItem所需要的内存大小
size_t GraphView::NodeItemBytes(cost Node *n);

//初始化节点
char* GraphView::InitializeNode(char* ptr, const Node* n);

//初始化GraphView，主要是初始化了node_offsets_和space_两个指针
void GraphView::Initialize(const Graph* g);

//设置内存分配的属性
Status GraphView::SetAllocAttrs(const Graph* g, const Device* device);

//ExecutorImpl类相关

//初始化执行器，首先初始化GraphView，然后构建帧的信息，预处理图中每个节点以便为op创造OpKernel，最后初始化PendingCounts信息
Status ExecutorImpl::Initialize();
```
## ExecutorState
在执行器的执行图计算的时候，需要一个结构来保存当前计算的即时信息，TF为此设计了类ExecutorState，它被用来保存每一个对ExecutorImpl::Run调用的状态信息。它会在一个节点已经准备好之后调度这个节点，并且保存每个节点尚未完成的输入信息。
下面让我们先来看一下这个类的结构：

```c++
class ExecutorState {
  public:
    ExecutorState(const Executor::Args& args, ExecutorImpl* impl);
    void RunAsync(Executor::DoneCallback done);
  private:
    DeviceContextMap device_context_map_;
    
    typedef gtl::InlinedVector<TaggedNode, 8> TaggedNodeSeq;
    typedef gtl::InlinedVector<Entry, 4> EntryVector;
    
    const bool vlog_;
    const bool log_memory_;
    int64 step_id_;
    
    //未拥有
    Rendezvous* rendezvous;
    SessionState* session_state_;
    TensorStore* tensor_store_;
    //每个执行步级别的容器
    ScopedStepContainer* step_container_;
    StepStatesCollector* stats_collector_;
    
    checkpoint::TensorSliceReaderCacheWrapper* slice_reader_cache_;
    FunctionCallFrame* call_frame;
    const ExecutorImpl* impl_;
    CancellationManager* cancellation_manager_;
    Executor::Args::Runner runner_;
    bool sync_on_finish_;
    
    //拥有
    bool dumped_on_error_ = false;
    //当前执行步骤开始的帧
    FrameState* root_frame_;
    //执行器结束时需要调用的回调函数
    Executor::DoneCallback done_cb_;
    std::atomic_int_fast32_t num_outstanding_ops_;
    mutex mu_;
    Status status_ GUARDED_BY(mu_);
    
    //从帧名称到实际帧的映射。在当前帧的某个迭代周期内，可能会产生一个新的帧。新的子帧的唯一键值必须由父帧的名称、迭代编号、以及由nodedef推断出来的新帧的名称组合而成
    gtl::FlatMap<string, FrameState*> outstanding_frames_ GUARDED_BY(mu_);
    
    //一个帧的名称
    inline string MakeFrameName(FrameState* frame, int64 iter_id, const string& name);
    
    //找到一个现存的帧，或者创建一个新帧，在帧frame的iter迭代周期
    void FindOrCreateChildFrame(FrameState* frame, int64 iter, const Node* node, FrameState** child);
    
    //删除一个帧，当帧调用结束时使用
    void DeleteFrame(FrameState* frame, TaggedNodeSeq* ready);
    
    //清除那些起源于帧frame和迭代iter的帧，当一个子帧结束时调用
    void CleanupFramesIterations(FrameState* frame, int64 iter, TaggedNodeSeq* ready);
    
    //在当前的线程中处理一个已准备好的节点
    void Process(TaggedNode node, int64 scheduled_usec);
    
    //在调用item->kernel之前，先填入其输入
    Status PrepareInputs(const NodeItem& item, Entry* first_input, TensorValueVec* inputs, DeviceContextVec* input_device_contexts, AllocatorAttributeVec* input_alloc_attrs, bool* is_input_dead);
    
    //在item->kernel计算结束之后，处理输出
    Status ProcessOutputs(const NodeItem& item, OpKernelContext* ctx, EntryVector* outputs, NodeExecStats* stats);
    
    //在处理完输出之后，将输入传递给下一个输入
    void PropagateOutputs(const TaggedNode& tagged_node, const NodeItem* item, EntryVector* outputs, TaggedNodeSeq* ready);
    
    //节点计算结束后，接管stats，如果执行完成则返回true
    bool NodeDone(const Status& s, const Node* node, const TaggedNodeSeq& ready, NodeExecStats* stats, TaggedNodeReadyQueue* inline_ready);
    
    //调度ready中的所有复杂节点，然后将ready中的非复杂节点放入inline_ready
    void ScheduleReady(const TaggedNodeSeq& ready, TaggedNodeReadyQueue* inline_ready);
    
    //仅用作调试或记录
    inline void MaybeMarkCompleted(FrameState* frame, int64 iter, int64 id);
    
    //输出一个未完成或者活跃节点的信息
    void DumpPendingNodeState(const int node_id, const Entry* input_vector, bool show_nodes_with_no_ready_inputs);
    void DumpActiveNodeState(const FrameState* frame, IterationState* iteration);
    
    //提供执行器的状态信息
    void DumpState();
    const Tensor* GetTensorValueForDump(const Entry& input);
    
    //当执行器结束执行时，清理
    void Finish();
};
```
从API上来看，ExecutorState几乎担当了执行器的职责，从后面的介绍也可以看出，实际上确实如此。执行器内部实际调用的就是ExecutorState内部的API。从类的结构中，我们还是看到了许多未曾相识的结构，下面我们先一一分析这些类的意义和结构。

首先来看Entry，Entry要么是一个张量指针，要么是一个张量值，为计算图中的节点的输入或输出提供了一种统一的类型。

```c++
struct Entry {
    Entry(const Entry& other);
    Entry& operator=(const Entry& other);
    
    void ClearVal();//清除val字段
    ManualConstructor<Tensor> val;//一个张量的值，如果val_filed_is_set是true的话
    Tensor* ref = nullptr;//一个张量引用
    mutext* ref_mu = nullptr;//为上述张量引用的互斥量
    bool has_value = false;//值是否存在，不论是val或者ref
    bool val_filed_is_set = false;//val字段是否被设置
    
    AllocatorAttributes alloc_attr;//为当前的张量分配内存的内存分配器的属性
    
    DeviceContext* device_context = nullptr;//包含了关于这个张量如何创建的设备相关的信息
};
```
接下来看看IterationState，它代表了一轮迭代的状态。

```c++
struct IterationState {
  public:
    //一轮迭代的状态，每个迭代轮次都由一个单独的拷贝。对于第k轮迭代，第i个节点的第j个输入在input_tensors[k][impl_->nodes[i].input_start+j]。注意，没有必要对input_tensors做互斥锁，其中的内容只会被边的前一个节点写入，被边的后一个节点擦除，而每条边的前后两个节点是不可能同时运行的
    Entry* input_tensors;
    
    //每一轮迭代中未完成的op数量
    size_t outstanding_ops;
    
    //每一轮迭代中未完成的帧数量
    int outstanding_frame_count;
    int pending(PendingCounts::Handle h);
    int decrement_pending(PendingCounts::Handle int v);
    
    //标记一个merge节点为live
    void mark_live(PendingCounts::Handle h);
    //标记一个节点为处理开始
    void mark_started(PendingCounts::Handle h);
    //标记一个节点为处理结束
    void mark_completed(PendingCounts::Handle h);
    //获取节点状态
    PendingCounts::NodeState node_state(PendingCounts::Handle h);
    int dead_count(PendingCounts::Handle h);
    void increment_dead_count(PendingCounts::Handle h);
    void adjust_for_activation(PendingCounts::Handle h, bool increment_dead, int* pending_result, int* dead_result);
  
  private:
    PendingCounts counts_;
};
```
接下来是FrameState，代表了一个帧的状态。对于帧和迭代轮次，有以下几点需要说明：

* 对于计算图中的循环来说，每个循环都需要创建一个新的帧。执行从第0个迭代开始。当第0个迭代的某个数值通过了一个NextIteration节点时，第1轮迭代就被创建并开始运行了。注意这时第0轮迭代可能仍在进行，所以多轮迭代可能会同时在运行。帧保持了多种数据结构来保存每轮迭代的状态。当第0轮迭代结束后，我们对其对应的状态进行垃圾回收。
* 一个帧，当它的所有输入都已经被传入，所有的迭代都被计算完成时，这个帧就被认为是完成了，可以被进行垃圾回收了。
* 一个帧保存了其中每一轮迭代的状态。如果以下三个条件都被满足，那么第i轮迭代就会被认为是已经完成了，第一，第i轮迭代已经没有未完成的节点了，第二，所有该轮的接收操作都已经完成了，第三，第i-1轮已完成。对于第0轮迭代，当帧的所有输入都已完成，我们就认为它已经结束了。
* 帧和迭代轮次在结束后，都会进行垃圾回收。我们需要保存的状态量，跟调度器允许的并行度高度相关。我们希望调度器能够动态的控制未完成的并行帧和迭代的数量。为了减少内存消耗，调度器可能需要优先调度内层的帧和较低的迭代轮次。
* 帧的状态一般总是在需要的时候才会被初始化，因此我们没有引入额外的损耗。

下面我们来具体看下FrameState的结构：

```c++
struct FrameState {
    const ExecutorImpl* executor = nullptr;//帧所在的执行器
    string frame_name;//当前帧的名称，是父帧，迭代轮次，和frame_name字段拼合起来得到的
    uint64 frame_id;//当前帧的唯一标识
    int64 parent_iter = -1;//父帧的迭代轮次，frame_name和parent_iter共同标识了当前的FrameState
    FrameState* parent_frame = nullptr;//父帧的FrameState
    const int 
    
    max_parallel_iterations;//最大允许的并行迭代数量
    int num_pending_inputs = 0;//当前帧仍然在等待的输入数量
    int64 iteration_count GUARDED_BY(mu) = 0;//当前帧中到达过的最大的迭代数量
    int num_outstanding_iterations GUARDED_BY(mu) = 1;//未完成的迭代数量
    
    gtl::InlinedVecotr<IterationState*,12> iterations;//当前帧活跃的迭代状态
    std::vector<std::pair<const Node*, Entry>> next_iter_roots GUARDED_BY(mu);
    std::vector<std::pair<const Node*, Entry>> inv_values GUARDED_BY(mu);
    std::vector<const Node*> dead_exits GUARDED_BY(mu);
    
    //属于当前帧的静态信息
    PendingCounts* pending_counts = nullptr;
    int total_input_tensors = 0;
    std::vector<const Node*>* nodes = nullptr;
    
    void InitializeFrameInfo(const string& enter_name);
    inline IterationState* GetInteration(int64 iter);
    inline void SetIteration(int64 iter, IterationState* state);
    
    //减少未完成的操作数量，清理帧中的迭代信息。如果帧执行结束则返回true
    inline bool DecrementOutputstandingOps(const GraphView* gview, int64 iter, TaggedNodeSeq* ready);
    inline bool DecrementOutstandingOpsLocked(const GraphView* gview, int64 iter, TaggedNodeSeq* ready);
    
    //如果帧中的计算都已经完成则返回true
    inline bool IsFrameDone();
    //如果迭代的计算已经结束则返回true
    bool IsIterationDone(int64 iter);
    //增加迭代的编号，如果是一个新迭代，就初始化它
    void IncrementIteration(const GraphView* gview, TaggedNodeSeq* ready);
    //激活一个新的迭代轮次中所有的NextIteration节点
    void ActivateNexts(const GraphView* gview, int64 iter, TaggedNodeSeq* ready);
    void ActivateLoopInvs(const GraphView* gview, int64 iter, TaggedNodeSeq* ready);
    void AddLoopInv(const NodeItem* item, const Entry& value, TaggedNodeSeq* ready);
    void ActivateNodes(const NodeItem* item, const bool is_dead, int64 iter, EntryVector* outputs, TaggedNodeSeq* ready);
    bool CleanupIterations(const GraphView* gview, int64 iter, TaggedNodeSeq* ready);
};
```

最后让我们来看下最后的两个结构体，TaggedNode和TaggedNodeReadyQueue。其中TaggedNode非常简单，就是一个<frame, iter, node>的结构体，而后者就是前者的一个Queue，用来表示已经准备好的节点的队列。

```c++
struct TaggedNode {
    const Node* node = nullptr;
    FrameState* input_frame = nullptr;
    int64 input_iter = -1;
    bool is_dead = false;
    
    TaggedNode(const Node* t_node, FrameState* in_frame, int64 in_iter, bool dead);
};
class TaggedNodeReadyQueue {
  public:
    void push_back(TaggedNode node);
    void pop_front();
    bool empty();
    const TaggedNode* begin();
    const TaggedNode* end();
  private:
    gtl::InlinedVector<TaggedNode, 16> ready_;
    int front_index_;
};

```
关于`TaggedNodeReadyQueue`，我们要说明一下，本来这里很自然的可以使用`std::deque`这个标准的双端列表来实现的，但因为在待运行序列中我们通常并没有太多的节点，所以为了效率我们只使用了一个`vector`来实现，并且省去了节点消耗后释放空间的麻烦。

## details

终于快要接近终点了。在前文中我们讲了那么多结构，最终计算图的执行过程究竟是怎样的，我们仍然不得而知。因为具体的实现细节都隐藏在函数的实现中，而我们上文中全部都在探讨接口。现在我们就来看下，具体的实现方法。
首先，执行器的入口是Run函数，先来看下`ExecutorImpl`中的`Run`函数是如何实现的吧。

```c++
void ExecutorImpl::RunAsync(const Args& args, DoneCallback done){
    (new ExecutorState(args,this))->RunAsync(std::move(done));
}
```
这验证了我们上文中提到的，ExecutorImpl仍然只是一个接口，真正的执行是被推到ExecutorState类中完成的。在上述函数中，我们首先定义了一个ExecutorState对象，然后调用了它的RunAsync函数。在构造函数中，首先初始化了root_frame和iteration 0，我们具体看看RunAsync是如何实现的：

```c++
void ExecutorState::RunAsync(Executor::DoneCallback done){
    const Graph* graph = impl_->graph_;//获取计算图指针
    TaggedNodeSeq ready;//构建ready节点序列
    
    //让设备填充设备上下文映射
    Device* device = impl_->params_.device;
    Status fill_status = device->FillContextMap(graph, &device_context_map_);
    if(!fill_status.ok()){
        done(fill_status);
        return;
    }
    
    //初始化ready队列
    for(const Node* n : impl_->root_nodes){
        DCHECK_EQ(n->in_edges().size(),0);
        ready.push_back(TaggedNode{n,root_frame_,0,false});
    }
    if(ready.empty()){
        done(Status::OK());
    } else {
        num_outstanding_ops = ready.size();
        root_frame_->iterations[0]->outstanding_ops = ready.size();
        done_cb_ = std::move(done);
        ScheduleReady(ready,nullptr);
    }
}
```
可见，主要做了两件事，第一是初始化了ready queue，第二是启动了ScheduleReady函数。
下面我们再来看一下SheduleReady函数的运行机制：

```c++
void ExecutorState::ScheduleReady(const TaggedNodeSeq& ready, TaggedNodeReadyQueue* inline_ready){
    if(ready.empty()) return ;
    
    int64 scheduled_usec = 0;
    if(stats_collector_){
        scheduled_usec = nodestats::NowInUsec();
    }
    if(inline_ready == nullptr){
        //在线程池中调度所有已经准备好的op
        for(auto& tagged_node : ready){
            runner_([=]() {Process(tagged_node, scheduled_usec);});
        }
        return;
    }
    const GraphView& gview = impl_->gview_;
    const TaggedNode* curr_expensive_node = nullptr;
    for(auto& tagged_node : ready){
        const NodeItem& item = *gview.node(tagged_node.node->id());
        if(tagged_node.is_dead || ! item.kernel_is_expensive){
            //内联化这个非复杂节点
            inline_ready->push_back(tagged_node);
        } else {
            if(curr_expensive_node){
                //将复杂节点丢给其它线程去做，因为当前线程还有很多事情要做
                runner_(std::bind(&ExecutorState::Process, this, *curr_expensive_node, scheduled_usec));
            }
            curr_expensive_node = &tagged_node;
        }
    }
    if(curr_expensive_node){
        if(inline_ready->empty()){
            //尾递归优化
            inline_ready->push_back(*curr_expensive_node);
        } else {
            //我们仍然有内联节点需要运行，因此把这个复杂节点丢给其它线程去运行
            runner_(std::bind(&ExecutorState::Process, this, *curr_expensive_node, scheduled_usec));
        }
    }
}
```
这个函数包含了两个输入，一个是待执行的节点队列，一个是待执行的内联节点序列。一共分两种情况处理：

* 第一种情况，inline_ready队列为空，这种情况下，我们会为ready队列中的每一个节点，单独新增一个执行线程，这也是ExecutorState::RunAsync函数调用调度函数时的执行方式。也就是说，执行的起点是，对于根执行队列中的节点，分别新增一个线程来执行；
* 第二种情况，inline_ready队列非空，这种情况下，我们需要明确一点，调度函数不会进行任何实际的执行，只会对执行进行分配。它会遍历ready中的每个节点，如果这个节点是非复杂节点或者节点已死亡，就放入inline_ready队列待执行，否则就单独开启一个线程执行它，同时，这个遍历过程进行完之后，会保留最后一个复杂节点（curr_expensive_node），这时候如果inline_ready队列是空的，就把这个复杂节点放入内联队列，否则就开启一个线程执行。

下面来看下Process函数，它是整个执行的核心，这个函数包含的代码量比较大，因为我们的核心目标是说明执行过程，所以细枝末节暂时略去，仅保留框架：
```c++
void ExecutorState::Process(TaggedNode tagged_node, int64 scheduled_usec){
    const GraphView& gview = impl_->gview_;
    TaggedNodeSeq ready;
    TaggedNodeReadyQueue inline_ready;
    
    //为OpKernel::Compute准备参数
    
    inline_ready.push_back(tagged_node);
    while(!inline_ready.empty()){
        //信息提取、记录处理
        
        //当这个节点是非死亡节点，或者这个节点是send/recv这样的数据传输节点时，才执行这个节点，对于传输节点，我们需要把dead这个字节传输下去
        bool launched_asynchronously = false;
        if(tagged_node.is_dead & !IsTransferNode(node)){
            outputs.resize(item.num_outputs);
        } else {
            //调用PreparedInputs准备输入
            //设置计算参数
            if(item.kernel_is_async){
                AsyncOpKernel* async = item.kernel->AsAsync();
                launched_asynchronously = true;
                AsyncState* state = new AsyncState(params, tagged_node, &item, first_input, stats);
                
                auto done = [this, state](){
                    //调用ProcessOutputs处理输出
                    //清理输入
                    //调用PropagateOutputs传递输出
                    //调用NodeDone清理战场
                };
                device->ComputeAsync(async, &state->ctx, done);
            } else {
                device->Compute(op_kernel,&ctx);
                //调用ProcessOutputs处理输出
            }//同步处理结束
        }//非死亡、非传输节点处理结束
        if(!launched_asynchronously){
            //清理输入
            //调用PropagateOutputs传递输出
            //调用NodeDone打扫战场
        }//后续处理结束
    }//while循环结束
}
```

这个函数把节点计算分为同步和异步计算分别处理，且同样遵循以下的处理方式：

```mermaid
graph LR
PrepareInput-->Compute
Compute-->ProcessOutput
ProcessOutput-->PropagateOutput
PropagateOutput-->NodeDone
```

下面我们分别来看一下这四个函数的实现，首先是准备输入函数和输出处理函数

```c++
Status ExecutorState::PrepareInputs(const NodeItem& item, Entry* first_input, TensorValueVec* inputs, DeviceContextVec* input_device_contexts, AllocatorAttributeVec* input_alloc_attrs, bool* is_input_dead);

Status ExecutorState::ProcessOutputs(const NodeItem& item, OpKernelContext* ctx, EntryVector* outputs, NodeExecStats* stats);
```
这两个函数没有什么花花肠子，只是对输入和输出的属性进行设置和填充，比较琐碎，感兴趣的读者可以去看看源码。
比较要紧的是推广输出和节点完成这两个函数，我们先看一下推广输出的函数：

```c++
void ExecutorState::PropagateOutputs(const TaggedNode& tagged_node, const NodeItem* item, EntryVector* outputs, TaggedNodeSeq* ready){
    //沿着输出边传递输出，把新准备好的节点放入ready队列
    
    //判断当前节点的类型，选择合适的处理方法，期间会调用ActivateNodes，DecrementOutstandingOpsLocked，AddLoopInv，DecrementOutstandingOps，IncrementIteration等函数进行处理
    
    //节点处理完成后，判断当前帧是否执行完毕，以及递归的判断父帧有没有执行完毕
}
```
其中，在ActivateNodes函数中，有这样一个结构：

```c++
for(size_t out_index=0; out_index<num_output_edges;out_index++){
    //其它处理
    if(dst_ready){
        if(dst_item->is_control_trigger)
            dst_dead = false;
        ready->push_back(TaggedNode(dst_item->node, this, iter, dst_dead));
        iter_state->outstanding_ops++;
    }
}
```
也就是说，在激活节点的时候，会顺势把该节点加入待执行队列ready。这一点很重要，因为在Process函数中，我们是有一个while循环对inline_ready队列（也就是激活节点函数中的ready队列）做处理的，但刚开始这个队列中只有一个节点，正式因为在PropagateOutputs函数中会调用ActivateNodes函数，不断向ready中添加节点，才使得这个while循环能够跑起来。
最后再看一下，节点完成这个函数：
```c++
bool ExecutorState::NodeDone(const Status& s, const Node* node, const TaggedNodeSeq& ready, NodeExecStats* stats, TaggedNodeReadyQueue* inline_ready){
    //其它处理
    if(s.ok()){
        ScheduleReady(ready, inline_ready);
    }
    return completed;
}
```
感觉又回到开头了？还记得刚开始讲到过的，`ExecutorState::RunAsync`函数中就调用了`ScheduleReady`这个函数吗？这个函数究竟起到了什么样的作用呢？
现在我们有必要把上述的过程总结一下了，这个过程也是本篇最核心的内容了：

* ExecutorImpl::RunAsync作为执行器的入口，其实它是把实际执行的工作交给了ExecutorState::RunAsync，这个函数进一步调用了ScheduleReady函数来调度执行，记住这个函数有两个输入，ready和inline_ready，在这里，inline_ready是空的。也就是说，这里调用ScheduleReady的作用是，给根节点队列里的节点，分别分配一个线程执行，执行的过程调用的是Process函数。
* 在Process函数内部，我们还要记住一点，这个函数的输入只有节点，没有ready和inline_ready，这两个变量都是在Process函数内部新创建的。也就是说，一旦把一个节点交给Process函数去处理，这个节点所在的队列跟Process函数就没有任何关系了。处理的过程分为输入准备、实际计算、输出准备、输出传递、节点完成五个步骤。其中只有输出传递和节点完成会对ready和inline_ready结构产生影响。
* 我们把节点分按照异步节点和同步节点分开处理。对于异步节点，NodeDone函数的最后一个参数inline_ready是空，也就是说，在异步执行时，调用NodeDone中的ScheduleReady时，跟RunAsync中的情形是一样的，直接调度ready中的节点就好了，不需要处理inline_ready的情况。对于同步节点，NodeDone函数的最后一个参数inline_ready是当前Process函数中新创建的inline_ready，也就是说，传递给ScheduleReady的inline_ready是非空的，这也就有可能对inline_ready的结构做修改，注意这里的inline_ready是从Process函数中创建的，每个Process函数都对应一个全新的线程，也就是说，每个全新的线程里面只有一个inline_ready结构，其中的函数不断的修改它的内容，然后不断的对它进行调度执行。注意Process中的while大循环是针对inline_ready队列执行的。
# 类图


# 文件

* [exector.h](../tensorflow/core/common_runtime/executor.h)
* [exector.cc](../tensorflow/core/common_runtime/executor.cc)
* [exector_factory.h](../tensorflow/core/common_runtime/exector_factory.h)