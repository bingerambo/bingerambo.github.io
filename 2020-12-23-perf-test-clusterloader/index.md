# K8S集群性能测试-clusterloader


如何使用perf-test的clusterloader进行性能测试

<!--more-->

<img src="/images/k8s/featured-image.jpg">

## 1 clusterloader准备

1. 从github上拉取[perf-test项目](https://github.com/kubernetes/perf-tests)，其中包含clusterloader2。perf-tests位置为：$GOPATH/src/k8s.io/perf-tests
    - 需要选择与测试k8s集群匹配的版本，这里选择了1.14版本
2. 进入clusterloader2目录，进行编译
```shell
go build -o clusterloader './cmd/'
```
3.  clusterloader2的测试配置文件在testing目录下。可以参考修改配置
4.  按修改后的测试配置文件，指定参数变量，执行clusterloader测试





## 2 clusterloader测试

### 1. 运行命令

**说明：运行命令前，需要根据测试场景，修改测试配置文件中的变量参数，配置文件包括有config.yaml， rc.yaml，deployment.yaml
具体配置参数说明，见下文。**

```shell
# 进入clusterloader可执行文件目录，配置文件也需转移到了此位置
cd /home/wangb/perf-test/clusterloader2
# ssh访问参数
export KUBE_SSH_KEY_PATH=/root/.ssh/id_rsa
# master节点信息
MASTER_NAME=node1
TEST_MASTER_IP=192.168.182.101
TEST_MASTER_INTERNAL_IP=192.168.182.101

KUBE_CONFIG=${HOME}/.kube/config
# 测试配置文件
TEST_CONFIG='/home/wangb/perf-test/clusterloader2/testing/density/config2.yaml'
# 测试报告
REPORT_DIR='./reports'
# 测试日志打印
LOG_FILE='test.log'

./clusterloader --kubeconfig=$KUBE_CONFIG \
    --mastername=$TEST_MASTER_IP \
    --masterip=$MASTER_IP \
    --master-internal-ip=TEST_MASTER_INTERNAL_IP \
    --testconfig=$TEST_CONFIG \
    --report-dir=$REPORT_DIR \
    --alsologtostderr 2>&1 | tee $LOG_FILE
```


### 2. 测试配置文件 test config

{{< admonition note "density 测试配置" >}}
* Steps is the procedures you defined. Each step might contain  phases, measurements
* Meansurement defines what you want to supervise or capture. 
* Phase describes the attributes of some certain tasks. 

{{< /admonition >}}


{{< admonition tip "This config defines the following steps:" >}}
1. Starting measurements
: don’t care about what happens during preparation.
2. Starting saturation pod measurements
: same as above
3. Creating saturation pods
: the first case is saturation pods
4. Collecting saturation pod measurements
5. Starting latency pod measurements
6. Creating latency pods
: the second case is latency pods
7. Waiting for latency pods to be running
8. Deleting latency pods
9. Waiting for latency pods to be deleted
10. Collecting pod startup latency
11. Deleting saturation pods
12. Waiting for saturation pods to be deleted
13. Collecting measurements
{{< /admonition >}}


**So we can see the testing mainly gathers measurements during the CRUD of saturation pods and latency pods:**


* saturation pods: pods in deployments with quite a large repliacas
* latency pods: pods in deployments with one replicas

So you see the differences between the two modes. When saturation pods are created, replicas-controller in kube-controller-manager is handling one event. But in terms of latency pods, it’s hundreds of events. But what’s the difference anyway? It’s because the various rate-limiter inside kubernetes affects the performance of scheduler and controller-manager.

In each case, what we’re concerned is the number of pods, deployments and namespaces. We all know that kubernetes limits the pods/node, pods/namespace, so it’s quite essential to adust relative parameters to achieve a reasonable load.


#### test config.yaml
```yaml
# ASSUMPTIONS:
# - Underlying cluster should have 100+ nodes.
# - Number of nodes should be divisible by NODES_PER_NAMESPACE (default 100).

#Constants
{{$DENSITY_RESOURCE_CONSTRAINTS_FILE := DefaultParam .DENSITY_RESOURCE_CONSTRAINTS_FILE ""}}
{{$NODE_MODE := DefaultParam .NODE_MODE "allnodes"}}
{{$NODES_PER_NAMESPACE := DefaultParam .NODES_PER_NAMESPACE 100}}
{{$PODS_PER_NODE := DefaultParam .PODS_PER_NODE 30}}
{{$DENSITY_TEST_THROUGHPUT := DefaultParam .DENSITY_TEST_THROUGHPUT 20}}
# LATENCY_POD_MEMORY and LATENCY_POD_CPU are calculated for 1-core 4GB node.
# Increasing allocation of both memory and cpu by 10%
# decreases the value of priority function in scheduler by one point.
# This results in decreased probability of choosing the same node again.
{{$LATENCY_POD_CPU := DefaultParam .LATENCY_POD_CPU 100}}
{{$LATENCY_POD_MEMORY := DefaultParam .LATENCY_POD_MEMORY 350}}
{{$MIN_LATENCY_PODS := 500}}
{{$MIN_SATURATION_PODS_TIMEOUT := 180}}
{{$ENABLE_CHAOSMONKEY := DefaultParam .ENABLE_CHAOSMONKEY false}}
{{$ENABLE_SYSTEM_POD_METRICS:= DefaultParam .ENABLE_SYSTEM_POD_METRICS true}}
{{$ENABLE_RESTART_COUNT_CHECK := DefaultParam .ENABLE_RESTART_COUNT_CHECK false}}
{{$RESTART_COUNT_THRESHOLD_OVERRIDES:= DefaultParam .RESTART_COUNT_THRESHOLD_OVERRIDES ""}}
#Variables
{{$namespaces := DivideInt .Nodes $NODES_PER_NAMESPACE}}
{{$podsPerNamespace := MultiplyInt $PODS_PER_NODE $NODES_PER_NAMESPACE}}
{{$totalPods := MultiplyInt $podsPerNamespace $namespaces}}
{{$latencyReplicas := DivideInt (MaxInt $MIN_LATENCY_PODS .Nodes) $namespaces}}
{{$totalLatencyPods := MultiplyInt $namespaces $latencyReplicas}}
{{$saturationRCTimeout := DivideFloat $totalPods $DENSITY_TEST_THROUGHPUT | AddInt $MIN_SATURATION_PODS_TIMEOUT}}
# saturationRCHardTimeout must be at least 20m to make sure that ~10m node
# failure won't fail the test. See https://github.com/kubernetes/kubernetes/issues/73461#issuecomment-467338711
{{$saturationRCHardTimeout := MaxInt $saturationRCTimeout 1200}}

name: density
automanagedNamespaces: {{$namespaces}}
tuningSets:
- name: Uniform5qps
  qpsLoad:
    qps: 5
{{if $ENABLE_CHAOSMONKEY}}
chaosMonkey:
  nodeFailure:
    failureRate: 0.01
    interval: 1m
    jitterFactor: 10.0
    simulatedDowntime: 10m
{{end}}
steps:
- measurements:
  - Identifier: APIResponsiveness
    Method: APIResponsiveness
    Params:
      action: reset
  - Identifier: TestMetrics
    Method: TestMetrics
    Params:
      action: start
      nodeMode: {{$NODE_MODE}}
      resourceConstraints: {{$DENSITY_RESOURCE_CONSTRAINTS_FILE}}
      systemPodMetricsEnabled: {{$ENABLE_SYSTEM_POD_METRICS}}
      restartCountThresholdOverrides: {{YamlQuote $RESTART_COUNT_THRESHOLD_OVERRIDES 4}}
      enableRestartCountCheck: {{$ENABLE_RESTART_COUNT_CHECK}}

# Create saturation pods
- measurements:
  - Identifier: SaturationPodStartupLatency
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group = saturation
      threshold: {{$saturationRCTimeout}}s
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: v1
      kind: ReplicationController
      labelSelector: group = saturation
      operationTimeout: {{$saturationRCHardTimeout}}s
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 1
    tuningSet: Uniform5qps
    objectBundle:
    - basename: saturation-rc
      objectTemplatePath: rc.yaml
      templateFillMap:
        Replicas: {{$podsPerNamespace}}
        Group: saturation
        CpuRequest: 1m
        MemoryRequest: 10M
- measurements:
  - Identifier: SchedulingThroughput
    Method: SchedulingThroughput
    Params:
      action: start
      labelSelector: group = saturation
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- measurements:
  - Identifier: SaturationPodStartupLatency
    Method: PodStartupLatency
    Params:
      action: gather
- measurements:
  - Identifier: SchedulingThroughput
    Method: SchedulingThroughput
    Params:
      action: gather
- name: Creating saturation pods
# Create latency pods
- measurements:
  - Identifier: PodStartupLatency
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group = latency
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: v1
      kind: ReplicationController
      labelSelector: group = latency
      operationTimeout: 15m
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: {{$latencyReplicas}}
    tuningSet: Uniform5qps
    objectBundle:
    - basename: latency-pod-rc
      objectTemplatePath: rc.yaml
      templateFillMap:
        Replicas: 1
        Group: latency
        CpuRequest: {{$LATENCY_POD_CPU}}m
        MemoryRequest: {{$LATENCY_POD_MEMORY}}M
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- name: Creating latency pods
# Remove latency pods
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 0
    tuningSet: Uniform5qps
    objectBundle:
    - basename: latency-pod-rc
      objectTemplatePath: rc.yaml
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- measurements:
  - Identifier: PodStartupLatency
    Method: PodStartupLatency
    Params:
      action: gather
- name: Deleting latancy pods
# Delete pods
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 0
    tuningSet: Uniform5qps
    objectBundle:
    - basename: saturation-rc
      objectTemplatePath: rc.yaml
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- name: Deleting saturation pods
# Collect measurements
- measurements:
  - Identifier: APIResponsiveness
    Method: APIResponsiveness
    Params:
      action: gather
  - Identifier: TestMetrics
    Method: TestMetrics
    Params:
      action: gather
      systemPodMetricsEnabled: {{$ENABLE_SYSTEM_POD_METRICS}}
      restartCountThresholdOverrides: {{YamlQuote $RESTART_COUNT_THRESHOLD_OVERRIDES 4}}
      enableRestartCountCheck: {{$ENABLE_RESTART_COUNT_CHECK}}

```

#### rc.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: {{.Name}}
  labels:
    group: {{.Group}}
spec:
  replicas: {{.Replicas}}
  selector:
    name: {{.Name}}
  template:
    metadata:
      labels:
        name: {{.Name}}
        group: {{.Group}}
    spec:
      containers:
      - image: k8s.gcr.io/pause:3.1
        imagePullPolicy: IfNotPresent
        name: {{.Name}}
        ports:
        resources:
          requests:
            cpu: {{.CpuRequest}}
            memory: {{.MemoryRequest}}
      # Add not-ready/unreachable tolerations for 15 minutes so that node
      # failure doesn't trigger pod deletion.
      tolerations:
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900

```

#### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{.Name}}
  labels:
    group: {{.Group}}
spec:
  replicas: {{.Replicas}}
  selector:
    matchLabels:
      name: {{.Name}}
  template:
    metadata:
      labels:
        name: {{.Name}}
        group: {{.Group}}
    spec:
      containers:
      - image: k8s.gcr.io/pause:3.1
        imagePullPolicy: IfNotPresent
        name: {{.Name}}
        ports:
        resources:
          requests:
            cpu: {{.CpuRequest}}
            memory: {{.MemoryRequest}}
      # Add not-ready/unreachable tolerations for 15 minutes so that node
      # failure doesn't trigger pod deletion.
      tolerations:
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900

```

### 3. clusterloader2 源码简析

解析测试配置信息，执行测试测试用例
clusterloader2/cmd/clusterloader.go
```golang
void main(){
  for _, clusterLoaderConfig.TestConfigPath = range testConfigPaths {
    test.RunTest(f, prometheusFramework, &clusterLoaderConfig)
    }
}
```

RunTest 又调用了 ExecuteTest，示例代码如下：

循环steps，按顺序执行ExecuteStep
```golang
// ExecuteTest executes test based on provided configuration.
func (ste *simpleTestExecutor) ExecuteTest(ctx Context, conf *api.Config)  {
  // 遍历steps，分步执行
  for i := range conf.Steps {
    if stepErrList := ste.ExecuteStep(ctx, &conf.Steps[i]); !stepErrList.IsEmpty() {
      errList.Concat(stepErrList)
      if isErrsCritical(stepErrList) {
        return errList
      }
    }
  }

  // 输出测试汇总信息
  for _, summary := range ctx.GetMeasurementManager().GetSummaries() {
    if ctx.GetClusterLoaderConfig().ReportDir == "" {
      klog.Infof("%v: %v", summary.SummaryName(), summary.SummaryContent())
    } else {
      // TODO(krzysied): Remember to keep original filename style for backward compatibility.
      filePath := path.Join(ctx.GetClusterLoaderConfig().ReportDir, summary.SummaryName()+"_"+conf.Name+"_"+summary.SummaryTime().Format(time.RFC3339)+"."+summary.SummaryExt())
      ioutil.WriteFile(filePath, []byte(summary.SummaryContent()), 0644)
    }
  }

}
```

可以看出 每个step中的Measurements和Phases都是并发执行的。

而且在每个step中，要么执行measurement.exec，要么执行phase.exec

clusterloader2/pkg/test/simple_test_executor.go
```golang

// ExecuteStep executes single test step based on provided step configuration.
func (ste *simpleTestExecutor) ExecuteStep(ctx Context, step *api.Step) *errors.ErrorList {
  var wg wait.Group
  errList := errors.NewErrorList()
  if len(step.Measurements) > 0 {
    for i := range step.Measurements {
      // index is created to make i value unchangeable during thread execution.
      index := i
      wg.Start(func() {
        err := ctx.GetMeasurementManager().Execute(step.Measurements[index].Method,
          step.Measurements[index].Identifier,
          step.Measurements[index].Params)
        if err != nil {
          errList.Append(fmt.Errorf("measurement call %s - %s error: %v", step.Measurements[index].Method, step.Measurements[index].Identifier, err))
        }
      })
    }
  } else {
    for i := range step.Phases {
      phase := &step.Phases[i]
      wg.Start(func() {
        if phaseErrList := ste.ExecutePhase(ctx, phase); !phaseErrList.IsEmpty() {
          errList.Concat(phaseErrList)
        }
      })
    }
  }
  wg.Wait()
  if step.Name != "" {
    klog.Infof("Step \"%s\" ended", step.Name)
  }
  return errList
}
```


### 部署测试

#### .1 k8s-2节点环境
在本地虚拟机2节点的测试环境中，需要修改测试配置文件和pod部署脚本。
测试配置文件主要修改参数有

1. NODES，测试工具会抓取实际环境中的节点数，进行设置
2. NODES_PER_NAMESPACE， 每个ns下的nodes数。**这里需注意: NODES > NODES_PER_NAMESPACE**
2. PODS_PER_NODE，每个节点下的pod数
3. MIN_LATENCY_PODS这个数值会跟 PODS_PER_NODE比较 选取最大的，作为LATENCY测试的参数。因为LATENCY测试一般使用较多pod
数，即$MIN_LATENCY_PODS

4. 测试中会有测试使用的资源参数，这里需要对实际情况进行config.yaml调整。
    - LATENCY_POD_CPU
    - LATENCY_POD_MEMORY
    - 其它自定义资源数量，可以在config.yaml或者rc.yaml和deployment文件中添加配置

##### config.yaml

这里主要修改如下：

* 上述的测试配置参数
* measurement-TestMetrics 注释掉，暂时规避解析收集Metrics操作导致的测试失败

```yaml 
# ASSUMPTIONS:
# - Underlying cluster should have 100+ nodes.
# - Number of nodes should be divisible by NODES_PER_NAMESPACE (default 100).

#Constants
{{$DENSITY_RESOURCE_CONSTRAINTS_FILE := DefaultParam .DENSITY_RESOURCE_CONSTRAINTS_FILE ""}}
#{{$NODE_MODE := DefaultParam .NODE_MODE "allnodes"}}
{{$NODE_MODE := DefaultParam .NODE_MODE "master"}}
{{$NODES_PER_NAMESPACE := DefaultParam .NODES_PER_NAMESPACE 1}}
{{$PODS_PER_NODE := DefaultParam .PODS_PER_NODE 10}}
{{$DENSITY_TEST_THROUGHPUT := DefaultParam .DENSITY_TEST_THROUGHPUT 20}}
# LATENCY_POD_MEMORY and LATENCY_POD_CPU are calculated for 1-core 4GB node.
# Increasing allocation of both memory and cpu by 10%
# decreases the value of priority function in scheduler by one point.
# This results in decreased probability of choosing the same node again.
{{$LATENCY_POD_CPU := DefaultParam .LATENCY_POD_CPU 5}}
{{$LATENCY_POD_MEMORY := DefaultParam .LATENCY_POD_MEMORY 3}}
{{$MIN_LATENCY_PODS := 20}}
{{$MIN_SATURATION_PODS_TIMEOUT := 180}}
{{$ENABLE_CHAOSMONKEY := DefaultParam .ENABLE_CHAOSMONKEY false}}
{{$ENABLE_SYSTEM_POD_METRICS:= DefaultParam .ENABLE_SYSTEM_POD_METRICS false}}
{{$ENABLE_RESTART_COUNT_CHECK := DefaultParam .ENABLE_RESTART_COUNT_CHECK false}}
{{$RESTART_COUNT_THRESHOLD_OVERRIDES:= DefaultParam .RESTART_COUNT_THRESHOLD_OVERRIDES ""}}
#Variables
{{$namespaces := DivideInt .Nodes $NODES_PER_NAMESPACE}}
{{$podsPerNamespace := MultiplyInt $PODS_PER_NODE $NODES_PER_NAMESPACE}}
{{$totalPods := MultiplyInt $podsPerNamespace $namespaces}}
{{$latencyReplicas := DivideInt (MaxInt $MIN_LATENCY_PODS .Nodes) $namespaces}}
{{$totalLatencyPods := MultiplyInt $namespaces $latencyReplicas}}
{{$saturationRCTimeout := DivideFloat $totalPods $DENSITY_TEST_THROUGHPUT | AddInt $MIN_SATURATION_PODS_TIMEOUT}}
# saturationRCHardTimeout must be at least 20m to make sure that ~10m node
# failure won't fail the test. See https://github.com/kubernetes/kubernetes/issues/73461#issuecomment-467338711
{{$saturationRCHardTimeout := MaxInt $saturationRCTimeout 1200}}

name: density
automanagedNamespaces: {{$namespaces}}
tuningSets:
- name: Uniform5qps
  qpsLoad:
    qps: 5
{{if $ENABLE_CHAOSMONKEY}}
chaosMonkey:
  nodeFailure:
    failureRate: 0.01
    interval: 1m
    jitterFactor: 10.0
    simulatedDowntime: 10m
{{end}}
steps:
- measurements:
  - Identifier: APIResponsiveness
    Method: APIResponsiveness
    Params:
      action: reset
#  - Identifier: TestMetrics
#    Method: TestMetrics
#    Params:
#      action: start
#      nodeMode: {{$NODE_MODE}}
#      resourceConstraints: {{$DENSITY_RESOURCE_CONSTRAINTS_FILE}}
#      systemPodMetricsEnabled: {{$ENABLE_SYSTEM_POD_METRICS}}
#      restartCountThresholdOverrides: {{YamlQuote $RESTART_COUNT_THRESHOLD_OVERRIDES 4}}
#      enableRestartCountCheck: {{$ENABLE_RESTART_COUNT_CHECK}}

# Create saturation pods
- measurements:
  - Identifier: SaturationPodStartupLatency
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group = saturation
      threshold: {{$saturationRCTimeout}}s
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: v1
      kind: ReplicationController
      labelSelector: group = saturation
      operationTimeout: {{$saturationRCHardTimeout}}s
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 1
    tuningSet: Uniform5qps
    objectBundle:
    - basename: saturation-rc
      objectTemplatePath: rc.yaml
      templateFillMap:
        Replicas: {{$podsPerNamespace}}
        Group: saturation
        CpuRequest: 1m
        MemoryRequest: 10M
- measurements:
  - Identifier: SchedulingThroughput
    Method: SchedulingThroughput
    Params:
      action: start
      labelSelector: group = saturation
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- measurements:
  - Identifier: SaturationPodStartupLatency
    Method: PodStartupLatency
    Params:
      action: gather
- measurements:
  - Identifier: SchedulingThroughput
    Method: SchedulingThroughput
    Params:
      action: gather
- name: Creating saturation pods
# Create latency pods
- measurements:
  - Identifier: PodStartupLatency
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group = latency
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: v1
      kind: ReplicationController
      labelSelector: group = latency
      operationTimeout: 15m
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: {{$latencyReplicas}}
    tuningSet: Uniform5qps
    objectBundle:
    - basename: latency-pod-rc
      objectTemplatePath: rc.yaml
      templateFillMap:
        Replicas: 1
        Group: latency
        CpuRequest: {{$LATENCY_POD_CPU}}m
        MemoryRequest: {{$LATENCY_POD_MEMORY}}M
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- name: Creating latency pods
# Remove latency pods
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 0
    tuningSet: Uniform5qps
    objectBundle:
    - basename: latency-pod-rc
      objectTemplatePath: rc.yaml
- measurements:
  - Identifier: WaitForRunningLatencyRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- measurements:
  - Identifier: PodStartupLatency
    Method: PodStartupLatency
    Params:
      action: gather
- name: Deleting latancy pods

# Delete pods
- phases:
  - namespaceRange:
      min: 1
      max: {{$namespaces}}
    replicasPerNamespace: 0
    tuningSet: Uniform5qps
    objectBundle:
    - basename: saturation-rc
      objectTemplatePath: rc.yaml
- measurements:
  - Identifier: WaitForRunningSaturationRCs
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
- name: Deleting saturation pods
# Collect measurements
- measurements:
  - Identifier: APIResponsiveness
    Method: APIResponsiveness
    Params:
      action: gather
#  - Identifier: TestMetrics
#    Method: TestMetrics
#    Params:
#      action: gather
#      systemPodMetricsEnabled: {{$ENABLE_SYSTEM_POD_METRICS}}
#      restartCountThresholdOverrides: {{YamlQuote $RESTART_COUNT_THRESHOLD_OVERRIDES 4}}
#      enableRestartCountCheck: {{$ENABLE_RESTART_COUNT_CHECK}}

```

##### rc.yaml
修改了image
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: {{.Name}}
  labels:
    group: {{.Group}}
spec:
  replicas: {{.Replicas}}
  selector:
    name: {{.Name}}
  template:
    metadata:
      labels:
        name: {{.Name}}
        group: {{.Group}}
    spec:
      containers:
      - image: 192.168.182.101:5000/com.inspur/pause-amd64:3.1
        imagePullPolicy: IfNotPresent
        name: {{.Name}}
        ports:
        resources:
          requests:
            cpu: {{.CpuRequest}}
            memory: {{.MemoryRequest}}
      # Add not-ready/unreachable tolerations for 15 minutes so that node
      # failure doesn't trigger pod deletion.
      tolerations:
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 900

```


##### 测试信息

命令运行报告信息，根据命令配置的report参数，进入report目录进行查看

```shell


[root@node1 clusterloader2]# ls
clusterloader  reports  testing  test.log
[root@node1 clusterloader2]# cd reports/
[root@node1 reports]# ll
total 44
-rw-r--r-- 1 root root 10093 Dec 11 15:38 APIResponsiveness_density_2020-12-11T15:38:47+08:00.json
-rw-r--r-- 1 root root  9792 Dec 11 16:29 APIResponsiveness_density_2020-12-11T16:29:41+08:00.json
-rw-r--r-- 1 root root   287 Dec 11 16:29 junit.xml
-rw-r--r-- 1 root root  1054 Dec 11 15:38 PodStartupLatency_SaturationPodStartupLatency_density_2020-12-11T15:37:54+08:00.json
-rw-r--r-- 1 root root  1048 Dec 11 16:29 PodStartupLatency_SaturationPodStartupLatency_density_2020-12-11T16:28:49+08:00.json
-rw-r--r-- 1 root root    64 Dec 11 15:38 SchedulingThroughput_density_2020-12-11T15:37:54+08:00.json
-rw-r--r-- 1 root root    64 Dec 11 16:29 SchedulingThroughput_density_2020-12-11T16:28:49+08:00.json

```

#### .2 kubemark节点环境






## 3 问题

1. 提示 Getting master name error: master node not found和 Getting master internal ip error: didn't find any InternalIP master IPs
mastername和 internalip 参数需要配置
```shell
I1211 11:10:31.302599  118141 clusterloader.go:105] ClusterConfig.Nodes set to 2
E1211 11:10:31.304485  118141 clusterloader.go:113] Getting master name error: master node not found
E1211 11:10:31.307705  118141 clusterloader.go:122] Getting master external ip error: didn't find any ExternalIP master IPs
E1211 11:10:31.309369  118141 clusterloader.go:131] Getting master internal ip error: didn't find any InternalIP master IPs
I1211 11:10:31.309388  118141 clusterloader.go:206] Using config: {ClusterConfig:{KubeConfigPath:/root/.kube/config Nodes:2 Provider: MasterIPs:[] MasterInternalIPs:[] MasterName: KubemarkRootKubeConfigPath:} ReportDir:./reports EnablePrometheusServer:false TearDownPrometheusServer:false TestConfigPath: TestOverridesPath:[] PrometheusConfig:{EnableServer:false TearDownServer:true ScrapeEtcd:false ScrapeNodeExporter:false ScrapeKubelets:false ScrapeKubeProxy:true SnapshotProject:}}
I1211 11:10:31.311334  118141 cluster.go:56] Listing cluster nodes:
I1211 11:10:31.311348  118141 cluster.go:68] Name: node1, clusterIP: 192.168.182.101, externalIP: , isSchedulable: true
I1211 11:10:31.311354  118141 cluster.go:68] Name: node2, clusterIP: 192.168.182.102, externalIP: , isSchedulable: true
I1211 11:10:31.314575  118141 clusterloader.go:167] --------------------------------------------------------------------------------
I1211 11:10:31.314588  118141 clusterloader.go:168] Running /home/wangb/perf-test/clusterloader2/testing/density/config.yaml
I1211 11:10:31.314591  118141 clusterloader.go:169] --------------------------------------------------------------------------------
```









2.  Errors: [measurement call TestMetrics - TestMetrics error: [unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}]

ssh问题，参数不正确，还需要自定义环境变量配置KUBE_SSH_KEY_PATH=/root/.ssh/id_rsa


```shell
E1211 11:34:39.085551   19551 test_metrics.go:185] TestMetrics: [unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}
unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}]
I1211 11:34:49.103215   19551 simple_test_executor.go:345] Resources cleanup time: 10.017395168s
E1211 11:34:49.103273   19551 clusterloader.go:177] --------------------------------------------------------------------------------
E1211 11:34:49.103291   19551 clusterloader.go:178] Test Finished
E1211 11:34:49.103295   19551 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config.yaml
E1211 11:34:49.103298   19551 clusterloader.go:180]   Status: Fail
E1211 11:34:49.103301   19551 clusterloader.go:182]   Errors: [measurement call TestMetrics - TestMetrics error: [unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}]
measurement call APIResponsiveness - APIResponsiveness error: top latency metric: there should be no high-latency requests, but: [got: {Resource:endpoints Subresource: Verb:GET Scope:namespace Latency:{Perc50:1.046ms Perc90:4.871ms Perc99:1.588679s} Count:33}; expected perc99 <= 1s]
measurement call TestMetrics - TestMetrics error: [unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}
unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}]]
E1211 11:34:49.103310   19551 clusterloader.go:184] --------------------------------------------------------------------------------
F1211 11:34:49.106925   19551 clusterloader.go:276] 1 tests have failed!
```



3. 测试流程最后，TestMetrics: [text format parsing error in line 1: invalid metric name]

https://github.com/kubernetes/perf-tests/issues/875
提的问题没有人解答

先把testMetic测试项关闭，暂时规避该问题。可能跟metric服务数据采集有关。待看源码分析下。
```shell
I1211 11:42:36.535182   30129 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1211 11:42:36.535218   30129 container_resource_gatherer.go:172] Closed stop channel. Waiting for 2 workers
I1211 11:42:36.535233   30129 resource_gather_worker.go:90] Closing worker for node2
I1211 11:42:36.535238   30129 resource_gather_worker.go:90] Closing worker for node1
I1211 11:42:36.535243   30129 container_resource_gatherer.go:180] Waitgroup finished.
I1211 11:42:36.535481   30129 system_pod_metrics.go:124] collecting system pod metrics...
I1211 11:42:36.545375   30129 system_pod_metrics.go:228] Loaded restart count threshold overrides: map[]
E1211 11:42:36.545827   30129 test_metrics.go:185] TestMetrics: [text format parsing error in line 1: invalid metric name]
I1211 11:42:46.580137   30129 simple_test_executor.go:345] Resources cleanup time: 10.03225584s
E1211 11:42:46.580213   30129 clusterloader.go:177] --------------------------------------------------------------------------------
E1211 11:42:46.580218   30129 clusterloader.go:178] Test Finished
E1211 11:42:46.580221   30129 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config.yaml
E1211 11:42:46.580225   30129 clusterloader.go:180]   Status: Fail
E1211 11:42:46.580227   30129 clusterloader.go:182]   Errors: [measurement call TestMetrics - TestMetrics error: [text format parsing error in line 1: invalid metric name]]
E1211 11:42:46.580229   30129 clusterloader.go:184] --------------------------------------------------------------------------------
F1211 11:42:46.581386   30129 clusterloader.go:276] 1 tests have failed!
```

## 4 附录

参考命令

批量删除k8s测试命名空间及其资源，这里测试数据默认使用了test-开头的命令规则
```shell
kubectl get ns |grep test- |awk '{print $1}' |xargs kubectl delete ns 
```


测试中如果出现异常，系统会残留有测试使用的资源参数，这里需要对实际情况进行调整

测试完成后的测试资源清理：
1. 测试ns、rc、pod清理
2. hollow-node 桩节点清理





参考文章

* [Kubernetes测试系列 - 性能测试](https://blog.csdn.net/qingdao666666/article/details/104625457)

* [kubernetes性能指标体系：clusterloader2](https://juejin.cn/post/6844904142389919758)

* [clusterloader2的漫漫踩坑路：最详细解析与使用指南](https://www.colabug.com/2020/0428/7329883/)

* [clusterloader2设计说明：Cluster loader vision](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/design.md)

