# K8S集群性能测试-kubemark


了解如何使用kubemark对k8s组件进行性能测试

<!--more-->

## 1 背景
项目想对k8s组件进行集群性能测试。原有组件如调度器，已有的测试工具多是单元测试。需要寻找一种可以对k8s集群进行性能测试。比如多多节点大集群规模下的调度器性能指标如何？

考虑使用k8s项目自带的性能测试组件kubemark。

## 2 kubemark

### 介绍
kubemark 是 K8s 官方给出的性能测试工具，能够不受任何资源限制，模拟出一个大规模 K8s 集群。其主要架构如图所示:需要一个外部 K8s 集群（external cluster） 以及一个机器节点运行 kubemark master，即另外一个 K8s 集群，但是只有一个 master 节点。我们需要在 external cluster 中部署运行 hollow pod，这些 pod 会主动向 kubemark 集群注册，并成为 kubemark 集群中的 hollow node(虚拟节点)。然后我们就可以在 kubemark 集群中进行 e2e 测试。虽然与真实集群的稍微有点误差，不过可以代表真实集群的数据。

{{< figure src ="/images/2020-12-08/kubemark.png">}}

<br>

**本文则只构造了kubemark组件，且只使用了测试集群，即外部 K8s 集群（external cluster），未使用第2个kubemark集群。目的为测试集群中的master组件，如调度器和控制器等。另外，此方式还可以自己使用第三方测试工具和框架**

<br>

{{< figure src ="/images/2020-12-08/k8s-perf.png">}}

### kubemark构造

#### 1. 编译kubemark

在 K8s 源码路径下构建 kubemark，生成的二进制文件在 _output/bin 目录下。
```shell
# KUBE_BUILD_PLATFORMS=linux/amd64 make kubemark GOFLAGS=-v GOGCFLAGS="-N -l"

 make kubemark GOGCFLAGS="-N -l"
```

#### 2. 构建kubemark镜像

将生成的 kubemark 二进制文件从 _output/bin 复制到 cluster/images/kubemark 目录下。 
```shell
cp _output/bin/kubemark cluster/images/kubemark/
```

并在该目录下执行构建镜像命令，生成镜像：staging-registry.cn-hangzhou.aliyuncs.com/google_containers/kubemark:v1.14.8。

```shell
# IMAGE_TAG=v1.14.3 make build
cd cluster/images/kubemark/
IMAGE_TAG=v1.14.8 make build

```

#### 3. 保存镜像至kubemark.tar


#### 4. kubemark部署到测试集群
在测试集群中的所有node节点中，导入该kubemark镜像。用于启动桩节点。
接下来进行桩节点hollow-node启动配置操作

``` shell
# 以下命令在测试集群的master节点上执行
# 从kubemark-master节点（191节点）拷贝过来kubeconfig文件，到测试集群的master节点中
scp -r 192.168.182.191:/root/.kube/config /home/wangb/

kubectl create ns kubemark

kubectl create configmap node-configmap -n kubemark --from-literal=content.type="test-cluster"

# kubectl create secret generic kubeconfig --type=Opaque --namespace=kubemark --from-file=kubelet.kubeconfig=config --from-file=kubeproxy.kubeconfig=config

kubectl create secret generic kubeconfig --type=Opaque --namespace=kubemark --from-file=kubelet.kubeconfig=/root/.kube/config --from-file=kubeproxy.kubeconfig=/root/.kube/config

```



#### 5. 在测试集群中启动hollow nodes
```shell
kubectl create -f hollow-node-sts.yaml -n kubemark
```




### 测试pod

启动桩节点，hollow-node-sts.yaml的默认配置如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hollow-node
  namespace: kubemark
spec:
  clusterIP: None
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: hollow-node
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: hollow-node
  namespace: kubemark
spec:
  podManagementPolicy: Parallel
  replicas: 6
  selector:
    matchLabels:
      name: hollow-node
  serviceName: hollow-node
  template:
    metadata:
      labels:
        name: hollow-node
    spec:
      initContainers:
      - name: init-inotify-limit
        image: docker.io/busybox:latest
        imagePullPolicy: IfNotPresent
        command: ['sysctl', '-w', 'fs.inotify.max_user_instances=200']
        securityContext:
          privileged: true
      volumes:
      - name: kubeconfig-volume
        secret:
          secretName: kubeconfig
      - name: logs-volume
        hostPath:
          path: /var/log
      containers:
      - name: hollow-kubelet
        image: staging-registry.cn-hangzhou.aliyuncs.com/google_containers/kubemark:v1.14.8
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 4194
        - containerPort: 10250
        - containerPort: 10255
        env:
        - name: CONTENT_TYPE
          valueFrom:
            configMapKeyRef:
              name: node-configmap
              key: content.type
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        command:
        - /bin/sh
        - -c
        - /kubemark --morph=kubelet --name=$(NODE_NAME) --kubeconfig=/kubeconfig/kubelet.kubeconfig $(CONTENT_TYPE) --alsologtostderr --v=2
        volumeMounts:
        - name: kubeconfig-volume
          mountPath: /kubeconfig
          readOnly: true
        - name: logs-volume
          mountPath: /var/log
        resources:
          requests:
            cpu: 20m
            memory: 50M
        securityContext:
          privileged: true
      - name: hollow-proxy
        image: staging-registry.cn-hangzhou.aliyuncs.com/google_containers/kubemark:v1.14.8
        imagePullPolicy: IfNotPresent
        env:
        - name: CONTENT_TYPE
          valueFrom:
            configMapKeyRef:
              name: node-configmap
              key: content.type
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        command:
        - /bin/sh
        - -c
        - /kubemark --morph=proxy --name=$(NODE_NAME) --use-real-proxier=false --kubeconfig=/kubeconfig/kubeproxy.kubeconfig $(CONTENT_TYPE) --alsologtostderr --v=2
        volumeMounts:
        - name: kubeconfig-volume
          mountPath: /kubeconfig
          readOnly: true
        - name: logs-volume
          mountPath: /var/log
        resources:
          requests:
            cpu: 20m
            memory: 50M
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
            nodeSelectorTerms:
            - matchExpressions:
              - key: name
                operator: NotIn
                values:
                - hollow-node
              - key: node-role.kubernetes.io/master
                operator: NotIn
                values:
                - "true"

```
由上可知，hollow-node实际上是启动过了kubelet和proxy的2个进程，后来分析源码确实如此。


写个测试pod，验证桩node是否可用，test-pod.yaml如下
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: docker.io/busybox:latest
    imagePullPolicy: IfNotPresent
    command: ['sleep', '3600']
    securityContext:
      privileged: true

  affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/node
                operator: NotIn
                values:
                - "true"
```


节点信息 hollow-node-0, 此信息为默认信息
```shell
Name:               hollow-node-0
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=hollow-node-0
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
CreationTimestamp:  Mon, 07 Dec 2020 17:25:15 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 08 Dec 2020 09:37:21 +0800   Mon, 07 Dec 2020 17:25:15 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 08 Dec 2020 09:37:21 +0800   Mon, 07 Dec 2020 17:25:15 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 08 Dec 2020 09:37:21 +0800   Mon, 07 Dec 2020 17:25:15 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Tue, 08 Dec 2020 09:37:21 +0800   Mon, 07 Dec 2020 17:25:15 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.233.96.39
  Hostname:    hollow-node-0
Capacity:
 cpu:                1
 ephemeral-storage:  0
 memory:             3840Mi
 pods:               110
Allocatable:
 cpu:                1
 ephemeral-storage:  0
 memory:             3840Mi
 pods:               110
System Info:
 Machine ID:
 System UUID:
 Boot ID:
 Kernel Version:             3.16.0-0.bpo.4-amd64
 OS Image:                   Debian GNU/Linux 7 (wheezy)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://1.13.1
 Kubelet Version:            v0.0.0-master+4d3c9e0c
 Kube-Proxy Version:         v0.0.0-master+4d3c9e0c
PodCIDR:                     10.233.98.0/24
Non-terminated Pods:         (3 in total)
  Namespace                  Name                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                               ------------  ----------  ---------------  -------------  ---
  kube-system                calico-node-29z7g                  150m (15%)    300m (30%)  64M (1%)         500M (12%)     16h
  kube-system                kube-proxy-8wcss                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                metrics-server-84d6dbd58b-6vqmf    50m (5%)      145m (14%)  125Mi (3%)       375Mi (9%)     16h

```


作业运行到hollow node 
```shell
[root@node1 .kube]# kubectl get po -A -owide
NAMESPACE     NAME                              READY   STATUS              RESTARTS   AGE    IP                NODE            NOMINATED NODE   READINESS GATES
default       myapp-pod                         1/1     Running             0          6s     10.94.251.209     hollow-node-1   <none>           <none>
kube-system   calico-node-77l7w                 0/1     Init:0/1            0          20m    10.233.96.20      hollow-node-3   <none>           <none>
kube-system   calico-node-fhlsn                 0/1     Init:0/1            0          20m    10.233.96.21      hollow-node-0   <none>           <none>
kube-system   calico-node-g9fx8                 0/1     Init:0/1            0          20m    10.233.96.19      hollow-node-5   <none>           <none>
kube-system   calico-node-gdrlk                 0/1     Init:0/1            0          20m    10.233.96.17      hollow-node-1   <none>           <none>
kube-system   calico-node-lnzc7                 0/1     Init:0/1            0          20m    10.233.96.22      hollow-node-4   <none>           <none>
kube-system   calico-node-lt9jh                 0/1     Init:0/1            0          20m    10.233.96.18      hollow-node-2   <none>           <none>
kube-system   calico-node-ms5t6                 1/1     Running             0          51m    192.168.182.102   node2           <none>           <none>
kube-system   calico-node-vxcb7                 1/1     Running             0          51m    192.168.182.101   node1           <none>           <none>
kube-system   coredns-654c8ccdc8-575qx          1/1     Running             0          53m    10.233.90.74      node1           <none>           <none>
kube-system   coredns-654c8ccdc8-m8ntn          0/1     Pending             0          174d   <none>            <none>          <none>           <none>
kube-system   dns-autoscaler-77445c9c8b-b565h   1/1     Running             0          53m    10.233.90.65      node1           <none>           <none>
kube-system   kube-apiserver-node1              1/1     Running             11         51m    192.168.182.101   node1           <none>           <none>
kube-system   kube-controller-manager-node1     1/1     Running             11         51m    192.168.182.101   node1           <none>           <none>
kube-system   kube-proxy-72xn7                  0/1     ContainerCreating   0          20m    10.233.96.21      hollow-node-0   <none>           <none>
kube-system   kube-proxy-8jw9r                  0/1     ContainerCreating   0          20m    10.233.96.19      hollow-node-5   <none>           <none>
kube-system   kube-proxy-92jzz                  0/1     ContainerCreating   0          20m    10.233.96.20      hollow-node-3   <none>           <none>
kube-system   kube-proxy-djxjg                  0/1     ContainerCreating   0          20m    10.233.96.18      hollow-node-2   <none>           <none>
kube-system   kube-proxy-dsghn                  1/1     Running             0          51m    192.168.182.101   node1           <none>           <none>
kube-system   kube-proxy-gjzq8                  1/1     Running             0          51m    192.168.182.102   node2           <none>           <none>
kube-system   kube-proxy-lpb8m                  0/1     ContainerCreating   0          20m    10.233.96.17      hollow-node-1   <none>           <none>
kube-system   kube-proxy-sfjlq                  0/1     ContainerCreating   0          20m    10.233.96.22      hollow-node-4   <none>           <none>
kube-system   kube-scheduler-node1              1/1     Running             11         51m    192.168.182.101   node1           <none>           <none>
kube-system   metrics-server-84d6dbd58b-rbrjd   0/2     ContainerCreating   0          20m    <none>            hollow-node-3   <none>           <none>
kubemark      hollow-node-0                     2/2     Running             0          20m    10.233.96.21      node2           <none>           <none>
kubemark      hollow-node-1                     2/2     Running             0          20m    10.233.96.17      node2           <none>           <none>
kubemark      hollow-node-2                     2/2     Running             0          20m    10.233.96.18      node2           <none>           <none>
kubemark      hollow-node-3                     2/2     Running             0          20m    10.233.96.20      node2           <none>           <none>
kubemark      hollow-node-4                     2/2     Running             0          20m    10.233.96.22      node2           <none>           <none>
kubemark      hollow-node-5                     2/2     Running             0          20m    10.233.96.19      node2           <none>           <none>
[root@node1 .kube]# kubectl describe po -ndefault       myapp-pod
Name:               myapp-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               hollow-node-1/10.233.96.17
Start Time:         Mon, 07 Dec 2020 16:33:50 +0800
Labels:             app=myapp
                    version=v1
Annotations:        <none>
Status:             Running
IP:                 10.94.251.209
Containers:
  app:
    Container ID:  docker://3033a0ef90209b22
    Image:         docker.io/busybox:latest
    Image ID:      docker://docker.io/busybox:latest
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Mon, 07 Dec 2020 16:33:54 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-fm7gj (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-fm7gj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-fm7gj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  17s   default-scheduler  Successfully assigned default/myapp-pod to hollow-node-1

```

上面的kubemark和测试hollow-node是缺省默认配置，无法满足项目的k8s测试需要，比如node的自定义资源描述等，所以还需对kubemark和测试hollow-node进行改造。

修改kubemark（kubelet），重新编译kubemark，重新部署kubemark，查看桩节点信息（*资源信息*）如下：

```shell
Name:               hollow-node-0
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=hollow-node-0
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
CreationTimestamp:  Tue, 08 Dec 2020 13:34:35 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 08 Dec 2020 13:34:35 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 08 Dec 2020 13:34:35 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 08 Dec 2020 13:34:35 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Tue, 08 Dec 2020 13:34:35 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.233.96.50
  Hostname:    hollow-node-0
Capacity:
 cpu:                  72
 ephemeral-storage:    15726578Mi
 inspur.com/gpu:       99999
 inspur.com/rdma-hca:  99999
 memory:               256Gi
 nvidia.com/gpu:       8
 pods:                 110
Allocatable:
 cpu:                  72
 ephemeral-storage:    15726578Mi
 inspur.com/gpu:       99999
 inspur.com/rdma-hca:  99999
 memory:               256Gi
 nvidia.com/gpu:       8
 pods:                 110
System Info:
 Machine ID:
 System UUID:
 Boot ID:
 Kernel Version:             3.16.0-0.bpo.4-amd64
 OS Image:                   Debian GNU/Linux 7 (wheezy)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://1.13.1
 Kubelet Version:            v0.0.0-master+4d3c9e0c
 Kube-Proxy Version:         v0.0.0-master+4d3c9e0c
PodCIDR:                     10.233.67.0/24
Non-terminated Pods:         (3 in total)
  Namespace                  Name                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                               ------------  ----------  ---------------  -------------  ---
  kube-system                calico-node-n5528                  150m (0%)     300m (0%)   64M (0%)         500M (0%)      14s
  kube-system                kube-proxy-fxcs4                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         14s
  kube-system                metrics-server-84d6dbd58b-sx7sl    50m (0%)      145m (0%)   125Mi (0%)       375Mi (0%)     8s
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource             Requests      Limits
  --------             --------      ------
  cpu                  200m (0%)     445m (0%)
  memory               195072k (0%)  893216k (0%)
  ephemeral-storage    0 (0%)        0 (0%)
  inspur.com/gpu       0             0
  inspur.com/rdma-hca  0             0
  nvidia.com/gpu       0             0
Events:
  Type    Reason    Age   From                       Message
  ----    ------    ----  ----                       -------
  Normal  Starting  16s   kube-proxy, hollow-node-0  Starting kube-proxy.

```


然后再运行测试pod，pod定义如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: docker.io/busybox:latest
    imagePullPolicy: IfNotPresent
    command: ['sleep', '3600']
    securityContext:
      privileged: true
    resources:
      limits:
        cpu: "1"
        memory: "2Mi"
        nvidia.com/gpu: "1"
      requests:
        cpu: "1"
        memory: "2Mi"
        nvidia.com/gpu: "1"        
  affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/node
                operator: NotIn
                values:
                - "true"
```

运行测试pod后的集群信息如下：

```shell
NAMESPACE     NAME                              READY   STATUS              RESTARTS   AGE    IP                NODE            NOMINATED NODE   READINESS GATES
default       myapp-pod                         1/1     Running             0          15s    10.142.241.4      hollow-node-3   <none>           <none>
kube-system   calico-node-6n95j                 0/1     Init:0/1            0          14m    10.233.96.49      hollow-node-5   <none>           <none>
kube-system   calico-node-g5qlq                 0/1     Init:0/1            0          14m    10.233.96.51      hollow-node-1   <none>           <none>
kube-system   calico-node-g6rqs                 0/1     Init:0/1            0          14m    10.233.96.46      hollow-node-3   <none>           <none>
kube-system   calico-node-m55zq                 0/1     Init:0/1            0          14m    10.233.96.47      hollow-node-2   <none>           <none>
kube-system   calico-node-ms5t6                 1/1     Running             1          22h    192.168.182.102   node2           <none>           <none>
kube-system   calico-node-n5528                 0/1     Init:0/1            0          14m    10.233.96.50      hollow-node-0   <none>           <none>
kube-system   calico-node-njrr9                 0/1     Init:0/1            0          14m    10.233.96.48      hollow-node-4   <none>           <none>
kube-system   calico-node-vxcb7                 1/1     Running             1          22h    192.168.182.101   node1           <none>           <none>
kube-system   coredns-654c8ccdc8-575qx          1/1     Running             1          22h    10.233.90.76      node1           <none>           <none>
kube-system   coredns-654c8ccdc8-m8ntn          0/1     Pending             0          175d   <none>            <none>          <none>           <none>
kube-system   dns-autoscaler-77445c9c8b-b565h   1/1     Running             1          22h    10.233.90.77      node1           <none>           <none>
kube-system   kube-apiserver-node1              1/1     Running             12         22h    192.168.182.101   node1           <none>           <none>
kube-system   kube-controller-manager-node1     1/1     Running             12         22h    192.168.182.101   node1           <none>           <none>
kube-system   kube-proxy-dsghn                  1/1     Running             1          22h    192.168.182.101   node1           <none>           <none>
kube-system   kube-proxy-dzll2                  0/1     ContainerCreating   0          14m    10.233.96.46      hollow-node-3   <none>           <none>
kube-system   kube-proxy-fqfqx                  0/1     ContainerCreating   0          14m    10.233.96.51      hollow-node-1   <none>           <none>
kube-system   kube-proxy-fxcs4                  0/1     ContainerCreating   0          14m    10.233.96.50      hollow-node-0   <none>           <none>
kube-system   kube-proxy-gjzq8                  1/1     Running             1          22h    192.168.182.102   node2           <none>           <none>
kube-system   kube-proxy-khv5c                  0/1     ContainerCreating   0          14m    10.233.96.48      hollow-node-4   <none>           <none>
kube-system   kube-proxy-l7hzh                  0/1     ContainerCreating   0          14m    10.233.96.49      hollow-node-5   <none>           <none>
kube-system   kube-proxy-r5lfv                  0/1     ContainerCreating   0          14m    10.233.96.47      hollow-node-2   <none>           <none>
kube-system   kube-scheduler-node1              1/1     Running             12         22h    192.168.182.101   node1           <none>           <none>
kube-system   metrics-server-84d6dbd58b-sx7sl   0/2     ContainerCreating   0          14m    <none>            hollow-node-0   <none>           <none>
kubemark      hollow-node-0                     2/2     Running             0          14m    10.233.96.50      node2           <none>           <none>
kubemark      hollow-node-1                     2/2     Running             0          14m    10.233.96.51      node2           <none>           <none>
kubemark      hollow-node-2                     2/2     Running             0          14m    10.233.96.47      node2           <none>           <none>
kubemark      hollow-node-3                     2/2     Running             0          14m    10.233.96.46      node2           <none>           <none>
kubemark      hollow-node-4                     2/2     Running             0          14m    10.233.96.48      node2           <none>           <none>
kubemark      hollow-node-5                     2/2     Running             0          14m    10.233.96.49      node2           <none>           <none>

```
*说明： 网络插件和proxy组件的非running态，并不影响集群测试使用，比如proxy使用了--use-real-proxier=false，并未使用真正的iptables*

测试pod调度到了hollow-node-3上，hollow-node-3节点信息如下
```shell
Name:               hollow-node-3
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=hollow-node-3
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
CreationTimestamp:  Tue, 08 Dec 2020 13:34:35 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 08 Dec 2020 13:49:06 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 08 Dec 2020 13:49:06 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 08 Dec 2020 13:49:06 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Tue, 08 Dec 2020 13:49:06 +0800   Tue, 08 Dec 2020 13:34:35 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.233.96.46
  Hostname:    hollow-node-3
Capacity:
 cpu:                  72
 ephemeral-storage:    15726578Mi
 inspur.com/gpu:       99999
 inspur.com/rdma-hca:  99999
 memory:               256Gi
 nvidia.com/gpu:       8
 pods:                 110
Allocatable:
 cpu:                  72
 ephemeral-storage:    15726578Mi
 inspur.com/gpu:       99999
 inspur.com/rdma-hca:  99999
 memory:               256Gi
 nvidia.com/gpu:       8
 pods:                 110
System Info:
 Machine ID:
 System UUID:
 Boot ID:
 Kernel Version:             3.16.0-0.bpo.4-amd64
 OS Image:                   Debian GNU/Linux 7 (wheezy)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://1.13.1
 Kubelet Version:            v0.0.0-master+4d3c9e0c
 Kube-Proxy Version:         v0.0.0-master+4d3c9e0c
PodCIDR:                     10.233.69.0/24
Non-terminated Pods:         (3 in total)
  Namespace                  Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                 ------------  ----------  ---------------  -------------  ---
  default                    myapp-pod            1 (1%)        1 (1%)      2Mi (0%)         2Mi (0%)       33s
  kube-system                calico-node-g6rqs    150m (0%)     300m (0%)   64M (0%)         500M (0%)      14m
  kube-system                kube-proxy-dzll2     0 (0%)        0 (0%)      0 (0%)           0 (0%)         14m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource             Requests      Limits
  --------             --------      ------
  cpu                  1150m (1%)    1300m (1%)
  memory               64548Ki (0%)  502097152 (0%)
  ephemeral-storage    0 (0%)        0 (0%)
  inspur.com/gpu       0             0
  inspur.com/rdma-hca  0             0
  nvidia.com/gpu       1             1
Events:
  Type    Reason    Age   From                       Message
  ----    ------    ----  ----                       -------
  Normal  Starting  14m   kube-proxy, hollow-node-3  Starting kube-proxy.
```

由上面测试集群信息，可以看到模拟了4个桩节点可用于测试pod调度。

### kubemark源码

#### 程序入口

kubemark根据参数Morph，可执行kubelet和proxy流程，从而实现节点组件功能。

cmd/kubemark/hollow-node.go
```golang
func run(config *hollowNodeConfig) {
  if !knownMorphs.Has(config.Morph) {
    klog.Fatalf("Unknown morph: %v. Allowed values: %v", config.Morph, knownMorphs.List())
  }

  // create a client to communicate with API server.
  clientConfig, err := config.createClientConfigFromFile()
  if err != nil {
    klog.Fatalf("Failed to create a ClientConfig: %v. Exiting.", err)
  }

  client, err := clientset.NewForConfig(clientConfig)
  if err != nil {
    klog.Fatalf("Failed to create a ClientSet: %v. Exiting.", err)
  }

  if config.Morph == "kubelet" {
    cadvisorInterface := &cadvisortest.Fake{
      NodeName: config.NodeName,
    }
    containerManager := cm.NewStubContainerManager()

    fakeDockerClientConfig := &dockershim.ClientConfig{
      DockerEndpoint:    libdocker.FakeDockerEndpoint,
      EnableSleep:       true,
      WithTraceDisabled: true,
    }

    hollowKubelet := kubemark.NewHollowKubelet(
      config.NodeName,
      client,
      cadvisorInterface,
      fakeDockerClientConfig,
      config.KubeletPort,
      config.KubeletReadOnlyPort,
      containerManager,
      maxPods,
      podsPerCore,
    )
    hollowKubelet.Run()
  }

  if config.Morph == "proxy" {
    client, err := clientset.NewForConfig(clientConfig)
    if err != nil {
      klog.Fatalf("Failed to create API Server client: %v", err)
    }
    iptInterface := fakeiptables.NewFake()
    sysctl := fakesysctl.NewFake()
    execer := &fakeexec.FakeExec{}
    eventBroadcaster := record.NewBroadcaster()
    recorder := eventBroadcaster.NewRecorder(legacyscheme.Scheme, v1.EventSource{Component: "kube-proxy", Host: config.NodeName})

    hollowProxy, err := kubemark.NewHollowProxyOrDie(
      config.NodeName,
      client,
      client.CoreV1(),
      iptInterface,
      sysctl,
      execer,
      eventBroadcaster,
      recorder,
      config.UseRealProxier,
      config.ProxierSyncPeriod,
      config.ProxierMinSyncPeriod,
    )
    if err != nil {
      klog.Fatalf("Failed to create hollowProxy instance: %v", err)
    }
    hollowProxy.Run()
  }
}
```

#### hollow_kubelet

pkg/kubemark/hollow_kubelet

```golang
type HollowKubelet struct {
  KubeletFlags         *options.KubeletFlags
  KubeletConfiguration *kubeletconfig.KubeletConfiguration
  KubeletDeps          *kubelet.Dependencies
}

func NewHollowKubelet(
  nodeName string,
  client *clientset.Clientset,
  cadvisorInterface cadvisor.Interface,
  dockerClientConfig *dockershim.ClientConfig,
  kubeletPort, kubeletReadOnlyPort int,
  containerManager cm.ContainerManager,
  maxPods int, podsPerCore int,
) *HollowKubelet {
  // -----------------
  // Static config
  // -----------------
  f, c := GetHollowKubeletConfig(nodeName, kubeletPort, kubeletReadOnlyPort, maxPods, podsPerCore)

  // -----------------
  // Injected objects
  // -----------------
  volumePlugins := emptydir.ProbeVolumePlugins()
  volumePlugins = append(volumePlugins, secret.ProbeVolumePlugins()...)
  volumePlugins = append(volumePlugins, projected.ProbeVolumePlugins()...)
  d := &kubelet.Dependencies{
    KubeClient:         client,
    HeartbeatClient:    client,
    DockerClientConfig: dockerClientConfig,
    CAdvisorInterface:  cadvisorInterface,
    Cloud:              nil,
    OSInterface:        &containertest.FakeOS{},
    ContainerManager:   containerManager,
    VolumePlugins:      volumePlugins,
    TLSOptions:         nil,
    OOMAdjuster:        oom.NewFakeOOMAdjuster(),
    Mounter:            mount.New("" /* default mount path */),
    Subpather:          &subpath.FakeSubpath{},
  }

  return &HollowKubelet{
    KubeletFlags:         f,
    KubeletConfiguration: c,
    KubeletDeps:          d,
  }
}

// Starts this HollowKubelet and blocks.
func (hk *HollowKubelet) Run() {
  if err := kubeletapp.RunKubelet(&options.KubeletServer{
    KubeletFlags:         *hk.KubeletFlags,
    KubeletConfiguration: *hk.KubeletConfiguration,
  }, hk.KubeletDeps, false); err != nil {
    klog.Fatalf("Failed to run HollowKubelet: %v. Exiting.", err)
  }
  select {}
}
```


#### hollow_proxy
pkg/kubemark/hollow_proxy
```golang
type HollowProxy struct {
  ProxyServer *proxyapp.ProxyServer
}

type FakeProxier struct{}

func (*FakeProxier) Sync() {}
func (*FakeProxier) SyncLoop() {
  select {}
}
func (*FakeProxier) OnServiceAdd(service *v1.Service)                        {}
func (*FakeProxier) OnServiceUpdate(oldService, service *v1.Service)         {}
func (*FakeProxier) OnServiceDelete(service *v1.Service)                     {}
func (*FakeProxier) OnServiceSynced()                                        {}
func (*FakeProxier) OnEndpointsAdd(endpoints *v1.Endpoints)                  {}
func (*FakeProxier) OnEndpointsUpdate(oldEndpoints, endpoints *v1.Endpoints) {}
func (*FakeProxier) OnEndpointsDelete(endpoints *v1.Endpoints)               {}
func (*FakeProxier) OnEndpointsSynced()                                      {}

func NewHollowProxyOrDie(
  nodeName string,
  client clientset.Interface,
  eventClient v1core.EventsGetter,
  iptInterface utiliptables.Interface,
  sysctl utilsysctl.Interface,
  execer utilexec.Interface,
  broadcaster record.EventBroadcaster,
  recorder record.EventRecorder,
  useRealProxier bool,
  proxierSyncPeriod time.Duration,
  proxierMinSyncPeriod time.Duration,
) (*HollowProxy, error) {
  // Create proxier and service/endpoint handlers.
  var proxier proxy.ProxyProvider
  var serviceHandler proxyconfig.ServiceHandler
  var endpointsHandler proxyconfig.EndpointsHandler

  if useRealProxier {
    // Real proxier with fake iptables, sysctl, etc underneath it.
    //var err error
    proxierIPTables, err := iptables.NewProxier(
      iptInterface,
      sysctl,
      execer,
      proxierSyncPeriod,
      proxierMinSyncPeriod,
      false,
      0,
      "10.0.0.0/8",
      nodeName,
      utilnode.GetNodeIP(client, nodeName),
      recorder,
      nil,
      []string{},
    )
    if err != nil {
      return nil, fmt.Errorf("unable to create proxier: %v", err)
    }
    proxier = proxierIPTables
    serviceHandler = proxierIPTables
    endpointsHandler = proxierIPTables
  } else {
    proxier = &FakeProxier{}
    serviceHandler = &FakeProxier{}
    endpointsHandler = &FakeProxier{}
  }

  // Create a Hollow Proxy instance.
  nodeRef := &v1.ObjectReference{
    Kind:      "Node",
    Name:      nodeName,
    UID:       types.UID(nodeName),
    Namespace: "",
  }
  return &HollowProxy{
    ProxyServer: &proxyapp.ProxyServer{
      Client:                client,
      EventClient:           eventClient,
      IptInterface:          iptInterface,
      Proxier:               proxier,
      Broadcaster:           broadcaster,
      Recorder:              recorder,
      ProxyMode:             "fake",
      NodeRef:               nodeRef,
      OOMScoreAdj:           utilpointer.Int32Ptr(0),
      ResourceContainer:     "",
      ConfigSyncPeriod:      30 * time.Second,
      ServiceEventHandler:   serviceHandler,
      EndpointsEventHandler: endpointsHandler,
    },
  }, nil
}

func (hp *HollowProxy) Run() {
  if err := hp.ProxyServer.Run(); err != nil {
    klog.Fatalf("Error while running proxy: %v\n", err)
  }
}
```

### kubemark总结
1. kubemark实际上是个K8S组件，包含了kubelet和一个controller，模拟桩节点主要使用了kubelet功能。
2. kubemark通过在真实节点上构造批量的hollow-node的pod方式，模拟运行了大量的桩节点。这些桩节点可以定时跟master同步状态和信息。
3. kubemark一般用于测试master节点上的组件的性能测试，比如测试调度器和控制器组件性能。
4. kubemark由于其构造方式，决定其不能测试node节点组件，比如kubelet性能和网络等。


## 3 测试框架

可参考k8s的perf-test

* [Kubernetes测试系列 - 性能测试](https://blog.csdn.net/qingdao666666/article/details/104625457)

* [kubernetes性能指标体系：clusterloader2](https://juejin.cn/post/6844904142389919758)

* [clusterloader2的漫漫踩坑路：最详细解析与使用指南](https://www.colabug.com/2020/0428/7329883/)

* [clusterloader2设计说明：Cluster loader vision](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/design.md)



## 4 附录

参考操作命令。。。

```shell
[root@test-master ~]# kubectl get secret -A |grep kubeconfig
kubemark          kubeconfig                                       Opaque                                2      94s

[root@test-master ~]# kubectl describe secret -nkubemark          kubeconfig
Name:         kubeconfig
Namespace:    kubemark
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
kubelet.kubeconfig:    5463 bytes
kubeproxy.kubeconfig:  5463 bytes

#### 如果要删除刚刚创建的secret和 configmap

kubectl delete secret -nkubemark          kubeconfig
kubectl delete configmap -n kubemark node-configmap 
kubectl delete ns kubemark


--force --grace-period=0

#### 强制删除资源
kubectl delete po --force --grace-period=0 -nkube-system   kube-proxy-p6k42 

### 删除kubemark命名空间下所有node资源
kubectl delete no --all -n kubemark


#### 设置master节点不可调度
# kubectl cordon nodename

kubectl cordon node1

# kubectl uncordon nodename       #取消


#### 节点打标签

kubectl label node  node1 accessswitch=switch1
kubectl label node  node1 groupId=defaultGroup
kubectl label node  node1 node-role.kubernetes.io/master=true
kubectl label node  node1 node-role.kubernetes.io/node=true
kubectl label node  node1 switchtype=ether

kubectl label node  node2 accessswitch=switch1
kubectl label node  node2 groupId=defaultGroup
kubectl label node  node2 node-role.kubernetes.io/node=true
kubectl label node  node2 switchtype=ether


```

修改hollow-node信息，不是node的全部信息都可以修改更新，如capacity等字段无法更新
```shell
kubectl patch node hollow-node-0 -p '{"spec":{"unschedulable":true}}'
```

e2e测试
编译e2e.test
make WHAT="test/e2e/e2e.test"

```shell
# 进入k8s项目，进行测试工具编译
make WHAT="test/e2e/e2e.test"

# 在目录下能够看到输出文件如下：
[root@node1 k8s1.14.8modify-wangb]# ll _output/bin/ -h
total 241M
-rwxr-xr-x. 1 root root 5.9M Dec  7 10:04 conversion-gen
-rwxr-xr-x. 1 root root 5.9M Dec  7 10:04 deepcopy-gen
-rwxr-xr-x. 1 root root 5.9M Dec  7 10:04 defaulter-gen
-rwxr-xr-x. 1 root root 110M Dec  7 17:01 e2e.test
-rwxr-xr-x. 1 root root 3.5M Dec  7 10:04 go2make
-rwxr-xr-x. 1 root root 2.0M Dec  7 10:04 go-bindata
-rwxr-xr-x. 1 root root  99M Dec  7 10:05 kubemark
-rwxr-xr-x. 1 root root  10M Dec  7 10:04 openapi-gen


# 把 e2e.test 文件拷贝到测试集群的master节点上


```
需要注意：网上的搜到的文章大多数都是编译e2e的二进制文件直接运行

``` shell
#./e2e.test --kube-master=192.168.182.101 --host=https://192.168.182.101:6443 --ginkgo.focus="\[Performance\]" --provider=local --kubeconfig=kubemark.kubeconfig --num-nodes=10 --v=3 --ginkgo.failFast --e2e-output-dir=. --report-dir=.


./e2e.test --kube-master=192.168.182.101 --host=https://192.168.182.101:6443 --ginkgo.focus="\[Performance\]" --provider=local --kubeconfig=/root/.kube/config --num-nodes=4 --v=3 --ginkgo.failFast --e2e-output-dir=. --report-dir=.
```
但其实e2e的性能用例已经被移出主库了 https://github.com/kubernetes/kubernetes/pull/83322，所以在2019.10.1之后出的版本用上面的命令是无法运行性能测试的


<br>


{{< admonition note "Deployment中pod创建的流程" >}}

1. apiserver收到创建deployment的请求，存储至etcd，告知controller-manager
2. controller-manager创建pod的壳子，打上creationTimeStamp，发送请求到apiserver
3. apiserver收到创建pod的请求，发送至etcd，推送到scheduler。
4. schduler选择node，填充nodeName，向apiserver更新pod信息。此时pod处于pending状态，pod也没有真正创建。
5. apiserver向etcd更新pod信息，同时推送到相应节点的kubelet
6. kubelet创建pod，填充HostIP与resourceVersion，向apiserver发送更新请求，pod处于pending状态
7. apiserver更新pod信息至etcd，同时kubelet继续创建pod。等到容器都处于running状态，kubelet再次发送pod的更新请求给apiserver，此时pod running
8. apiserver收到请求，更新到etcd中，并推送到informer中，informer记录下watchPhase


{{< /admonition >}}
