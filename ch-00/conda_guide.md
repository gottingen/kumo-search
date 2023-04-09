conda guide
====

# why conda

It is a long term to explain that why I select conda to manage these
project.

at first, I use [cget](https://github.com/pfultz2/cget) to manage project dependencies.
it works well when a few project. When projects grow. it is difficult to
do these because it takes long time to compile all the project and if a mistake curred,
we need completely restart. Any way, cmake external_project can do the some
thing, but it need more coding every time and download project source code every time during 
develop.

so we desire to using a binary manage system, so that, we can depend a project
by release tag.

conda is a good choose, on anaconda, we can ge some free and easy upload and 
download our binaries, farther more, it provide sample configure for us to build
binary.

# install conda

It is easy to install, first you need to run `uname -m` in commandline to known
you computer arch. next confirm the package name `MINICONDA_FILENAME`.
after these two step, you can run the below, to install conda.
```shell
# Download and init conda
MINICONDA_FILENAME=Miniconda3-latest-MacOSX-x86_64.sh
curl -L -o $MINICONDA_FILENAME \
    "https://repo.continuum.io/miniconda/$MINICONDA_FILENAME"
bash ${MINICONDA_FILENAME} -b -f -p $HOME/miniconda3
export PATH=$HOME/miniconda3/bin:$PATH
```

run source $HOME/miniconda3/bin/activate to activate conda environment. 
```shell
source $HOME/miniconda3/bin/activate
```

# how to prepare a develop environment

see a example like [lambda environment](https://github.com/gottingen/lambda/blob/master/conda/environment.yaml) like this:
```yaml
name: lambda-dev
channels:
  - conda-forge
  - mgottingen
dependencies:
  - turbo
  - protobuf <=3.20.0
```
channels express that the source channel to get the binary packages.
like `turbo` package, it exists in the `mgottingen` channel.

protobuf <=3.20.0 express that protobuf version must less than 3.20.0,
you can also describe it like `protobuf >=3.6.0,<=3.20.0`, both set minus version
and max version.

when we got this file, we can do below to create the environment.
```shell
conda env create -f environment.yaml
conda activate lambda-dev
```
so you working in a environment have installed the packages you want.

# build a conda package