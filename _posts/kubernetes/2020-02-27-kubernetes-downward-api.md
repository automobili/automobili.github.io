---
title: "Kubernetes Downward Api"
tags: 
  - "kubernetes"
published: true
---

## 介绍

想要容器可以知道一些自己的信息，但又不需要跟k8s过度耦合（也就是不希望在容器中调用k8s的api）。

有两种方式可以将Pod和Container的信息暴漏给运行中的容器。

* Environment variables
* DownwardAPIVolumeFiles

他们叫做 downward API。

-----

#### Environment variables

通过Downward API来将 POD 的 IP、名称以及所对应的 namespace 注入到容器的环境变量中去，然后我们在容器中打印全部的环境变量来进行验证，对应的yaml文件如下:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: test-env-pod
    namespace: kube-system
spec:
    containers:
    - name: test-env-pod
      image: busybox:latest
      command: ["/bin/sh", "-c", "env"]
      env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
```
我们需要注意的是 POD 的 name 和 namespace 属于元数据，是在 POD 创建之前就已经定下来了的，所以我们使用 metata 获取就可以了，但是对于 POD 的 IP 则不一样，因为我们知道 POD IP 是不固定的，POD 重建了就变了，它属于状态数据，所以我们使用 status 去获取。
> 除了使用fieldRef获取 POD 的基本信息外，还可以通过resourceFieldRef去获取容器的资源请求和资源限制信息。

-----

#### DownwardAPIVolumeFiles

Downward API除了提供环境变量的方式外，还提供了通过Volume挂载的方式去获取 POD 的基本信息。接下来我们通过Downward API将 POD 的 Label、Annotation 等信息通过 Volume 挂载到容器的某个文件中去，然后在容器中打印出该文件的值来验证。

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: test-volume-pod
    namespace: kube-system
    labels:
        k8s-app: test-volume
        node-env: test
    annotations:
        build: test
        own: qikqiak
spec:
    containers:
    - name: test-volume-pod-container
      image: busybox:latest
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 3600;
        done;
      volumeMounts:
      - name: podinfo
        mountPath: /etc/podinfo
    volumes:
    - name: podinfo
      downwardAPI:
        items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
```
如果你的应用有获取 POD 的基本信息的需求，一般我们就可以利用Downward API来获取基本信息，然后编写一个启动脚本或者利用initContainer将 POD 的信息注入到我们容器中去，然后在我们自己的应用中就可以正常的处理相关逻辑了。

-----

#### Capabilities of the Downward API
下面这些信息可以通过环境变量和DownwardAPIVolumeFiles提供给容器：

能通过fieldRef获得的： 
* metadata.name - Pod名称 
* metadata.namespace - Pod名字空间 
* metadata.uid - Pod的UID, 版本要求 v1.8.0-alpha.2 
* metadata.labels['<KEY>'] - 单个 pod 标签值 <KEY> (例如, metadata.labels['mylabel']); 版本要求 Kubernetes 1.9+ 
* metadata.annotations['<KEY>'] - 单个 pod 的标注值 <KEY> (例如, metadata.annotations['myannotation']); 版本要求 Kubernetes 1.9+

能通过resourceFieldRef获得的： * 容器的CPU约束值 * 容器的CPU请求值 * 容器的内存约束值 * 容器的内存请求值 * 容器的临时存储约束值, 版本要求 v1.8.0-beta.0 * 容器的临时存储请求值, 版本要求 v1.8.0-beta.0

此外，以下信息可通过DownwardAPIVolumeFiles从fieldRef获得：

* metadata.labels - all of the pod’s labels, formatted as label-key="escaped-label-value" with one label per line
* metadata.annotations - all of the pod’s annotations, formatted as annotation-key="escaped-annotation-value" with one annotation per line
* metadata.labels - 所有Pod的标签，以label-key="escaped-label-value"格式显示，每行显示一个label
* metadata.annotations - Pod的注释，以annotation-key="escaped-annotation-value"格式显示，每行显示一个标签

以下信息可通过环境变量从fieldRef获得：

* status.podIP - 节点IP
* spec.serviceAccountName - Pod服务帐号名称, 版本要求 v1.4.0-alpha.3
* spec.nodeName - 节点名称, 版本要求 v1.4.0-alpha.3
* status.hostIP - 节点IP, 版本要求 v1.7.0-alpha.1