framework graph
===

# graph

graph是TF计算设计的载体，如果拿TF代码的执行和Java代码执行相比，它相当于Java的字节码。关于graph的执行过程，我们在这里简单介绍一下。在graph构建完成，并进行了一些简单优化之后，会对图进行分割，实际上就是执行一个节点分配的过程，然后在各设备上分别对子图进行运行前的优化，最后调用各设备的执行器，调度运行运行子图。在node章节我们讲过，node本身是自带图结构的，一个node的集合就能复原一个完整的graph。所以，graph本身添加的数据内容并不多。下面我们看下GraphDef的定义：

```c++
message GraphDef {
    repeated NodeDef node = 1;
    VersionDef versions = 4;
    int32 version = 3 [deprecated = true];
    FunctionDefLibrary library = 2;
};
```
可以看到，这个结构里除了添加了一个函数定义库之外，没有其它数据了。

# 图构建辅助函数

TF为了方便构建GraphDef，提供了很多辅助函数，下面我们来看下：
```c++
//生成一个可读性好的关于GraphDef的描述，而不是返回一个可读性差的proto文本
string SummarizeGraphDef(const GraphDef& graph_def);

//校验一个GraphDef
Status ValidateExternalGraphDefSyntax(const GraphDef& graph_def);

//从节点索引node_offset开始，为GraphDef中的节点加入默认参数值，节点对应的op的默认参数值在op_registry中
Status AddDefaultAttrsToGraphDef(GraphDef* graph_def, const OpRegistryInterface& op_registry, int node_offset);

//从GraphDef中除去那些，在producer_op_registry中出现过，但在consumer_op_registry中未出现过的默认参数值
Status RemoveNewDefaultAttrsFromGraphDef(GraphDef* graph_def, const OpRegistryInterface& consumer_op_registry, const OpRegistryInterface& producer_op_registry, std::set<std::pair<string,string>>* op_attr_removed);

//收集图使用的op，以字符串集合的形式返回
void OpsUsedByGraph(const GraphDef& graph_def, std::set<string>* ops_used_in_graph);

//将graph_def中出现过，同时也在op_registry中出现过的op放入stripped_op_list
Status StrippedOpListForGraph(const GraphDef& graph_def, const OpRegistryInterface& op_registry, OpList* stripped_op_list);
```
其中有很多是跟图中op的默认参数值相关的函数。这些函数出现的背景是这样的，假设在一个分布式的环境下，master需要workder执行一个子图，但这个图是master产生的，图中操作的默认值使用的是master所在机器的运行时环境中，OpRegistry中注册的操作所包含的默认值，但关键是，workder所在机器使用的运行时环境，跟master可能不一样！比如，master机器及时对TF进行了升级，但workder却没有。而升级之后，master所在运行时的op参数，可能之前没有默认值，现在加上了默认值，或者之前的默认值改成了现在的默认值，这时候，为了让这张子图具有向前兼容特性，即为了让它能够在workder机器上运行，需要对这张子图进行处理，删除仅在master运行时的OpRegistry中出现的op参数默认值。于是就出现了最后的三个函数。

# graph_transfer_info
为了让我们定义的计算图能够在其它设备（比如DSP）上运行，需要对图结构进行一些转换。

```c++
message GraphTransferInfo {
    enum Destination {
        NOP = 0;
        HEXAGON = 1;
    }
    message NodeInput {
        int32 node_id = 1;
        int32 output_port = 2;
    }
    message NodeInfo {
        string name = 1;
        int32 node_id = 2;
        string type_name = 3;
        int32 soc_op_id = 4;
        int32 padding_id = 5;
        int32 input_count = 6;
        int32 output_count = 7;
    };
    message ConstNodeInfo {
        string name = 1;
        int32 node_id = 2;
        repeated int64 shape = 3;
        bytes data = 4;
        DataType dtype = 5;
    };
    message NodeInputInfo {
        int32 node_id = 1;
        repeated NodeInput node_input = 2;
    };
    message NodeOutputInfo {
        int32 node_id = 1;
        repeated int32 max_byte_size = 2;
    };
    message GraphInputNodeInfo {
        string name = 1;
        repeated int64 shape = 2;
        DataType dtype = 3;
    };
    message GraphOutputNodeInfo {
        string name = 1;
        repeated int64 shape = 2;
        DataType dtype = 3;
    };
    repeated NodeInfo node_info = 1;
    repeated ConstNodeInfo const_node_info = 2;
    repeated NodeInputInfo node_input_info = 3;
    repeated NodeOutputInfo node_output_info = 4;
    repeated GraphInputNodeInfo graph_input_node_info = 5;
    repeated GraphOutputNodeInfo graph_output_node_info = 6;
    Destination destination = 7;
};
```

# 文件

* [graph.proto](../tensorflow/core/framework/graph.proto)
* [graph_def_util.h](../tensorflow/core/framework/graph_def_util.h)
* [graph_transfer_info.proto](../tensorflow/core/framework/graph_transfer_info.proto)
* [graph_to_functiondef.h](../tensorflow/core/framework/graph_to_functiondef.h)