简易安装
===

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