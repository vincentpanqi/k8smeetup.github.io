---
cn-approvers:
- tianshapjq
title: kubeadm 故障排查
---


{% capture overview %}


在任何程序中，您都有可能遇到使用或操作上的错误。下面我们列出常见的失败情况，并提供了一些步骤，以帮助您理解并希望能解决这些问题。


如果您的问题未在下面列出，请按照以下步骤操作：


- 如果您认为您的问题是一个和 kubeadm 相关的错误：
  - 到 [github.com/kubernetes/kubeadm](https://github.com/kubernetes/kubeadm/issues) 查找目前已存在的 issue。
  - 如果没有相关的 issue，请 [创建一个](https://github.com/kubernetes/kubeadm/issues/new) 并遵照 issue 的模板进行填写。


- 如果您不确定 kubeadm 或者 kubernetes 是如何工作的，并且希望能够得到关于您问题的帮助，请在 Slack 的 #kubeadm 频道上提问，或者在 StackOverflow 上创建一个问题。请在问题中包含类似 `#kubernetes` 或者 `#kubeadm` 等相关标签以便其他人能够给你提供帮助。


如果您的集群正处于错误状态，并且看到 Pod 的状态类似于 `RunContainerError`、`CrashLoopBackOff` 或者 `Error`，那么您很可能在配置上出现了问题。如果是这种情况，请参阅下文。

{% endcapture %}


#### 在安装的时候提示找不到 `ebtables` 或者 `ethtool`


如果您在运行 `kubeadm init` 的时候看到如下警告

```
[preflight] WARNING: ebtables not found in system path                          
[preflight] WARNING: ethtool not found in system path                           
```


那么可能是在您的 Linux 机器上缺少了 ebtables 和 ethtool。您可以通过以下命令进行安装：

```
# For ubuntu/debian users, try 
apt install ebtables ethtool

# For CentOS/Fedora users, try 
yum install ebtables ethtool
```


#### Pod 处于 `RunContainerError`、`CrashLoopBackOff` 或者 `Error` 状态


`kubeadm init` 结束之后是不应该有这样状态的 Pod 的。如果在 `kubeadm init` _结束之后_ 出现这样的 Pod，那么请在 kubeadm 库上创建一个 issue 描述此问题。在您部署网络解决方案之前，`kube-dns` 应该处于 `Pending` 状态。但是，如果在部署网络解决方案后 Pod 仍然处于 `RunContainerError`、`CrashLoopBackOff` 或者 `Error` 状态，并且 `kube-dns` 状态没有任何变化，那么很可能是您安装的 Pod 网络解决方案出了问题。您可能需要赋予它更多的 RBAC 权限或者使用一个更新的版本。请您在 Pod 网络提供商的 issue 跟踪处创建一个 issue 并对它进行归类。


#### `kube-dns` 阻塞在 `Pending` 状态


这是 **期望** 的状态并且也是设计的一部分。kubeadm 并不提供网络解决方案，所以管理员需要 [安装 pod 网络解决方案](/docs/concepts/cluster-administration/addons/)。您需要在 `kube-dns` 完全部署之前安装一个 pod 网络。因此 `kube-dns` 在网络建立之前是处于 `Pending` 状态的。


#### `HostPort` 服务不可用


`HostPort` 和 `HostIP` 功能是否可用取决于您的 Pod 网络提供商。请联系该 Pod 网络解决方案的作者以确定 `HostPort` 和 `HostIP` 功能是否可用。


已经过验证的 HostPort CNI 提供商：
- Calico
- Canal
- Flannel


如果需要更多信息，请参阅 [CNI portmap documentation](https://github.com/containernetworking/plugins/blob/master/plugins/meta/portmap/README.md).


如果您的网络提供商不支持 portmap CNI 插件，那么您可能需要使用 [NodePort feature of
services](/docs/concepts/services-networking/service/#type-nodeport) 或者使用 `HostNetwork=true`。


#### 通过 Service IP 无法访问 Pod


目前一些网络组件还没开启 [hairpin 模式](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#a-pod-cannot-reach-itself-via-service-ip)，该模式能够让 pod 在不知道 podIP 的情况下通过 Service IP 访问到它们自身。这是一个和 [CNI](https://github.com/containernetworking/cni/issues/476) 相关的 issue。请联系网络组件的提供商，及时获得它们是否支持 hairpin 模式的信息。


如果您正在使用 VirtualBox （直接使用或通过 Vagrant），您需要确定 `hostname -i` 返回的是一个可路由的 IP 地址 （即第二个网络接口的地址，而不是第一个）。默认情况下，VirtualBox 并不这样做，从而使得 kubelet 因为使用第一个非回环的网络接口地址而停止，该地址通常是可 NAT 的。解决方法：修改 `/etc/hosts` 文件，可参阅 `Vagrantfile`[ubuntu-vagrantfile](https://github.com/errordeveloper/k8s-playground/blob/22dd39dfc06111235620e6c4404a96ae146f26fd/Vagrantfile#L11) 查看修改方法。


#### TLS 证书错误


以下错误表明证书可能不匹配

```
# kubectl get po
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```


验证 `$HOME/.kube/config` 文件是否包含有效证书，并在必要时重新生成证书。另外一个解决方法是覆盖 "admin" 用户的默认 `kubeconfig` 文件：

```
mv  $HOME/.kube $HOME/.kube.bak
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


#### 在 CentOS 上建立 master 时出错


如果您正在使用 CentOS 并且在建立 master 节点时遇到问题，请确认您的 Docker cgroup 驱动匹配 kubelet 的配置：

```bash
docker info | grep -i cgroup
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```


如果 Docker cgroup 驱动和 kubelet 的配置不匹配，那么需要修改 kubelet 的配置使其能够匹配 Docker cgroup 驱动。您需要设置 `--cgroup-driver` 参数。如果参数已经被设置了，您可以通过以下方式更新：

```bash
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```


否则，您需要打开 systemd 文件然后添加该参数到现有的环境变量中。


然后重启 kubelet：

```bash
systemctl daemon-reload
systemctl restart kubelet
```


`kubectl describe pod` 或者 `kubectl logs` 命令能够帮助你分析错误。例如：

```bash
kubectl -n ${NAMESPACE} describe pod ${POD_NAME}

kubectl -n ${NAMESPACE} logs ${POD_NAME} -c ${CONTAINER_NAME}
```