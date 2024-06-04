cnetos-7 系统环境配置
==============================

```shell
yum update -y
yum install -y epel-release
yum install -y git gcc gcc-c++ make openssl11 openssl11-devel  wget unzip zlib-devel bzip2-devel lz4-devel zstd-devel xz-devel
yum install -y rpm-build
# 安装 cmake
wget https://github.com/Kitware/CMake/releases/download/v3.24.3/cmake-3.24.3.zip
unzip cmake-3.24.3.zip
cd cmake-3.24.3
./bootstrap --prefix=/usr/local
make -j$(nproc)
make install
cd ..
rm -rf cmake-3.24.3 cmake-3.24.3.zip
# 安装 python 依赖
yum install -y libffi-devel ncurses-devel sqlite-devel readline-devel
yum install -y centos-release-scl
yum install -y devtoolset-9-gcc*
scl enable devtoolset-9 bash
ln -sf /usr/lib64/pkgconfig/openssl11.pc /usr/lib64/pkgconfig/openssl.pc
ln -sf /usr/bin/openssl11 /usr/bin/openssl
# 安装 python
wget https://registry.npmmirror.com/-/binary/python/3.10.6/Python-3.10.6.tgz
tar -zxvf Python-3.10.6.tgz
cd Python-3.10.6
./configure --enable-optimizations
make -j$(nproc)
make install
cd ..
rm -rf Python-3.10.6 Python-3.10.6.tgz
pip3 install --upgrade pip
pip3 install --upgrade setuptools
pip3 install carbin
# 设置启动时加载 gcc9
echo "source /opt/rh/devtoolset-9/enable" >> ~/.bashrc
```

[返回](inf.md)

[下一页 ubuntu](ubuntu.md)

[回到首页](../../README.md)
