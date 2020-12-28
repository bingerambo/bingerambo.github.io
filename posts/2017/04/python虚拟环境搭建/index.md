# Python虚拟环境搭建


>用 virtualenv 来管理多个开发环境，virtualenvwrapper 使得virtualenv变得更好用

<!--more-->

## python虚拟环境搭建


```shell
# 安装虚拟环境
pip install virtualenv
```

### 安装配置

* 安装:
(sudo) pip install virtualenv virtualenvwrapper
Linux/Mac OSX 下：

修改~/.bash_profile或其它环境变量相关文件(如 .bashrc 或用 ZSH 之后的 .zshrc)，添加以下语句
```shell
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/workspace
source /usr/local/bin/virtualenvwrapper.sh
#修改后使之立即生效(也可以重启终端使之生效)：

source ~/.bash_profile
```

Windows 下：
```shell
pip install virtualenvwrapper-win
```

* 1.设置环境变量
    - 设置WORK_HOME环境变量：例如，WORK_HOME ： D:\virtualenv
    
* 2.新建虚拟环境
```shell
mkvirtualenv virtualtest
```
注：因为前一步设置了WORK_HOME，所有虚拟环境将安装到 E:\virtualenv
* 3.查看安装的所有虚拟环境
```shell
workon
```

* 使用方法：
mkvirtualenv env_test：创建运行环境env_test

workon env_test: 工作在 env_test 环境 或 从其它环境切换到 env_test 环境

deactivate: 退出终端环境



* 其它的：

rmvirtualenv ENV：删除运行环境ENV

mkproject mic：创建mic项目和运行环境mic

mktmpenv：创建临时运行环境

lsvirtualenv: 列出可用的运行环境

lssitepackages: 列出当前环境安装了的包

创建的环境是独立的，互不干扰


* 列出所有虚拟环境
lsvirtualenv

* 激活虚拟环境
workon venv
　
* 进入虚拟环境目录
cdvirtualenv

* 进入虚拟环境的site-packages目录
cdsitepackages

* 列出site-packages目录的所有软件包
lssitepackages

* 停止虚拟环境
deactivate

*  删除虚拟环境
rmvitualenv venv

### 重建Python环境

* 冻结环境

所谓 冻结(freeze) 环境，就是将当前环境的软件包等固定下来:
```shell
# 安装包列表保存到文件packages.txt中
pip freeze >d:\packages.txt　　　
```

* 重建环境

重建(rebuild) 环境就是在部署的时候，在生产环境安装好对应版本的软件包，不要出现版本兼容等问题:
```shell
pip install -r d:\packages.txt # 配合pip，可以批量安装对应版本的软件包，快速重建环境，完成部署。
```




