common runtime direct session
===

# 概念

读过之前文章的读者应该还记得，[session](session.md)是一个执行代理。我们把计算图和输入交给session，由它来调度执行器，执行计算产生结果。TF给我们提供了一个最简单的执行器direction_session。按照当前的理解，我们觉得direction_session的实现应该是非常简单而直接的，毕竟执行器的复杂结构我们在executor那篇已经见到了。但实际上，问题的难点在于，有时候我们只是希望以计算图中某些节点为输入，某些节点为输出，来执行图中的一小部分计算，而不需要执行整张图，另外一个方面，这种对图部分执行的任务，在同一张图上可能同时存在多个。为了应对这种情况，direct_session就衍生出了很多辅助数据。

# direct_session

## API
DirectSession类提供了丰富的数据和接口，以下为了表达简洁，我们略去了部分函数的形参：
```c++
class DirectSession : public Session {
  public:
    DirectionSession(const SessionOptions& options, const Device* device_mgr, DirectSessionFactory* factory);
    
    Status Create(const GraphDef& graph) override;
    Status Extend(const GraphDef& graph) override;
    Status Run(...) override;//运行图
    
    Status PRunSetup(...);//部分运行图准备
    Status PRun(...);//部分运行图
    
    Status Reset(const std::vector<string>& containers);//清空device_mgr中的containers，如果containers本身就是空的，那么清空默认容器
    
    Status ListDevice(...) override;
    Status Close() overrides;
    Status LocalDeviceManager(const DeviceMgr** output) overrides;
    
    void ExportCostModels(...);

  private:
    Status MaybeInitializeExecutionState(...);//给定graph之后，如果执行器状态没有初始化，则初始化基础的执行器状态
    
    Status GetOrCreateExecutors(...);//对于一组给定的输入和输出，在一个给定的执行器集合中检索，是否存在合适的执行器，如果没有，则创造一个
    
    Status CreateGraphs(...);//给定graph_def_和设备，以及输入和输出，创造多张图，这些新创建的图共享一个公共的函数库flib_def
    
    Status ExtendLocked(const GraphDef& graph);//Extend的内部执行类
    
    Status ResourceHandleToInputTensor(...);
    
    Status SendPRunInputs(...);//将更多的输入提供给执行器，启动后续的执行
    
    Status RecvPRunOutputs(...);//从执行器中获取更多的输出，它会等待直到输出张量计算完成
    
    Status CheckFetch(...);//检查需求的输出能否根据给定的输入计算出来
    
    Status WaitForNotification(...);
    Status CheckNotClosed();
    
    const SessionOptions options_;
    
    //设备相关的结构
    const std::unique_ptr<const DeviceMgr> device_mgr_;
    std::vector<Device*> devices_;
    DeviceSet device_set_;
    
    string session_handle_;
    bool graph_created_ GUARDED_BY(graph_def_lock_) = false;
    mutex graph_def_lock_;
    GraphDef graph_def_ GUARDED_BY(graph_def_lock_);
    
    std::vector<std::pair<thread::ThreadPool*, bool>> thread_pools_;//被用来执行op的线程池，用一个布尔值来标志，是否拥有这个线程池
    
    Status init_error_;
    
    bool sync_on_finish_ = true;//如果为真，阻塞线程直到设备已经完成了某个步骤内的所有队列中的操作
    void SchedClosure(thread::ThreadPool* pool, std::function<void()> c);//在线程池中调度c
    
    mutex executor_lock_;//保护执行器
    
    std::unordered_map<string, std::shared_ptr<ExecutorsAndkeys>> executor_ GUARDED_BY(executor_lock_);//由签名映射到它的执行器，签名包括了部分执行图的输入和输出，由这两个就能唯一确定一个部分执行图
    
    std::unordered_map<string, std::shared_ptr<RunState>> partial_runs_ GUARDED_BY(executor_lock_);//从签名到部分执行状态，每一个部分执行都会有一个专门保存其状态的结构
    
    SessionState session_state_;//保存了所有当前在会话中正在存活的张量
    
    DirectSessionFactory* const factory_;
    CancellationManager* cancellation_manager_;
    
    std::unordered_map<string, string> stateful_placements_ GUARDED_BY(graph_def_lock_);//对于有状态的节点（比如params和queue），保存节点名称到节点所在设备的映射，一旦这些节点被放置在了某个设备上，是不允许再移动的
    
    std::unique_ptr<SimpleGraphExecutionState> execution_state_ GUARDED_BY(graph_def_lock_);//放置整张图时使用
    
    std::unique_ptr<FunctionLibraryDefinition> flib_def_;//在任何的重写或优化之前的函数库，特别是，CreateGraphs函数会修改函数库
    
    mutex closed_lock_;
    bool closed_ GUARDED_BY(closed_lock_) = false;//如果会话已经被关闭，则为true
    
    //为这个会话生成唯一的名字
    std::atomic<int64> edge_name_counter_ = {0};
    std::atomic<int64> handle_name_counter_ = {0};
    
    static std::atomic_int_fast64_t step_id_counter_;//为所有的会话生成唯一的step id
    
    const int64 operation_timeout_in_ms_ = 0;//全局对阻塞操作的超时阈值
    
    CostModelManager cost_model_manager_;//为当前会话中执行的图管理所有的损失模型
}
```
可见，DirectSession里面的很多内容都是为部分执行准备的。由于计算图仅是一个计算的规划，我们可以通过为同一张图选取不同的输入和输出，来执行不同的计算。而不同的计算需要不同的执行器，也需要不同的存储结构来保存各个计算的当前状态。为此，TF专门给出了几个结构体，首先我们来看一下对不同计算执行器的封装：

```c++
//为每一个partition准备的执行器和函数运行时库
struct PerPartionExecutorAndLib {
    Graph* graph = nullptr;
    std::unique_ptr<FunctionLibraryRuntime> flib;
    std::unique_ptr<Executor> executor;
};

//为每一次计算提供的数据结构
struct ExecutorsAndKeys {
    std::atomic_int_fast64_t step_count;
    std::unique_ptr<Graph> graph;
    NameNodeMap name_to_node;
    std::unique_ptr<FunctionLibraryDefinition> flib_def;
    std::vector<PerPartitionExecutorsAndLib> items;
    std::unordered_map<string, size_t> input_name_to_index;
    std::unordered_map<string, string> input_name_to_rendezvous_key;
    std::unordered_map<string, size_t> output_name_to_index;
    std::unordered_map<string, string> output_name_to_rendezvous_key;
    
    DataTypeVector input_types;
    DataTypeVector output_types;
};
```
对于一张计算图来说，我们的每一次计算的执行，不论是完整图的计算还是部分图的计算，都有可能是跨设备的，因此都需要先做节点放置，把图的节点分割到不同的设备上，每一个设备上放置了一个图的partition，每个partition有对应的运行时函数库和执行器。而对于每一种计算来说，我们需要一个vector把不同partition的信息存储起来。
另外，刚才提到我们还需要为每一次计算提供保存当前状态的结构，下面就来看一下：

```c++
//对于每一个partition内的执行，会话保存了一个RunState
struct RunState {
    mutex mu_;
    Status status GUARDED_BY(mu_);
    IntraProcessRendezvous* rendez = nullptr;
    std::unique_ptr<StepStatsCollector> collector;
    Notification executors_done;
    std::unordered_map<string, bool> pending_inputs;//如果已经提供了输入，则为true
    std::unordered_map<string, bool> pending_outputs;//如果已经获得了输出，则为true
    TensorStore tensor_store;
    ScopedStepContainer step-container;
    //...
};

struct RunStateArgs {
    RunStateArgs(const DebugOption& options) : debug_options(options) {}
    bool is_partial_run = false;
    string handle;
    std::unique_ptr<Graph> graph;
    const DebugOptions& debug_options;
};
```
其中，RunState为每一个partition的执行提供了状态保存的功能，而RunStateArgs则为前者提供了用于调试的参数和配置。

## 内部类

在源文件里，给出了DirectSessionFactory的定义，它提供了对于DirectSession进行生成和管理的功能，简要摘录如下：
```c++
class DirectSessionFactory : public SessionFactory {
  public:
    Session* NewSession(const SessionOptions& options) override;
    Status Reset(...) override;
    void Deregister(const DirectSession* session);
  private:
    mutex session_lock_;
    std::vector<DirectSession*> session_ GUARDED_BY(sessions_lock_);//用于存储生成的DirectSession
};
```
另外，还提供了一个对于直接工厂注册的类：

```c++
class DirectSessionRegistrar {
  public:
    DirectSessionRegistrar() {
        SessionFactory::Register("DIRECT_SESSION", new DirectSessionFactory());
    }
};
static DirectSessionRegistrar registrar;
```
下面，我们会按照顺序对DirectSession内重要的函数，进行拆解，由于部分函数细节比较多，除了核心代码之外，我们仅给出功能解释：

```c++
DirectSession::DirectSession(const SessionOptions& options, const DeviceMgr* device_mgr, DirectSessionFactory* const factory){
    //根据options准备线程池
    //根据device_mgr准备device_和device_set_和每个设备的op_segment()
}

Status DirectSession::Run(...){
    //提取对于当前会话的本次运行的输入的名称
    //检查对于所需的输入输出，是否已经存在现成的执行器
    //构造一个调用帧（call frame），方便会话与执行器之间传递输入和输出
    //创建一个运行时状态的结构（RunState）
    //开始并行执行，核心代码如下
    for(const auto& item : executors_and_keys->items){
        item.executor->RunAsync(args, barrier->Get());
    }
    //获取输出
    //保存本次运行中我们希望保存的输出张量
    //创建并返回损失模型（cost model）
    //如果RunOptions中有相关配置，输出分割后的图
}

Status DirectSession::GetOrCreateExecutors(...){
    //快速查找路径
    //慢查找路径，对输入和输出做排序，使得相同输入和输出集合会得到相同的签名
    //如果未找到，则创建这个执行器并缓存
    //构建执行图，核心代码如下
    CreateGraphs(options, &graphs, &ek->flib_def, run_state_args, &ek->input_types, &ek->output_types));
    //为各子图准备运行时环境
}

Status DirectSession::CreateGraphs(...){
    //前期预处理
    //图分割算法，核心代码如下
    Partition(popts, &client_graph->graph, &partitions);
    //检查分割结果的有效性
    //图优化遍历，核心代码如下
    OptimizationPassRegistry::Global()->RunGrouping(OptimizationPassRegistry::POST_PARTITIONING, optimization_options);
    //允许设备重写它拥有的子图
}
```
可见，具体的执行过程是在Run函数内部，调用executor->RunAsync函数来实现的，在具体执行之前，我们还需要通过GetOrCreateExecutors函数获得执行器，在这个函数内部，我们通过CreateGraphs函数对原图进行了分割，并利用图优化遍历算法对图进行了优化。

# 文件

* [direct_session.h](../tensorflow/core/common_runtime/direct_session.h)
* [direct_session.cc](../tensorflow/core/common_runtime/direct_session.cc)
* [session_factory.h](../tensorflow/core/common_runtime/session_factory.h)
* [session.h](../tensorflow/core/public/session.h)
