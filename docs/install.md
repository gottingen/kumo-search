安装与使用
==================

# EA 依赖

`EA`(Elastic automic infrastructure architecture)系列的应用和服务都会使用静态编译的方式进行部署，这样可以减少运行时的依赖，提高运行效率。
`EA`为了方便项目依赖和用户安装部署，提供了[carbin][1]工具，用于下载文件和安装，具体参开[carbin docs][2]. cmake构建系统
在`cicd`应用请参考[cmake有点甜](cicd/sweet_cmake.md)
使用carbin工具不仅可以安装第三方
依赖库，更重要的功能，carbin可以一键创建工程的cmake 构建系统，方便用户进行编译和部署。

`EA`基础开发环境的运行分为三大部分： `基础系统`、`基础依赖安装`、`EA inf安装`。

* cmake >= 3.24
* gcc/g++ >= 9.3
* oneapi >= 2024.1
* carbin
* gottingen/essential
    * gflags
    * benchmark-1.8.4
    * boost-1.85.0
    * faiss-1.8.0
    * gflags-2.2.2
    * googletest-release-1.12.1
    * leveldb-1.23
    * lz4-1.9.4
    * openssl-OpenSSL_1_1_1k
    * protobuf-3.20.0
    * rocksdb-6.29.3
    * snappy-1.2.0
    * zlib-ng-2.1.6
    * zstd-1.5.6
* gottingen/turbo
* gottingen/collie
* gottingen/alkaid
* gottingen/melon

# 依赖安装

## 基础系统

系统依赖：

* cmake >= 3.24
* gcc/g++ >= 9.3

## 基础依赖安装

install carbin

```shell
pip install carbin
```

install oneapi

on centos [intel original document][3]

```shell
  tee > /tmp/oneAPI.repo << EOF
  [oneAPI]
  name=Intel® oneAPI repository
  baseurl=https://yum.repos.intel.com/oneapi
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
  EOF
  sudo mv /tmp/oneAPI.repo /etc/yum.repos.d
  yum install intel-oneapi-mkl
```

on ubuntu [intel original document][4]

```shell
  sudo apt update
  sudo apt install -y gpg-agent wget
  wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null
  echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
  sudo apt update
  sudo apt install intel-oneapi-mkl
  sudo apt install intel-oneapi-mkl-devel
```

安装`gottingen/essential`依赖库。

```shell
  git clone https://github.com/gottingen/essential.git
  cd essential
  ./install.sh
```
注意：`gottingen/essential`依赖中的依赖库不与系统中的库冲突，都会被安装到`/opt/EA/inf`目录下。为了方便集成，对各个库的编译构建有稍许改动
* 静态编译
* 静态库增加`-fPIC`编译选项
* 增加rtti支持
* 增加`-std=c++17`编译选项

不要直接使用github上相应的库替代，编译选项和集成方式有所不同。更改他们的主要目的是为了让我们的应用能随时随地的部署，不受系统环境的影响。

## EA inf安装

EA inf的项目都是使用`carbin`工具进行构建的，所有项目都支持`deb`和`rpm`两种打包方式，方便用户部署。
到相应的github仓库中下载对应的`deb`或`rpm`包 centos使用 `rpm -ivh --nodeps` 安装，ubuntu使用 `dpkg -i xxx.deb` 安装。
安装顺序按照如下顺序：
* gottingen/turbo
* gottingen/collie
* gottingen/alkaid
* gottingen/melon

注意：rpm安装时，如果之前安装过，比如turbo，可能会报错，先删除之前的包再安装。
注意：rpm安装时，如果依赖库不全，可能会报错，增加`--nodeps`选项强制安装。这个主要原因是在centos7上使用scl工具安装gcc9.3，系统
的gcc是4.8，rpm再检查时检查的是系统的gcc，所以会报错。

EA inf也可以使用源码安装，例如：turbo

```shell
  git clone https://github.com/gottingen/turbo.git
  cd turbo
  git checkout xxx
  mkdir build
  cd build
  cmake .. --CMAKE_INSTALL_PREFIX=/opt/EA/inf
  make -j
  make package
  rpm -ivh --nodeps package/turbo-xxx.rpm # or make install
```

[1]: https://github.com/gottingen/carbin

[2]: https://carbin.readthedocs.io/en/latest/

[3]: https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&linux-install=yum

[4]: https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&linux-install=apt