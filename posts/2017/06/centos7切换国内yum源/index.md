# centos7切换国内yum源


>配置国内阿里yum源

<!--more-->

## yum源配置步骤
根据官网的说明，分别有 CentOS 6、CentOS 7、CentOS 8等配置操作步骤。

### 1. 备份操作

备份，将 CentOS-Base.repo 为CentOS-Base.repo.backup
```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

### 2. 下载yum源配置文件
下载新的 <http://mirrors.aliyun.com/repo/Centos-7.repo>，并命名为CentOS-Base.repo
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
或者
```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

### 3. 清除缓存
```shell
# 清除系统所有的yum缓存
yum clean all   
# 生成yum缓存  
yum makecache     
```


## epel源 安装和配置
### 1. 查看可用的epel源
```shell
yum list | grep epel-release
```

示例：
```shell
[java@localhost yum.repos.d]$ yum list | grep epel-release
epel-release.noarch                         7-11                       extras   
[java@localhost yum.repos.d]$ 
```

### 2. 安装 epel
```shell
yum install -y epel-release

```

### 3. 配置阿里镜像提供的epel源
```shell
wget -O /etc/yum.repos.d/epel-7.repo  http://mirrors.aliyun.com/repo/epel-7.repo
```
或者
```shell
curl -o /etc/yum.repos.d/epel-7.repo  http://mirrors.aliyun.com/repo/epel-7.repo
```

### 4. 清除缓存
```shell
# 清除系统所有的yum缓存
yum clean all
# 生成yum缓存    
yum makecache     
```

### 5. 其它命令

```shell
#查看所有的yum源：

yum repolist all

#查看可用的yum源：

yum repolist enabled
```
