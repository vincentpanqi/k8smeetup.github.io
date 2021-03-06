---
title: 通过文件暴露 Pod 信息给容器
cn-approvers:
- pigletfly
---


{% capture overview %}


本文展示了如何使用 DownwardAPIVolumeFile 暴露 Pod 自身信息给运行在这个 Pod 中的容器。DownwardAPIVolumeFile 可以暴露 Pod 的字段和 Container 字段。

{% endcapture %}


{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}


## Downward API

有两种方法可以暴露 Pod 和 Container 字段给一个运行的容器：

* [环境变量](/docs/tasks/configure-pod-container/environment-variable-expose-pod-information/)
* DownwardAPIVolumeFiles

这两种暴露 Pod 和 Container 字段的方法被称为 *Downward API* 。


## 存储 Pod 字段

在本练习中，您创建了一个 Pod，里面运行了一个容器。
下面是这个 Pod 的配置文件：

{% include code.html language="yaml" file="dapi-volume.yaml" ghlink="/docs/tasks/inject-data-application/dapi-volume.yaml" %}


在这个配置文件中，您可以看到这个 Pod 有一个 `downwardAPI` 卷，容器把卷挂载在 `/etc` 。


看下在 `downwardAPI` 下的 `items` 数组。这个数组中的每个元素是一个 [DownwardAPIVolumeFile](/docs/resources-reference/{{page.version}}/#downwardapivolumefile-v1-core)。
第一个元素指定了 Pod 的 `metadata.labels` 字段的值应该存储在一个名字为 `labels` 的文件中。
第二个元素指定了 Pod 的 `annotations` 字段的值应该存储在一个名字为 `annotations` 的文件中。


**注意:**  在本例中的字段是 Pod 的字段。它们不是 Pod 中 Container 的字段。
{: .note}


创建 Pod：

```shell
kubectl create -f https://k8s.io/docs/tasks/inject-data-application/dapi-volume.yaml
```


验证 Pod 中的容器是否运行：

```shell
kubectl get pods
```


查看容器的日志：

```shell
kubectl logs kubernetes-downwardapi-volume-example
```


输出展示了 `labels` 和 `annotations` 的文件内容：

```shell
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"

build="two"
builder="john-doe"
```


获取您的 Pod 中运行的容器的 shell：

```
kubectl exec -it kubernetes-downwardapi-volume-example -- sh
```


在您的 shell 中，查看 `labels` 文件：

```shell
/# cat /etc/labels
```


输出展示了这个 Pod 的全部 labels 都被写入了  `labels` 文件：

```shell
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```


相似的，查看 `annotations` 文件：

```shell
/# cat /etc/annotations
```


查看 `/etc` 目录下的文件：

```shell
/# ls -laR /etc
```


在这个输出中，您可以看到 `labels` 和 `annotations` 文件是在一个临时的子目录中：在本例中，是 `..2982_06_02_21_47_53.299460680` 目录。
在 `/etc` 目录中, `..data` 是链接到临时子目录的符号链接。在 `/etc` 目录中, `labels` 和 `annotations` 也是符号链接。


```
drwxr-xr-x  ... Feb 6 21:47 ..2982_06_02_21_47_53.299460680
lrwxrwxrwx  ... Feb 6 21:47 ..data -> ..2982_06_02_21_47_53.299460680
lrwxrwxrwx  ... Feb 6 21:47 annotations -> ..data/annotations
lrwxrwxrwx  ... Feb 6 21:47 labels -> ..data/labels

/etc/..2982_06_02_21_47_53.299460680:
total 8
-rw-r--r--  ... Feb  6 21:47 annotations
-rw-r--r--  ... Feb  6 21:47 labels
```


使用符号链接可以实现元数据的动态原子刷新；更新被写入了一个临时目录，并且 `..data` 符号链接通过使用 [rename(2)](http://man7.org/linux/man-pages/man2/rename.2.html) 自动更新。


退出 shell：

```shell
/# exit
```


## 存储 Container 字段

在前面的练习中，您将 Pod 字段存储在 DownwardAPIVolumeFile 中。
在下一个练习中，您将存储 Container 字段。这里是包含一个容器的 Pod 的配置文件：

{% include code.html language="yaml" file="dapi-volume-resources.yaml" ghlink="/docs/tasks/inject-data-application/dapi-volume-resources.yaml" %}


在这个配置文件中，您可以看到这个 Pod 有一个 `downwardAPI` 卷，容器把卷挂载在 `/etc` 。


看下在 `downwardAPI` 下的 `items` 数组。这个数组中的每个元素是一个 [DownwardAPIVolumeFile](/docs/resources-reference/{{page.version}}/#downwardapivolumefile-v1-core)。

第一个元素指定了名字为 `client-container` 容器的 `limits.cpu` 字段应该存储在名为 `cpu_limit` 的文件中。


创建 Pod：

```shell
kubectl create -f https://k8s.io/docs/tasks/inject-data-application/dapi-volume-resources.yaml
```


获取您的 Pod 中运行的容器的 shell：

```
kubectl exec -it kubernetes-downwardapi-volume-example-2 -- sh
```


在您的 shell 中，查看 `cpu_limit` 文件：

```shell
/# cat /etc/cpu_limit
```

您可以使用相似的命令查看 `cpu_request`、`mem_limit` 和 `mem_request` 文件。

{% endcapture %}

{% capture discussion %}


## Downward API 的能力


以下信息可以通过环境变量和 DownwardAPIVolumeFiles 提供给容器：


* Node 名称
* Node IP
* Pod 名称
* Pod namespace
* Pod IP 地址
* Pod 的 serviceaccount 名称
* Pod 的 UID
* 容器的 CPU limit
* 容器的 CPU request
* 容器的内存 limit
* 容器的内存 request


另外，通过 DownwardAPIVolumeFiles 还可以提供以下信息：


* Pod 的 labels
* Pod 的 annotations


**注意:** 如果 CPU 和 内存 limits 没有指定，Downward API 的默认值是 Node 可分配的 CPU 和内存。
{: .note}

## Project keys to specific paths and file permissions

You can project keys to specific paths and specific permissions on a per-file
basis. For more information, see
[Secrets](/docs/concepts/configuration/secret/).


## Downward API 的动机


对于容器来说，有时在不和 Kubernetes 过度耦合的情况下获得关于它自己的信息非常有用。Downward API 允许容器获取有关自己或集群的信息，而不使用 Kubernetes 客户端
或 API server。


一个例子是假定一个已经存在的应用程序拥有一个特别的知名环境变量，这个环境变量有唯一的标识符。一种可能性是包装应用程序，
但是很乏味而且容易出错，它违反了低耦合的目标。一个更好的选择是，使用 Pod 名称作为标识符，然后将 Pod 名称转入到这环境变量中。



{% endcapture %}


{% capture whatsnext %}

* [PodSpec](/docs/resources-reference/{{page.version}}/#podspec-v1-core)
* [Volume](/docs/resources-reference/{{page.version}}/#volume-v1-core)
* [DownwardAPIVolumeSource](/docs/resources-reference/{{page.version}}/#downwardapivolumesource-v1-core)
* [DownwardAPIVolumeFile](/docs/resources-reference/{{page.version}}/#downwardapivolumefile-v1-core)
* [ResourceFieldSelector](/docs/resources-reference/{{page.version}}/#resourcefieldselector-v1-core)

{% endcapture %}

{% include templates/task.md %}

