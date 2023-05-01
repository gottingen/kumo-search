简易安装
===

# 安装
前面介绍了如何在自己服务器上编译tensorflow，如果借助conda工具，能帮助我们快速的安装和部署tensorflow。

创建conda环境并激活环境,python 版本可根据自己需要更改
```shell
conda create -n tf_env python=3.8
conda activate tf_env
```

安装tensorflow
```shell
conda install -c conda-forge libtensorflow_cc
```
这里要注意使用`conda-forge` channel

到这里tensorflow就安装成功了。接下来， 我们在使用c++接口时，需要注意，
tensoflow是被安装在 `miniconda3/envs/tf_env`目录中。
在cmake或是构建工具里，需要将头文件和lib路径引入即可。

conda提供环境变量`$CONDA_PREFIX`指向上面环境的目录。

cmake引入

```cmake
include_directories($CONDA_PREFIX/include)
link_directories($CONDA_PREFIX/lib)
```

# 编译

conda-forge同样提供了简易编译的模版，但是需要注意的是，通过这种方式，我们必要在conda环境下使用编译的包。

参照github上conda-forge的[编译模版](https://github.com/conda-forge/tensorflow-feedstock)
仓库会根据tensorflow当前的版本，一直保持最新的版本，如果要使用历史的版本，可以自己fork出来，finetune一下meta.yaml文件，进行本地编译。对于linux上的编译，确保编译前已经安装了docker。
```shell
python build-locally.py
```

官方的编译不会增加cpu指令集的选项，为了提升性能，我们要定制化的编译，也可以finetune编译模版，编译定制的版本的，编译后的产物，我们自己可以存放到自定义的conda channel中，用conda前端统一安装部署。