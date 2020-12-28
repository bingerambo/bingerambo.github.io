# 安装NVIDIA Docker2(NVIDIA Container V2)和NVIDIA K8S-GPU插件



Centos 7环境下，安装NVIDIA Container和K8S的GPU插件的操作命令

<!--more-->

- **Setting up NVIDIA Container Toolkit**

    NVIDIA Docker参考NVIDIA官网教程

    [NVIDIA Container Toolkit 官方安装说明](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#install-guide)

- **NVIDIA k8s-device-plugin 参考项目地址**

    [k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)

## NVIDIA Docker依赖

```shell
sudo yum install -y tar bzip2 make automake gcc gcc-c++ vim pciutils elfutils-libelf-devel libglvnd-devel iptables

### Setup the official Docker CE repository:

sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

### Now you can observe the packages available from the docker-ce repo:
sudo yum repolist -v

#### 生成yum缓存  
sudo yum makecache  
```

## NVIDIA Docker2



``` shell

### Clear installed old version package

# rpm -qa|grep nvidia
# yum info installed |grep nvidia

sudo yum remove -y nvidia-docker
sudo yum remove -y nvidia-docker2

## 如果原有版本使用rpm方式安装，则清理rpm包

rpm -qa|grep nvidia |grep -E "libnvidia-container|nvidia-container-runtime" |xargs rpm -e




### Setup the stable repository and the GPG key:

distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo


sudo yum clean expire-cache

### 生成yum缓存  
#sudo yum makecache  

sudo yum install -y nvidia-docker2


### Restart the Docker daemon to complete the installation after setting the default runtime:


sudo systemctl restart docker


```


## 验证

```shell
### t this point, a working setup can be tested by running a base CUDA container:
sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

安装成功，如下结果
```markdown

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.80.02    Driver Version: 450.80.02    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla P100-PCIE...  Off  | 00000000:3B:00.0 Off |                  Off |
| N/A   37C    P0    33W / 250W |      0MiB / 16280MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla P100-PCIE...  Off  | 00000000:86:00.0 Off |                  Off |
| N/A   37C    P0    32W / 250W |      0MiB / 16280MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   2  Tesla P100-PCIE...  Off  | 00000000:D8:00.0 Off |                  Off |
| N/A   36C    P0    27W / 250W |      0MiB / 16280MiB |      4%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+


```


## NVIDIA K8S Device plugin

这里使用镜像方式，更多方式，参考[k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)

### 拉取镜像

``` shell
docker pull nvidia/k8s-device-plugin:v0.7.3
docker tag nvidia/k8s-device-plugin:v0.7.3 nvidia/k8s-device-plugin:devel
```

### 运行镜像

以下方式2选1：

Without compatibility for the CPUManager static policy:
```shell
docker run \
    -it \
    --security-opt=no-new-privileges \
    --cap-drop=ALL \
    --network=none \
    -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins \
    nvidia/k8s-device-plugin:devel
```


With compatibility for the CPUManager static policy:
```shell
docker run \
    -it \
    --privileged \
    --network=none \
    -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins \
    nvidia/k8s-device-plugin:devel --pass-device-specs
```


## 附录

手动安装nvidia-docker(在有外网机器上面进行)，
**未测试验证，仅供参考**
```shell
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo

yum install --downloadonly nvidia-docker2 --downloaddir=/tmp/nvidia

##在拷贝到没有网路的服务器上面执行以下命令
rpm -ivh libnvidia-container1-1.1.1-1.x86_64.rpm libnvidia-container-tools-1.1.1-1.x86_64.rpm

rpm -ivh nvidia-container-runtime-3.2.0-1.x86_64.rpm nvidia-container-toolkit-1.1.2-2.x86_64.rpm 
```
