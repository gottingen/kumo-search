![][50]

<p align="center">
    <a> <font face="黑体" color=#6628ff size=4>|&nbsp; </font></a>
    <a href="docs/product.md"><font face="黑体" color=#0099fc size=4>应用</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-&nbsp; </font></a>
    <a href="docs/install.md"><font face="黑体" color=#0099fc size=4>安装</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-&nbsp; </font></a>
    <a href="docs/develop.md"><font face="黑体" color=#0099fc size=4>开发</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-&nbsp; </font></a>
    <a href="docs/docs.md"><font face="黑体" color=#0099fc size=4>文档</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-&nbsp; </font></a>
    <a href="docs/lecture.md"><font face="黑体" color=#0099fc size=4>深度学习</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-&nbsp; </font></a>
    <a href="docs/faq/faq.md"><font face="黑体" color=#0099fc size=4>FAQ</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-</font></a>
    <a href="docs/tips/tips.md"><font face="黑体" color=#0099fc size=4>TIPS</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-</font></a>
    <a href="docs/lecture.md"><font face="黑体" color=#0099fc size=4>EA半小时</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;-</font></a>
    <a href="docs/lecture.md"><font face="黑体" color=#0099fc size=4>技术专题</font></a>
    <a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
</p>

# 端到端搜索

`kumo search`
是一个端到端搜索引擎框架，支持全文检索、倒排索引、正排索引、排序、缓存、索引分层、干预系统、特征收集、离线计算、存储系统等功能。`kumo search`
运行在 `EA`(Elastic automic infrastructure architecture)
平台上，支持在多机房、多集群上实现`工程自动化`、`服务治理`、`实时数据`、`服务降级与容灾`等功能。

随着互联网的发展，全网搜索已经不再是获取信息的唯一途径。很多垂直的信息服务，如电商、社交、新闻等，都有自己的搜索引擎。
这些搜索引擎的特点是：数据量中等，业务复杂，用户体验要求高。这些搜索引擎的开发，需要大量的工程和算法支持。`kumo search`旨在
提供一套开箱即用的搜索引擎框架，帮助用户快速搭建自己的搜索引擎。在这个框架上，用户可以通过项目内的AOT编译器，用
`python`编写业务逻辑，框架会自动生成`c++`代码，并生成二进制动态库，动态更新到搜索引擎中。从而实现搜索引擎的快速迭代。

# 功能展示

![][51]

# 项目概览

## 基础库项目

| 序号 | 项目名                   | 说明                                        | 说明                                 |
|:---|:----------------------|:------------------------------------------|:-----------------------------------|
| 1  | [collie][1]           | 引用外部header only library 如jason，toml等，统一管理 |                                    |
| 2  | [turbo][2]            | hash，log，容器类，字符串相关操作                      |                                    |
| 3  | [melon][3]            | rpc通信                                     |                                    |
| 4  | [alkaid][4]           | 文件系统封装、本地文件，hdfs，s3等                      | 文件系统统一api，zlib，lz4，zst unified api |
| 5  | [mizar][5]            | 基于rocksdb，toplingdb存储引擎内核                 | 待开发wisekey功能，暂时先用rocksdb官方版本       |
| 6  | alioth玉衡              | 表格内存                                      | 开发中                                |
| 7  | megrez天权              | 数据集读写                                     | hdf5 cvs bin已完成，待封装高级c++api        |
| 8  | [phekda][8]           | 统一向量引擎访问api UnifiedIndex，简化接口             | 支持snapshot，过滤插件                    |
| 9  | merak天璇               | 综合搜索引擎内核                                  | 待开发                                |
| 10 | dubhe 天枢              | nlp内核                                     | 待开发                                |
| 11 | [flare][11]           | gpu、cpu高维张量计算，等计算                         |                                    |
| 12 | [theia][12]           | 基于opengl图形图像显示，服务端不可用（无显示设备）              |                                    |
| 13 | [dwarf][13]           | jupyter协议c++内核                            |                                    |
| 14 | [exodus][14]          | hercules and other jupyter应用              | 完成                                 |
| 15 | [hercules][15]        | python aot编译器                             |                                    |
| 16 | [carbin][16]          | c++包管理器，cmake生成器                          | 完成                                 |
| 17 | [carbin-template][17] | cmake模板库                                  | 完成                                 |
| 18 | [carbin-recipes][18]  | carbin recipes 依赖库自定义配置                   | 完成                                 |
| 18 | hadar                 | suggest 搜索建议服务 内核                         | 接近完成，商用不开源                         |

## kumo search 服务项目

| 序号 | 项目名          | 说明                                  | 进度        |
|:---|:-------------|:------------------------------------|:----------|
| 1  | [sirius][31] | EA元数据服务器 服务发现，全局时钟服务，全局配置服务， 全局id服务 | 完成        |
| 2  | polaris      | 向量引擎单机服务                            | 完成        |
| 3  | elnath       | 综合搜索引单机服务                           | 开发中       |
| 4  | vega         | 向量引擎数据库集群版                          | 完成 商用不开源  |
| 5  | arcturus     | 综合搜索引擎集群版                           | 开发中 商用不开源 |
| 6  | pollux       | 综合引擎业务控制台                           | 开发中 商用不开源 |
| 7  | capella      | ltr排序服务                             | 开发中 商用不开源 |
| 8  | aldebaran    | suggest搜索建议服务集群                     | 开发中 商用不开源 |
| 9  | nunki        | nlp服务                               | 开发中 商用不开源 |

# [EA半小时](docs/halfanhour.md)

* [a001-hala-ea](docs/half/a001-hala-ea.md) - 基础环境安装，使用carbin创建项目
* [a002-hala-ea](docs/half/a002-hala-ea.md) - 创建一个c++应用，在cmake创建库并使用
* [a003-hala-ea](docs/half/a003-hala-ea.md) - 创建一个c++库，使用googletest进行单元测试
* [a004-hala-restful](docs/half/a004-hala-restful.md) - 使用melon库, 创建一个restful服务
* [a005-hala-echo](docs/half/a005-hala-echo.md) - 使用melon库, 创建一个echo服务
* [a006-hala-vue](docs/half/a006-hala-vue.md) - 创建一个cache服务，并提供浏览器访问界面
* [a007-hala-vue-ext](docs/half/a007-hala-vue-ext.md) - cache服务源码解读

# [搜索架构之旅](https://juejin.cn/user/42219472170896/columns)

## 走近AI：向量检索系列
* [一 向量检索为何受到工业界和开发者青睐？](https://juejin.cn/post/7381372851121831974?searchId=20240619101012ED67E7AC3C349325CB2F)

# 基础环境与CI/CD

`EA`是服务端应用的基础架构，`EA`目前支持`centos`和`ubuntu`两种操作系统，`mac`系统目前在开发中， 尽最大可能支持`mac`
系统。但目前并没有
尝试，为方便编译和ide开发，后续部分功能可能进行尝试兼容。基础环境部署参见[安装与使用](docs/inf/inf.md)

`EA`体系的`cicd`使用[carbin][4]工具进行管理。`carbin`是一个`c++`包管理器，`cmake`生成器，`cicd`工具。`carbin`可以下载第三方依赖库，
生成`cmake`构建系统，进行工程编译和部署。`carbin`的使用参见[carbin docs](https://carbin.readthedocs.io/zh-cn/latest/)

|       | carbin        | conda       | cmake   | CPM     | conan         | bazel       |
|-------|---------------|-------------|---------|---------|---------------|-------------|
| 使用复杂度 | easy          | middle      | hard    | middle  | hard          | hard        |
| 安装难度  | pip easy      | binary easy | NA easy | cmake   | pip easy      | binary hard |
| 依赖模式  | source/binary | binary      | source  | source  | source/binary | source      |
| 依赖树   | support       | support     | support | support | support       | support     |
| 本地源码  | support       | NA          | support | support | NA            | support     |
| 兼容性   | good          | middle      | good    | good    | good          | poor        |
| 速度    | good          | middle      | poor    | poor    | good          | poor        |
|       |               |             |         |         |               |             |

conda是一款不错的管理工具，没有选择conda，是因为conda的编译依赖项比较复杂，而且编译选项经常会出现问题，不太适合c++工程的编译。
cmake自带的管理工具，不太适合大型工程的管理，每次重新编译项目可能导致重新下载依赖库，编译时间过长。CPM是一个c++包管理器，同样，在国内的网络
环境下，下载依赖库速度较慢，不太适合大型工程的管理。conan是一个c++包管理器，但是conan的依赖库下载速度较慢，不太适合大型工程的管理。

同时carbin也是非常适合c++工程的管理，carbin能够快速生成c++项目管理cmake体系，统一了项目编译过程，选项配置，以及编译后安装导出的变量规则，
`EA`体系的项目可以通过固定规则`find_package`找到项目和项目对象.当时也适合任何基于`cmake`的项目使用。

如果基于docker开发，`EA` 提供了已经基础开发[ea inf](https://hub.docker.com/repository/docker/lijippy/ea_inf/general)容器:

centos7-openssl11-python-310-gcc-9.3:

    lijippy/ea_inf:c7_base_v1

# [服务与应用](docs/product.md)

* 天空中最亮的星 ———— [集群元数据服务](docs/application/sirius/sirius.md) - 服务发现，全局时钟服务，全局配置服务，全局id服务

# 技术专题

* [cmake有点甜](cicd/sweet_cmake.md) - 利用cmake构建系统进行工程编译和部署，实现cicd自动化。
* [走近AI：向量检索](vecsearch/vector.md) - 向量检索是一种基于向量相似度的检索技术，本文介绍了向量检索的基本原理和应用场景,
  以及kumo搜索引擎的实现。

## 作者

* @author Jeff.li vicky codejie
* @email bohuli2048@gmail.com

[1]: https://github.com/gottingen/collie

[2]: https://github.com/gottingen/turbo

[3]: https://github.com/gottingen/melon

[4]: https://github.com/gottingen/alkaid

[5]: https://github.com/gottingen/mizar

[8]: https://github.com/gottingen/phekda

[11]: https://github.com/gottingen/flare

[12]: https://github.com/gottingen/theia

[13]: https://github.com/gottingen/dwarf

[14]: https://github.com/gottingen/exodus

[15]: https://github.com/gottingen/hercules

[16]: https://github.com/gottingen/carbin

[17]: https://github.com/gottingen/carbin-template

[18]: https://github.com/gottingen/carbin-recipes

[31]: https://github.com/gottingen/sirius


[50]: images/kumo_search_logo.png

[51]: images/kumo_search.gif

[52]: images/K_64x64.png