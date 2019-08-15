# VNPY

## 安装

### Centos篇

#### 版本

- [vnpy](https://github.com/vnpy/vnpy) v2.0.3
- Python v3.7.0
- centos 7

#### 下载vnpy

```sh
git clone —branch v2.0.3 https://github.com/vnpy/vnpy.git
```

#### 安装相关依赖

```shell
$ pip install --upgrade pip setuptools wheel
$ wget https://artiya4u.keybase.pub/TA-lib/ta-lib-0.4.0-src.tar.gz
$ tar -xvf ta-lib-0.4.0-src.tar.gz
$ cd ta-lib/
$ ./configure --prefix=/usr
$ make
$ sudo make install
$ pip install numpy
$ pip install --pre --extra-index-url https://rquser:ricequant99@py.ricequant.com/simple/ rqdatac
$ pip install ta-lib
$ pip install https://vnpy-pip.oss-cn-shanghai.aliyuncs.com/colletion/ibapi-9.75.1-py3-none-any.whl
$ sudo yum install postgresql-devel
$ pip install -r requirements.txt
```

验证下ta-lib是否安装成功

```shell
$ python
import talib
```

如果找不到报错libta_lib.so.0，则设置环境变量

```shell
$ vim ~/.bashrc
# 添加 
# export LD_LIBRARY_PATH=/usr/lib
$ source ~/.bashrc
```

最后安装vnpy模块

```shell
$ pip install .
```

这时候大概率报错，ubuntu是没看出啥错误信息的，centos报错gcc的命令错误不支持std=c++17

说明gcc版本太低，so升级gcc

```shell
$ yum -y install centos-release-scl
$ yum -y install devtoolset-7-gcc devtoolset-7-gcc-c++ devtoolset-7-binutils
$ scl enable devtoolset-7 bash
```

最后再次安装vnpy模块就可以了！