![][1]

<p align="center">
    <a></a> <font face="黑体" color=#6628ff size=4>|&nbsp; </font></a>
    <a href="docs/product.md"><font face="黑体" color=#0099fc size=4>功能</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
    <a href="docs/install.md"><font face="黑体" color=#0099fc size=4>安装</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
    <a href="docs/develop.md"><font face="黑体" color=#0099fc size=4>开发</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
    <a href="docs/docs.md"><font face="黑体" color=#0099fc size=4>文档</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
    <a href="docs/lecture.md"><font face="黑体" color=#0099fc size=4>系列</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|&nbsp; </font></a>
    <a href="acknowledgments.md"><font face="黑体" color=#0099fc size=4>FAQ</font></a>
    <a></a> <font face="黑体" color=#6628ff size=4> &nbsp;|</font></a>
</p>

# 端到端搜索

`kumo search`是一个端到端搜索引擎框架，支持全文检索、倒排索引、正排索引、排序、缓存、索引分层、干预系统、特征收集、离线计算、存储系统等功能。`kumo search`
运行在 `EA`(Elastic automic infrastructure architecture)平台上，支持在多机房、多集群上实现`工程自动化`、`服务治理`、`实时数据`、`服务降级与容灾`等功能。

随着互联网的发展，全网搜索已经不再是获取信息的唯一途径。很多垂直的信息服务，如电商、社交、新闻等，都有自己的搜索引擎。
这些搜索引擎的特点是：数据量中等，业务复杂，用户体验要求高。这些搜索引擎的开发，需要大量的工程和算法支持。`kumo search`旨在
提供一套开箱即用的搜索引擎框架，帮助用户快速搭建自己的搜索引擎。在这个框架上，用户可以通过项目内的AOT编译器，用
`python`编写业务逻辑，框架会自动生成`c++`代码，并生成二进制动态库，动态更新到搜索引擎中。从而实现搜索引擎的快速迭代。

# 功能展示

![][2]

# 技术专题

* [cmake有点甜](cicd/sweet_cmake.md) - 利用cmake构建系统进行工程编译和部署，实现cicd自动化。
* [走近AI：向量检索](vecsearch/vector.md) - 向量检索是一种基于向量相似度的检索技术，本文介绍了向量检索的基本原理和应用场景, 以及kumo搜索引擎的实现。


# 项目概览

## 基础库项目

| 序号 | 项目名             | 说明                           | 进度                           |
|:---|:----------------|:-----------------------------|:-----------------------------|
| 1  | collie          | 头文件库                         | 完成                           |
| 2  | turbo           | 容器、日志                        | 完成                           |
| 3  | melon           | rpc网络库                       | 主体完成，待完善周边功能                 |
| 4  | alkaid 瑶光       | 文件操作                         | 本地部分完成，待增加hdfs，s3等存储支持       |
| 5  | mizar开阳         | kv引擎                         | 待开发wisekey功能，暂时先用rocksdb官方版本 |
| 6  | alioth玉衡        | 表格内存                         | 开发中                          |
| 7  | megrez天权        | 数据集读写                        | hdf5 cvs bin已完成，待封装高级c++api  |
| 8  | phecda 天玑       | 向量引擎内核                       | 开发中，部分功能暂时使用官方faiss          |
| 9  | merak天璇         | 综合搜索引擎内核                     | 待开发                          |
| 10 | dubhe 天枢        | nlp内核                        | 待开发                          |
| 11 | flare           | 张量计算                         | 完成，cpu版本待优化                  |
| 12 | theia           | 图像显示                         | 完成                           |
| 13 | dwarf           | jupyter协议c++内核               | 完成                           |
| 14 | exodus          | hercules and other jupyter应用 | 完成                           |
| 15 | hercules        | python aot编译器                | 完成,c++ interpreter 开发中       |
| 16 | carbin          | c++包管理器，cmake生成器             | 完成                           |
| 17 | carbin-template | cmake模板库                     | 完成                           |
| 18 | hadar           | suggest 搜索建议服务 内核            | 接近完成，商用不开源                   |

## kumo search 服务项目

| 序号 | 项目名       | 说明                                  | 进度        |
|:---|:----------|:------------------------------------|:----------|
| 1  | sirius    | EA元数据服务器 服务发现，全局时钟服务，全局配置服务， 全局id服务 | 完成        |
| 2  | polaris   | 向量引擎单机服务                            | 完成        |
| 3  | elnath    | 综合搜索引单机服务                           | 开发中       |
| 4  | vega      | 向量引擎数据库集群版                          | 完成 商用不开源  |
| 5  | arcturus  | 综合搜索引擎集群版                           | 开发中 商用不开源 |
| 6  | pollux    | 综合引擎业务控制台                           | 开发中 商用不开源 |
| 7  | capella   | ltr排序服务                             | 开发中 商用不开源 |
| 8  | aldebaran | suggest搜索建议服务集群                     | 开发中 商用不开源 |
| 9  | nunki     | nlp服务                               | 开发中 商用不开源 |

## 作者
* @author Jeff.li vicky codejie
* @email bohuli2048@gmail.com

[1]: images/kumo_search_logo.png
[2]: images/kumo_search.gif
[3]: images/K_64x64.png