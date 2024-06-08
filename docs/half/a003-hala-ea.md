hala ea 03
==============================

本节要点:

* 使用`carbin`管理项目依赖库 `gtest`, `gmock`, `benchmark`
* 为项目集成单元测试。
* gtest, gmock 使用。

服务的稳定性和可靠性是我们一直在追求的目标，然而要让一个服务稳定可靠，并非一件容易得事情，服务的稳定，可靠，依赖每个组件的稳定，
可靠，而每个组件的稳定，可靠。组件稳定性方面，单元测试是一个非常重要的手段，通过单元测试，我们可以保证组件的稳定性，可靠性。

经常听到有人说，这个功能我写完了，但是不知道对不对，怎么办？这个时候，单元测试就派上用场了，通过单元测试，我们可以验证我们的功能是否正确。

目前业界使用最广泛的单元测试框架是`gtest`，`gmock`，`benchmark`，这三个库是google开源的，非常好用，而且功能强大。
其他的单元测试框架也有很多，比如`catch2`，`doctest`等等，这里不做介绍。`EA`中，我们使用`gtest`，`gmock`，`benchmark`。

本节代码仓库位于[hallo-ea-03][1]


## 初始化仓库

```shell
mkdir halaltest
cd halaltest
carbin create --name halaltest --test --benchmark --example --requirements
```

## 安装依赖

carbin 自动创建了`carbin_deps.txt`文件，默认已经集成经过`EA`封装的`gottingen/carbin-recipes@ea`，`testing`库,
已经集成了`gtest`, `gmock`, `benchmark`库，保持不变即可,注意

* `#`开头的行是注释行，不会被`carbin`处理。
*  `EA`集成的`gtest`, `gmock`, `benchmark`库，对官方的库修改了编译的选项，官方的库默认使用`c++14`编译，`EA`使用`c++17`编译。
   使用`c++14`编译，会导致在测试c++17的数据结构时报错，比如`std::optional`，`std::variant`， `std::string_view`等等。

carbin_deps.txt 内容如下:

```text
############
# ea recipes
############
gottingen/carbin-recipes@ea

#########################
# testing tools googlegtest benchmark
testing
```

在`halaltest`目录下安装依赖库

```shell
carbin install
```
运行`carbin install`命令后，`carbin`会自动下载依赖库，解压，编译，安装，安装在当前目录的`carbin`目录下。**不要忘记在`.gitignore`文件中添加`carbin`目录**。

## 编写代码

与前面的例子`a002-hala-ea`很类似，这次我们只需要常见一个库，提供自有计算机以来，最简单的`add`函数。具体参见源码。

## 编写cmake构建文件

修改`halatest`目录下的`CMakeLists.txt`文件，添加`halalib`库的构建方式。

```cmake
carbin_cc_library(
        NAMESPACE halatest
        NAME add
        SOURCES
        add.cc
        CXXOPTS
        ${CARBIN_CXX_OPTIONS}
        PLINKS
        ${CARBIN_DEPS_LINK}
        PUBLIC
)
```

写到这里，我们可以尝试编译一下，看看是否能够通过编译。
在项目的根目录下创建`build`目录，进入`build`目录，执行`cmake`命令。
```shell
mkdir build
cd build
cmake ..
```
编译通过，我们可以继续下一步。

## 编写单元测试

`carbin`工具已经为我们创建了一个`tests`目录，我们在`tests`目录下创建一个`add_test`文件，添加如下代码:

```c++
#include <gtest/gtest.h>

#include <halatest/add.h>

TEST(AddTest, Add) {
    EXPECT_EQ(hala::add(1, 2), 3);
    EXPECT_EQ(hala::add(2, 3), 5);
    EXPECT_EQ(hala::add(3, 4), 7);
}
```

## 编写cmake构建文件

修改`tests`目录下的`CMakeLists.txt`文件，添加`add_test`的构建方式。

```cmake
find_package(GTest REQUIRED)

carbin_cc_test(
        NAME add_test
        MODULE base
        SOURCES add_test.cc
        LINKS halatest::add GTest::gtest GTest::gtest_main
)
```

## 编译测试

在`halatest/build`目录下执行`cmake`命令，编译测试。

```shell
rm -rf *
cmake .. -DCARBIN_BUILD_TEST=ON
make
make test
```
可以看到测试通过了。
这里要注意的是，`cmake`在第一次编译时，已经生成一个`CMakeCache.txt`文件，这个文件里并没有`CARBIN_BUILD_TEST`这个变量，所以我们需要
在第一次编译时，删除`build`目录下的`CMakeCache.txt`文件，然后重新执行`cmake`命令。
看到这里，你已经可以使用`carbin`管理你的项目了，`carbin`提供了很多的构建函数，并且可以完成90%的单元测试，基准测试的构建工作。


## 测试扩展，继续添加测试

接下来，上面我们测试了不需要输入参数的测试，90%的情况下，我们的测试都是不需要输入参数的，但是有时候，我们需要输入参数的测试，这个时候，
我们可以使用`carbin`的`carbin_cc_test`函数，以及`carbin`的`carbin_cc_test_ext`函数配合使用，构建我们的测试。

我们继续添加一个测试，测试`carbin`对`test`的支持, 我们编译一个args_test.cc文件，测试`carbin`对`test`的支持。

```c++
#include <signal.h>
#include <stdio.h>

int main(int argc, char**argv)
{
    if(argc !=2) {
        fprintf(stdout, "Failed\n");
        fflush(stdout);  /* ensure the output buffer is seen */
        return 0;
    }
    fprintf(stdout, "%s\n", argv[1]);
    fflush(stdout);  /* ensure the output buffer is seen */
    //raise(SIGABRT);
    return 0;
}
```
这个文件的内容，很简单，接受一个参数，打印出来，然后退出。接下来的不要眨眼睛，`CMakelists.txt`编写才是我们的重头戏。
继续在 `CMakelists.txt`文件中添加如下代码:
```cmake
carbin_cc_test(
        NAME args_test
        MODULE base
        SOURCES args_test.cc
        EXT
)

# base_args_test_fail fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS fail
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
)

# base_args_test_fail fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS fail_args
        ARGS "Failed"
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
)

# base_args_test_skip fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS skip_fail
        PASS_EXP "pass;Passed"
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
        SKIP_EXP "[^a-z]Skip" "SKIP" "Skipped"
)

# base_args_test_skip fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS skip
        ARGS "SKIP"
        PASS_EXP "pass;Passed"
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
        SKIP_EXP "[^a-z]Skip" "SKIP" "Skipped"
)

# base_args_test_skip fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS skip_pass
        ARGS "Passed"
        PASS_EXP "pass;Passed"
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
        SKIP_EXP "[^a-z]Skip" "SKIP" "Skipped"
)

# base_args_test_skip fail
carbin_cc_test_ext(
        NAME args_test
        MODULE base
        ALIAS skip_diabled
        ARGS "Passed"
        PASS_EXP "pass;Passed"
        FAIL_EXP "[^a-z]Error;ERROR;Failed"
        SKIP_EXP "[^a-z]Skip" "SKIP" "Skipped"
        DISABLE
)
```
`carbin_cc_test`细心的你会发现和上一个例子的`carbin_cc_test`不同，多了一个`EXT`参数，这个参数是用来扩展测试的，`carbin_cc_test_ext`函数
是`carbin`提供的一个扩展测试的函数，可以用来扩展测试的功能，比如测试参数，测试失败，测试跳过等等。

`carbin_cc_test_ext`函数有很多参数，这里我们只介绍几个常用的参数:

* `ALIAS` 用来给测试取别名，这个别名可以用来区分测试，比如`fail`, `skip`, `skip_fail`, `skip_pass`, `skip_disabled`等等。
* `ARGS` 用来给测试传递参数，这个参数可以用来测试测试参数的功能。
* `PASS_EXP` 用来测试测试通过的条件，这个条件可以是一个正则表达式，也可以是一个字符串。
* `FAIL_EXP` 用来测试测试失败的条件，这个条件可以是一个正则表达式，也可以是一个字符串。
* `SKIP_EXP` 用来测试测试跳过的条件，这个条件可以是一个正则表达式，也可以是一个字符串。
* `DISABLE` 用来禁用测试，这个测试不会被执行。

这里我们测试了`args_test`测试，测试了测试参数，测试失败，测试跳过等等。这个测试是一个非常简单的测试，但是却包含了很多的测试用例。
重新编译测试，运行测试，看看测试结果。

```shell
rm -rf *
cmake .. -DCARBIN_BUILD_TEST=ON
make
make test
```
我们定义了7个测试用例，其中一个是禁用的，所以只有6个测试用例被执行，最终的结果会运行6个测试用例，名字包含`fail`的测试用例会失败。

结合自己的应用项目，尝试添加更多的测试用例，测试参数，测试失败，测试跳过等等。


## 总结

`carbin`提供了很多的构建函数，可以帮助我们快速构建一个c++应用，一个c++库，一个c++测试，一个c++基准测试，`carbin`提供了很多的构建函数，
可以帮助我们快速构建一个c++应用，一个c++库，一个c++测试，一个c++基准测试，`carbin`提供了很多的构建函数。

目前为止，我们介绍了`carbin`的一些基本用法，有好奇心的朋友，会比较好奇，`carbin_cc_test`，`carbin_cc_library`,`carbin_cc_binary`函数是如何实现的，接下来的部分会陆续展开
`carbin template`中的一些实现细节, 感兴趣的朋友可以继续关注。



[1]:https://github.com/gottingen/ea-half-an-hour/tree/master/a003-hala-ea