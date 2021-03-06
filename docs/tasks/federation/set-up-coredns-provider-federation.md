---
cn-approvers:
- lichuqiang
title: 设置 CoreDNS 作为集群联邦的 DNS 提供商
---


{% capture overview %}


本文展示了如何配置并部署 CoreDNS 作为集群联邦的 DNS 提供商。

{% endcapture %}


{% capture objectives %}


* 配置并部署 CoreDNS 服务器
* 以 CoreDNS 作为 dns 提供商创建联邦
* 在名称服务器查找链中配置 CoreDNS 服务器

{% endcapture %}


{% capture prerequisites %}


* 您需要拥有一个运行的 Kubernetes 集群 （它被引用为 host 集群）。
请查看 [入门](/docs/setup/) 指导，根据您的平台选择其中一种安装说明。

* 为启用 `CoreDNS` 来实现跨联邦集群的服务发现，联邦的成员集群中必须支持 `LoadBalancer` 服务。

{% endcapture %}


{% capture lessoncontent %}


## 部署 CoreDNS 和 etcd charts

我们可以按照不同的配置部署 CoreDNS。 下面是一种参考，您可以对其进行调整，以适应平台和集群联邦的需求：


我们可以利用 helm charts 来部署 CoreDNS。 CoreDNS 部署时会以 [etcd](https://coreos.com/etcd)
作为后端，并且 etcd 应预先安装。 etcd 也可以利用 helm charts 进行部署。
下面展示了部署 etcd 的指导。

    helm install --namespace my-namespace --name etcd-operator stable/etcd-operator
    helm upgrade --namespace my-namespace --set cluster.enabled=true etcd-operator stable/etcd-operator


*注意：可以覆盖默认的部署配置，来适应 host 集群。*


部署成功后，可以在 host 集群内通过 [http://etcd-cluster.my-namespace:2379](http://etcd-cluster.my-namespace:2379) 端点访问 etcd。


您需要定制 CoreDNS 默认配置，以适应联邦。
下面展示的是 Values.yaml，它覆盖了 CoreDNS chart 的默认配置参数。

{% include code.html language="yaml" file="Values.yaml" ghlink="/docs/tasks/federation/Values.yaml" %}


上面的配置文件需要一些说明：


 - `isClusterService` 指定 CoreDNS 是否以集群服务的形式部署（默认为是）。
您需要将其设置为 "false"，以使 CoreDNS 以 Kubernetes 应用服务的形式部署。

 - `serviceType` 指定为 CoreDNS 创建的 Kubernetes 服务类型。
您需要选择 "LoadBalancer" 或 "NodePort"，以使得 CoreDNS 服务能够从 Kubernetes 集群外部访问。

 - `middleware.kubernetes` 默认是启用的，通过将 `middleware.kubernetes.enabled`
设置为 "false" 来禁用 `middleware.kubernetes`。

 - 通过将 `middleware.etcd.enabled` 设置为 "true" 来启用 `middleware.etcd`。

 - 按照上面展示的，通过设置 `middleware.etcd.zones` 来配置 CoreDNS 被授权的 DNS 区域（联邦区域）。

 - 通过设置 `middleware.etcd.endpoint` 来设置先前部署的 etcd 的端点。


现在通过运行以下命令来部署 CoreDNS：

    helm install --namespace my-namespace --name coredns -f Values.yaml stable/coredns


验证 etcd 和 CoreDNS pod 按照预期在运行。



## 使用 CoreDNS 作为 DNS 提供商来部署联邦

可以使用 `kubefed init` 来部署联邦控制平面。 可以通过指定两个附加参数来选择 CoreDNS
作为 DNS 提供商。

    --dns-provider=coredns
    --dns-provider-config=coredns-provider.conf


coredns-provider.conf 格式如下：

    [Global]
    etcd-endpoints = http://etcd-cluster.my-namespace:2379
    zones = example.com.
    coredns-endpoints = <coredns-server-ip>:<port>


 - `etcd-endpoints` 是访问 etcd 的端点。

 - `zones` 是 CoreDNS 被授权的联邦区域，其值与 `kubefed init` 的 --dns-zone-name 参数相同。

 - `coredns-endpoints` 是访问 CoreDNS 服务器的端点。 这是一个 1.7 版本开始引入的可选参数。


*注意：CoreDNS 配置中的 middleware.etcd.zones 与 kubefed init 的 --dns-zone-name 参数应匹配。*



## 在名称服务器 resolv.conf 链中设置 CoreDNS 服务器

*注意：以下章节只适应于 1.7 以前的版本，且如上面章节所描述的，如果在 `coredns-provider.conf`
中配置了 `coredns-endpoints` 参数，会自动将 CoreDNS 服务器配置到名称服务器 resolv.conf 链中。*


由于这种自主的 CoreDNS 服务器不能够公开发现，一旦联邦控制平面部署完成，且要联合的集群加入了联邦，
您需要将 CoreDNS 服务器加入到所有联邦集群的 pod 名称服务器 resolv.conf 链中。
这可以通过将下面的内容加入到 `kube-dns` deployment 中的 `dnsmasq` 容器参数来实现。


    --server=/example.com./<CoreDNS 端点>


用联邦的域替换上面的 `example.com` 。



现在联邦集群已经准备好跨集群服务发现了！

{% endcapture %}

{% include templates/tutorial.md %}
