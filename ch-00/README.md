develop environment prepare
====

When develop through all projects, we using `conda` and `cmake` to manage
the projects.

all the project release its binary on anaconda by group `mgottingen`.
for example, the project need `turbo`, you can attach the conda environment,
and run:
```shell
conda install -c mgottingen turbo
```
so, you will install turbo to you conda environment, it can be found
in `$CONDA_PREFIX/include` and `$CONDA_PREFIX/lib`

so easily, all the project can be installed via the same pattern.

For more advanced and detailed usage, please refer to the `cmake` 
and `conda` detailed documentation below

* [cmake guide](cmake_guide.md)
* [conda guide](conda_guide.md)

[back](../README.md)