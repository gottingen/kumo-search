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

before to build a conda package, a `meta.yaml` file is needed.
also take `libtext` package as example. see [libtext/conda](https://github.com/gottingen/libtext/tree/master/conda)
`meta.yaml`, `conda_build_config.yaml`, `bld.sh` the three files(assume that you are using posix system, if using windows
 please see bld.bat).

just change workdir to the libtext/conda and run:
```shell
conda build .
```
it will generate a conda package.

let see `meta.yaml` as below
```yaml
{% set version = environ.get('GIT_DESCRIBE_TAG').lstrip('v') %}
{% set number = GIT_DESCRIBE_NUMBER %}

package:
  name: libtext-pkg
  version: {{ version }}

build:
  number: {{ number }}

about:
  home: https://github.com/gottingen/libtext
  license: Apache License 2
  license_family: APACHE
  license_file: LICENSE
  summary: A c++ library for text processing for searching

source:
  git_url: ../

outputs:
  - name: libtext
    script: bld.sh   # [not win]
    script: bld.bat  # [win]
    build:
      string: "h{{ GIT_DESCRIBE_HASH }}_{{ number }}"
      run_exports:
        - {{ pin_compatible('libtext', exact=True) }}
    requirements:
      build:
        - turbo
      host:
        - turbo
      run:
        - turbo

    test:
      commands:
      #  - test -f $PREFIX/lib/libturbo.so              # [linux]
      #  - test -f $PREFIX/lib/libturbo.dylib           # [osx]
      #  - conda inspect linkages -p $PREFIX $PKG_NAME  # [not win]
      #  - conda inspect objects -p $PREFIX $PKG_NAME   # [osx]
```

the most important is the out put section
* mark the package name 'libtext'
* mark script 'bld.sh' this told conda how build it and install package to the location that conda tools specified
* requirements means what package deps. at this case, only deps turbo.

at this time, all things seems good, but, we knows that, it would not work.
because where conda can got turbo package. so we need to configure another
file `conda_build_config.yaml`.

```yaml
CONDA_BUILD_SYSROOT:   # [osx and x86_64]
  - /opt/MacOSX10.15.sdk        # [osx and x86_64]

MACOSX_SDK_VERSION:  # [osx and x86_64]
  - "10.15"  # [osx and x86_64]

MACOSX_DEPLOYMENT_TARGET:  # [osx and x86_64]
  - "10.15"  # [osx and x86_64]
python:
  - 3.6
  - 3.7
  - 3.8
channel_sources:
  - mgottingen
  - conda-forge
```
in this file, `channel_sources`section told conda what channel to get the package
the project deps, for example `turbo` will be ge in `mgottingen`.

the build script like
```shell
#!/bin/bash
set -e

mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$PREFIX \
        -DCMAKE_BUILD_TYPE=Release \
        -DTURBO_BUILD_TESTING=OFF \
        -DCMAKE_INSTALL_LIBDIR=lib \
        -DTURBO_BUILD_EXAMPLE=OFF \
        -DBUILD_SHARED_LIBS=ON

cmake --build .
cmake --build . --target install
```

this script do two things, build and install. caution that, you need
add `$CONDA_PREFIX` to your `CMAKE_PREFIX_PATH` before `find_package` in your cmake
script.