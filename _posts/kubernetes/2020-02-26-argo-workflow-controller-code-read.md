---
title: "Argo Workflow Controller 源码阅读"
published: true
tags: 
  - "kubernetes"
  - "cicd"
  - "cncf"
---

Argo V2 版本通过 Kubernetes CRD（Custom Resource Definition）来进行实现的，所以我们可以通过 kubectl 工具来管理 Argo 工作流，当然就可以和其他 Kubernetes 资源对象直接集成了，比如 Volumes、Secrets、RBAC 等等。新版本的 Argo 更加轻量级，安装也十分简单，但是依然提供了完整的工作流功能，包括参数替换、制品、Fixture、循环和递归工作流等功能。

Argo 中的工作流自动化是通过使用 ADSL（Argo 领域特定语言）设计的 YAML 模板（因为 Kubernetes 主要也使用相同的 DSL 方式，所以非常容易使用）进行驱动的。ADSL 中提供的每条指令都被看作一段代码，并与代码仓库中的源码一起托管。Argo 支持6中不同的 YAML 结构:

* 容器模板:根据需要创建单个容器和参数
* 工作流模板:定义一个作业任务（工作流中的一个步骤可以是一个容器）
* 策略模板:触发或者调用作业/通知的规则
* 部署模板:创建一个长期运行的应用程序模板
* Fixture 模板:整合 Argo 外部的第三方资源
* 项目模板:可以在 Argo 目录中访问的工作流定义
  
Argo 支持几种不同的方式来定义 Kubernetes 资源清单：Ksonnect、Helm Chart 以及简单的 YAML/JSON 资源清单目录。

> 代码段基于argo-2.4版本

------

## Workflow Controller 进程启动

> cmd/workflow-controller/main.go

```go
wfController := controller.NewWorkflowController(config, kubeclientset, wflientset, namespace, executorImage, executorImagePullPolicy, configMap)
```

## Workflow Controller Config 参数

> workflow/config/config.go

```go
type WorkflowControllerConfig struct {
	// 执行workflow每个单元的基础镜像
	ExecutorImage string `json:"executorImage,omitempty"`

	// 基础镜像的拉取策略
	ExecutorImagePullPolicy string `json:"executorImagePullPolicy,omitempty"`

	// Executor holds container customizations for the executor to use when running pods
	Executor *apiv1.Container `json:"executor,omitempty"`

	// ExecutorResources specifies the resource requirements that will be used for the executor sidecar
	// DEPRECATED: use `executor.resources` in configmap instead
	ExecutorResources *apiv1.ResourceRequirements `json:"executorResources,omitempty"`

	// kube client实例配置
	KubeConfig *KubeConfig `json:"kubeConfig,omitempty"`

	// ContainerRuntimeExecutor specifies the container runtime interface to use, default is docker
	ContainerRuntimeExecutor string `json:"containerRuntimeExecutor,omitempty"`

	// kubelet监听端口
	KubeletPort int `json:"kubeletPort,omitempty"`

	// KubeletInsecure disable the TLS verification of the kubelet containerRuntimeExecutor, default to false
	KubeletInsecure bool `json:"kubeletInsecure,omitempty"`

	// 制品仓库
	ArtifactRepository ArtifactRepository `json:"artifactRepository,omitempty"`

	// controller工作ns
	Namespace string `json:"namespace,omitempty"`

    // 多个workflow controller 实例标记
	InstanceID string `json:"instanceID,omitempty"`

	MetricsConfig metrics.PrometheusConfig `json:"metricsConfig,omitempty"`

	TelemetryConfig metrics.PrometheusConfig `json:"telemetryConfig,omitempty"`

	// 最大并行workflow数量
	Parallelism int `json:"parallelism,omitempty"`

	// Persistence contains the workflow persistence DB configuration
	Persistence *PersistConfig `json:"persistence,omitempty"`

	// 自定义docker socket路径
	DockerSockPath string `json:"dockerSockPath,omitempty"`
}
```

## Controller 主要逻辑

> workflow/controller/controller.go

WorkflowController结构体:
```go
type WorkflowController struct {
	// namespace of the workflow controller
	namespace string
	// configMap is the name of the config map in which to derive configuration of the controller from
	configMap string
	// Config is the workflow controller's configuration
	Config config.WorkflowControllerConfig

	// cliExecutorImage is the executor image as specified from the command line
	cliExecutorImage string

	// cliExecutorImagePullPolicy is the executor imagePullPolicy as specified from the command line
	cliExecutorImagePullPolicy string

	// restConfig is used by controller to send a SIGUSR1 to the wait sidecar using remotecommand.NewSPDYExecutor().
	restConfig    *rest.Config
	kubeclientset kubernetes.Interface
	wfclientset   wfclientset.Interface

	// datastructures to support the processing of workflows and workflow pods
	wfInformer     cache.SharedIndexInformer
	wftmplInformer wfextvv1alpha1.WorkflowTemplateInformer
	podInformer    cache.SharedIndexInformer
	wfQueue        workqueue.RateLimitingInterface
	podQueue       workqueue.RateLimitingInterface
	completedPods  chan string
	gcPods         chan string // pods to be deleted depend on GC strategy
	throttler      Throttler
	wfDBctx        sqldb.DBRepository
}
```

