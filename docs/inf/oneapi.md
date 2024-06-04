onapi安装
======================

install oneapi

on centos [intel original document][1]

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

on ubuntu [intel original document][2]

```shell
  sudo apt update
  sudo apt install -y gpg-agent wget
  wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null
  echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
  sudo apt update
  sudo apt install intel-oneapi-mkl
  sudo apt install intel-oneapi-mkl-devel
```

[1]: https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&linux-install=yum

[2]: https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&linux-install=apt