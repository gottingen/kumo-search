hala ea 004
==============================

本节要点:

* 使用`carbin`管理项目依赖库树状结构管理，依赖`melon`
* 创建restful服务服务端以及客户端，使用`melon`进行通信

代码仓库位于[hallo-ea-04][1]

## 初始化仓库

```shell
mkdir halalrestful
cd halalrestful
carbin create --name halalrestful --requirements
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
注意： `melon`也有依赖，

melon依赖：
* turbo
* gtest
* gmock
* benchmark
* gflags
* leveldb
* protobuf
* zlib

但是这里我们不需要关心`melon`有什么依赖，`carbin`会自动解决依赖下载安装。

在`halalrestful`目录下安装依赖库

```shell
carbin install
```
大约需要5分钟时间，正好可以点一小支，`carbin`会自动下载依赖库，解压，编译，安装，安装在当前目录的`carbin`目录下。**不要忘记在`.gitignore`文件中添加`carbin`目录**。

## 编写代码

### 服务端

melon 提供一个restful的基类，继承这个基类，并实现自己的process方法。

```cpp
namespace myservice {

class MyService : public melon::RestfulService {
private:
    void process(const melon::RestfulRequest *request, melon::RestfulResponse *response) override {
        auto path = request->unresolved_path();
        response->set_status_code(200);
        response->set_header("Content-Type", "text/plain");
        if(path == "version") {
            response->set_body(HALAREST_VERSION_STRING);
            response->append_body("\n");
            return;
        } else {
            response->set_body("Hello, my restful\n");
            response->append_body("Request path: ");
            response->append_body(path);
            response->append_body("\n");
        }
    }
};
}  // namespace myservice
```
解析一下，RestfulService再处理时，会调用process方法，传入request和response对象，
我们可以通过request对象获取请求的路径，参数等信息，通过response对象设置返回的状态码，头部，body等信息。

`RestfulRequest`和`RestfulResponse`是两个代理类，提供getter和setter方法，方便我们操作。也无需关注这两个对象的生命周期，他们只引用
外层的controller,具体可以参见`melon`源码。

使用时，需要注意set_body方法，会覆盖之前的body内容，如果需要追加内容，可以使用append_body方法。我在编写时，就犯了这个错误，导致
version信息没有返回。浪费了一根小支的时间。

ok，进入下一步，我们需要创建一个main函数，启动服务。

```cpp
int main(int argc, char* argv[]) {
    google::ParseCommandLineFlags(&argc, &argv, true);
    turbo::setup_rotating_file_sink("http_server.log", 100, 10, true, 60);
    // Generally you only need one Server.
    melon::Server server;
    myservice::MyService service;
    auto rs = service.register_server("/ea", &server);
    if(!rs.ok()) {
        LOG(ERROR) << "register server failed: " << rs;
        return -1;
    }

    melon::ServerOptions options;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    options.mutable_ssl_options()->default_cert.certificate = FLAGS_certificate;
    options.mutable_ssl_options()->default_cert.private_key = FLAGS_private_key;
    options.mutable_ssl_options()->ciphers = FLAGS_ciphers;
    if (server.Start(FLAGS_port, &options) != 0) {
        LOG(ERROR) << "Fail to start HttpServer";
        return -1;
    }
    server.RunUntilAskedToQuit();
    return 0;
}
```

这里面用到几个gflags的定义，在这里,要放在文件头部。
    
```cpp
DEFINE_int32(port, 8018, "TCP Port of this server");
DEFINE_int32(idle_timeout_s, -1, "Connection will be closed if there is no "
"read/write operations during the last `idle_timeout_s'");

DEFINE_string(certificate, "cert.pem", "Certificate file path to enable SSL");
DEFINE_string(private_key, "key.pem", "Private key file path to enable SSL");
DEFINE_string(ciphers, "", "Cipher suite used for SSL connections");
```
### 客户端

接下来，我们需要编写一个客户端，来测试我们的服务。
客户端代码很简单,下面来看下步骤，
1. 是创建一个`Channel`对象，初始化，是为链接服务器。
2. 创建一个`RestfulClient`对象，设置`Channel`对象，创建一个`RestfulRequest`对象，设置请求的url，方法，参数等信息。
3. 调用`client.create_request()`方法，创建一个`RestfulRequest`对象，设置请求的url，方法，参数等信息。
4. 调用`client.do_request(req)`方法，发送请求，返回一个`RestfulResponse`对象，我们可以通过这个对象获取返回的状态码，头部，body等信息。
5. 使用返回的`RestfulResponse`对象，获取返回的body信息。

```cpp

#include <gflags/gflags.h>
#include <turbo/log/logging.h>
#include <melon/rpc/channel.h>
#include <melon/rpc/restful_service.h>

DEFINE_string(d, "", "POST this data to the http server");
DEFINE_string(load_balancer, "", "The algorithm for load balancing");
DEFINE_int32(timeout_ms, 2000, "RPC timeout in milliseconds");
DEFINE_int32(max_retry, 3, "Max retries(not including the first RPC)");
DEFINE_string(protocol, "http", "Client-side protocol");

namespace melon {
    DECLARE_bool(http_verbose);
}

int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    google::ParseCommandLineFlags(&argc, &argv, true);

    if (argc != 2) {
        LOG(ERROR) << "Usage: ./restful_client \"http(s)://www.foo.com\"";
        return -1;
    }
    char* url = argv[1];

    // A Channel represents a communication line to a Server. Notice that
    // Channel is thread-safe and can be shared by all threads in your program.
    melon::Channel channel;
    melon::ChannelOptions options;
    options.protocol = FLAGS_protocol;
    options.timeout_ms = FLAGS_timeout_ms/*milliseconds*/;
    options.max_retry = FLAGS_max_retry;

    // Initialize the channel, nullptr means using default options.
    // options, see `melon/rpc/channel.h'.
    if (channel.Init(url, FLAGS_load_balancer.c_str(), &options) != 0) {
        LOG(ERROR) << "Fail to initialize channel";
        return -1;
    }

    // We will receive response synchronously, safe to put variables
    // on stack.
    melon::RestfulClient client;
    client.set_channel(&channel);
    auto _rs = client.create_request();
    if (!_rs.ok()) {
        LOG(ERROR) << "Fail to create request: " << _rs.status();
        return -1;
    }
    auto req = _rs.value();
    req.set_uri(url);
    if (!FLAGS_d.empty()) {
        req.set_method(melon::HTTP_METHOD_POST);
        req.set_body(FLAGS_d);
    } else {
        req.set_method(melon::HTTP_METHOD_GET);
    }

    auto res = client.do_request(req);

    if (res.failed()) {
        std::cerr << res.failed_reason() << std::endl;
        return -1;
    }
    // If -http_verbose is on, melon already prints the response to stderr.
    if (!melon::FLAGS_http_verbose) {
        std::cout << res.body() << std::endl;
    }
    return 0;
}
```

## 编译

在`CMakelists.txt`文件中添加如下代码:

```cmake

file(COPY key.pem
        DESTINATION ${PROJECT_BINARY_DIR})
file(COPY cert.pem
        DESTINATION ${PROJECT_BINARY_DIR})

carbin_cc_binary(
        NAMESPACE halarest
        NAME restful_server
        SOURCES
        restful_server.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        ${CARBIN_DEPS_LINK}
        PUBLIC
)

carbin_cc_binary(
        NAMESPACE halarest
        NAME restful_client
        SOURCES
        restful_client.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        ${CARBIN_DEPS_LINK}
        PUBLIC
)
```

看起来是不是特别的简单，`carbin_cc_binary`是`carbin`提供的一个函数，用来编译二进制文件，`NAMESPACE`是命名空间，通常是我们的项目名，
`NAME`是二进制文件的名字，`SOURCES`是源文件，`CXXOPTS`是编译选项，`LINKS`是链接的依赖库，`PUBLIC`是公共依赖库。

重点来了，`LINKS`这一项，我们需要添加`melon`的依赖，
在`${PROJECT_SOURCE_DIR}/cmake`目录下创建一个`halarest_deps.cmake`文件，找到
`find_package(Thread REQUIRED)`这一行，添加如下代码:

```cmake
find_package(melon REQUIRED)
include_directories(${melon_INCLUDE_DIR})
```
将

```cmake
set(CARBIN_DEPS_LINK
        #${TURBO_LIB}
        ${CARBIN_SYSTEM_DYLINK}
        )
```

改为
```cmake
set(CARBIN_DEPS_LINK
        ${MELON_STATIC_LIBRARIES}
        ${CARBIN_SYSTEM_DYLINK}
        )
```

ok，到此为止，我们的代码已经编译构建已经完成。接下来，我们可以编译运行了。

# 编译运行

编译
```shell
mkdir build
cd build
cmake ..
make
```
运行
```shell
./restful_server
```

## 在浏览器中访问

在浏览器中输入`http://localhost:8018/ea/version`，可以看到返回的信息。
继续访问`http://localhost:8018/ea/abc`，可以看到返回的信息。

## 在客户端中访问

在终端中输入`./restful_client http://localhost:8018/ea/version`，可以看到返回的信息。
继续输入`./restful_client http://localhost:8018/ea/abc`，可以看到返回的信息。

到此为止，我们的服务端和客户端已经编写完成，可以正常运行了。

# 总结

特别之处在于，引入了`melon`依赖，`melon`也有很多依赖，但是我们不需要关心，而是直接使用`${MELON_STATIC_LIBRARIES}`来链接`melon`以及
`melon`的依赖库。节省我们很多心智上的负担，同时也提高了我们的开发效率。这部分的内容也是`EA`体系化的一个体现，需要关心的是，需要什么
，而不是怎么去做， `EA`帮你去做。在`EA`体系中，依赖的库以及他依赖的，都会在`${${PROJECT_NAME}_STATIC_LIBRARIES}`,这是一条很明确
的规则，我们只需要关心这个变量，就可以了。当然如果你有兴趣参加`EA`体系的开发，可以参考`melon`或者其他`EA`体系的项目。这个实现是利用
`cmake`目标导出实现，参考这个文件`halarest_config.cmake.in`,类似的，melon相应目录下有一个`melon_config.cmake.in`文件。
实现的方法也很朴素，就是将`melon`在`melon_deps.cmake`文件中的`find_package`等操作，在这个文件中重新执行一遍，然后将结果导出到相应
的变量中即可，具体可以参考`melon`的源码。

# 后记

经过四节的内容，我们已经可以我们的一些输出，写入到浏览器中，方便我们查看，也可以通过客户端来访问我们的服务，这是一个很好的开始，后续
我们会继续深入，结合生产环境，来实现一些更加复杂的功能，比如，服务发现，全局时钟服务，全局配置服务，全局id服务等等，这些都是我们
在生产环必不可少的功能，我们会一步一步的来实现。

另外，现在为止，我们基本上可以初步使用`carbin`来管理我们的项目，后续在`cmake有点甜`系列文章中，会详细的讲解`carbin`设计和实现。

同时，再次的说明一次，虽然我们是半小时系列，半小时能够完成很多实用的功能，但这个背后，需要我们花费很多的时间去做基础性能的积累和抽象，总结。
不要因此觉得，`哦，原来程序员上班都在摸鱼`，其实不然，为了能够在半小时内完成一个功能，我们需要花费很多的时间去学习，去实践，去总结，去抽象，
甚至背后的工作是几年的积累和总结。


[1]: https://github.com/gottingen/ea-half-an-hour/tree/master/a004-hala-restful
