# K8S基于NUMA亲和性的资源分配特性测试



1.20版本已经有了kubelet的numa亲和性资源（CPU和GPU）分配功能（与1.18版本的beta接口相同），本文记录操作要点

<!--more-->

## 配置kubelet

1. 添加kubelet中numa相关的运行命令参数
```shell
--cpu-manager-policy=static --topology-manager-policy=best-effort
```
kubelet的cpu-manager策略默认是none，会分配系统全部cpuset。这里需要显示指定策略

topology-manager-policy这里根据项目场景需要，配置best-effort：优选分配numa拓扑亲和性的资源，如果numa亲和性不满足，则分配系统可用资源。


2. cpu-manager策略默认配置
```markdown
[root@gpu53 ~]# cat  /var/lib/kubelet/cpu_manager_state
{"policyName":"none","defaultCpuSet":"","checksum":1353318690}
```

3. cpu-manager策略static配置
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-27","entries":{"39b37746-7f5e-4064-b8e1-eebd2bfaa003":{"app":"1-3"}},"checksum":3300516549}
```


## topology-manager-policy
{{< admonition note >}} 
:(far fa-bookmark fa-fw): 说明

- none: this policy will not attempt to do any alignment of resources. It will act the same as if the TopologyManager were not present at all. This is the default policy.
- best-effort: with this policy, the TopologyManager will attempt to align allocations on NUMA nodes as best it can, but will always allow the pod to start even if some of the allocated resources are not aligned on the same NUMA node.
- restricted: this policy is the same as the best-effort policy, except it will fail pod admission if allocated resources cannot be aligned properly. Unlike with the single-numa-node policy, some allocations may come from multiple NUMA nodes if it is impossible to ever satisfy the allocation request on a single NUMA node (e.g. 2 devices are requested and the only 2 devices on the system are on different NUMA nodes).
- single-numa-node: this policy is the most restrictive and will only allow a pod to be admitted if all requested CPUs and devices can be allocated from exactly one NUMA node.

{{< /admonition >}}


## kubelet.env配置示例
/etc/kubernetes/kubelet.env

即在原有配置上增加  --cpu-manager-policy=static --topology-manager-policy=best-effort 

```markdown

[root@node2 kubelet]# cat /etc/kubernetes/kubelet.env
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=2"
KUBELET_ADDRESS="--node-ip=10.151.11.61"
KUBELET_HOSTNAME="--hostname-override=node2"



KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
--config=/etc/kubernetes/kubelet-config.yaml \
--kubeconfig=/etc/kubernetes/kubelet.conf \
--pod-infra-container-image=k8s.gcr.io/pause:3.2 \
--authentication-token-webhook \
--enforce-node-allocatable="" \
--client-ca-file=/etc/kubernetes/ssl/ca.crt \
--rotate-certificates \
--node-status-update-frequency=10s \
--cgroup-driver=systemd \
--cgroups-per-qos=False \
--max-pods=110 \
--anonymous-auth=false \
--read-only-port=0 \
--fail-swap-on=True \
--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice \
 --cluster-dns=10.233.0.3 --cluster-domain=cluster.local --resolv-conf=/etc/resolv.conf --node-labels=   --eviction-hard=""  --image-gc-high-threshold=100 --image-gc-low-threshold=99 --kube-reserved cpu=100m --system-reserved cpu=100m \
--cpu-manager-policy=static --topology-manager-policy=best-effort  \
   "
KUBELET_NETWORK_PLUGIN="--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
KUBELET_CLOUDPROVIDER=""

PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin


```

## kubelet重启

**注意：kubelet修改cpu_manager策略配置，一定要停掉kubelet服务，并删除/var/lib/kubelet/cpu_manager_state文件，再重启kubelet，否则会导致kubelet服务重启失败。**


## 启动GPU k8s插件

需要支持CPUManager static policy
这里采用镜像方式启动，详细操作参考[**K8S GPU DEVICEPLUGIN**](/posts/2020/12/%E5%AE%89%E8%A3%85nvidia-docker2nvidia-container-v2%E5%92%8Cnvidia-k8s-gpu%E6%8F%92%E4%BB%B6/)

```shell
docker run \
    -it \
    --privileged \
    --network=none \
    -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins \
    nvidia/k8s-device-plugin:devel --pass-device-specs

```



## kubelet的快照文件

- cpu_manager_state：CPU管理器快照文件，包含cpu分配策略和已分配pod的cpuset信息
- device-plugins/kubelet_internal_checkpoint：deviceplugin的快照信息，这里关注测试numa亲和性分配相关的TOPO分配信息



## GPU命令

### GPU uuid
```shell
nvidia-smi -L
```
显示如下，查询到INDEX -> UUID：
```markdown
[root@node2 ~]# nvidia-smi -L
GPU 0: Tesla P100-PCIE-16GB (UUID: GPU-77a702db-e37f-3a74-d46d-c5713f66058c)
GPU 1: Tesla P100-PCIE-16GB (UUID: GPU-9b341c59-f96b-ba85-c137-78c3652fea65)
GPU 2: Tesla P100-PCIE-16GB (UUID: GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841)
```

### GPU 详细信息
```shell
lspci | grep -i nvidia
```

```markdown
[root@node2 ~]# lspci | grep -i nvidia
3b:00.0 3D controller: NVIDIA Corporation GP100GL [Tesla P100 PCIe 16GB] (rev a1)
86:00.0 3D controller: NVIDIA Corporation GP100GL [Tesla P100 PCIe 16GB] (rev a1)
d8:00.0 3D controller: NVIDIA Corporation GP100GL [Tesla P100 PCIe 16GB] (rev a1)

```

前边的序号 "3b:00.0"是显卡的代号;

查看指定显卡的详细信息用以下指令：
```shell
lspci -v -s 3b:00.0
```

这里能看到NUMA node 1
```markdown
[root@node2 ~]# lspci -v -s d8:00.0
d8:00.0 3D controller: NVIDIA Corporation GP100GL [Tesla P100 PCIe 16GB] (rev a1)
        Subsystem: NVIDIA Corporation Device 118f
        Flags: bus master, fast devsel, latency 0, IRQ 441, NUMA node 1
        Memory at fa000000 (32-bit, non-prefetchable) [size=16M]
        Memory at 39f800000000 (64-bit, prefetchable) [size=16G]
        Memory at 39fc00000000 (64-bit, prefetchable) [size=32M]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable+ Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [258] L1 PM Substates
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [420] Advanced Error Reporting
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Capabilities: [900] #19
        Kernel driver in use: nvidia
        Kernel modules: nouveau, nvidia_drm, nvidia

```



### GPU拓扑

```shell
nvidia-smi topo -mp
```

GPU0属于NUMA组0，GPU1和GPU2属于NUMA组1

```markdown
[root@node2 numa_test]# nvidia-smi topo -mp
        GPU0    GPU1    GPU2    CPU Affinity    NUMA Affinity
GPU0     X      SYS     SYS     0-13    0
GPU1    SYS      X      NODE    14-27   1
GPU2    SYS     NODE     X      14-27   1

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge

```



## 测试


### CPU numa亲和性

1. 资源占用和释放：启动pod[3c]，并删除该pod

占用3个cpu后，再释放：
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-27","entries":{"39b37746-7f5e-4064-b8e1-eebd2bfaa003":{"app":"1-3"}},"checksum":3300516549}
[root@node2 kubelet]# kubectl delete po cpu-numa-batch-pod
pod "cpu-numa-batch-pod" deleted
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-27","checksum":273146150}
```

2. 环境资源未占用
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,14-27","entries":{"c0c5c4b3-3f63-4677-ba68-52da74012371":{"app":"1-13"}},"checksum":1954249489}
```

3. 占用一个numa组的cpu资源，14个cpu
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13","entries":{"6c5f3038-adfc-485d-9943-3fd5e825300d":{"app":"14-27"}},"checksum":3451722052}
```
4. 启动2个pod，pod1 占用14c，pod2占用12c
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,13","entries":{"55784671-0e4e-49e2-b4d6-c0377ca14c81":{"app":"1-12"},"6c5f3038-adfc-485d-9943-3fd5e825300d":{"app":"14-27"}},"checksum":3558029577}
```



### GPU+CPU numa亲和性

1. pod请求2个GPU，0个cpu

```
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-27","checksum":273146150}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"9a15d2b5-c152-46b9-96e0-d57032629e1f","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CmsKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSUUdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEsR1BVLTliMzQxYzU5LWY5NmItYmE4NS1jMTM3LTc4YzM2NTJmZWE2NRokCg4vZGV2L252aWRpYWN0bBIOL2Rldi9udmlkaWFjdGwaAnJ3GiYKDy9kZXYvbnZpZGlhLXV2bRIPL2Rldi9udmlkaWEtdXZtGgJydxoyChUvZGV2L252aWRpYS11dm0tdG9vbHMSFS9kZXYvbnZpZGlhLXV2bS10b29scxoCcncaLgoTL2Rldi9udmlkaWEtbW9kZXNldBITL2Rldi9udmlkaWEtbW9kZXNldBoCcncaIAoML2Rldi9udmlkaWExEgwvZGV2L252aWRpYTEaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":2530956716}[root@node2 kubelet]#
[root@node2 kubelet]#

```


查看容器信息 docker inspect，已分配GPU资源

```json
 "Devices": [
                {
                    "PathOnHost": "/dev/nvidia1",
                    "PathInContainer": "/dev/nvidia1",
                    "CgroupPermissions": "rw"
                },
                {
                    "PathOnHost": "/dev/nvidia2",
                    "PathInContainer": "/dev/nvidia2",
                    "CgroupPermissions": "rw"
                }
            ]

```


结果：2个GPU都分配到了同1个numa组，cpu资源无指定则使用全部cpuset

2. pod请求1个GPU，3个cpu


```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-27","entries":{"513cb897-0262-4868-826f-aa943ee45a38":{"app":"1-3"}},"checksum":1982473279}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"513cb897-0262-4868-826f-aa943ee45a38","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMBIML2Rldi9udmlkaWEwGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":133412836}[root@node2 kubelet]#

```


查看容器信息 docker inspect，分配了GPU0

结果：资源充足时，1个GPU，3个cpu都分配到了numa组0，同时满足numa亲和性

3. pod请求2个GPU，3个cpu 

```markdown

[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13,17-27","entries":{"de6df8b8-a6b7-41cc-97a6-19d0fbd44714":{"app":"14-16"}},"checksum":3366848516}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"de6df8b8-a6b7-41cc-97a6-19d0fbd44714","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CmsKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSUUdQVS05YjM0MWM1OS1mOTZiLWJhODUtYzEzNy03OGMzNjUyZmVhNjUsR1BVLWMxZTlmMjQ5LWIzN2ItODFjMi1hOGQ5LWJhNWNhMDI5NDg0MRokCg4vZGV2L252aWRpYWN0bBIOL2Rldi9udmlkaWFjdGwaAnJ3GiYKDy9kZXYvbnZpZGlhLXV2bRIPL2Rldi9udmlkaWEtdXZtGgJydxoyChUvZGV2L252aWRpYS11dm0tdG9vbHMSFS9kZXYvbnZpZGlhLXV2bS10b29scxoCcncaLgoTL2Rldi9udmlkaWEtbW9kZXNldBITL2Rldi9udmlkaWEtbW9kZXNldBoCcncaIAoML2Rldi9udmlkaWExEgwvZGV2L252aWRpYTEaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":4219022648}[root@node2 kubelet]#
[root@node2 kubelet]# ll device-plugins/kubelet_internal_checkpoint


```

查看容器信息 docker inspect，分配了GPU1和2

结果：资源充足时，2个GPU，3个cpu都分配到了numa组1，同时满足numa亲和性


4. 启动2个pod

    - pod1：请求1个GPU，3个cpu [场景2]
    - pod2：请求2个GPU，3个cpu [场景3]

```markdown

[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-13,17-27","entries":{"513cb897-0262-4868-826f-aa943ee45a38":{"app":"1-3"},"94283d1b-ce5a-4797-bff8-0cf0c7143b2b":{"app":"14-16"}},"checksum":1623972425}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"513cb897-0262-4868-826f-aa943ee45a38","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMBIML2Rldi9udmlkaWEwGgJydw=="},{"PodUID":"94283d1b-ce5a-4797-bff8-0cf0c7143b2b","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841","GPU-9b341c59-f96b-ba85-c137-78c3652fea65"]},"AllocResp":"CmsKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSUUdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEsR1BVLTliMzQxYzU5LWY5NmItYmE4NS1jMTM3LTc4YzM2NTJmZWE2NRokCg4vZGV2L252aWRpYWN0bBIOL2Rldi9udmlkaWFjdGwaAnJ3GiYKDy9kZXYvbnZpZGlhLXV2bRIPL2Rldi9udmlkaWEtdXZtGgJydxoyChUvZGV2L252aWRpYS11dm0tdG9vbHMSFS9kZXYvbnZpZGlhLXV2bS10b29scxoCcncaLgoTL2Rldi9udmlkaWEtbW9kZXNldBITL2Rldi9udmlkaWEtbW9kZXNldBoCcncaIAoML2Rldi9udmlkaWExEgwvZGV2L252aWRpYTEaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":2442330366}[root@node2 kubelet]#


```


查看容器信息 docker inspect，pod1的计算资源分配到了numa组0；pod2的计算资源分配到了numa组1

结果：资源充足时，2个pod的计算资源分配满足numa亲和性


5. 启动2个pod 2，结果同上
    - pod1：请求1个GPU，3个cpu 
    - pod2：请求1个GPU，3个cpu

```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-13,17-27","entries":{"513cb897-0262-4868-826f-aa943ee45a38":{"app":"1-3"},"f22736b4-45a9-4852-8fdd-feb948918597":{"app":"14-16"}},"checksum":2054609245}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"513cb897-0262-4868-826f-aa943ee45a38","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMBIML2Rldi9udmlkaWEwGgJydw=="},{"PodUID":"f22736b4-45a9-4852-8fdd-feb948918597","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-9b341c59-f96b-ba85-c137-78c3652fea65"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS05YjM0MWM1OS1mOTZiLWJhODUtYzEzNy03OGMzNjUyZmVhNjUaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMRIML2Rldi9udmlkaWExGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841","GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65"]}},"Checksum":3082069630}[root@node2 kubelet]#

```

查看容器信息 docker inspect，pod1的计算资源分配到了numa组0；pod2的计算资源分配到了numa组1

结果：资源充足时，2个pod的计算资源分配满足numa亲和性


6. 启动2个pod 3
    - pod1：请求0个GPU，3个cpu 已占用了numa组1
    - pod2：请求1个GPU，3个cpu 测试pod2被分配到哪个numa组


pod2资源都分配到了numa组0，满足numa亲和性

```markdown

[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13,17-27","entries":{"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":2485662466}




[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,4-13,17-27","entries":{"78a1a5c8-39ee-47c3-8e95-9328f1398693":{"app":"1-3"},"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":3632910195}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"f21fe02b-e6e2-4d04-9a4a-9e57367fa324","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="},{"PodUID":"78a1a5c8-39ee-47c3-8e95-9328f1398693","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMBIML2Rldi9udmlkaWEwGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":3193682924}[root@node2 kubelet]#
```


### numa资源不足场景测试

#### cpu某num组资源不足

1. 启动2个pod
启动2个pod
    - pod1：请求0个GPU，12个cpu 
    - pod2：请求1个GPU，3个cpu

pod1分配到了numa组0，且基本上占满numa组0的cpu资源；
这时pod2再分配资源（cpu和GPU）时，根据numa亲和性策略，要分配到numa组1的cpu和GPU资源

```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,13,17-27","entries":{"77025d90-6e46-4a87-ad3a-bf0c02c6713c":{"app":"1-12"},"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":874856219}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"f21fe02b-e6e2-4d04-9a4a-9e57367fa324","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":2941906560}[root@node2 kubelet]#
[root@node2 kubelet]#

```



2. 启动2个pod 2

启动2个pod

- pod1：请求1个GPU，3个cpu, 已占numa组1
- pod2：请求1个GPU，12个cpu 


第2个pod 9388acc6-a396-4f03-a353-ce153da46aaf 的cpu资源 占用了numa组0和1，gpu资源占用了numa组0，如下

```
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13,17-27","entries":{"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":2485662466}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,20-27","entries":{"9388acc6-a396-4f03-a353-ce153da46aaf":{"app":"1-13,17-19"},"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":4055801500}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"f21fe02b-e6e2-4d04-9a4a-9e57367fa324","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="},{"PodUID":"9388acc6-a396-4f03-a353-ce153da46aaf","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMBIML2Rldi9udmlkaWEwGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":4148283274}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#


```


此时的拓扑管理器的策略结果输出如下，虽然有部分cpu和gpu不在同一个numa组，认为cpu和gpu的合并分配结果仍满足numa亲和性

```markdown

Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.740680  117175 topology_manager.go:187] [topologymanager] Topology Admit Handler
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.740755  117175 scope_container.go:80] [topologymanager] TopologyHints for pod '16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf)', container 'app': map[nvidia.com/gpu:[{01 true} {10 true} {11 false}]]
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.740986  117175 policy_static.go:379] [cpumanager] TopologyHints generated for pod '16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf)', container 'app': [{11 true}]
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.741009  117175 scope_container.go:80] [topologymanager] TopologyHints for pod '16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf)', container 'app': map[cpu:[{11 true}]]
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.741037  117175 scope_container.go:88] [topologymanager] ContainerTopologyHint: {01 true}
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.741054  117175 scope_container.go:53] [topologymanager] Best TopologyHint for (pod: 16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf) container: app): {01 true}
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.741072  117175 scope_container.go:63] [topologymanager] Topology Affinity for (pod: 16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf) container: app): {01 true}
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.743378  117175 policy_static.go:221] [cpumanager] static policy: Allocate (pod: 16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf), container: app)
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.743418  117175 policy_static.go:232] [cpumanager] Pod 16cpu-numa-batch-pod_default(9388acc6-a396-4f03-a353-ce153da46aaf), Container app Topology Affinity is: {01 true}
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.743448  117175 policy_static.go:259] [cpumanager] allocateCpus: (numCPUs: 16, socket: 01)
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.743841  117175 state_mem.go:88] [cpumanager] updated default cpuset: "0,20-27"
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.744701  117175 policy_static.go:294] [cpumanager] allocateCPUs: returning "1-13,17-19"
Dec 29 15:13:13 node2 kubelet[117175]: I1229 15:13:13.744761  117175 state_mem.go:80] [cpumanager] updated desired cpuset (pod: 9388acc6-a396-4f03-a353-ce153da46aaf, container: app, cpuset: "1-13,17-19")


```



#### GPU某numa组资源不足


启动2个pod

- pod1：请求1个GPU，3个cpu 占有numa组1
- pod2：请求2个GPU，0个cpu 




pod2 [08b4a90a-534a-4fc6-90c3-b57ee777071d]的2个GPU分配到了numa组0和1，在best-effort策略下，虽不满足numa亲和性，但仍按系统可用资源进行分配
```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13,17-27","entries":{"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":2485662466}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"08b4a90a-534a-4fc6-90c3-b57ee777071d","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"],"1":["GPU-9b341c59-f96b-ba85-c137-78c3652fea65"]},"AllocResp":"CmsKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSUUdQVS05YjM0MWM1OS1mOTZiLWJhODUtYzEzNy03OGMzNjUyZmVhNjUsR1BVLTc3YTcwMmRiLWUzN2YtM2E3NC1kNDZkLWM1NzEzZjY2MDU4YxokCg4vZGV2L252aWRpYWN0bBIOL2Rldi9udmlkaWFjdGwaAnJ3GiYKDy9kZXYvbnZpZGlhLXV2bRIPL2Rldi9udmlkaWEtdXZtGgJydxoyChUvZGV2L252aWRpYS11dm0tdG9vbHMSFS9kZXYvbnZpZGlhLXV2bS10b29scxoCcncaLgoTL2Rldi9udmlkaWEtbW9kZXNldBITL2Rldi9udmlkaWEtbW9kZXNldBoCcncaIAoML2Rldi9udmlkaWEwEgwvZGV2L252aWRpYTAaAnJ3GiAKDC9kZXYvbnZpZGlhMRIML2Rldi9udmlkaWExGgJydw=="},{"PodUID":"f21fe02b-e6e2-4d04-9a4a-9e57367fa324","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":3342579237}[root@node2 kubelet]#
```

拓扑管理器的策略分配结果，如下，不满足numa亲和性。

```markdown
Dec 29 14:50:05 node2 kubelet[117175]: I1229 14:50:05.825370  117175 scope_container.go:80] [topologymanager] TopologyHints for pod '2gpu-numa-batch-pod_default(08b4a90a-534a-4fc6-90c3-b57ee777071d)', container 'app': map[]
Dec 29 14:50:05 node2 kubelet[117175]: I1229 14:50:05.825386  117175 policy.go:70] [topologymanager] Hint Provider has no preference for NUMA affinity with any resource
Dec 29 14:50:05 node2 kubelet[117175]: I1229 14:50:05.825403  117175 scope_container.go:88] [topologymanager] ContainerTopologyHint: {11 false}
```


#### GPU和cpu numa组资源都不满足


启动2个pod
- pod1：请求1个GPU，3个cpu 占有numa组1
- pod2：请求2个GPU，16个cpu 

pod2 [27a7e589-4c5e-4c47-813e-be1a118d3d80] 的cpu分配到了2个numa组，GPU也同样分配到了2个numa组，不满足numa亲和性了。

```markdown
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0-13,17-27","entries":{"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":2485662466}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat device-plugins/kubelet_internal_checkpoint
{"Data":{"PodDeviceEntries":[{"PodUID":"f21fe02b-e6e2-4d04-9a4a-9e57367fa324","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"1":["GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]},"AllocResp":"CkIKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSKEdQVS1jMWU5ZjI0OS1iMzdiLTgxYzItYThkOS1iYTVjYTAyOTQ4NDEaJAoOL2Rldi9udmlkaWFjdGwSDi9kZXYvbnZpZGlhY3RsGgJydxomCg8vZGV2L252aWRpYS11dm0SDy9kZXYvbnZpZGlhLXV2bRoCcncaMgoVL2Rldi9udmlkaWEtdXZtLXRvb2xzEhUvZGV2L252aWRpYS11dm0tdG9vbHMaAnJ3Gi4KEy9kZXYvbnZpZGlhLW1vZGVzZXQSEy9kZXYvbnZpZGlhLW1vZGVzZXQaAnJ3GiAKDC9kZXYvbnZpZGlhMhIML2Rldi9udmlkaWEyGgJydw=="},{"PodUID":"27a7e589-4c5e-4c47-813e-be1a118d3d80","ContainerName":"app","ResourceName":"nvidia.com/gpu","DeviceIDs":{"0":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c"],"1":["GPU-9b341c59-f96b-ba85-c137-78c3652fea65"]},"AllocResp":"CmsKFk5WSURJQV9WSVNJQkxFX0RFVklDRVMSUUdQVS03N2E3MDJkYi1lMzdmLTNhNzQtZDQ2ZC1jNTcxM2Y2NjA1OGMsR1BVLTliMzQxYzU5LWY5NmItYmE4NS1jMTM3LTc4YzM2NTJmZWE2NRokCg4vZGV2L252aWRpYWN0bBIOL2Rldi9udmlkaWFjdGwaAnJ3GiYKDy9kZXYvbnZpZGlhLXV2bRIPL2Rldi9udmlkaWEtdXZtGgJydxoyChUvZGV2L252aWRpYS11dm0tdG9vbHMSFS9kZXYvbnZpZGlhLXV2bS10b29scxoCcncaLgoTL2Rldi9udmlkaWEtbW9kZXNldBITL2Rldi9udmlkaWEtbW9kZXNldBoCcncaIAoML2Rldi9udmlkaWEwEgwvZGV2L252aWRpYTAaAnJ3GiAKDC9kZXYvbnZpZGlhMRIML2Rldi9udmlkaWExGgJydw=="}],"RegisteredDevices":{"nvidia.com/gpu":["GPU-77a702db-e37f-3a74-d46d-c5713f66058c","GPU-9b341c59-f96b-ba85-c137-78c3652fea65","GPU-c1e9f249-b37b-81c2-a8d9-ba5ca0294841"]}},"Checksum":4286867880}[root@node2 kubelet]#
[root@node2 kubelet]#
[root@node2 kubelet]# cat cpu_manager_state
{"policyName":"static","defaultCpuSet":"0,6-13","entries":{"27a7e589-4c5e-4c47-813e-be1a118d3d80":{"app":"1-5,17-27"},"f21fe02b-e6e2-4d04-9a4a-9e57367fa324":{"app":"14-16"}},"checksum":3981361486}[root@node2 kubelet]#
[root@node2 kubelet]#

```

拓扑管理器的策略分配结果，如下，不满足numa亲和性。

```markdown
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719228  117175 topology_manager.go:187] [topologymanager] Topology Admit Handler
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719322  117175 scope_container.go:80] [topologymanager] TopologyHints for pod '16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80)', container 'app': map[nvidia.com/gpu:[{11 false}]]
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719555  117175 policy_static.go:379] [cpumanager] TopologyHints generated for pod '16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80)', container 'app': [{11 true}]
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719581  117175 scope_container.go:80] [topologymanager] TopologyHints for pod '16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80)', container 'app': map[cpu:[{11 true}]]
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719605  117175 scope_container.go:88] [topologymanager] ContainerTopologyHint: {11 false}
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719622  117175 scope_container.go:53] [topologymanager] Best TopologyHint for (pod: 16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80) container: app): {11 false}
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.719639  117175 scope_container.go:63] [topologymanager] Topology Affinity for (pod: 16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80) container: app): {11 false}
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.721990  117175 policy_static.go:221] [cpumanager] static policy: Allocate (pod: 16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80), container: app)
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.722024  117175 policy_static.go:232] [cpumanager] Pod 16cpu-2gpu-numa-kubebatch-pod_default(27a7e589-4c5e-4c47-813e-be1a118d3d80), Container app Topology Affinity is: {11 false}
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.722052  117175 policy_static.go:259] [cpumanager] allocateCpus: (numCPUs: 16, socket: 11)
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.722773  117175 state_mem.go:88] [cpumanager] updated default cpuset: "0,6-13"
Dec 29 15:37:43 node2 kubelet[117175]: I1229 15:37:43.723694  117175 policy_static.go:294] [cpumanager] allocateCPUs: returning "1-5,17-27"

```




## 附录

### kubelet numa拓扑亲和性资源分配方案：

[Kubernetes Topology Manager Moves to Beta - Align Up!](https://kubernetes.io/blog/2020/04/01/kubernetes-1-18-feature-topoloy-manager-beta/)


### 测试pod 配置

16cpu-2gpu-numa-kubebatch-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: 16cpu-2gpu-numa-kubebatch-pod
  labels:
    app: myapp
    version: v1
spec:
  schedulerName: kube-batch
  containers:
  - name: app
    image: docker.io/busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
    resources:
      limits:
        cpu: "16"
        memory: "100Mi"
        nvidia.com/gpu: 2
      requests:
        cpu: "16"
        memory: "100Mi"
        nvidia.com/gpu: 2
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
