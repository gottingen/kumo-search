安装与使用
==================

# 系统环境

`EA`(Elastic automic infrastructure architecture) 系列的应用和服务都是基于`linux`系统开发的，所以需要在`linux`
系统上运行。`EA` 系列的应用和服务都是基于`x86_64`架构开发的，
所以需要在`x86_64`架构的`linux`系统上运行。

`EA` 系列的应用和服务都是基于`cmake`构建的，所以需要在`cmake`版本大于等于`3.24`的系统上运行。 `EA`
系列的应用和服务都是基于`gcc`编译的，
所以需要在`gcc`版本大于等于`9.3`的系统上运行。 centos7系统的gcc版本是4.8，部分c++依赖如 `boost`，`gflags`
等c++库，在gcc9.3上编译时会出现
`ABI`不兼容的问题，所以c++的依赖库都以源码形式进行编译依赖，部分C程序库如`openssl`，`zlib`等c库，不存在`ABI`不兼容问题，所以可以直接使用。

centos7 上自带的openssl版本是1.0.2，python版本是2.7，这两个版本都比较老，`EA` 需要python版本大于等于3.8，openssl版本大于等于1.1.1。
详细的配置参见[centos7](inf/centos7.md)。

ubuntu系统上自带的openssl版本是1.1.1，python版本是3.8，这两个版本都比较新，`EA` 需要python版本大于等于3.8，openssl版本大于等于1.1.1。
详细的配置参见[ubuntu](inf/ubuntu.md)。

# 基础开发环境准备

`EA`(Elastic automic infrastructure architecture)系列的应用和服务都会使用静态编译的方式进行部署，这样可以减少运行时的依赖，提高运行效率，尽量减少
部署的依赖。

`EA`为了方便项目依赖和用户安装部署，提供了[carbin][1]工具，用于下载文件和安装，具体参开[carbin docs][2]. cmake构建系统
在`cicd`应用请参考[cmake有点甜](cicd/sweet_cmake.md)
使用carbin工具不仅可以安装第三方
依赖库，更重要的功能，carbin可以一键创建工程的cmake 构建系统，方便用户进行编译和部署。

`EA`基础开发环境的运行分为三大部分： `基础系统`、`基础依赖安装`、`EA inf安装`。

依赖库

* googletest-release-1.12.1
* benchmark-1.8.4
* protobuf-3.20.0
* leveldb-1.23
* boost-1.85.0
* gflags

基础依赖安装：

```shell
cd /opt/EA/
carbin install gottingen/carbin-recipes@ea
carbin install ea/gflags  --prefix=/opt/EA/inf
carbin install ea/boost --prefix=/opt/EA/inf
carbin install ea/leveldb --prefix=/opt/EA/inf
carbin install ea/protobuf --prefix=/opt/EA/inf
carbin install ea/benchmark --prefix=/opt/EA/inf
```

**注意：** 以上依赖库都是使用carbin工具安装的，carbin工具会自动下载源码，编译，安装，所以需要联网。
**注意：** 以上依赖库是依据``EA``
的项目依赖对编译选项和源码进行了修改，不要使用原始的依赖库，否则会出现编译错误。源码仓库位于[ea inf][3]组织中。

# MKL安装

`EA`的一些应用和服务需要使用`MKL`库，`MKL`库是`Intel`公司提供的数学库，提供了高效的数学计算函数，`EA`
的一些应用和服务需要使用`MKL`库来提高计算效率。
具体安装步骤参见[安装oneAPI](inf/oneapi.md)。

# EA 安装

`EA` 提供的库都使用了`carbin`进行管理，所以安装`EA`的库也是使用`carbin`进行安装。如安装 `turbo` 库：

```shell
  carbin install gottingen/turbo@v0.5.6 --prefix=/opt/EA/inf
```

`EA` 大部分库都编译了动态库和静态库，静态库编译时都会带上`-fPIC`选项，所以可以直接使用静态库进行编译。并且非常方便`cmake`
构建系统的使用。
cmake中使用`find_package`命令就可以找到`EA`的库。所有库都会导出`${xxx_INCLUDE_DIRS}`, `${xxx_LIBRARIES}`等变量，方便用户使用。

如`turbo`库的使用：

```cmake
find_package(turbo REQUIRED)
target_include_directories(${PROJECT_NAME} PRIVATE ${turbo_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} turbo::turbo)
```

或者使用carbin提供的模版：

```cmake
find_package(turbo REQUIRED)
carbin_cc_library(
    NAMESPACE you_project
    NAME your_lib
    SOURCES ${your_lib_srcs}
    PUBLIC_LINKS turbo::turbo # or turbo::turbo_static
    INCLUDES ${turbo_INCLUDE_DIRS}
    PUBLIC
)
```


导出的对象 `turbo::turbo` 就是`turbo`库的动态库，`turbo::turbo_static` 就是`turbo`库的静态库。其他项目也是类似。


[1]: https://github.com/gottingen/carbin

[2]: https://carbin.readthedocs.io/en/latest/

[3]: https://github.com/eainf