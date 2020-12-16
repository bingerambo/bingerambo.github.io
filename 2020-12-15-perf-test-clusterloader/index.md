# K8S集群性能测试-clusterloader


如何使用perf-test的clusterloader进行性能测试

<!--more-->

<img src="/images/k8s/featured-image.jpg">

## 1 K8S的性能指标：SLIs/SLOs

K8S的SLI (服务等级指标) 和 SLO (服务等级目标)：
Kubernetes 社区提供的K8S系统性能测试指标定义。

社区参考文档：[Kubernetes scalability and performance SLIs/SLOs](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)

目前社区提供的`官方正式`的性能指标有3个，如下表：

| Status | SLI | SLO |
| ------ | ----------- | ----------- |
| Official | Latency of mutating API calls for single objects for every (resource, verb) pair, measured as 99th percentile over last 5 minutes | In default Kubernetes installation, for every (resource, verb) pair, excluding virtual and aggregated resources and Custom Resource Definitions, 99th percentile per cluster-day1 <= 1s |
| Official | Latency of non-streaming read-only API calls for every (resource, scope pair, measured as 99th percentile over last 5 minutes | In default Kubernetes installation, for every (resource, scope) pair, excluding virtual and aggregated resources and Custom Resource Definitions, 99th percentile per cluster-day1 (a) <= 1s if scope=resource (b) <= 5s if scope=namespace (c) <= 30s if scope=cluster |
| Official | Startup latency of schedulable stateless pods, excluding time to pull images and run init containers, measured from pod creation timestamp to when all its containers are reported as started and observed via watch, measured as 99th percentile over last 5 minutes | In default Kubernetes installation, 99th percentile per cluster-day1 <= 5s |




## 2 clusterloader准备

1. 从github上拉取[perf-test项目](https://github.com/kubernetes/perf-tests)，其中包含clusterloader2。perf-tests位置为：$GOPATH/src/k8s.io/perf-tests
    - 需要选择与测试k8s集群匹配的版本，这里选择了1.14版本
2. 进入clusterloader2目录，进行编译
```shell
export GOPATH=/home/wangb/goprojects
cd $GOPATH/src/k8s.io/perf-tests/clusterloader2
go build -o clusterloader './cmd/'
```
3.  clusterloader2的测试配置文件在testing目录下。可以参考修改配置
4.  按修改后的测试配置文件，指定参数变量，执行clusterloader测试



## 3 clusterloader测试

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
# 测试报告目录位置
REPORT_DIR='./reports'
# 测试日志打印文件
LOG_FILE='test.log'


./clusterloader --kubeconfig=$KUBE_CONFIG \
    --mastername=$TEST_MASTER_IP \
    --masterip=$MASTER_IP \
    --master-internal-ip=TEST_MASTER_INTERNAL_IP \
    --testconfig=$TEST_CONFIG \
    --report-dir=$REPORT_DIR \
    --alsologtostderr 2>&1 | tee $LOG_FILE
```

运行命令可以指定nodes数量，不过这里默认使用集群全部节点。


### 2. 测试配置文件 test config（默认）

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


#### test config.yaml（默认配置）
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

    // 构造clusterLoaderConfig

    // 构造framework，即各种k8s client
    f, err := framework.NewFramework(
        &clusterLoaderConfig.ClusterConfig,
        getClientsNumber(clusterLoaderConfig.ClusterConfig.Nodes),
    )
    // 遍历测试配置文件（可多个），按配置用例运行测试
    for _, clusterLoaderConfig.TestConfigPath = range testConfigPaths {
        test.RunTest(f, prometheusFramework, &clusterLoaderConfig)
    }
}
```


```go
// RunTest runs test based on provided test configuration.
func RunTest(clusterFramework, prometheusFramework *framework.Framework, clusterLoaderConfig *config.ClusterLoaderConfig) *errors.ErrorList {

    // simpleContext上下文信息
    ctx := CreateContext(clusterLoaderConfig, clusterFramework, prometheusFramework, state.NewState())
    testConfigFilename := filepath.Base(clusterLoaderConfig.TestConfigPath)
    // 按参数 设置override config 和 nodes参数
    mapping, errList := config.GetMapping(clusterLoaderConfig)
    if errList != nil {
        return errList
    }
    // 使用emplateProvider根据mapping信息把testConfig的模板文件渲染成可用的api.Config
    testConfig, err := ctx.GetTemplateProvider().TemplateToConfig(testConfigFilename, mapping)
    if err != nil {
        return errors.NewErrorList(fmt.Errorf("config reading error: %v", err))
    }
    return Test.ExecuteTest(ctx, testConfig)
}

// api.Config 定义
// Config is a structure that represents configuration
// for a single test scenario.
type Config struct {
    // Name of the test case.
    Name string `json: name`
    // AutomanagedNamespaces is a number of automanaged namespaces.
    AutomanagedNamespaces int32 `json: automanagedNamespaces`
    // Steps is a sequence of test steps executed in serial.
    Steps []Step `json: steps`
    // TuningSets is a collection of tuning sets that can be used by steps.
    TuningSets []TuningSet `json: tuningSets`
    // ChaosMonkey is a config for simulated component failures.
    ChaosMonkey ChaosMonkeyConfig `json: chaosMonkey`
}
```



RunTest 又调用了 ExecuteTest，示例代码如下：
循环steps，按顺序执行ExecuteStep
```golang
// ExecuteTest executes test based on provided configuration.
func (ste *simpleTestExecutor) ExecuteTest(ctx Context, conf *api.Config)  {

    // auto set test namespace
  ctx.GetClusterFramework().SetAutomanagedNamespacePrefix(fmt.Sprintf("test-%s", util.RandomDNS1123String(6)))
  // clear test resource
  defer cleanupResources(ctx)  
  // create test namespace
  err = ctx.GetClusterFramework().CreateAutomanagedNamespaces(int(conf.AutomanagedNamespaces))
  // 遍历steps，分步执行，如果某step出错stepErr，则退出。
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


measurement Execute
根据 methodName, identifier 创建measurementInstance。目前有17种measurementInstance。比如：apiResponsivenessMeasurement；podStartupLatencyMeasurement等等
```go
// Execute executes measurement based on provided identifier, methodName and params.
func (mm *MeasurementManager) Execute(methodName string, identifier string, params map[string]interface{}) error {
    measurementInstance, err := mm.getMeasurementInstance(methodName, identifier)
    if err != nil {
        return err
    }
    config := &MeasurementConfig{
        ClusterFramework:    mm.clusterFramework,
        PrometheusFramework: mm.prometheusFramework,
        Params:              params,
        TemplateProvider:    mm.templateProvider,
        Identifier:          identifier,
        CloudProvider:       mm.clusterLoaderConfig.ClusterConfig.Provider,
    }
    summaries, err := measurementInstance.Execute(config)
    mm.summaries = append(mm.summaries, summaries...)
    return err
}
```

#### Measurement处理
以podStartupLatencyMeasurement为例分析，action参数分为start和gather，分别表示测试启动和测试收集

```go
// Execute supports two actions:
// - start - Starts to observe pods and pods events.
// - gather - Gathers and prints current pod latency data.
// Does NOT support concurrency. Multiple calls to this measurement
// shouldn't be done within one step.
func (p *podStartupLatencyMeasurement) Execute(config *measurement.MeasurementConfig) ([]measurement.Summary, error) {
    action, err := util.GetString(config.Params, "action")
    if err != nil {
        return nil, err
    }

    switch action {
    case "start":
        p.namespace, err = util.GetStringOrDefault(config.Params, "namespace", metav1.NamespaceAll)
        if err != nil {
            return nil, err
        }
        p.labelSelector, err = util.GetStringOrDefault(config.Params, "labelSelector", "")
        if err != nil {
            return nil, err
        }
        p.fieldSelector, err = util.GetStringOrDefault(config.Params, "fieldSelector", "")
        if err != nil {
            return nil, err
        }
        p.threshold, err = util.GetDurationOrDefault(config.Params, "threshold", defaultPodStartupLatencyThreshold)
        if err != nil {
            return nil, err
        }
        return nil, p.start(config.ClusterFramework.GetClientSets().GetClient())
    case "gather":
        return p.gather(config.ClusterFramework.GetClientSets().GetClient(), config.Identifier)
    default:
        return nil, fmt.Errorf("unknown action %v", action)
    }

}
// 测试启动
func (p *podStartupLatencyMeasurement) start(c clientset.Interface) error {
    if p.isRunning {
        klog.Infof("%s: pod startup latancy measurement already running", p)
        return nil
    }
    p.selectorsString = measurementutil.CreateSelectorsString(p.namespace, p.labelSelector, p.fieldSelector)
    klog.Infof("%s: starting pod startup latency measurement...", p)
    p.isRunning = true
    p.stopCh = make(chan struct{})
    p.informer = informer.NewInformer(
        c,
        "pods",
        p.namespace,
        p.fieldSelector,
        p.labelSelector,
        // 使用checkPod回调处理，一旦pod状态为running，则更新测试工具保存的pod的start时间
        p.checkPod,
    )
    // 启动了监听该范围内的pod资源
    go p.informer.Run(p.stopCh)
    timeoutCh := make(chan struct{})
    timeoutTimer := time.AfterFunc(informerSyncTimeout, func() {
        close(timeoutCh)
    })
    defer timeoutTimer.Stop()
    if !cache.WaitForCacheSync(timeoutCh, p.informer.HasSynced) {
        return fmt.Errorf("timed out waiting for caches to sync")
    }
    return nil
}


// 测试收集
func (p *podStartupLatencyMeasurement) gather(c clientset.Interface, identifier string) ([]measurement.Summary, error) {
    klog.Infof("%s: gathering pod startup latency measurement...", p)
    // 检查podStartupLatencyMeasurement 是否已启动
    if !p.isRunning {
        return nil, fmt.Errorf("metric %s has not been started", podStartupLatencyMeasurementName)
    }

    scheduleLag := make([]measurementutil.LatencyData, 0)
    startupLag := make([]measurementutil.LatencyData, 0)
    watchLag := make([]measurementutil.LatencyData, 0)
    schedToWatchLag := make([]measurementutil.LatencyData, 0)
    e2eLag := make([]measurementutil.LatencyData, 0)

    p.stop()
    // 通过schedEvents方式获取调度器事件的create时间
    if err := p.gatherScheduleTimes(c); err != nil {
        return nil, err
    }
    // 遍历pod-createTime map，按pod生命周期逻辑，进行条件检查
    for key, create := range p.createTimes {
        sched, hasSched := p.scheduleTimes[key]
        if !hasSched {
            klog.Infof("%s: failed to find schedule time for %v", p, key)
        }
        run, ok := p.runTimes[key]
        if !ok {
            klog.Infof("%s: failed to find run time for %v", p, key)
            continue
        }
        watch, ok := p.watchTimes[key]
        if !ok {
            klog.Infof("%s: failed to find watch time for %v", p, key)
            continue
        }
        node, ok := p.nodeNames[key]
        if !ok {
            klog.Infof("%s: failed to find node for %v", p, key)
            continue
        }
        // 计算各种延时，重要。。。
        if hasSched {
            scheduleLag = append(scheduleLag, podLatencyData{Name: key, Node: node, Latency: sched.Time.Sub(create.Time)})
            startupLag = append(startupLag, podLatencyData{Name: key, Node: node, Latency: run.Time.Sub(sched.Time)})
            schedToWatchLag = append(schedToWatchLag, podLatencyData{Name: key, Node: node, Latency: watch.Time.Sub(sched.Time)})
        }
        watchLag = append(watchLag, podLatencyData{Name: key, Node: node, Latency: watch.Time.Sub(run.Time)})
        e2eLag = append(e2eLag, podLatencyData{Name: key, Node: node, Latency: watch.Time.Sub(create.Time)})
    }
    // 把各个pod的各延时指标值进行排序，排序目的是为了方便进行数据统计
    sort.Sort(measurementutil.LatencySlice(scheduleLag))
    sort.Sort(measurementutil.LatencySlice(startupLag))
    sort.Sort(measurementutil.LatencySlice(watchLag))
    sort.Sort(measurementutil.LatencySlice(schedToWatchLag))
    sort.Sort(measurementutil.LatencySlice(e2eLag))

    p.printLatencies(scheduleLag, "worst create-to-schedule latencies")
    p.printLatencies(startupLag, "worst schedule-to-run latencies")
    p.printLatencies(watchLag, "worst run-to-watch latencies")
    p.printLatencies(schedToWatchLag, "worst schedule-to-watch latencies")
    p.printLatencies(e2eLag, "worst e2e latencies")

    podStartupLatency := &podStartupLatency{
        CreateToScheduleLatency: measurementutil.ExtractLatencyMetrics(scheduleLag),
        ScheduleToRunLatency:    measurementutil.ExtractLatencyMetrics(startupLag),
        RunToWatchLatency:       measurementutil.ExtractLatencyMetrics(watchLag),
        ScheduleToWatchLatency:  measurementutil.ExtractLatencyMetrics(schedToWatchLag),
        E2ELatency:              measurementutil.ExtractLatencyMetrics(e2eLag),
    }
    // 成功率
    var err error
    if successRatio := float32(len(e2eLag)) / float32(len(p.createTimes)); successRatio < successfulStartupRatioThreshold {
        err = fmt.Errorf("only %v%% of all pods were scheduled successfully", successRatio*100)
        klog.Errorf("%s: %v", p, err)
    }
    // 设置阈值，这里各百分位数使用的都是相同阈值
    podStartupLatencyThreshold := &measurementutil.LatencyMetric{
        Perc50: p.threshold,
        Perc90: p.threshold,
        Perc99: p.threshold,
    }
    // 进行延时指标验证，判断是否满足slos，不满足则输出错误信息
    if slosErr := podStartupLatency.E2ELatency.VerifyThreshold(podStartupLatencyThreshold); slosErr != nil {
        err = errors.NewMetricViolationError("pod startup", slosErr.Error())
        klog.Errorf("%s: %v", p, err)
    }

    content, jsonErr := util.PrettyPrintJSON(podStartupLatencyToPerfData(podStartupLatency))
    if err != nil {
        return nil, jsonErr
    }
    summary := measurement.CreateSummary(fmt.Sprintf("%s_%s", podStartupLatencyMeasurementName, identifier), "json", content)
    return []measurement.Summary{summary}, err
}


// VerifyThreshold verifies latency metric against given percentile thresholds.
func (metric *LatencyMetric) VerifyThreshold(threshold *LatencyMetric) error {
    if metric.Perc50 > threshold.Perc50 {
        return fmt.Errorf("too high latency 50th percentile: %v", metric.Perc50)
    }
    if metric.Perc90 > threshold.Perc90 {
        return fmt.Errorf("too high latency 90th percentile: %v", metric.Perc90)
    }
    if metric.Perc99 > threshold.Perc99 {
        return fmt.Errorf("too high latency 99th percentile: %v", metric.Perc99)
    }
    return nil
}

```


#### Phases处理
Phases处理，实际上就是对配置中的ObjectBundle进行处理, ste.ExecuteObject

```go
// ExecutePhase executes single test phase based on provided phase configuration.
func (ste *simpleTestExecutor) ExecutePhase(ctx Context, phase *api.Phase) *errors.ErrorList {
    // TODO: add tuning set
    errList := errors.NewErrorList()
    nsList := createNamespacesList(ctx, phase.NamespaceRange)
    tuningSet, err := ctx.GetTuningSetFactory().CreateTuningSet(phase.TuningSet)

    instances, exists := ctx.GetState().GetNamespacesState().Get(nsName, id)
    // // ExecuteObject executes single test object operation based on provided object configuration.
    ste.ExecuteObject(ctx, &phase.ObjectBundle[j], nsName, replicaIndex, XXX)

    tuningSet.Execute(actions)
    
}
```


### 4. 部署测试

#### .1 k8s-2节点环境
在本地虚拟机2节点的测试环境中，需要修改测试配置文件和pod部署脚本。
测试配置文件主要修改参数有

1. Nodes，属于配置文件上下文参数，如果不指定，测试工具会抓取实际环境中的可用的节点数，进行设置
2. NODES_PER_NAMESPACE， 每个ns下的nodes数。**这里需注意: NODES > NODES_PER_NAMESPACE**
2. PODS_PER_NODE，每个节点下的pod数
3. MIN_LATENCY_PODS这个数值会跟 PODS_PER_NODE比较 选取最大的，作为LATENCY测试的参数。因为LATENCY测试一般使用较多pod
数，即$MIN_LATENCY_PODS

4. 测试中会有测试使用的资源参数，这里需要对实际情况进行config.yaml调整。
    - LATENCY_POD_CPU
    - LATENCY_POD_MEMORY
    - 其它自定义资源数量，可以在config.yaml或者rc.yaml和deployment文件中添加配置

##### .1 部署config.yaml

这里主要修改如下：

* 上述的测试配置参数
* 主要修改参数有
  - NODES_PER_NAMESPACE
  - PODS_PER_NODE
  - MIN_LATENCY_PODS
  - LATENCY_POD_CPU
  - LATENCY_POD_MEMORY
  - DENSITY_TEST_THROUGHPUT

* measurement-TestMetrics 原有测试工具解析收集Metrics操作异常导致测试失败，详见后面问题描述

```yaml 
# ASSUMPTIONS:
# - Underlying cluster should have 100+ nodes.
# - Number of nodes should be divisible by NODES_PER_NAMESPACE (default 100).

#Constants
{{$DENSITY_RESOURCE_CONSTRAINTS_FILE := DefaultParam .DENSITY_RESOURCE_CONSTRAINTS_FILE ""}}
#{{$NODE_MODE := DefaultParam .NODE_MODE "allnodes"}}
{{$NODE_MODE := DefaultParam .NODE_MODE "master"}}
{{$NODES_PER_NAMESPACE := DefaultParam .NODES_PER_NAMESPACE 1}}
{{$PODS_PER_NODE := DefaultParam .PODS_PER_NODE 2}}
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

##### .2 部署rc.yaml
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


##### .3 部署测试信息

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



按上面的config.yaml，在2节点的本地虚拟机环境执行density测试，完整信息如下：

```shell

I1214 15:48:46.685903  129068 clusterloader.go:105] ClusterConfig.Nodes set to 2
E1214 15:48:46.690191  129068 clusterloader.go:122] Getting master external ip error: did not find any ExternalIP master IPs
I1214 15:48:46.690809  129068 clusterloader.go:206] Using config: {ClusterConfig:{KubeConfigPath:/root/.kube/config Nodes:2 Provider: MasterIPs:[] MasterInternalIPs:[TEST_MASTER_INTERNAL_IP] MasterName:192.168.182.101 KubemarkRootKubeConfigPath:} ReportDir:./reports EnablePrometheusServer:false TearDownPrometheusServer:false TestConfigPath: TestOverridesPath:[] PrometheusConfig:{EnableServer:false TearDownServer:true ScrapeEtcd:false ScrapeNodeExporter:false ScrapeKubelets:false ScrapeKubeProxy:true SnapshotProject:}}
I1214 15:48:46.693242  129068 cluster.go:56] Listing cluster nodes:
I1214 15:48:46.693254  129068 cluster.go:68] Name: node1, clusterIP: 192.168.182.101, externalIP: , isSchedulable: true
I1214 15:48:46.693260  129068 cluster.go:68] Name: node2, clusterIP: 192.168.182.102, externalIP: , isSchedulable: true
I1214 15:48:46.696447  129068 clusterloader.go:167] --------------------------------------------------------------------------------
I1214 15:48:46.696469  129068 clusterloader.go:168] Running /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
I1214 15:48:46.696472  129068 clusterloader.go:169] --------------------------------------------------------------------------------
I1214 15:48:46.697804  129068 simple_test_executor.go:50] AutomanagedNamespacePrefix: test-tteu8b
I1214 15:48:46.729142  129068 etcd_metrics.go:76] EtcdMetrics: starting etcd metrics collecting...
I1214 15:48:46.729168  129068 scheduler_latency.go:77] SchedulingMetrics: resetting latency metrics in scheduler...
I1214 15:48:46.729274  129068 api_responsiveness.go:70] APIResponsiveness: resetting latency metrics in apiserver...
I1214 15:48:46.936885  129068 resource_usage.go:106] ResourceUsageSummary: starting resource usage collecting...
I1214 15:48:46.948058  129068 system_pod_metrics.go:82] skipping collection of system pod metrics
I1214 15:48:46.948107  129068 pod_startup_latency.go:131] PodStartupLatency: labelSelector(group = saturation): starting pod startup latency measurement...
I1214 15:48:47.048405  129068 wait_for_controlled_pods.go:163] WaitForControlledPodsRunning: starting wait for controlled pods measurement...
I1214 15:48:47.607326  129068 scheduling_throughput.go:107] SchedulingThroughput: starting collecting throughput data
I1214 15:48:47.607393  129068 wait_for_controlled_pods.go:196] WaitForControlledPodsRunning: waiting for controlled pods measurement...
I1214 15:48:52.226268  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=saturation-rc-0): Pods: 2 out of 2 created, 2 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:52.408575  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=saturation-rc-0): Pods: 2 out of 2 created, 2 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:52.607705  129068 scheduling_throughput.go:123] SchedulingThroughput: labelSelector(group = saturation): 4 pods scheduled
I1214 15:48:52.611510  129068 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 2, deleted 0, timeout: 0, unknown: 0
I1214 15:48:52.611535  129068 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 2/2 ReplicationControllers are running with all pods
I1214 15:48:52.611582  129068 pod_startup_latency.go:163] PodStartupLatency: labelSelector(group = saturation): gathering pod startup latency measurement...
I1214 15:48:52.630805  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 4 worst create-to-schedule latencies: [{test-tteu8b-2/saturation-rc-0-t2t27 node2 0s} {test-tteu8b-1/saturation-rc-0-vxxc6 node2 0s} {test-tteu8b-1/saturation-rc-0-pqmh6 node1 0s} {test-tteu8b-2/saturation-rc-0-phkng node1 0s}]
I1214 15:48:52.630877  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 0s, perc90: 0s, perc99: 0s; threshold: 3m0s
I1214 15:48:52.630887  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 4 worst schedule-to-run latencies: [{test-tteu8b-2/saturation-rc-0-t2t27 node2 1s} {test-tteu8b-1/saturation-rc-0-vxxc6 node2 1s} {test-tteu8b-1/saturation-rc-0-pqmh6 node1 1s} {test-tteu8b-2/saturation-rc-0-phkng node1 1s}]
I1214 15:48:52.630894  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 1s, perc90: 1s, perc99: 1s; threshold: 3m0s
I1214 15:48:52.630903  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 4 worst run-to-watch latencies: [{test-tteu8b-2/saturation-rc-0-t2t27 node2 1.202667785s} {test-tteu8b-1/saturation-rc-0-vxxc6 node2 1.22260201s} {test-tteu8b-1/saturation-rc-0-pqmh6 node1 1.292204887s} {test-tteu8b-2/saturation-rc-0-phkng node1 1.30166796s}]
I1214 15:48:52.630909  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 1.22260201s, perc90: 1.30166796s, perc99: 1.30166796s; threshold: 3m0s
I1214 15:48:52.630912  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 4 worst schedule-to-watch latencies: [{test-tteu8b-2/saturation-rc-0-t2t27 node2 2.202667785s} {test-tteu8b-1/saturation-rc-0-vxxc6 node2 2.22260201s} {test-tteu8b-1/saturation-rc-0-pqmh6 node1 2.292204887s} {test-tteu8b-2/saturation-rc-0-phkng node1 2.30166796s}]
I1214 15:48:52.630920  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 2.22260201s, perc90: 2.30166796s, perc99: 2.30166796s; threshold: 3m0s
I1214 15:48:52.630922  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 4 worst e2e latencies: [{test-tteu8b-2/saturation-rc-0-t2t27 node2 2.202667785s} {test-tteu8b-1/saturation-rc-0-vxxc6 node2 2.22260201s} {test-tteu8b-1/saturation-rc-0-pqmh6 node1 2.292204887s} {test-tteu8b-2/saturation-rc-0-phkng node1 2.30166796s}]
I1214 15:48:52.630926  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 2.22260201s, perc90: 2.30166796s, perc99: 2.30166796s; threshold: 3m0s
I1214 15:48:52.631183  129068 scheduling_throughput.go:136] SchedulingThroughput: gathering data
I1214 15:48:52.631231  129068 simple_test_executor.go:128] Step "Creating saturation pods" ended
I1214 15:48:52.631261  129068 pod_startup_latency.go:131] PodStartupLatency: labelSelector(group = latency): starting pod startup latency measurement...
I1214 15:48:52.731639  129068 wait_for_controlled_pods.go:163] WaitForControlledPodsRunning: starting wait for controlled pods measurement...
I1214 15:48:56.854842  129068 wait_for_controlled_pods.go:196] WaitForControlledPodsRunning: waiting for controlled pods measurement...
I1214 15:48:57.893711  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-0): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:58.090743  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-1): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:58.290905  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-2): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:58.489711  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-3): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:58.690296  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-4): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:58.889999  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-5): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:59.089820  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-6): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:59.289666  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-7): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:59.490967  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-8): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:59.690953  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-9): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:48:59.892106  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-0): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:00.093290  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-1): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:00.296933  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-2): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:00.498491  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-3): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:00.700027  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-4): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:00.963786  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-5): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:01.107629  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-6): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:01.311728  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-7): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:01.509745  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-8): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:01.710317  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-9): Pods: 1 out of 1 created, 0 running, 1 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:02.893967  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-0): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:03.090943  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-1): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:03.290972  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-2): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:03.490268  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-3): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:03.690609  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-4): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:03.890985  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-5): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:04.090049  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-6): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:04.290203  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-7): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:04.691160  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-9): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:05.093608  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-1): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:05.499007  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-3): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:05.964020  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-5): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:06.312641  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-7): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:06.710758  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-9): Pods: 1 out of 1 created, 1 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:06.710803  129068 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 20, deleted 0, timeout: 0, unknown: 0
I1214 15:49:06.715769  129068 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 20/20 ReplicationControllers are running with all pods
I1214 15:49:06.720533  129068 simple_test_executor.go:128] Step "Creating latency pods" ended
I1214 15:49:10.750656  129068 wait_for_controlled_pods.go:196] WaitForControlledPodsRunning: waiting for controlled pods measurement...
I1214 15:49:11.801519  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-0): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:11.986805  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-1): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:12.184772  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-2): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:12.382919  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-3): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:12.589442  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-4): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:12.785810  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-5): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:12.983926  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-6): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:13.184330  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-7): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:13.384069  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-8): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:13.586112  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-9): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:13.790252  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-0): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:13.987269  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-1): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:14.192313  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-2): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:14.397369  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-3): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:14.605276  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-4): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:14.798794  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-5): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:14.997796  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-6): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:15.209917  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-7): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:15.394405  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-8): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:15.594518  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-9): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:17.184991  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-2): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:17.383303  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-3): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:17.786423  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-5): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:17.984309  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-6): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:18.184626  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-7): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:18.586402  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-9): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:18.987494  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-1): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:19.397720  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-3): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:19.799365  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-5): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:20.210268  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-7): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:20.594992  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=latency-pod-rc-9): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:22.984472  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-6): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:23.587181  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=latency-pod-rc-9): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:23.587247  129068 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 0, deleted 20, timeout: 0, unknown: 0
I1214 15:49:23.587289  129068 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 0/0 ReplicationControllers are running with all pods
I1214 15:49:23.612305  129068 pod_startup_latency.go:163] PodStartupLatency: labelSelector(group = latency): gathering pod startup latency measurement...
I1214 15:49:23.650456  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = latency): 20 worst create-to-schedule latencies: [{test-tteu8b-1/latency-pod-rc-0-lnrtg node2 0s} {test-tteu8b-1/latency-pod-rc-7-646cq node2 0s} {test-tteu8b-2/latency-pod-rc-4-tbszn node1 0s} {test-tteu8b-1/latency-pod-rc-5-mjbsr node2 0s} {test-tteu8b-1/latency-pod-rc-3-42gdw node2 0s} {test-tteu8b-2/latency-pod-rc-7-c7rst node2 0s} {test-tteu8b-1/latency-pod-rc-4-ngh2t node2 0s} {test-tteu8b-2/latency-pod-rc-1-4j655 node2 0s} {test-tteu8b-2/latency-pod-rc-9-xmct4 node2 0s} {test-tteu8b-1/latency-pod-rc-1-nk24t node2 0s} {test-tteu8b-1/latency-pod-rc-2-rj7h5 node2 0s} {test-tteu8b-1/latency-pod-rc-9-ww659 node2 0s} {test-tteu8b-2/latency-pod-rc-3-k8g85 node2 0s} {test-tteu8b-2/latency-pod-rc-0-wdhxz node1 0s} {test-tteu8b-2/latency-pod-rc-2-pwgsb node1 0s} {test-tteu8b-2/latency-pod-rc-8-c5m4q node1 0s} {test-tteu8b-2/latency-pod-rc-6-x7d4t node1 0s} {test-tteu8b-1/latency-pod-rc-8-4kx24 node1 0s} {test-tteu8b-1/latency-pod-rc-6-wskwq node2 0s} {test-tteu8b-2/latency-pod-rc-5-xx7qn node2 1s}]
I1214 15:49:23.650562  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = latency): perc50: 0s, perc90: 0s, perc99: 1s; threshold: 5s
I1214 15:49:23.650571  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = latency): 20 worst schedule-to-run latencies: [{test-tteu8b-1/latency-pod-rc-8-4kx24 node1 1s} {test-tteu8b-2/latency-pod-rc-2-pwgsb node1 1s} {test-tteu8b-2/latency-pod-rc-0-wdhxz node1 2s} {test-tteu8b-2/latency-pod-rc-6-x7d4t node1 2s} {test-tteu8b-2/latency-pod-rc-4-tbszn node1 2s} {test-tteu8b-2/latency-pod-rc-8-c5m4q node1 2s} {test-tteu8b-1/latency-pod-rc-0-lnrtg node2 3s} {test-tteu8b-1/latency-pod-rc-1-nk24t node2 3s} {test-tteu8b-2/latency-pod-rc-5-xx7qn node2 4s} {test-tteu8b-2/latency-pod-rc-9-xmct4 node2 4s} {test-tteu8b-1/latency-pod-rc-3-42gdw node2 4s} {test-tteu8b-2/latency-pod-rc-7-c7rst node2 4s} {test-tteu8b-1/latency-pod-rc-2-rj7h5 node2 4s} {test-tteu8b-1/latency-pod-rc-6-wskwq node2 4s} {test-tteu8b-1/latency-pod-rc-9-ww659 node2 5s} {test-tteu8b-2/latency-pod-rc-1-4j655 node2 5s} {test-tteu8b-1/latency-pod-rc-4-ngh2t node2 5s} {test-tteu8b-1/latency-pod-rc-5-mjbsr node2 5s} {test-tteu8b-1/latency-pod-rc-7-646cq node2 5s} {test-tteu8b-2/latency-pod-rc-3-k8g85 node2 5s}]
I1214 15:49:23.650585  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = latency): perc50: 4s, perc90: 5s, perc99: 5s; threshold: 5s
I1214 15:49:23.650588  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = latency): 20 worst run-to-watch latencies: [{test-tteu8b-2/latency-pod-rc-1-4j655 node2 560.606582ms} {test-tteu8b-2/latency-pod-rc-0-wdhxz node1 721.939052ms} {test-tteu8b-2/latency-pod-rc-8-c5m4q node1 833.924156ms} {test-tteu8b-2/latency-pod-rc-6-x7d4t node1 854.39232ms} {test-tteu8b-1/latency-pod-rc-5-mjbsr node2 1.012828114s} {test-tteu8b-2/latency-pod-rc-5-xx7qn node2 1.411960877s} {test-tteu8b-1/latency-pod-rc-7-646cq node2 1.647809261s} {test-tteu8b-2/latency-pod-rc-3-k8g85 node2 1.812513167s} {test-tteu8b-2/latency-pod-rc-2-pwgsb node1 1.832899011s} {test-tteu8b-2/latency-pod-rc-4-tbszn node1 1.865941757s} {test-tteu8b-1/latency-pod-rc-9-ww659 node2 2.009388509s} {test-tteu8b-1/latency-pod-rc-2-rj7h5 node2 2.082626218s} {test-tteu8b-1/latency-pod-rc-4-ngh2t node2 2.140752308s} {test-tteu8b-2/latency-pod-rc-9-xmct4 node2 2.210054751s} {test-tteu8b-1/latency-pod-rc-8-4kx24 node1 2.508582695s} {test-tteu8b-2/latency-pod-rc-7-c7rst node2 2.611408502s} {test-tteu8b-1/latency-pod-rc-6-wskwq node2 2.625719954s} {test-tteu8b-1/latency-pod-rc-1-nk24t node2 3.066876643s} {test-tteu8b-1/latency-pod-rc-3-42gdw node2 3.538575976s} {test-tteu8b-1/latency-pod-rc-0-lnrtg node2 5.586231683s}]
I1214 15:49:23.650601  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = latency): perc50: 1.865941757s, perc90: 3.066876643s, perc99: 5.586231683s; threshold: 5s
I1214 15:49:23.650604  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = latency): 20 worst schedule-to-watch latencies: [{test-tteu8b-2/latency-pod-rc-0-wdhxz node1 2.721939052s} {test-tteu8b-2/latency-pod-rc-2-pwgsb node1 2.832899011s} {test-tteu8b-2/latency-pod-rc-8-c5m4q node1 2.833924156s} {test-tteu8b-2/latency-pod-rc-6-x7d4t node1 2.85439232s} {test-tteu8b-1/latency-pod-rc-8-4kx24 node1 3.508582695s} {test-tteu8b-2/latency-pod-rc-4-tbszn node1 3.865941757s} {test-tteu8b-2/latency-pod-rc-5-xx7qn node2 5.411960877s} {test-tteu8b-2/latency-pod-rc-1-4j655 node2 5.560606582s} {test-tteu8b-1/latency-pod-rc-5-mjbsr node2 6.012828114s} {test-tteu8b-1/latency-pod-rc-1-nk24t node2 6.066876643s} {test-tteu8b-1/latency-pod-rc-2-rj7h5 node2 6.082626218s} {test-tteu8b-2/latency-pod-rc-9-xmct4 node2 6.210054751s} {test-tteu8b-2/latency-pod-rc-7-c7rst node2 6.611408502s} {test-tteu8b-1/latency-pod-rc-6-wskwq node2 6.625719954s} {test-tteu8b-1/latency-pod-rc-7-646cq node2 6.647809261s} {test-tteu8b-2/latency-pod-rc-3-k8g85 node2 6.812513167s} {test-tteu8b-1/latency-pod-rc-9-ww659 node2 7.009388509s} {test-tteu8b-1/latency-pod-rc-4-ngh2t node2 7.140752308s} {test-tteu8b-1/latency-pod-rc-3-42gdw node2 7.538575976s} {test-tteu8b-1/latency-pod-rc-0-lnrtg node2 8.586231683s}]
I1214 15:49:23.650616  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = latency): perc50: 6.066876643s, perc90: 7.140752308s, perc99: 8.586231683s; threshold: 5s
I1214 15:49:23.650620  129068 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = latency): 20 worst e2e latencies: [{test-tteu8b-2/latency-pod-rc-0-wdhxz node1 2.721939052s} {test-tteu8b-2/latency-pod-rc-2-pwgsb node1 2.832899011s} {test-tteu8b-2/latency-pod-rc-8-c5m4q node1 2.833924156s} {test-tteu8b-2/latency-pod-rc-6-x7d4t node1 2.85439232s} {test-tteu8b-1/latency-pod-rc-8-4kx24 node1 3.508582695s} {test-tteu8b-2/latency-pod-rc-4-tbszn node1 3.865941757s} {test-tteu8b-2/latency-pod-rc-1-4j655 node2 5.560606582s} {test-tteu8b-1/latency-pod-rc-5-mjbsr node2 6.012828114s} {test-tteu8b-1/latency-pod-rc-1-nk24t node2 6.066876643s} {test-tteu8b-1/latency-pod-rc-2-rj7h5 node2 6.082626218s} {test-tteu8b-2/latency-pod-rc-9-xmct4 node2 6.210054751s} {test-tteu8b-2/latency-pod-rc-5-xx7qn node2 6.411960877s} {test-tteu8b-2/latency-pod-rc-7-c7rst node2 6.611408502s} {test-tteu8b-1/latency-pod-rc-6-wskwq node2 6.625719954s} {test-tteu8b-1/latency-pod-rc-7-646cq node2 6.647809261s} {test-tteu8b-2/latency-pod-rc-3-k8g85 node2 6.812513167s} {test-tteu8b-1/latency-pod-rc-9-ww659 node2 7.009388509s} {test-tteu8b-1/latency-pod-rc-4-ngh2t node2 7.140752308s} {test-tteu8b-1/latency-pod-rc-3-42gdw node2 7.538575976s} {test-tteu8b-1/latency-pod-rc-0-lnrtg node2 8.586231683s}]
I1214 15:49:23.650631  129068 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = latency): perc50: 6.082626218s, perc90: 7.140752308s, perc99: 8.586231683s; threshold: 5s
E1214 15:49:23.651429  129068 pod_startup_latency.go:243] PodStartupLatency: labelSelector(group = latency): pod startup: too high latency 50th percentile: 6.082626218s
I1214 15:49:23.651671  129068 simple_test_executor.go:128] Step "Deleting latancy pods" ended
I1214 15:49:24.054984  129068 wait_for_controlled_pods.go:196] WaitForControlledPodsRunning: waiting for controlled pods measurement...
I1214 15:49:28.720087  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=saturation-rc-0): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 1 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:28.910121  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-2), labelSelector(name=saturation-rc-0): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:33.720408  129068 wait_for_pods.go:141] WaitForControlledPodsRunning: namespace(test-tteu8b-1), labelSelector(name=saturation-rc-0): Pods: 0 out of 0 created, 0 running, 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady
I1214 15:49:33.723457  129068 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 0, deleted 2, timeout: 0, unknown: 0
I1214 15:49:33.723518  129068 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 0/0 ReplicationControllers are running with all pods
I1214 15:49:33.723540  129068 simple_test_executor.go:128] Step "Deleting saturation pods" ended
I1214 15:49:33.833335  129068 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:8.831ms Perc90:13.349ms Perc99:36.302ms} Count:48}; threshold: 1s
I1214 15:49:33.833401  129068 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:GET Scope:namespace Latency:{Perc50:1.705ms Perc90:5.171ms Perc99:32.127ms} Count:184}; threshold: 1s
I1214 15:49:33.833407  129068 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:status Verb:PATCH Scope:namespace Latency:{Perc50:2.893ms Perc90:8.968ms Perc99:20.499ms} Count:105}; threshold: 1s
I1214 15:49:33.833412  129068 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:replicationcontrollers Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:4.333ms Perc90:10.995ms Perc99:19.334ms} Count:22}; threshold: 1s
I1214 15:49:33.833421  129068 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:binding Verb:POST Scope:namespace Latency:{Perc50:2.979ms Perc90:9.545ms Perc99:18.153ms} Count:24}; threshold: 1s
I1214 15:49:34.572385  129068 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1214 15:49:34.574852  129068 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1214 15:49:34.574936  129068 resource_gather_worker.go:90] Closing worker for node1
I1214 15:49:34.574949  129068 container_resource_gatherer.go:180] Waitgroup finished.
I1214 15:49:34.576009  129068 system_pod_metrics.go:82] skipping collection of system pod metrics
I1214 15:49:44.638344  129068 simple_test_executor.go:345] Resources cleanup time: 10.059218091s
I1214 15:49:44.641394  129068 clusterloader.go:177] --------------------------------------------------------------------------------
I1214 15:49:44.641440  129068 clusterloader.go:178] Test Finished
I1214 15:49:44.641461  129068 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
I1214 15:49:44.641465  129068 clusterloader.go:180]   Status: Success
I1214 15:49:44.641467  129068 clusterloader.go:184] --------------------------------------------------------------------------------

```

##### .4 测试报告

```shell
[root@node1 clusterloader2]# ll reports/ -h
total 260K
-rw-r--r-- 1 root root 9.9K Dec 14 16:26 APIResponsiveness_density_2020-12-14T16:26:20+08:00.json
-rw-r--r-- 1 root root 1.6K Dec 14 16:26 EtcdMetrics_density_2020-12-14T16:26:21+08:00.json
-rw-r--r-- 1 root root  287 Dec 14 16:26 junit.xml
-rw-r--r-- 1 root root 224K Dec 14 16:26 MetricsForE2E_density_2020-12-14T16:26:21+08:00.json
-rw-r--r-- 1 root root 1.1K Dec 14 16:26 PodStartupLatency_SaturationPodStartupLatency_density_2020-12-14T16:25:43+08:00.json
-rw-r--r-- 1 root root    3 Dec 14 16:26 ResourceUsageSummary_density_2020-12-14T16:26:21+08:00.json
-rw-r--r-- 1 root root  400 Dec 14 16:26 SchedulingMetrics_density_2020-12-14T16:26:21+08:00.json
-rw-r--r-- 1 root root   72 Dec 14 16:26 SchedulingThroughput_density_2020-12-14T16:25:43+08:00.json

```


* 测试的指标大多根据 metrics 获取
* 也有数据从 event 获取，比如 podStartupLatency

#### .2 kubemark节点环境


kubemark + clusterloader2方式测试

测试环境：总节点数8个：本地虚拟机2节点 + 6个kubemark节点
虚拟机内存大小为2G

1. 启动kubemark节点
```shell
[root@node1 wangb]# kubectl get no
NAME            STATUS   ROLES         AGE     VERSION
hollow-node-0   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
hollow-node-1   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
hollow-node-2   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
hollow-node-3   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
hollow-node-4   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
hollow-node-5   Ready    <none>        9s      v0.0.0-master+4d3c9e0c
node1           Ready    master,node   7d22h   v1.14.8
node2           Ready    node          7d22h   v1.14.8

```


2. clusterloader test config.yaml

##### node-pod状态


* 首先能够看到hollow-node和真实node上(共8个节点)都运行了pod
```shell
test-l3zhzg-2   saturation-rc-0-gmx45             1/1     Running             0          2m19s   10.152.121.76     hollow-node-1   <none>           <none>
test-l3zhzg-2   saturation-rc-0-grbkl             1/1     Running             0          2m19s   10.233.90.157     node1           <none>           <none>
test-l3zhzg-2   saturation-rc-0-jtw77             1/1     Running             0          2m17s   10.233.96.78      node2           <none>           <none>
test-l3zhzg-2   saturation-rc-0-krzkt             1/1     Running             0          2m19s   10.88.194.43      hollow-node-0   <none>           <none>
test-l3zhzg-2   saturation-rc-0-kskvv             1/1     Running             0          2m16s   10.177.65.132     hollow-node-4   <none>           <none>
test-l3zhzg-2   saturation-rc-0-l4zbp             1/1     Running             0          2m19s   10.16.150.60      hollow-node-0   <none>           <none>
test-l3zhzg-2   saturation-rc-0-l89cz             1/1     Running             0          2m13s   10.118.8.133      hollow-node-1   <none>           <none>
test-l3zhzg-2   saturation-rc-0-lp8tp             1/1     Running             0          2m19s   10.220.39.235     hollow-node-3   <none>           <none>
test-l3zhzg-2   saturation-rc-0-ls9zw             1/1     Running             0          2m19s   10.15.197.103     hollow-node-2   <none>           <none>
test-l3zhzg-2   saturation-rc-0-m64hg             1/1     Running             0          2m14s   10.172.167.195    hollow-node-2   <none>           <none>
test-l3zhzg-2   saturation-rc-0-m7qct             1/1     Running             0          2m17s   10.41.26.226      hollow-node-4   <none>           <none>
test-l3zhzg-2   saturation-rc-0-mct9d             1/1     Running             0          2m18s   10.126.207.219    hollow-node-4   <none>           <none>
test-l3zhzg-2   saturation-rc-0-ml2fw             1/1     Running             0          2m13s   10.245.91.90      hollow-node-2   <none>           <none>
test-l3zhzg-2   saturation-rc-0-mndt2             1/1     Running             0          2m19s   10.177.227.228    hollow-node-3   <none>           <none>
test-l3zhzg-2   saturation-rc-0-n6tzb             1/1     Running             0          2m13s   10.76.229.141     hollow-node-3   <none>           <none>
test-l3zhzg-2   saturation-rc-0-nfq2d             1/1     Running             0          2m19s   10.172.121.37     hollow-node-4   <none>           <none>
test-l3zhzg-2   saturation-rc-0-nn4wn             1/1     Running             0          2m18s   10.22.201.220     hollow-node-0   <none>           <none>
test-l3zhzg-2   saturation-rc-0-nvpz7             1/1     Running             0          2m16s   10.3.93.180       hollow-node-5   <none>           <none>
test-l3zhzg-2   saturation-rc-0-pfw58             1/1     Running             0          2m16s   10.144.139.248    hollow-node-1   <none>           <none>
test-l3zhzg-2   saturation-rc-0-pqbd6             1/1     Running             0          2m17s   10.207.147.215    hollow-node-0   <none>           <none>
test-l3zhzg-2   saturation-rc-0-q9x97             1/1     Running             0          2m14s   10.233.90.149     node1           <none>           <none>
test-l3zhzg-2   saturation-rc-0-qslqs             1/1     Running             0
```


* pod的调度延时
```shell
I1215 13:59:14.776944   27607 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 2/2 ReplicationControllers are running with all pods
I1215 13:59:14.777038   27607 pod_startup_latency.go:163] PodStartupLatency: labelSelector(group = saturation): gathering pod startup latency measurement...
I1215 13:59:14.795377   27607 pod_startup_latency.go:312] PodStartupLatency: labelSelector(group = saturation): 40 worst create-to-schedule latencies: [{test-bijvxv-1/saturation-rc-0-gktmn hollow-node-1 0s} {test-bijvxv-1/saturation-rc-0-gv9qb hollow-node-2 0s} {test-bijvxv-1/saturation-rc-0-n4cm5 hollow-node-4 0s} {test-bijvxv-2/saturation-rc-0-bn9z4 hollow-node-4 0s} {test-bijvxv-2/saturation-rc-0-srlf6 hollow-node-0 0s} {test-bijvxv-2/saturation-rc-0-btbng hollow-node-1 0s} {test-bijvxv-1/saturation-rc-0-kkcr7 hollow-node-5 0s} {test-bijvxv-2/saturation-rc-0-dctpp hollow-node-0 0s} {test-bijvxv-1/saturation-rc-0-5m8b2 hollow-node-1 0s} {test-bijvxv-1/saturation-rc-0-fpg6d hollow-node-2 0s} {test-bijvxv-1/saturation-rc-0-tjkvs hollow-node-4 0s} {test-bijvxv-2/saturation-rc-0-z6vtd hollow-node-2 0s} {test-bijvxv-2/saturation-rc-0-hrpwd hollow-node-3 0s} {test-bijvxv-2/saturation-rc-0-hzvk9 hollow-node-1 0s} {test-bijvxv-1/saturation-rc-0-x9b4c hollow-node-0 0s} {test-bijvxv-2/saturation-rc-0-4cqkz hollow-node-0 0s} {test-bijvxv-2/saturation-rc-0-4nppr node2 0s} {test-bijvxv-1/saturation-rc-0-rz69k hollow-node-2 0s} {test-bijvxv-2/saturation-rc-0-btsq4 hollow-node-3 0s} {test-bijvxv-1/saturation-rc-0-ll8sk hollow-node-3 0s} {test-bijvxv-1/saturation-rc-0-zjs57 hollow-node-1 0s} {test-bijvxv-1/saturation-rc-0-dq7x4 hollow-node-5 0s} {test-bijvxv-2/saturation-rc-0-fgxjz hollow-node-4 0s} {test-bijvxv-1/saturation-rc-0-g2bcp node1 0s} {test-bijvxv-2/saturation-rc-0-qp9pg hollow-node-3 0s} {test-bijvxv-1/saturation-rc-0-bdqzd hollow-node-5 0s} {test-bijvxv-1/saturation-rc-0-j5s2q hollow-node-4 0s} {test-bijvxv-2/saturation-rc-0-cp2wn node1 0s} {test-bijvxv-2/saturation-rc-0-2z8ct node1 0s} {test-bijvxv-2/saturation-rc-0-q6hd5 hollow-node-1 0s} {test-bijvxv-2/saturation-rc-0-68wwc hollow-node-5 0s} {test-bijvxv-2/saturation-rc-0-5t2wb hollow-node-4 0s} {test-bijvxv-1/saturation-rc-0-lt8ws hollow-node-0 0s} {test-bijvxv-1/saturation-rc-0-dwqz4 hollow-node-3 0s} {test-bijvxv-1/saturation-rc-0-vj68z node2 0s} {test-bijvxv-2/saturation-rc-0-96l5s hollow-node-5 0s} {test-bijvxv-1/saturation-rc-0-v7jq6 hollow-node-3 0s} {test-bijvxv-2/saturation-rc-0-8g4hx hollow-node-5 0s} {test-bijvxv-2/saturation-rc-0-hj7c7 hollow-node-2 0s} {test-bijvxv-1/saturation-rc-0-gx72w hollow-node-0 0s}]
I1215 13:59:14.795463   27607 pod_startup_latency.go:313] PodStartupLatency: labelSelector(group = saturation): perc50: 0s, perc90: 0s, perc99: 0s; threshold: 3m2s

```




##### 测试1
- NODES_PER_NAMESPACE 4
- PODS_PER_NODE 5
- MIN_LATENCY_PODS 20


```shell
I1215 14:05:35.050407   35029 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 0/0 ReplicationControllers are running with all pods
I1215 14:05:35.050777   35029 simple_test_executor.go:128] Step "Deleting saturation pods" ended
I1215 14:05:35.159705   35029 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:POST Scope:namespace Latency:{Perc50:2.613ms Perc90:11.315ms Perc99:52.045ms} Count:180}; threshold: 1s
I1215 14:05:35.159757   35029 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:5.669ms Perc90:15.912ms Perc99:36.005ms} Count:360}; threshold: 1s
I1215 14:05:35.159762   35029 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:binding Verb:POST Scope:namespace Latency:{Perc50:2.547ms Perc90:14.111ms Perc99:26.474ms} Count:180}; threshold: 1s
I1215 14:05:35.159767   35029 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:nodes Subresource:status Verb:PATCH Scope:cluster Latency:{Perc50:8.703ms Perc90:20.129ms Perc99:23.056ms} Count:14}; threshold: 1s
I1215 14:05:35.159772   35029 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:status Verb:PATCH Scope:namespace Latency:{Perc50:2.698ms Perc90:7.41ms Perc99:19.599ms} Count:905}; threshold: 1s
I1215 14:05:35.470022   35029 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1215 14:05:35.470058   35029 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1215 14:05:35.470072   35029 resource_gather_worker.go:90] Closing worker for node1
I1215 14:05:35.470079   35029 container_resource_gatherer.go:180] Waitgroup finished.
I1215 14:05:35.470202   35029 system_pod_metrics.go:82] skipping collection of system pod metrics
I1215 14:05:45.494628   35029 simple_test_executor.go:345] Resources cleanup time: 10.023988649s
I1215 14:05:45.494661   35029 clusterloader.go:177] --------------------------------------------------------------------------------
I1215 14:05:45.494665   35029 clusterloader.go:178] Test Finished
I1215 14:05:45.494669   35029 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config3.yaml
I1215 14:05:45.494673   35029 clusterloader.go:180]   Status: Success
I1215 14:05:45.494677   35029 clusterloader.go:184] --------------------------------------------------------------------------------


```


##### 测试2
- NODES_PER_NAMESPACE 4
- PODS_PER_NODE 20
- MIN_LATENCY_PODS 200

能够看到 apiserver的响应延时比上面的用例要大

```shell

I1215 14:16:30.674465   49564 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:6.124ms Perc90:17.215ms Perc99:36.206ms} Count:720}; threshold: 1s
I1215 14:16:30.674524   49564 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:POST Scope:namespace Latency:{Perc50:2.713ms Perc90:12.849ms Perc99:35.964ms} Count:360}; threshold: 1s
I1215 14:16:30.674530   49564 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:binding Verb:POST Scope:namespace Latency:{Perc50:2.481ms Perc90:11.292ms Perc99:29.995ms} Count:360}; threshold: 1s
I1215 14:16:30.674535   49564 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:replicationcontrollers Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:3.972ms Perc90:8.978ms Perc99:28.475ms} Count:202}; threshold: 1s
I1215 14:16:30.674543   49564 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:status Verb:PATCH Scope:namespace Latency:{Perc50:2.61ms Perc90:7.179ms Perc99:27.04ms} Count:1891}; threshold: 1s
I1215 14:16:31.136547   49564 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1215 14:16:31.136595   49564 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1215 14:16:31.136612   49564 resource_gather_worker.go:90] Closing worker for node1
I1215 14:16:31.136619   49564 container_resource_gatherer.go:180] Waitgroup finished.
I1215 14:16:31.136708   49564 system_pod_metrics.go:82] skipping collection of system pod metrics
I1215 14:16:41.156280   49564 simple_test_executor.go:345] Resources cleanup time: 10.019116668s
I1215 14:16:41.156316   49564 clusterloader.go:177] --------------------------------------------------------------------------------
I1215 14:16:41.156321   49564 clusterloader.go:178] Test Finished
I1215 14:16:41.156326   49564 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config3.yaml
I1215 14:16:41.156330   49564 clusterloader.go:180]   Status: Success
I1215 14:16:41.156334   49564 clusterloader.go:184] --------------------------------------------------------------------------------

```


##### 测试3

- NODES_PER_NAMESPACE 4
- PODS_PER_NODE 25
- MIN_LATENCY_PODS 200

```shell

I1215 16:19:38.812083   32636 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:subjectaccessreviews Subresource: Verb:POST Scope:cluster Latency:{Perc50:396µs Perc90:14.48ms Perc99:66.33ms} Count:37}; threshold: 1s
I1215 16:19:38.812151   32636 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:nodes Subresource:status Verb:PATCH Scope:cluster Latency:{Perc50:6.048ms Perc90:32.049ms Perc99:54.374ms} Count:24}; threshold: 1s
I1215 16:19:38.812159   32636 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:POST Scope:namespace Latency:{Perc50:3.118ms Perc90:12.026ms Perc99:34.693ms} Count:400}; threshold: 1s
I1215 16:19:38.812169   32636 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:endpoints Subresource: Verb:PUT Scope:namespace Latency:{Perc50:2.467ms Perc90:7.957ms Perc99:34.158ms} Count:166}; threshold: 1s
I1215 16:19:38.812174   32636 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:DELETE Scope:namespace Latency:{Perc50:6.106ms Perc90:13.581ms Perc99:33.557ms} Count:800}; threshold: 1s
I1215 16:19:39.258943   32636 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1215 16:19:39.258993   32636 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1215 16:19:39.259012   32636 resource_gather_worker.go:90] Closing worker for node1
I1215 16:19:39.259027   32636 container_resource_gatherer.go:180] Waitgroup finished.
I1215 16:19:39.259125   32636 system_pod_metrics.go:82] skipping collection of system pod metrics
I1215 16:19:49.287916   32636 simple_test_executor.go:345] Resources cleanup time: 10.028043195s
I1215 16:19:49.287943   32636 clusterloader.go:177] --------------------------------------------------------------------------------
I1215 16:19:49.287946   32636 clusterloader.go:178] Test Finished
I1215 16:19:49.287948   32636 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config3.yaml
I1215 16:19:49.287951   32636 clusterloader.go:180]   Status: Success
I1215 16:19:49.287954   32636 clusterloader.go:184] --------------------------------------------------------------------------------

```


##### 测试4

- NODES_PER_NAMESPACE 4
- PODS_PER_NODE 30
- MIN_LATENCY_PODS 500

API访问超时，虚拟机OOM报错，无法再运行测试用例



## 4 问题

### 1. 提示 Getting master name error: master node not found和 Getting master internal ip error: didn't find any InternalIP master IPs
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









### 2.  Errors: [measurement call TestMetrics - TestMetrics error: [unexpected error (code: 0) in ssh connection to master: &errors.errorString{s:"error getting signer for provider : 'GetSigner(...) not implemented for '"}]

测试配置了TestMetrics measurement，但是没有通过。

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



### 3. 告警提示：Master node is not registered. Grabbing metrics from Scheduler, ControllerManager and ClusterAutoscaler is disabled.





```shell

W1214 10:00:44.212402   40729 metrics_grabber.go:81] Master node is not registered. Grabbing metrics from Scheduler, ControllerManager and ClusterAutoscaler is disabled.
I1214 10:00:44.268795   40729 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1214 10:00:44.268822   40729 container_resource_gatherer.go:172] Closed stop channel. Waiting for 0 workers
I1214 10:00:44.268851   40729 container_resource_gatherer.go:180] Waitgroup finished.
I1214 10:00:44.268935   40729 system_pod_metrics.go:82] skipping collection of system pod metrics
E1214 10:00:44.268946   40729 test_metrics.go:185] TestMetrics: [text format parsing error in line 1: invalid metric name]
I1214 10:00:54.301192   40729 simple_test_executor.go:345] Resources cleanup time: 10.031663914s
E1214 10:00:54.301219   40729 clusterloader.go:177] --------------------------------------------------------------------------------
E1214 10:00:54.301222   40729 clusterloader.go:178] Test Finished
E1214 10:00:54.301225   40729 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
E1214 10:00:54.301227   40729 clusterloader.go:180]   Status: Fail
E1214 10:00:54.301229   40729 clusterloader.go:182]   Errors: [measurement call TestMetrics - TestMetrics error: [text format parsing error in line 1: invalid metric name]]
E1214 10:00:54.301233   40729 clusterloader.go:184] --------------------------------------------------------------------------------
F1214 10:00:54.305222   40729 clusterloader.go:276] 1 tests have failed!

```


排查过程，结合分析源码：

如果没有注册master节点，则测试不会统计调度器和controllers等组件信息
分处理逻辑，发现clusterloader2对master节点的判断条件不符合测试集群环境，如下。需要修改下clusterloader2的代码
```golang

// TODO: find a better way of figuring out if given node is a registered master.
func IsMasterNode(nodeName string) bool {
  // We are trying to capture "master(-...)?$" regexp.
  // However, using regexp.MatchString() results even in more than 35%
  // of all space allocations in ControllerManager spent in this function.
  // That's why we are trying to be a bit smarter.
  if strings.HasSuffix(nodeName, "master") {
    return true
  }
  if len(nodeName) >= 10 {
    return strings.HasSuffix(nodeName[:len(nodeName)-3], "master-")
  }
  return false
}

```

原有代码程序对master节点判断逻辑为：nodename为master或者master-开头
**修改代码：在system.IsMasterNode(node.Name) 引用处，新增条件： node.Labels["node-role.kubernetes.io/master"] == "true" ，作为master节点判断**


### 4. EtcdMetrics信息获取不到：EtcdMetrics: failed to collect etcd database size 

```shell

E1214 11:06:03.936128    2312 etcd_metrics.go:121] EtcdMetrics: failed to collect etcd database size

```


或者上报错误：TestMetrics: [text format parsing error in line 1: invalid metric name]

```shell
E1211 11:42:36.545827   30129 test_metrics.go:185] TestMetrics: [text format parsing error in line 1: invalid metric name]
```


https://github.com/kubernetes/perf-tests/issues/875
提的问题没有人解答

最初先把testMetic测试项关闭，暂时规避该问题。可能跟metric服务数据采集有关。后来排查了下日志打印信息，发现有多处报错，要逐个排查。


分析源码应该是获取不到etcd的metrics导致，修改代码如下：
measurement/common/simple/etcd_metrics

```golang
func (e *etcdMetricsMeasurement) getEtcdMetrics(host, provider string) ([]*model.Sample, error) {
  // Etcd is only exposed on localhost level. We are using ssh method
  if provider == "gke" {
    klog.Infof("%s: not grabbing etcd metrics through master SSH: unsupported for gke", e)
    return nil, nil
  }

  // In https://github.com/kubernetes/kubernetes/pull/74690, mTLS is enabled for etcd server
  // http://localhost:2382 is specified to bypass TLS credential requirement when checking
  // etcd /metrics and /health.
  //if samples, err := e.sshEtcdMetrics("curl http://localhost:2382/metrics", host, provider); err == nil {
  //  return samples, nil
  //}

  // fix: 问题错误信息：EtcdMetrics: failed to collect etcd database size
  // 这里需要根据实际测试环境情况，进行硬编码配置。 add by wangb
  // 先ssh，再执行metrics的cmd
  if samples, err := e.sshEtcdMetrics("curl https://localhost:2379/metrics -k --cert /etc/ssl/etcd/ssl/ca.pem --key /etc/ssl/etcd/ssl/ca-key.pem", host, provider); err == nil {
    return samples, nil
  }

  // Use old endpoint if new one fails.
  return e.sshEtcdMetrics("curl http://localhost:2379/metrics", host, provider)
}
```
按上述修改后，再重新编译，问题解决


### 5.报错找不到资源  TestMetrics: [the server could not find the requested resource (get pods kube-scheduler-192.168.182.101:10251)]


```shell
I1214 14:14:20.039016  126597 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1214 14:14:20.039058  126597 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1214 14:14:20.039075  126597 resource_gather_worker.go:90] Closing worker for node1
I1214 14:14:20.039082  126597 container_resource_gatherer.go:180] Waitgroup finished.
I1214 14:14:20.039181  126597 system_pod_metrics.go:82] skipping collection of system pod metrics
E1214 14:14:20.039193  126597 test_metrics.go:185] TestMetrics: [the server could not find the requested resource (get pods kube-scheduler-192.168.182.101:10251)]
I1214 14:14:30.103890  126597 simple_test_executor.go:345] Resources cleanup time: 10.064213743s
E1214 14:14:30.104163  126597 clusterloader.go:177] --------------------------------------------------------------------------------
E1214 14:14:30.104170  126597 clusterloader.go:178] Test Finished
E1214 14:14:30.104173  126597 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
E1214 14:14:30.104176  126597 clusterloader.go:180]   Status: Fail
E1214 14:14:30.104178  126597 clusterloader.go:182]   Errors: [measurement call TestMetrics - TestMetrics error: [the server could not find the requested resource (delete pods kube-scheduler-192.168.182.101:10251)]
measurement call TestMetrics - TestMetrics error: [the server could not find the requested resource (get pods kube-scheduler-192.168.182.101:10251)]]
E1214 14:14:30.104180  126597 clusterloader.go:184] --------------------------------------------------------------------------------
F1214 14:14:30.104658  126597 clusterloader.go:276] 1 tests have failed!

```

分析可能是 view resource no match 查询资源url不正确导致？
分析代码如下，可能是在msternode下构造request时有问题，定位原因为restclient构造url有问题。改用curl方式（可本地测试通过）直接获取调度器metrics
common/simple/scheduler_latency.go
```golang
// Sends request to kube scheduler metrics
func (s *schedulerLatencyMeasurement) sendRequestToScheduler(c clientset.Interface, op, host, provider, masterName string) (string, error) {
  opUpper := strings.ToUpper(op)
  if opUpper != "GET" && opUpper != "DELETE" {
    return "", fmt.Errorf("unknown REST request")
  }

  nodes, err := c.CoreV1().Nodes().List(metav1.ListOptions{})
  if err != nil {
    return "", err
  }

  var masterRegistered = false
  for _, node := range nodes.Items {
    if node.Labels["node-role.kubernetes.io/master"] == "true" || system.IsMasterNode(node.Name) {
      masterRegistered = true
    }
  }

  var responseText string
  // masterRegistered时，client接口处理有问题，统一改使用curl -X 方式处理GET和DELETE  add by wangb start
  _ = masterRegistered
  //if masterRegistered {
  //  ctx, cancel := context.WithTimeout(context.Background(), singleRestCallTimeout)
  //  defer cancel()
  //
  //  body, err := c.CoreV1().RESTClient().Verb(opUpper).
  //    Context(ctx).
  //    Namespace(metav1.NamespaceSystem).
  //    Resource("pods").
  //    Name(fmt.Sprintf("kube-scheduler-%v:%v", masterName, ports.InsecureSchedulerPort)).
  //    SubResource("proxy").
  //    Suffix("metrics").
  //    Do().Raw()
  //
  //  if err != nil {
  //    return "", err
  //  }
  //  responseText = string(body)
  //} else {
  //  // If master is not registered fall back to old method of using SSH.
  //  if provider == "gke" {
  //    klog.Infof("%s: not grabbing scheduler metrics through master SSH: unsupported for gke", s)
  //    return "", nil
  //  }
  //
  //  cmd := "curl -X " + opUpper + " http://localhost:10251/metrics"
  //  sshResult, err := measurementutil.SSH(cmd, host+":22", provider)
  //  if err != nil || sshResult.Code != 0 {
  //    return "", fmt.Errorf("unexpected error (code: %d) in ssh connection to master: %#v", sshResult.Code, err)
  //  }
  //  responseText = sshResult.Stdout
  //}

  // curl http://localhost:10251/metrics 这个命令测试可用
  cmd := "curl -X " + opUpper + " http://localhost:10251/metrics"
  sshResult, err := measurementutil.SSH(cmd, host+":22", provider)
  if err != nil || sshResult.Code != 0 {
    return "", fmt.Errorf("unexpected error (code: %d) in ssh connection to master: %#v", sshResult.Code, err)
  }
  responseText = sshResult.Stdout
  // masterRegistered时，client接口处理有问题，统一改使用curl -X 方式处理GET和DELETE  add by wangb end
  return responseText, nil
}
```


### 6. 测试结果指标异常输出
不是问题，这是测试工具成功生效，并返回提示断言信息

```shell
I1214 15:25:56.117594   96634 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 0, deleted 2, timeout: 0, unknown: 0
I1214 15:25:56.117625   96634 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 0/0 ReplicationControllers are running with all pods
I1214 15:25:56.124212   96634 simple_test_executor.go:128] Step "Deleting saturation pods" ended
I1214 15:25:56.245924   96634 api_responsiveness.go:119] APIResponsiveness: WARNING Top latency metric: {Resource:endpoints Subresource: Verb:PUT Scope:namespace Latency:{Perc50:2.65ms Perc90:22.594ms Perc99:1.122221s} Count:22}; threshold: 1s
I1214 15:25:56.245949   96634 api_responsiveness.go:119] APIResponsiveness: WARNING Top latency metric: {Resource:namespaces Subresource: Verb:GET Scope:cluster Latency:{Perc50:11.99ms Perc90:1.005472s Perc99:1.084129s} Count:13}; threshold: 1s
I1214 15:25:56.245957   96634 api_responsiveness.go:119] APIResponsiveness: WARNING Top latency metric: {Resource:nodes Subresource:status Verb:PATCH Scope:cluster Latency:{Perc50:1.00345s Perc90:1.00345s Perc99:1.00345s} Count:1}; threshold: 1s
I1214 15:25:56.245962   96634 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:status Verb:PATCH Scope:namespace Latency:{Perc50:3.777ms Perc90:13.656ms Perc99:173.072ms} Count:88}; threshold: 1s
I1214 15:25:56.245966   96634 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:GET Scope:namespace Latency:{Perc50:1.88ms Perc90:11.522ms Perc99:87.668ms} Count:156}; threshold: 1s
I1214 15:25:56.821263   96634 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1214 15:25:56.823909   96634 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1214 15:25:56.824075   96634 resource_gather_worker.go:90] Closing worker for node1
I1214 15:25:56.824118   96634 container_resource_gatherer.go:180] Waitgroup finished.
I1214 15:25:56.824313   96634 system_pod_metrics.go:82] skipping collection of system pod metrics
I1214 15:26:06.865304   96634 simple_test_executor.go:345] Resources cleanup time: 10.040658542s
E1214 15:26:06.865325   96634 clusterloader.go:177] --------------------------------------------------------------------------------
E1214 15:26:06.865328   96634 clusterloader.go:178] Test Finished
E1214 15:26:06.865330   96634 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
E1214 15:26:06.865335   96634 clusterloader.go:180]   Status: Fail
E1214 15:26:06.865338   96634 clusterloader.go:182]   Errors: [measurement call APIResponsiveness - APIResponsiveness error: top latency metric: there should be no high-latency requests, but: [got: {Resource:endpoints Subresource: Verb:PUT Scope:namespace Latency:{Perc50:2.65ms Perc90:22.594ms Perc99:1.122221s} Count:22}; expected perc99 <= 1s got: {Resource:namespaces Subresource: Verb:GET Scope:cluster Latency:{Perc50:11.99ms Perc90:1.005472s Perc99:1.084129s} Count:13}; expected perc99 <= 1s got: {Resource:nodes Subresource:status Verb:PATCH Scope:cluster Latency:{Perc50:1.00345s Perc90:1.00345s Perc99:1.00345s} Count:1}; expected perc99 <= 1s]]
E1214 15:26:06.865341   96634 clusterloader.go:184] --------------------------------------------------------------------------------
F1214 15:26:06.866736   96634 clusterloader.go:276] 1 tests have failed!

```

由上看出，由于时延性能指标超过门限值1s，测试工具认为测试不通过。

修改下 测试配置文件中的PODS_PER_NODE参数，由10改为2，负载变小，则测试通过

```shell
I1214 15:35:53.874477  111782 wait_for_controlled_pods.go:235] WaitForControlledPodsRunning: running 0, deleted 2, timeout: 0, unknown: 0
I1214 15:35:53.874751  111782 wait_for_controlled_pods.go:249] WaitForControlledPodsRunning: 0/0 ReplicationControllers are running with all pods
I1214 15:35:53.874765  111782 simple_test_executor.go:128] Step "Deleting saturation pods" ended
I1214 15:35:53.956315  111782 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:replicationcontrollers Subresource:status Verb:PUT Scope:namespace Latency:{Perc50:3.108ms Perc90:9.428ms Perc99:11.831ms} Count:11}; threshold: 1s
I1214 15:35:53.956378  111782 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource:status Verb:PATCH Scope:namespace Latency:{Perc50:2.767ms Perc90:7.49ms Perc99:9.821ms} Count:21}; threshold: 1s
I1214 15:35:53.956384  111782 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:GET Scope:namespace Latency:{Perc50:1.797ms Perc90:5.05ms Perc99:9.388ms} Count:36}; threshold: 1s
I1214 15:35:53.956388  111782 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:pods Subresource: Verb:POST Scope:namespace Latency:{Perc50:7.554ms Perc90:8.879ms Perc99:8.879ms} Count:4}; threshold: 1s
I1214 15:35:53.956392  111782 api_responsiveness.go:119] APIResponsiveness: Top latency metric: {Resource:services Subresource: Verb:GET Scope:namespace Latency:{Perc50:8.757ms Perc90:8.757ms Perc99:8.757ms} Count:1}; threshold: 1s
I1214 15:35:54.364397  111782 resource_usage.go:124] ResourceUsageSummary: gathering resource usage...
I1214 15:35:54.364444  111782 container_resource_gatherer.go:172] Closed stop channel. Waiting for 1 workers
I1214 15:35:54.364648  111782 resource_gather_worker.go:90] Closing worker for node1
I1214 15:35:54.364655  111782 container_resource_gatherer.go:180] Waitgroup finished.
I1214 15:35:54.366369  111782 system_pod_metrics.go:82] skipping collection of system pod metrics
I1214 15:36:04.390697  111782 simple_test_executor.go:345] Resources cleanup time: 10.023892674s
I1214 15:36:04.390750  111782 clusterloader.go:177] --------------------------------------------------------------------------------
I1214 15:36:04.390753  111782 clusterloader.go:178] Test Finished
I1214 15:36:04.390756  111782 clusterloader.go:179]   Test: /home/wangb/perf-test/clusterloader2/testing/density/config2.yaml
I1214 15:36:04.390758  111782 clusterloader.go:180]   Status: Success
I1214 15:36:04.390760  111782 clusterloader.go:184] --------------------------------------------------------------------------------
```


## 5 总结

1. perf-test clusterloader2工具主要提供了性能压测，可配置性好，方便编写测试用例，并且统计了相应的性能指标
2. clusterloader2内置实现了k8s指标采集处理和指标阈值定义，参考文档：[Kubernetes scalability and performance SLIs/SLOs](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)
3. clusterloader2没有详细的使用说明文档，目前来看不是可以拿来直接运行使用。所遇到问题一般只能依靠自己解决。
4. 由于上面第3点，所遇问题较多，一般多涉及测试工具环境配置参数，另外clusterloader2对一些参数使用的是硬编码方式，导致无法直接使用原有工具，只能修改源码进行测试适配。
5. 测试使用clusterloader2，需要详细了解其设计方案，才能运行测试用例
6. 进行集群测试，需要了解集群测试指标定义，再编写测试配置
7. 测试时需要预估下测试pod数量和内存占用情况，否则会引起OOM。
7. **clusterloader2并不是一个拿来即用的测试工具，还需结合测试环境进行改造适配，更像是K8S内部使用的类似脚手架的东西**




## 6 附录

参考命令

批量删除k8s测试命名空间及其资源，这里测试数据默认使用了test-开头的命令规则
```shell
kubectl get ns |grep test- |awk '{print $1}' |xargs kubectl delete ns --force --grace-period=0
```


测试中如果出现异常，系统会残留有测试使用的资源参数，这里需要对实际情况进行调整

测试完成后的测试资源清理（如果测试后有测试数据资源残留的话）：
1. 测试ns、rc、pod清理
2. hollow-node 桩节点清理


#### K8S的SLI (服务等级指标) 和 SLO (服务等级目标) 

Kubernetes 社区提供了 SLI (服务等级指标) 和 SLO (服务等级目标) 系统性能测试、分析文档 [Kubernetes scalability and performance SLIs/SLOs](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)。模拟出一个 K8s cluster（Kubemark cluster），不受资源限制。cluster 中 master 是真实的机器，所有的 nodes 是 Hollow nodes。Hollow nodes 不会调用Docker，测试一套 K8s API 调用的完整流程，不会真正创建 pod。

 

社区开发了 perf-test/clusterloader2，可配置性好，并且统计了相应的性能指标

kubemark 不调用 CRI 接口之外，其它行为和 kubelet 基本一致


### Etcd监控指标
参考: https://github.com/coreos/etcd/blob/master/Documentation/metrics.md

#### 领导者相关

etcd_server_has_leader etcd是否有leader
etcd_server_leader_changes_seen_total etcd的leader变换次数
etcd_debugging_mvcc_db_total_size_in_bytes 数据库的大小
process_resident_memory_bytes 进程驻留内存

#### 网络相关

grpc_server_started_total grpc(高性能、开源的通用RPC(远程过程调用)框架)服务器启动总数
etcd_network_client_grpc_received_bytes_total 接收到grpc客户端的字节总数
etcd_network_client_grpc_sent_bytes_total 发送给grpc客户端的字节总数
etcd_network_peer_received_bytes_total etcd网络对等方接收的字节总数(对等网络，即对等计算机网络，是一种在对等者（Peer）之间分配任务和工作负载的分布式应用架构，是对等计算模型在应用层形成的一种组网或网络形式)
etcd_network_peer_sent_bytes_total etcd网络对等方发送的字节总数

#### 提案相关

etcd_server_proposals_failed_total 目前正在处理的提案(提交会议讨论决定的建议。)数量
etcd_server_proposals_pending 失败提案总数
etcd_server_proposals_committed_total 已落实共识提案的总数。
etcd_server_proposals_applied_total 已应用的共识提案总数。


#### 这些指标描述了磁盘操作的状态。

etcd_disk_backend_commit_duration_seconds_sum etcd磁盘后端提交持续时间秒数总和
etcd_disk_backend_commit_duration_seconds_bucket etcd磁盘后端提交持续时间

#### 快照

etcd_debugging_snap_save_total_duration_seconds_sum etcd快照保存用时

#### 文件

process_open_fds{service="etcd-k8s"} 打开文件描述符的数量
process_max_fds{service="etcd-k8s"} 打开文件描述符的最大数量
etcd_disk_wal_fsync_duration_seconds_sum Wal(预写日志系统)调用的fsync(将文件数据同步到硬盘)的延迟分布
etcd_disk_wal_fsync_duration_seconds_bucket 后端调用的提交的延迟分布





参考文章

* [Kubernetes测试系列 - 性能测试](https://blog.csdn.net/qingdao666666/article/details/104625457)

* [kubernetes性能指标体系：clusterloader2](https://juejin.cn/post/6844904142389919758)

* [clusterloader2的漫漫踩坑路：最详细解析与使用指南](https://www.colabug.com/2020/0428/7329883/)

* [clusterloader2设计说明：Cluster loader vision](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/design.md)
* [etcd指标监控，参考文章](https://sysdig.com/blog/monitor-etcd/)


