# tensorflow 编译安装

tensorflow 官方提供python、C、Java、Go的发布包,并没有提供c++版本的发型包。官方提供的包对指令集的支持也有限。要深入的tensorflow进行训练和推理，我们需要调试各种问题，自主编译tensorflow是一项必须得工作。
选择tensorflow的版本`v2.10.0`
# 环境准备

我们选择ubutntu 22.04作操作系统。

为了后续的环境维护方便，我们采用两层环境，第一层是操作系统本身的环境，这个环境下，提供bazel，gcc，以及c/c++的基础包如git，tar等。

第二层环境，使用conda作为工具，提供python环境为主。如numpy， python3等。

系统的环境配置
```shell
    sudo apt install gcc g++ git
    sudo apt install bazel-5.1.1
```
conda安装参见[conda guide](../../ch-00/conda_guide.md)

安装conda后，创建一个tensorflow的编译环境
```shell
conda create -n tf_build python=3.8
conda activate tf_build
```
# 编译

## 下载tensorflow

```shell
    git clone https://github.com/tensorflow/tensorflow.git
    git checkout v2.10.0
    cd tensorflow
```

## 编译配置

目录下有`configure.py`运行该文件，根据自己的需求选择配置，如果无gpu等特殊需求，一路按回车即可。
编译过程中，tensorflow会下载依赖，过程会比较长，要耐心等待，另外，也要注意bazel默认会启动电脑的所有cpu
进行编译，可能造成死机，可以增加一个`--local_cpu_resources`选项，选择一个cpu的数量小于机器的数量即可。

编译不需要GPU包
```shell
bazel build --config=opt //tensorflow:libtensorflow_cc.so //tensorflow/tools/pip_package:build_pip_package
```
编译需要GPU包
```shell
bazel build --config=opt --config=cuda //tensorflow:libtensorflow_cc.so //tensorflow/tools/pip_package:build_pip_package
```
构建whl包
```shell
./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

## 安装

安装tensorflow的pip包

```shell
cd /tmp/tensorflow_pkg
python3 -m pip install --user tensorflow-2.4.0-cp39-cp39-linux_x86_64.whl
```

编译成功后，多出来的几个目录分别是：`bazel-out` `bazel-testlogs` `bazel-bin` `bazel-tensorflow`

```shell
ls bazel-bin/tensorflow
_api      core                                                                 distribute   __init__.py.original               libtensorflow_cc.so.2.10.0           libtensorflow_framework.so.2.10.0           stream_executor
c         create_tensorflow.python_api_tf_python_api_gen_v2                    dtensor      install_headers.genrule_script.sh  libtensorflow_cc.so.2.10.0-2.params  libtensorflow_framework.so.2.10.0-2.params  tensorflow_cc.lib
cc        create_tensorflow.python_api_tf_python_api_gen_v2.runfiles           include      libtensorflow_cc.so                libtensorflow_framework.so           lite                                        tools
compiler  create_tensorflow.python_api_tf_python_api_gen_v2.runfiles_manifest  __init__.py  libtensorflow_cc.so.2              libtensorflow_framework.so.2         python

```

为了方便使用，整理了一套tensorflow编译安装的脚本，参见[tensorflow build](https://github.com/gottingen/tensorflow-build)。[简易安装](simple_install.md)




