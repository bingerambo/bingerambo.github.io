# 脚本部署Python3




脚本一键安装部署Python3

<!--more-->

## 安装脚本
- centos系统自带默认python2
- py3命令需要跟py2进行区别

```shell
#! /bin/bash
yum -y install zlib-devel bzip2-devel libffi-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel wget gcc

wget https://www.python.org/ftp/python/3.7.1/Python-3.7.1.tgz

mkdir -p /usr/local/python3
tar -xf Python-3.7.1.tgz
yum install libffi-devel -y
cd Python-3.7.1
pwd
./configure --prefix=/usr/local/python3
make
make install
ln -s /usr/local/python3/bin/python3 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
echo 'PATH=$PATH:$HOME/bin:/usr/local/python3/bin' >>/etc/profile
echo 'export PATH' >>/etc/profile
source /etc/profile
```


