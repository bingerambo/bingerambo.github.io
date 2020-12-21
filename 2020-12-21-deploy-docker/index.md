# 脚本部署Docker



脚本一键安装部署docker19.03

<!--more-->

## 安装脚本
- 使用阿里云镜像源
- docker参数 native.cgroupdriver=systemd

```shell
#!/bin/bash

# 安装docker

# VAR SET
DOCKER_VERSION="19.03.8"

echo "START to install docker $DOCKER_VERSION"
export REGISTRY_MIRROR=https://registry.cn-hangzhou.aliyuncs.com
# a) 检查和卸载旧版本(如果之前有安装docker)
echo "check and uninstall old docker..."
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# b) 配置yum repository
echo "config yum repository..."
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# c) 安装并启动docker

echo "install docker $DOCKER_VERSION"
yum install -y docker-ce-$DOCKER_VERSION docker-ce-cli-$DOCKER_VERSION containerd.io
systemctl enable docker
systemctl start docker


# d) 修改docker Cgroup Driver为systemd
echo "config docker Cgroup Driver: systemd"
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# e) 设置 docker 镜像，提高 docker 镜像下载速度和稳定性
echo "set docker mirror..."
curl -sSL https://kuboard.cn/install-script/set_mirror.sh | sh -s ${REGISTRY_MIRROR}
systemctl daemon-reload
systemctl restart docker
docker version
```


