a007 hala vue ext
==============================

本节代码仓库位于[hallo-ea-06][1]

本节继续a006-hala-vue的内容。对于源码进行解读。

# 编译选项

本项目引入另外一个依赖库`alkaid`，这个库是对文件系统，数据压缩的封装。提供了统一的文件系统操作接口，目前
支持`local`本地文件系统，压缩文件系统`zip`，`zstd`，`lz4`，`bz2`。

`carbin_deps.txt`文件内容如下:

```text
############
# ea recipes
############
gottingen/carbin-recipes@ea

#########################
# testing tools googlegtest benchmark
testing
gottingen/collie@v0.2.38 -DCARBIN_BUILD_TEST=OFF -DCARBIN_BUILD_BENCHMARK=OFF -DCARBIN_BUILD_EXAMPLES=OFF
gottingen/melon@v0.5.11 -DCARBIN_BUILD_TEST=OFF -DCARBIN_BUILD_BENCHMARK=OFF -DCARBIN_BUILD_EXAMPLES=OFF
gottingen/alkaid@v0.1.7 -DCARBIN_BUILD_TEST=OFF -DCARBIN_BUILD_BENCHMARK=OFF -DCARBIN_BUILD_EXAMPLES=OFF

[1]: https://github.com/gottingen/ea-half-an-hour/tree/master/a006-hala-vue
```
执行

```shell
carbin install
```
安装依赖。

修改`cmake/halavue_deps.cmake`文件，添加`alkaid`依赖。

```cmake
find_package(Threads REQUIRED)
find_package(melon REQUIRED)
find_package(alkaid REQUIRED)
find_package(turbo REQUIRED)
include_directories(${melon_INCLUDE_DIR})
include_directories(${melon_INCLUDE_DIRS})
############################################################
#
# add you libs to the CARBIN_DEPS_LINK variable eg as turbo
# so you can and system pthread and rt, dl already add to
# CARBIN_SYSTEM_DYLINK, using it for fun.
##########################################################
set(CARBIN_DEPS_LINK
        #${TURBO_LIB}
        ${MELON_STATIC_LIBRARIES}
        ${ALKAID_LIBRARIES}
        turbo::turbo_static
        ${CARBIN_SYSTEM_DYLINK}
        )
list(REMOVE_DUPLICATES CARBIN_DEPS_LINK)
carbin_print_list_label("Denpendcies:" CARBIN_DEPS_LINK)
```
# 代码实现

主函数启动，与前面的restful服务类似,都是启动一个restful服务。不同的是这次的restful服务增加了新的功能，根据不同的请求路径，选择提供restful服务，还是提供前端服务。

* 缓存服务 访问 `http://localhost:8018/ea/cache` 是一个restful服务，用于设置和读取缓存数据。
* 前端服务 访问 `http://localhost:8018/ea/ui` 是一个vue前端服务，浏览器访问这个服务，可以访问缓存服务。
这在上一节也提到，这个判断处理过程在`CacheService::process`函数中。在讲解这个函数之前，先介绍一个前置的小知识点，还记得main函数里注册的`CacheService`吗？

```c++
    halavue::CacheService cache_service(&cache, &vue_service);
    rs = cache_service.register_server("/ea", &server);
    if(!rs.ok()) {
        LOG(ERROR) << "register server failed: " << rs;
        return -1;
    }
```
这里要关注的是，我们注册的路径和我们restful服务的请求路径。本例中，我们注册的路径是`/ea`，那么，请求为`http://host:port/ea/...`的请求，
都会交给`CacheService`处理。这个处理过程在`CacheService::process`函数中。当然也可以注册其他路径,比如:

* `/ea/v1`，那么请求为`http://host:port/ea/v1/...`的请求，
* `/ea/v2`，那么请求为`http://host:port/ea/v2/...`的请求，

但是要注意的是，restful服务只能注册一次，如果注册多次，会注册失败。`process`函数的`request`参数提供接口`unresolved_path`来获取请求的路径。
它是这样返回结果，比如 

* `http://localhost:8018/ea/cache` ,这个函数返回的值是`cache`
* `http://localhost:8018/ea/ui` ,这个函数返回的值是`ui`
* `http://localhost:8018/ea/ui/index.html` ,这个函数返回的值是`ui/index.html`
这点很重要，`process`的路由处理就是基于这个路径来处理的。

现在是时候来进入真正的处理过程了，`CacheService::process`函数。

```c++
     auto &path = request->unresolved_path();
        response->set_header("Access-Control-Allow-Origin", "*");
        // index
        if(path.empty()) {
            response->set_status_code(200);
            response->set_header("Content-Type", "text/html");
            response->set_body(R"(
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Cache Service</title>
</head>
<body>
    <br> <a href="/ea/cache">cache</a> </br>
    <br> <a href="/ea/ui">ui</a> </br>
</body>
</html>
            )");
            return;
        }
        if(turbo::starts_with(path, "ui")) {
            _web_service->process(request, response);
            return;
        }
        if(path != "cache") {
            response->set_status_code(404);
            response->set_body("not found");
            return;
        }
        // get method
        response->set_header("Content-Type", "application/json");
        if(request->method() ==melon::HTTP_METHOD_GET) {
            auto &uri = request->uri();
            auto *key = uri.GetQuery("key");
            if(key == nullptr) {
                response->set_status_code(200);
                nlohmann::json j;
                j["code"] = turbo::StatusCode::kInvalidArgument;
                j["msg"] = "no key";
                j["value"] = "";
                response->set_body(j.dump());
                return;
            }
            // get key from cache
            auto r = _cache_service->get(*key);
            if(r.second) {
                response->set_status_code(200);
                nlohmann::json j;
                j["code"] = turbo::StatusCode::kOk;
                j["msg"] = "ok";
                j["value"] = r.first;
                response->set_body(j.dump());
                return;
            } else {
                response->set_status_code(200);
                nlohmann::json j;
                j["code"] = turbo::StatusCode::kNotFound;
                j["msg"] = turbo::str_cat(*key, " not found");
                j["value"] = "";
                response->set_body(j.dump());
                return;
            }
        }

        // set method
        if(request->method() == melon::HTTP_METHOD_POST) {
            auto &uri = request->uri();
            auto *key = uri.GetQuery("key");
            auto &value = request->body();
            if(key == nullptr || value.empty()) {
                response->set_status_code(200);
                nlohmann::json j;
                j["code"] = turbo::StatusCode::kInvalidArgument;
                j["msg"] = "no key or value";
                j["value"] = "";
                response->set_body(j.dump());
                return;
            }
            _cache_service->set(*key, value.to_string());
            response->set_status_code(200);
            nlohmann::json j;
            j["code"] = turbo::StatusCode::kOk;
            j["msg"] = "ok";
            j["value"] = "";
            response->set_body(j.dump());
            return;
        }
        response->set_status_code(403);
        response->set_body("not support method");
    }
```
首先先获取请求路径，如果路径为空，返回一个简单的html页面，这个页面提供了两个链接，一个是`cache`，一个是`ui`。这个页面是一个简单的导航页面，用于用户选择服务。这一项
主要是为了浏览器访问方便，如果直接访问 `http://localhost:8018/ea` ，会直接返回这个页面。
`response->set_header("Access-Control-Allow-Origin", "*");`这个是设置跨域访问，关于跨域访问，可以参考[跨域访问](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)。
简单一句概括一下就是为了让浏览器可以访从服务下载资源。服务本身不要太关注这个，只要设置这个就可以了。

```c++
        if(turbo::starts_with(path, "ui")) {
            _web_service->process(request, response);
            return;
        }
        if(path != "cache") {
            response->set_status_code(404);
            response->set_body("not found");
            return;
        }
```
这段代码是路由请求路径，如果请求路径以`ui`开头，那么交给`_web_service`处理，本函数的执行结束。接下来判断请求路径是否为`cache`，如果不是，返回404错误，请求结束。
这个函数剩下的部分是处理`cache`请求，这个请求是一个restful服务，提供了两个接口，一个是设置缓存，一个是读取缓存。这个接口是一个简单的键值对接口，通过`key`来设置和读取缓存。
如果请求是 http get，则是查询缓存，如果请求是 http post，则是设置缓存。缓存代码在`cache.h`文件中，调用了`turbo`中的`LRCCache`类，这个类是一个LRU缓存。不要分心，
这个缓存的实现不是本文的重点，这里只是简单的调用。

返回值，设置为json格式，`{"code":0, "msg":"ok"， "value":""}`value字段始终为空。这个返回值是一个标准的restful服务返回值，code为0表示成功，msg为ok表示成功，value为返回值。
`response->set_header("Content-Type", "application/json");` 这个是设置返回值的类型，这个是一个json格式的返回值。

缓存的服务到目前为止，我们就完成，为了能更直观的让浏览器访问，我们需要一个前端服务。

**web服务**

`WebServie`这个类只有一个成员变量，是`vue`工程的文件路径。这个服务，与缓存服务是实际上是一点关系都没有，只是一个文件服务器，将请求的路径映射到本地的文件路径。将文件发送给浏览器。
还记得前面我们提到的`unresolved_path`函数吗？这个函数返回的值，就是请求的路径。这个路径是相对路径，相对于`vue`工程的路径。这个路径是相对路径，相对于`vue`工程的路径。但是这里，
根据请求的规则，我们要做一些转换
* 只请求`iu`根目录 `http://localhost:8018/ea/ui` ,我们需要补全路径，转换为`http://localhost:8018/ea/ui/index.html` 。 映射到本地文件路径为`www/index.html`。
* 请求有文件名字的路径，比如 `http://localhost:8018/ea/ui/css/app.css` ，映射到本地文件路径为 `www/css/app.css`。

在这个过程中，`/ea`被框架处理，用于定位服务，`/ui/*`文件路径被`WebServie`处理，用于定位文件。
那对应的规则就很明确了，"unresolved_path"返回的路径，这个函数里必须是'ui'开头，替换为函数的成员变量 `_root`即可，如果`_root`文件根目录，补全路径，返回文件内容。

name，按照这个逻辑，我们可以实现我们的处理逻辑:

```c++
  void WebServie::process(const melon::RestfulRequest *request, melon::RestfulResponse *response) const {
        // process request
        auto unresolved_path = request->unresolved_path();
        // path must starts with "ui"
        if(unresolved_path.size() >= 3) {
            unresolved_path = turbo::strip_prefix(unresolved_path, "ui/");
        } else {
            unresolved_path.clear();
        }

        if(unresolved_path.empty()) {
            unresolved_path = "index.html";
        }
        auto path = alkaid::filesystem::path(_root) / unresolved_path;
        LOG(INFO)<<"path: "<<path.string();
        std::error_code ec;
        if(!alkaid::filesystem::exists(path, ec)) {
            response->set_status_code(404);
            response->set_header("Content-Type", "text/html; charset=utf-8");
            response->set_header("Vary", "Accept-Encoding");
            response->set_body("404 not found");
            return;
        }
        auto lfs = alkaid::Filesystem::localfs();
        if(!lfs) {
            response->set_status_code(500);
            response->set_header("Content-Type", "text/html; charset=utf-8");
            response->set_header("Vary", "Accept-Encoding");
            response->set_body("500 internal error");
            return;
        }
        std::string content;
        auto rs =lfs->read_file(path.string(),&content);
        if(!rs.ok()) {
            response->set_status_code(500);
            response->set_header("Content-Type", "text/html; charset=utf-8");
            response->set_header("Vary", "Accept-Encoding");
            response->set_body("500 internal error");
            return;
        }
        std::string ct = "text/html; charset=utf-8";
        if(path.extension() == ".css") {
            ct = "text/css; charset=utf-8";
        } else if(path.extension() == ".js") {
            ct = "application/javascript; charset=utf-8";
        } else if(path.extension() == ".png") {
            ct = "image/png";
        } else if(path.extension() == ".jpg") {
            ct = "image/jpeg";
        } else if(path.extension() == ".gif") {
            ct = "image/gif";
        } else if(path.extension() == ".ico") {
            ct = "image/x-icon";
        }
        LOG(INFO)<<"file path: "<<path.string()<<" content size: "<<content.size();
        response->set_status_code(200);
        response->set_header("Content-Type", ct);
        response->set_header("Vary", "Accept-Encoding");
        response->set_body(content);
    }
```

这里要注意一下，`Content-Type`,前端解析会有要求，比如js文件需要将`content-type`设置为`application/javascript`，否则浏览器不会解析js文件。
我在调试时候就犯了这个错误，导致前端页面无法居中显示，浪费了一根小支的时间。

到目前为止，我们的后端服务已经全部完成了，为了让前端页面能够访问，我们还需要有一个能够和后端服务通信的前端页面。

**前端页面**

这也是第一接触前端，好在这个时代不缺学习资源，我在网上找到了一些资料，我花了半天时间，查阅资料，在bilibili上找到了一个前端视频教程学习，也算做了一个可以实现基本
功能要求的前端页面。这个页面是一个简单的缓存页面，提供了两个输入框，一个按钮，一个显示框。输入框用于输入缓存的key和value，按钮用于设置缓存，显示框用于显示缓存的内容。
这个工程部署在`cache`目录下，生成的前端工程在`cache/dist`目录下，拷贝到`a007-hala-vue-ext`目录下`www`目录中。在编译过程中会拷贝到`build`目录下。

需要注意的是，`npm run build`之前，需要设只一下`vue.config.js`文件，设置`publicPath`为`/ea/ui/`，这个是为了让前端页面能够访问到后端服务。

```javascript
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
    transpileDependencies: true,
    publicPath: '/ea/ui/',
})
```

我在第一次调试时，没有修改这个这个这配置，导致前端页面无法访问后端服务。

原本的计划是在c++中写入前端代码，像builtin服务一样。但是实际情况还是有很大区别，builtin服务是个固定的服务，提供框架信息查询，但是这类的
业务前端，改动会很频繁，而且前端的开发和维护，需要专业的前端工程师，考虑到这点，我还是决定将前端代码单独部署，提供一个简易的web server即可，
提供业务数据的交互，并不需要太多的渲染，如果是大型前端项目，还应该是使用专业的前端服务工具，如nginx，apache等。

总结：

我们实现一个带有浏览器交互的缓存服务，不必使用命令行去访问服务，通过浏览器访问服务，更加直观，更加方便。但是追求完美的我们不会就此，满足，
因为我们的API还不能在各个语言的应用的中完全互通，设置我们将来更新api，都会给客户带来困难，下一节，在我们这一节的基础上，让我们的缓存服务
能够更变多方便的在各个语言之间实现调用，接口以更好的方式暴露给用户。实现二进制协议、http协议互通， python，c++互通调用。