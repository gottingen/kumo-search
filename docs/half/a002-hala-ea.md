hala ea 02
==============================

本节主要介绍如何创建一个c++应用，以及如何使用`carbin`管理项目依赖库, 源码仓库位于[hallo-ea-02][1]

# 创建一个库

本地创建一个库，库的名字是`halalib
    
    ```shell
    mkdir halalapp
    cd halalapp
    mkdir lib
    cd lib
    carbin create --name halalib --test --benchmark --example --requirements
    ```
    
删除掉`lib/halalib`中的cc和h文件，`注意，不要删除`lib/halalib/version.h.in`文件，这个文件是库的版本文件，`carbin`会自动更新这个文件的版本号。
参照源码文件中的源码创建和修改文件，修改`CMakeLists.txt`文件，添加`halalib`构建方式。完成后，进入`lib`目录，创建`build`目录，编译库。

    ```shell
    mkdir build
    cd build
    cmake ..
    make
    ```
编译通过后，可以在`build`目录下看到生成的库文件，`halalib/libhala.so`和`halalib/libhala.a`文件。

到目前为止，一个库就创建完成了。 这个库提供一个函数`std::string hala_version()`，返回库的版本号。
声明在`halalib/hala.h`文件中，实现在`halalib/hala.cc`文件中。

接下来，创建一个应用，使用这个库。

# 创建一个应用

在`halalapp`目录下创建一个应用，应用的名字是`halalapp`

    ```shell
    mkdir app
    cd app
    carbin create --name halalapp --test --benchmark --example --requirements
    ```
## 安装依赖
修改目录下的`carbin_deps.txt`文件，添加`halalib`库的依赖。本次我们使用本地路径，在最后一行添加`../lib即可。
此时，`carbin_deps.txt` 引用了三个依赖:
* gottingen/carbin-recipes@ea
* testing
* ../lib

../lib就是我们刚才创建的`halalib`库。
gottingen/carbin-recipes@ea 是`EA`一个菜谱，封装一些库，方便用户使用， 其中testing就是gottingen/carbin-recipes@ea封装的一个库，
包括`gtest`,`gmock`,`benchmark`等基础测试库。

在app 目录安装依赖库
    
        ```shell
        carbin install
        ```
是的， 这一条命令就依赖库安装好了，安装在当前目录的`carbin`目录下。通常，`carbin`目录里的文件会比较多，注意在`.gitignore`文件中添加`carbin`目录,
不要将`carbin`目录提交到代码仓库中。

## 在cmake中查找依赖库

修改`app/cmake/halaapp_deps.cmake`文件，添加`halalib`库的查找路径。在`find_package(Threads REQUIRED)`后添加`find_package(halalib REQUIRED)`

    ```cmake
    find_package(halalib REQUIRED)
    include_directories(${halalib_INCLUDE_DIR})
    ```
`include_directories(${halalib_INCLUDE_DIR})`是为了在编译时，找到`halalib`库的头文件。相当于`-I`选项。

## 编写应用代码

在`app/halaapp`目录下创建`app.cc`文件，添加如下代码

    ```c++
    #include <halalib/hala.h>
    #include <halaapp/version.h>
    #include <iostream>
    
    int main() {
        std::cout<<"hala lib version: "<<hala::hala_version()<<std::endl;
        std::cout<<"hala app version: "<<HALAAPP_VERSION_STRING<<std::endl;
        return 0;
    }
    ```
接下来编写app的构建文件`app/CMakeLists.txt`文件，添加如下代码

    ```cmake
    carbin_cc_binary(
        NAMESPACE halaapp
        NAME halaapp_static
        SOURCES
        app.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        halalib::hala_static
        ${CARBIN_DEPS_LINK}
        PUBLIC
    )

    carbin_cc_binary(
        NAMESPACE halaapp
        NAME halaapp_dynamic
        SOURCES
        app.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        LINKS
        halalib::hala_shared
        ${CARBIN_DEPS_LINK}
        PUBLIC
    )
    ```
这里构建了两个应用，一个是静态库，一个是动态库。`carbin_cc_binary`是`carbin-template`提供的一个构建函数，可以快速构建一个c++应用。这个函数
封装了`cmake`的构建过程，用户只需要提供应用的名字，源码文件，依赖库，编译选项等信息，就可以构建一个c++应用,后面的`PUBLIC`是为了将`halaapp`的应用
导出为cmake对象，其他应用可以通过`find_package(halaapp)`, `halaapp::halaapp_static`, `halaapp::halaapp_dynamic`来找到这个应用。

## 编译应用

在`app`目录下创建`build`目录，编译应用

    ```shell
    mkdir build
    cd build
    cmake ..
    make
    ```
此时，我们构建了第一个应用，可以看到`build/halaapp/`目录下生成了两个应用`halaapp_static`和`halaapp_dynamic`。
尝试运行一下

    ```shell
    ./halaapp_static
    ./halaapp_dynamic
    ```
在linux系统上可以通过`ldd`命令查看动态库的依赖关系

    ```shell
    ldd halaapp_dynamic
    ```
可以看到`halaapp_dynamic`依赖了`hala.so`库。

再通过 ``ls -l``查看观察两个应用的小大，可以看到`halaapp_static`比`halaapp_dynamic`大很多，这是因为`halaapp_static`包含了`halalib`库的代码，

# 总结

本节主要介绍了如何创建一个c++库和一个c++应用，以及如何使用`carbin`管理项目依赖库。`carbin`是一个c++项目管理工具，可以帮助用户快速创建c++项目，
管理项目依赖库，以及编译项目。`carbin`的使用非常简单，只需要一个命令就可以创建一个c++项目。

下一节将介绍如何创建一个如何基于 `EA` inf创建一个 http 服务，以及如何使用`carbin`管理有树状依赖的项目。

[1]:https://github.com/gottingen/ea-half-an-hour/tree/master/a002-hala-ea