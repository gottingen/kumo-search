common runtime device
===

# 概念 

在framework部分，介绍了DeviceAttributes和DeviceBase两个结构，这些其实是为了要介绍的Device类做准备的。可以去回顾下前面讲过的[内容](framework_device.md)。Device类只是对DeviceBase类的继承，没有添加更多新的数据成员，但提供了Compute计算接口。DeviceSet是一个设备集合类，而DeviceMgr与DeviceSet的不同点在于，它提供了设备管理的功能，为设备查找和计数提供了便利的数据结构。最后，DeviceFactory是为了产生某种类型的设备准备的工厂类，同样的设备类型（比如CPU）会对应不同的工厂，意味着不同的实现，而不同的工厂有着不同的权重。这里的权重是为了辅助我们选择某种类型的设备用的。

# device

Device类，除了包含对内部私有数据的访问API之外，还包含了核心的计算API Compute，我们先来看一下它的结构：

```c++
class Device : public DeviceBase {
  public:
    virtual void Compute(OpKernel* op_kernel, OpKernelContext* context){
        op_kernel->Compute(context);
    }
    virtual void ComputeAsync(AsyncOpKernel* op_kernel, OpKernelContext* context, AsyncOpKernel::DoneCallback done){
        op_kernel->ComputeAsync(context, std::move(done));
    }
    //...
  private:
    const DeviceAttributes device_attributes_;
    DeviceNameUtils::ParsedName parsed_name_;
    OpSegment op_seg_;
    ResourceMgr* rmgr_ = nullptr;
}
```
TF对于设备名称是有要求的，它必须满足这种格式：`/job:_/replica:_/task:_/(gpu|cpu):_`，举个例子：`/job:train/replica:0/task:3/gpu:2`。其中，`Device`类的数据成员`parsed_name_`就是对这种设备名称的拆解，感兴趣的读者可以自行看下`ParsedName`的定义。`ResourceMgr`和`OpSegment`在`framework`部分也都介绍过了。所以从数据角度讲，`Device`没有什么新鲜的，只是对原有的关于设备的数据做了一个整合。但从API的角度讲，它包含了一个计算接口`Compute`，实际上也就是对`OpKernel`中的`Compute`接口的封装。

# device_set
`DeviceSet`是一个容器类，用于管理一个模型使用的不同设备。这个类相对比较简单，我们看它的结构：
```c++
class DeviceSet {
  public:
    //...
  private:
    std::vector<Device*> devices_;
    std::unordered_map<string, Device*> device_by_name_;
    Device* client_device_ = nullptr;
}
```
其中，device_by_name_是一个从设备全称到设备指针的映射，而client_device_是我们从devices_中挑选的，默认的客户端设备。

# device_mgr
`DeviceMgr`顾名思义是一个设备管理类，其实它主要是提供了一系列数据结构来提高API的效率，比如，我们要查找一个给定设备名的设备指针，或者要对某种类型的设备计数。针对这种高频操作，DeviceMgr为其准备了高效的数据结构。类的结构如下：
```c++
class DeviceMgr {
  public:
    //...
  private:
    typedef gtl::InlinedVector<Device*, 8> DeviceVec;
    DeviceVec devices_;
    std::unordered_map<StringPiece, Device*, StringPiece::Hasher> device_map_;
    core::Arena name_backing_store_;
    std::unordered_map<string, int> device_type_counts_;
}
```
device_map_是为了提高查找指定名称的设备的效率，device_type_counts_是为了提高查找指定类型的设备数的效率。

# DeviceFactory

正如刚才提到过的，DeviceFactory代表了某种设备（比如CPU）的某种实现的工厂类。下面我们看下DeviceFactory类的结构：

```c++
class DeviceFactory {
  public:
    static void Register(const string& device_type, DeviceFactory* factory, int priority);
    static DeviceFactory* GetFactory(const string& device_type);
    static Status AddDevices(const SessionOptions& options, const string& name_prefix, std::vector<Device*>* devices);
    static Device* NewDevice(const string& type, const SessionOptions& options, const string& name_prefix);
    virtual Status CreateDevices(const SessionOptions& options, const string& name_prefix, std::vector<Device*>* devices) = 0;
    static int32 DevicePriority(const string& device_type);
};
```
看完这个类，我们感觉很疑惑，它提供了很多的API，但是没有数据成员，那它注册的那些工厂，存储在哪里呢？
别慌，我们在device_factory.cc文件中，找到了这样的定义：

```c++
struct FactoryItem {
    std::unique_ptr<DeviceFactory> factory;
    int priority;
};
std::unordered_map<string, FactoryItem>& device_factories(){
    static std::unordered_map<string, FactoryItem>* factories = new std::unordered_map<string, FactoryItem>;
    return *factories;
}
```
对于第二个函数，它内部定义了一个静态成员，因此相当于提供了一个全局的从设备类型名称到其生产工厂的映射。每当我们需要这个映射时，就调用这个函数。实际上，DeviceFactory的很多成员函数，就是这样实现的。
另外，TF还提供了一个Registrar类，为DeviceFactory提供了注册的入口：
```c++
template<class Factory> class Registrar {
  public:
    explicit Registrar(const string& device_type, int priority=50){
        DeviceFactory::Register(device_type, new Factory(), priority);
    }
};
```
它实际上是为某种设备类型注册其设备工厂。
关于设备工厂类，我们在代码中经常看到`priority`，对于权重，我们详细说明一下：
* 对于同样一种设备类型，不同的注册可以由不同的权重，即同一个设备类型的不同实现，可以拥有不同的权重。权重主要被应用于以下两个方面：
* 当我们需要为某一个特定的设备类型选择工厂时，拥有最高权重的工厂将会被选择。例如，如果有如下的两种注册信息
`Registrar<CPUFactory1>("CPU", 125)`;和
`Registrar<CPUFactory2>("CPU", 150)`;
那么当调用`DeviceFactory::GetFactory("CPU")`时，`CPUFactory2`将会被返回。
* 当需要在DeviceSet中选择一种设备类型时，选择的顺序由权重priority决定。例如，对于以下的两种注册：
`Registrar<CPUFactory>("CPU",100)`;和
`Registrar<GPUFactory>("GPU",200)`;
则`DeviceType("GPU")`将会被优先选择。
* 不同设备的默认权重如下：`GPU:200`，`SYCL:200`，`GPUCompatibleCPU:70`，`ThreadPoolDevice:60`，`Default:50`。