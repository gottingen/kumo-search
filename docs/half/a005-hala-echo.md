hala ea 005
==============================

上节， 我们创建了一个restful服务，使用`melon`库，这节我们创建一个echo服务，使用`melon`库。

要点:

* 引入`protobuf`
* 使用`carbin_cc_proto`生成protobuf c++代码
* 编译生成的protobuf c++代码并在项目中使用

代码仓库位于[hallo-ea-05][1]

## 初始化仓库

```shell
mkdir halalecho
cd halalecho
carbin create --name halalecho --requirements
```

修改依赖文件`carbin_deps.txt`，添加`melon`依赖

```text
############
# ea recipes
############
gottingen/carbin-recipes@ea

#########################
# testing tools googlegtest benchmark
testing
gottingen/melon@v0.5.11 -DCARBIN_BUILD_TEST=OFF -DCARBIN_BUILD_BENCHMARK=OFF -DCARBIN_BUILD_EXAMPLES=OFF
```
安装依赖库

```shell
carbin install
```
前面步骤和上节一样，这里不再赘述。

## 编写代码

### proto文件`echo.proto`

```protobuf
syntax="proto2";
package halaecho;

option cc_generic_services = true;

message EchoRequest {
      required string message = 1;
};

message EchoResponse {
      required string message = 1;
};

service EchoService {
      rpc Echo(EchoRequest) returns (EchoResponse);
};
```

### 编译proto文件

很多编译器具有提示功能，为了提升开发体验，我先将`echo.proto`文件编译为`echo.pb.cc`和`echo.pb.h`文件。
在项目的`CMakeLists.txt`文件中，在`add_subdirectory(halaecho)`之上，添加如下代码:

```cmake
set(PROTO_FILES
        halaecho/echo.proto
)

set(PROTOC_FLAGS ${PROTOC_FLAGS} -I${PROTOBUF_INCLUDE_DIR})
carbin_cc_proto(PROTO_HDRS PROTO_SRCS ${PROJECT_BINARY_DIR}
        ${PROJECT_BINARY_DIR}/output/include
        ${PROJECT_SOURCE_DIR}
        "${PROTO_FILES}")
carbin_cc_object(
        NAMESPACE halaecho
        NAME proto_obj
        SOURCES
        ${PROTO_SRCS}
        CXXOPTS
        ${MELON_CXX_OPTIONS}
)
```

这里要注意，carbin_cc_proto的第一个参数是生成的头文件，第二个参数是生成的源文件，第三个参数是生成的目录，第四个参数是proto文件的目录，第五个参数是proto文件列表。
后续如果有多个proto文件，可以将`PROTO_FILES`添加多个文件。直接拷贝上面的代码即可。聪明的你可能已经发现`EA`的规则，很多功能都是直接拷贝，修改输入参数即可，这也是
`EA`提升效率的一个重要因素，不需要过多的学习更多的规则，从流程上能够理解按照什么步骤去做就可以了。剩下的事情，`EA`会帮你完成。

`carbin_cc_object`这个函数是用来编译proto生成的源文件的，这个函数的参数和`carbin_cc_library`是一样的，只是`carbin_cc_object`是用来编译源文件的，
`carbin_cc_library`是用来编译库的。**这里要注意，必须在这里调用，不可以在`halaecho`目录下调用，否则不会生成proto文件**。

`carbin`的模版规则，必须要有一个对象导出，因此，我们需要在`halaecho`目录下的`CMakeLists.txt`文件先创建一个可以导出的对象。用直白的表达，就要生成一个
库或者是二进制文件。这里我们生成一个库。

```cmake
carbin_cc_library(
        NAMESPACE halaecho
        NAME proto
        DEPS
        proto_obj
        OBJECTS
        proto_obj
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        PLINKS
        ${CARBIN_DEPS_LINK}
        PUBLIC
)
```

先编译proto文件

```shell
mkdir build
cd build
cmake ..
make
```

如果clion提示找不到头文件，可以使用clion IDE，如果找不到proto文件，在setting中设置proto的引用路径。

## 编写echo服务

### server

```c++
#include <gflags/gflags.h>
#include <turbo/log/logging.h>
#include <melon/rpc/server.h>
#include <melon/json2pb/pb_to_json.h>
#include <halaecho/echo.pb.h>

DEFINE_bool(echo_attachment, true, "Echo attachment as well");
DEFINE_int32(port, 8000, "TCP Port of this server");
DEFINE_string(listen_addr, "", "Server listen address, may be IPV4/IPV6/UDS."
            " If this is set, the flag port will be ignored");
DEFINE_int32(idle_timeout_s, -1, "Connection will be closed if there is no "
             "read/write operations during the last `idle_timeout_s'");

// Your implementation of example::EchoService
// Notice that implementing melon::Describable grants the ability to put
// additional information in /status.
namespace halaecho {
class EchoServiceImpl : public EchoService {
public:
    EchoServiceImpl() {}
    virtual ~EchoServiceImpl() {}
    virtual void Echo(google::protobuf::RpcController* cntl_base,
                      const EchoRequest* request,
                      EchoResponse* response,
                      google::protobuf::Closure* done) {
        // This object helps you to call done->Run() in RAII style. If you need
        // to process the request asynchronously, pass done_guard.release().
        melon::ClosureGuard done_guard(done);

        melon::Controller* cntl =
            static_cast<melon::Controller*>(cntl_base);

        // optional: set a callback function which is called after response is sent
        // and before cntl/req/res is destructed.
        cntl->set_after_rpc_resp_fn(std::bind(&EchoServiceImpl::CallAfterRpc,
            std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));

        // The purpose of following logs is to help you to understand
        // how clients interact with servers more intuitively. You should 
        // remove these logs in performance-sensitive servers.
        LOG(INFO) << "Received request[log_id=" << cntl->log_id()
                  << "] from " << cntl->remote_side() 
                  << " to " << cntl->local_side()
                  << ": " << request->message()
                  << " (attached=" << cntl->request_attachment() << ")";

        // Fill response.
        response->set_message(request->message());

        // You can compress the response by setting Controller, but be aware
        // that compression may be costly, evaluate before turning on.
        // cntl->set_response_compress_type(melon::COMPRESS_TYPE_GZIP);

        if (FLAGS_echo_attachment) {
            // Set attachment which is wired to network directly instead of
            // being serialized into protobuf messages.
            cntl->response_attachment().append(cntl->request_attachment());
        }
    }

    // optional
    static void CallAfterRpc(melon::Controller* cntl,
                        const google::protobuf::Message* req,
                        const google::protobuf::Message* res) {
        // at this time res is already sent to client, but cntl/req/res is not destructed
        std::string req_str;
        std::string res_str;
        json2pb::ProtoMessageToJson(*req, &req_str, NULL);
        json2pb::ProtoMessageToJson(*res, &res_str, NULL);
        LOG(INFO) << "req:" << req_str
                    << " res:" << res_str;
    }
};
}  // namespace halaecho

int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    google::ParseCommandLineFlags(&argc, &argv, true);

    // Generally you only need one Server.
    melon::Server server;

    // Instance of your service.
    halaecho::EchoServiceImpl echo_service_impl;

    // Add the service into server. Notice the second parameter, because the
    // service is put on stack, we don't want server to delete it, otherwise
    // use melon::SERVER_OWNS_SERVICE.
    if (server.AddService(&echo_service_impl, 
                          melon::SERVER_DOESNT_OWN_SERVICE) != 0) {
        LOG(ERROR) << "Fail to add service";
        return -1;
    }

    mutil::EndPoint point;
    if (!FLAGS_listen_addr.empty()) {
        if (mutil::str2endpoint(FLAGS_listen_addr.c_str(), &point) < 0) {
            LOG(ERROR) << "Invalid listen address:" << FLAGS_listen_addr;
            return -1;
        }
    } else {
        point = mutil::EndPoint(mutil::IP_ANY, FLAGS_port);
    }
    // Start the server.
    melon::ServerOptions options;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    if (server.Start(point, &options) != 0) {
        LOG(ERROR) << "Fail to start EchoServer";
        return -1;
    }

    // Wait until Ctrl-C is pressed, then Stop() and Join() the server.
    server.RunUntilAskedToQuit();
    return 0;
}
```
这里要注意的`set_after_rpc_resp_fn`函数，这个函数是在response发送之后，但是在`cntl/req/res`对象析构之前调用的，这个函数是用来处理一些
response发送之后的逻辑，比如日志记录等。这个函数是可选的，如果不需要，可以不设置。

### client

```c++

#include <gflags/gflags.h>
#include <turbo/log/logging.h>
#include <melon/utility/time.h>
#include <melon/rpc/channel.h>
#include <halaecho/echo.pb.h>

DEFINE_string(attachment, "", "Carry this along with requests");
DEFINE_string(protocol, "melon_std", "Protocol type. Defined in melon/rpc/options.proto");
DEFINE_string(connection_type, "", "Connection type. Available values: single, pooled, short");
DEFINE_string(server, "0.0.0.0:8000", "IP Address of server");
DEFINE_string(load_balancer, "", "The algorithm for load balancing");
DEFINE_int32(timeout_ms, 100, "RPC timeout in milliseconds");
DEFINE_int32(max_retry, 3, "Max retries(not including the first RPC)"); 
DEFINE_int32(interval_ms, 1000, "Milliseconds between consecutive requests");

int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    google::ParseCommandLineFlags(&argc, &argv, true);
    
    // A Channel represents a communication line to a Server. Notice that 
    // Channel is thread-safe and can be shared by all threads in your program.
    melon::Channel channel;
    
    // Initialize the channel, NULL means using default options.
    melon::ChannelOptions options;
    options.protocol = FLAGS_protocol;
    options.connection_type = FLAGS_connection_type;
    options.timeout_ms = FLAGS_timeout_ms/*milliseconds*/;
    options.max_retry = FLAGS_max_retry;
    if (channel.Init(FLAGS_server.c_str(), FLAGS_load_balancer.c_str(), &options) != 0) {
        LOG(ERROR) << "Fail to initialize channel";
        return -1;
    }

    // Normally, you should not call a Channel directly, but instead construct
    // a stub Service wrapping it. stub can be shared by all threads as well.
    halaecho::EchoService_Stub stub(&channel);

    // Send a request and wait for the response every 1 second.
    int log_id = 0;
    while (!melon::IsAskedToQuit()) {
        // We will receive response synchronously, safe to put variables
        // on stack.
        halaecho::EchoRequest request;
        halaecho::EchoResponse response;
        melon::Controller cntl;

        request.set_message("hello world");

        cntl.set_log_id(log_id ++);  // set by user
        // Set attachment which is wired to network directly instead of 
        // being serialized into protobuf messages.
        cntl.request_attachment().append(FLAGS_attachment);

        // Because `done'(last parameter) is NULL, this function waits until
        // the response comes back or error occurs(including timedout).
        stub.Echo(&cntl, &request, &response, NULL);
        if (!cntl.Failed()) {
            LOG(INFO) << "Received response from " << cntl.remote_side()
                << " to " << cntl.local_side()
                << ": " << response.message() << " (attached="
                << cntl.response_attachment() << ")"
                << " latency=" << cntl.latency_us() << "us";
        } else {
            LOG(WARNING) << cntl.ErrorText();
        }
        usleep(FLAGS_interval_ms * 1000L);
    }

    LOG(INFO) << "EchoClient is going to quit";
    return 0;
}
```

## 修改CMakeLists.txt

在`halaecho`目录下的`CMakeLists.txt`文件中，添加如下代码:

```cmake

carbin_cc_binary(
        NAMESPACE halaecho
        NAME echo_cli
        SOURCES
       client.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        ${CARBIN_DEPS_LINK}
        halaecho::proto_static
        PUBLIC
)

carbin_cc_binary(
        NAMESPACE halaecho
        NAME echo_server
        SOURCES
        server.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        ${CARBIN_DEPS_LINK}
        halaecho::proto_static
        PUBLIC
)
```
记得不要写在`proto`库的编译之前.

## 编译

```shell
cd build
rm -rf *
cmake ..
make
```

可以看到，在halalecho目录下生成了`echo_cli`和`echo_server`两个二进制文件。

## 运行
打开个shell
```shell
./halaecho/echo_server
```

打开另一个shell
```shell
./halaecho/echo_cli
```
打开浏览器，输入`http://localhost:8000/ea/status`，可以看到服务的状态信息。

到此为止，我们已经完成了一个echo服务，使用`melon`库。

## 总结

通过本节和上一节的学习，我们已经了解了如何使用`melon`库创建一个restful服务和echo服务。`restful`不使用`protobuf`，`echo`使用`protobuf`。
这两节涵盖了大部分业务场景的服务协议形式，内部应用之间可以使用`protobuf`，对外应用可以使用`restful`提供标准的http接口。

## 下节预告

下节开始，我们正式进入企业生产级应用服务的开发。计划如下

* a006-hala-cache - 使用`turbo`库，创建一个内存缓存服务
* a007-hala-kv - 使用`turbo`库，创建一个内存kv存储服务
* a008-hala-proxy - 使用`melon`库，创建一个代理服务,代理后端服务
* a009-hala-dkv - 使用`melon`库，创建一个分布式内存kv存储服务

[1]: https://github.com/gottingen/ea-half-an-hour/tree/master/a005-hala-echo