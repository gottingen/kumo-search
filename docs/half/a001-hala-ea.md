hala ea 01
==============================

本节主要介绍如何建立第一个c++项目，以及如何使用`carbin`管理项目依赖库。

* 安装基础环境
* 使用carbin创建项目
* 编译项目

## 安装基础环境

基础环境安装参见[基础环境](../inf/inf.md)。这部分内容在20分钟内应该可以完成，如果使用docker，可以直接使用`lijippy/ea_inf:c7_base_v1`镜像。
可以省略这一步。如果是初学者，对linux还是不太熟悉，建议``先试用docker进行下一步操作``, 熟悉之后再进行本地安装一次，熟悉linux的基本操作和基础环境
的安装。

## 使用carbin创建项目

`carbin`是一个c++项目管理工具，可以帮助用户快速创建c++项目，管理项目依赖库，以及编译项目。`carbin`的使用非常简单，只需要一个命令就可以创建一个c++项目。

```shell
    mkdir hala_ea
    cd hala_ea
    carbin create --name halaea --test --benchmark --example --requirements
```
查看本地目录结构：

```shell
    tree -L 1
```

```shell
    .
    ├── CMakeLists.txt
    ├── README.md
    ├── benchmark
    ├── carbin_cmake
    ├── cmake
    ├── halaea
    ├── examples
    ├── carbin_deps.txt
    └── tests
```

运行cmake 命令编译项目

```shell
    mkdir build
    cd build
    cmake ..
    make
```

为项目构建发布版本

```shell
   make package
```
这时候会在build目录下生成一个package目录，里面包含了项目的发布版本。centos7系统上的发布版本是一个`.sh`文件，和一个`rpm`文件， ubuntu系统上的发布版本是一个`.deb`文件， 和一个',sh`文件。
如果分发安装，建议使用rpm或者deb文件分发。

到此为止，第一个c++项目就建立完成了。下一节将介绍如何创建一个c++应用，以及如何使用`carbin`管理项目依赖库。

目前为止，很多朋友可能会比较惊奇，为什么要使用`carbin`能这么快的构建一个c++项目，很好，带着这个问题，我们继续下一节的内容。在未来的4节内容之后，自然会有答案。

[返回首页](../../README.md) 

[下一节](a002-hala-ea.md) - 创建一个c++应用，在cmake创建库并使用库                                   [专题主页](../halfanhour.md) - EA半小时
