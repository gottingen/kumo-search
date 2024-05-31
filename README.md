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

## 作者
* @author Jeff.li
* @email bohuli2048@gmail.com

[1]: images/kumo_search_logo.png
[2]: images/kumo_search.gif
[3]: images/K_64x64.png