基础环境
=============================

# 系统基础环境

* git
* cmake >= 3.24
* gcc >= 9.3
* g++ >= 9.3
* make
* python >= 3.8
* openssl >= 1.1.1
* zlib
* bz2
* lz4
* zstd
* carbin

centos-7 安装参考[centos](centos.md)， ubuntu 安装参考[ubuntu](ubuntu.md)

``EA`` 提供了centos7基础环境的容器，可以直接使用

```shell
docker pull lijippy/ea_inf:c7_base_v1
docker run -it lijippy/ea_inf:c7_base_v1 /bin/bash
``` 

[返回首页](../../README.md)
