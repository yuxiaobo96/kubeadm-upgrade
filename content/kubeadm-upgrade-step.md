

## kubeadm跨版本升级集群

在进行升级之前需要了解各版本之间的关系：

1. kubernetes版本的命名方式用 XYZ 来表示，其中X为主要版本，Y表示次要版本，Z表示补丁版本，比如 v1.16.3
2. kubeadm的更新是不支持跨多个主版本的。只能在相邻主版本之间升级或者同一主版本的相邻次要版本间升级。如：将 k8s 集群从1.16.x版本升级到1.17.x版本，以及从版本1.17.x升级到1.17.y版本，其中 y>x。
3. 在升级k8s集群时，最先升级的核心组件是kube-apiserver，k8s中的其他组件（kube-scheduler，kube-controller-manager等）的版本号不得高于kube-apiserver；这些组件的版本号可低于kube-apiserver的1个次要版本，比如kube-apiserver是v1.16.3，其他的组件可以为1.16.x和1.15.x。但建议其他的组件与kube-apiserver的版本号完全一致。

在升级过程中主要分为三个步骤：

1. 升级master节点（主控制平面节点）
2. 升级其他的控制平面节点
3. 升级node节点（工作节点）

具体的升级流程：

1. 先确定要升级到kubeadm的哪个版本
2. 升级主控制平面节点的组件（master节点）
3. 升级主控制平面节点上的kubelet和kubectl
4. 升级其他的控制平面节点
5. 升级node节点（工作节点）
6. 验证集群是否更新正确

注意：

- kubeadm upgrade 只会更新k8s内部的组件，并不会影响您的工作负载。但建议备份etcd数据库。
- 在节点上禁用Swap（如果不禁用，重启kubelet服务时会报错）
- 升级后，因为容器的 spec 哈希值已更改，所有的容器都会重新启动
- 集群控制平面应使用pod和etcd pod或外部pod

使用`kubeadm upgrade -h`查看集群升级命名帮助

```go
$ kubeadm upgrade -h
Upgrade your cluster smoothly to a newer version with this command

Usage:
  kubeadm upgrade [flags]
  kubeadm upgrade [command]

Available Commands:
  apply       Upgrade your Kubernetes cluster to the specified version
  diff        Show what differences would be applied to existing static pod manifests. See also: kubeadm upgrade apply --dry-run
  node        Upgrade commands for a node in the cluster
  plan        Check which versions are available to upgrade to and validate whether your current cluster is upgradeable. To skip the internet check, pass in the optional [version] parameter

Flags:
  -h, --help   help for upgrade

Global Flags:
      --add-dir-header           If true, adds the file directory to the header
      --log-file string          If non-empty, use this log file
      --log-file-max-size uint   Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0, the maximum file size is unlimited. (default 1800)
      --rootfs string            [EXPERIMENTAL] The path to the 'real' host root filesystem.
      --skip-headers             If true, avoid header prefixes in the log messages
      --skip-log-headers         If true, avoid headers when opening log files
  -v, --v Level                  number for the log level verbosity

Use "kubeadm upgrade [command] --help" for more information about a command.

```

其中的命令解析：

- 第9行，apply：升级kubernetes集群到指定的版本
- 第10行，diff：显示即将应用的静态pod文件清单和正在运行的静态pod清单文件不同
- 第11行，node：升级集群中的node节点
- 第12行，plan：验证当前的集群知否可升级，并检查可升级到哪些版本。要跳过互联网检查，需传递可选的[version]参数

除此之外，可以使用`kubeadm upgrade [command] --help`获取子命令的详细信息。

当前使用的操作环境：

- 系统：Ubuntu18.04
- kubernetes集群(v1.16.3)：一master，一node

## 执行k8s集群升级流程(示例从v1.16.3升级到v1.17.2)

### 升级控制平面节点（master节点）

1. 使用命令`kubeadm config view `查看集群的配置信息，如下：

```shell
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.16.3
networking:
  dnsDomain: cluster.local
  podSubnet: 10.0.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

此处的dns使用的类型是CoreDNS（在 v1.11.0 版本后就默认使用 coredns）；

当前的k8s版本为v1.16.3。

2. 查找可升级的版本

```shell
$ apt update
$ apt-cache policy kubeadm
kubeadm:
  已安装：1.16.3-00
  候选： 1.17.2-00
  版本列表：
     1.17.2-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
     1.17.1-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
...
```

可升级为最新的1.17.2-00版本

3. 在master节点上将kubeadm升级到1.17.2-00：

```shell
$ apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.17.2-00 && \
apt-mark hold kubeadm
```

4. 验证kubeadm是否已升级到预期版本：

```shell
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2", GitCommit:"59603c6e503c87169aea6106f57b9f242f64df89", GitTreeState
...
```

5. drain 控制平面节点，将控制平面节点调整为不可调度

```shell
#  <cp-node-name> 代表的是您的控制平面节点的名称
$ kubectl drain <cp-node-name> --ignore-daemonsets
```

6. 运行升级计划命令：检测集群是否可以升级，及获取到升级的版本

```shell
$ sudo kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.16.3
[upgrade/versions] kubeadm version: v1.17.2
W0209 20:53:48.456912   22601 version.go:101] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable.txt": Get https://dl.k8s.io/release/stable.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0209 20:53:48.456968   22601 version.go:102] falling back to the local client version: v1.17.2
[upgrade/versions] Latest stable version: v1.17.2
W0209 20:53:58.539873   22601 version.go:101] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.16.txt": Get https://dl.k8s.io/release/stable-1.16.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0209 20:53:58.539941   22601 version.go:102] falling back to the local client version: v1.17.2
[upgrade/versions] Latest version in the v1.16 series: v1.17.2

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     2 x v1.16.3   v1.17.2

Upgrade to the latest version in the v1.16 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.16.3   v1.17.2
Controller Manager   v1.16.3   v1.17.2
Scheduler            v1.16.3   v1.17.2
Kube Proxy           v1.16.3   v1.17.2
CoreDNS              1.6.2     1.6.5
Etcd                 3.3.15    3.4.3-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.17.2

_____________________________________________________________________

```

7. 升级控制平面各组件，包括etcd(此处要拉取各组件的镜像及相关证书的更新，需要科学上网)

```shell
$ sudo kubeadm upgrade apply v1.17.2
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/version] You have chosen to change the cluster version to "v1.17.2"
[upgrade/versions] Cluster version: v1.16.3
[upgrade/versions] kubeadm version: v1.17.2
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Will prepull images for components [kube-apiserver kube-controller-manager kube-scheduler etcd]
[upgrade/prepull] Prepulling image for component etcd.
[upgrade/prepull] Prepulling image for component kube-apiserver.
[upgrade/prepull] Prepulling image for component kube-controller-manager.
[upgrade/prepull] Prepulling image for component kube-scheduler.
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-apiserver
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-controller-manager
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-etcd
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-etcd
[upgrade/prepull] Prepulled image for component kube-controller-manager.
[upgrade/prepull] Prepulled image for component etcd.
[upgrade/prepull] Prepulled image for component kube-apiserver.
[upgrade/prepull] Prepulled image for component kube-scheduler.
[upgrade/prepull] Successfully prepulled the images for all the control plane components
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.17.2"...
Static pod: kube-apiserver-yuxiaobo-master hash: 0dd2684decdad0c71291df8a0eab9d9f
Static pod: kube-controller-manager-yuxiaobo-master hash: ddf4d7dd458032b91c6abc5f65ef3eb3
Static pod: kube-scheduler-yuxiaobo-master hash: 4e1bd6e5b41d60d131353157588ab020
[upgrade/etcd] Upgrading to TLS for etcd
Static pod: etcd-yuxiaobo-master hash: c1a31e9fc74a43fc862192d921baf512
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-02-10-10-48-49/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: etcd-yuxiaobo-master hash: c1a31e9fc74a43fc862192d921baf512
Static pod: etcd-yuxiaobo-master hash: d5a03d2c703bca8f245eae11740b6419
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests049444808"
W0210 10:49:13.185153   23047 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-02-10-10-48-49/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-apiserver-yuxiaobo-master hash: 0dd2684decdad0c71291df8a0eab9d9f
Static pod: kube-apiserver-yuxiaobo-master hash: 2e3508621802e44482ef38c60d2f1101
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-02-10-10-48-49/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-controller-manager-yuxiaobo-master hash: ddf4d7dd458032b91c6abc5f65ef3eb3
Static pod: kube-controller-manager-yuxiaobo-master hash: 104ca85f84ee2ddc289732d63c86740a
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-02-10-10-48-49/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-scheduler-yuxiaobo-master hash: 4e1bd6e5b41d60d131353157588ab020
Static pod: kube-scheduler-yuxiaobo-master hash: 9c994ea62a2d8d6f1bb7498f10aa6fcf
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons]: Migrating CoreDNS Corefile
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.17.2". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

根据上面输出的最后两行，升级集群到v1.17.2成功

`kubeadm upgrade apply`执行了如下操作：

- 检测集群是否可以升级：
  API Service是否可用、
  所有的Node节点是不是处于Ready、
  控制平面是否healthy。
- 强制实施版本的skew policies。
- 确保控制平面镜像可用且拉取到机器上。
- 通过更新/etc/kubernetes/manifests下清单文件来升级控制平面组件，如果升级失败，则将清单文件还原。
- 应用新的kube-dns和kube-proxy配置清单文件，及创建相关的RBAC规则。
- 为API Server创建新的证书和key，并把旧的备份一份(如果它们将在180天后过期)。

8. 运行完之后，验证集群版本：

```shell
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.3", GitCommit:"b3cbbae08ec52a7fc73d334838e18d17e8512749", GitTreeState:"clean", BuildDate:"2019-11-13T11:23:11Z", GoVersion:"go1.12.12", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2", GitCommit:"59603c6e503c87169aea6106f57b9f242f64df89", GitTreeState:"clean", BuildDate:"2020-01-18T23:22:30Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"linux/amd64"}
```

此处可以发现服务端的版本已经升级为v1.17.2。

查看控制平面节点上的组件状态：

```shell
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

9. 检查CNI提供的程序插件是否需要升级

示例中使用的网络插件是[calico](https://docs.projectcalico.org/introduction/)。kubec

```shell
$ kubectl get pods -n kube-system
...
$ kubectl describe pod calico-node-jcbfl -n kube-system
...
    Image:         calico/cni:v3.10.1
...
```

当前使用的calico镜像是v3.10.1版本的，最新版本是v3.12.0，有需要可根据[官网](https://docs.projectcalico.org/release-notes/)进行升级。

不同的容器网络接口（CNI）提供程序的升级说明也不同。检查[插件](https://kubernetes.io/docs/concepts/cluster-administration/addons/)页面以找到您的CNI提供程序，并查看是否需要其他升级步骤。如果CNI提供程序作为DaemonSet运行，则在其他控制平面节点上不需要此步骤。

10. Uncordon 控制平面节点，恢复控制平面节点

```shell
# <cp-node-name> 代表的是您的控制平面节点的名称
$ kubectl uncordon <cp-node-name>
```

11. 升级该控制平面上的kubelet和kubectl（root权限）

```shell
# replace x in 1.17.x-00 with the latest patch version
$ apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.17.2-00 kubectl=1.17.2-00 && \
apt-mark hold kubelet kubectl
```

12. 重启kubelet

```shell
$ sudo systemctl restart kubelet
```

13. 查看控制平面节点（master）是否升级成功

```shell
$ kubectl get nodes
NAME              STATUS   ROLES    AGE   VERSION
k8s-node1         Ready    <none>   67d   v1.16.3
yuxiaobo-master   Ready    master   68d   v1.17.2
```

到此为止，主控制平面节点的升级已完成

### 升级其他控制平面节点

1. 与在主控制平面节点的操作相同，但使用以下命令进行升级

```shell
$ sudo kubeadm upgrade node experimental-control-plane
```

`kubeadm upgrade node`在其他控制平面节点上做以下工作：

- 从集群中获取kubeadm 的`ClusterConfiguration`
- 备份kube-apiserver证书（可选）。
- 升级控制平面核心组件的静态Pod清单文件

2. 在其他的控制平面节点上也需要升级kubelet和kubectl（root权限）

```shell
# replace x in 1.17.x-00 with the latest patch version
$ apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.17.2-00 kubectl=1.17.2-00 && \
apt-mark hold kubelet kubectl
```

3. 重启kubelet

```shell
$ sudo systemctl restart kubelet
```

### 升级工作节点

在不牺牲运行工作负载所需的最小容量的前提下，工作节点上的升级过程应该一次执行一个节点，或者一次执行几个节点。

1. 在所有工作节点上升级kubeadm（root权限）

```shell
# replace x in 1.17.x-00 with the latest patch version
$ apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.17.2-00 && \
apt-mark hold kubeadm
```

2. 将工作节点标记为不可用，使其处于维护状态

```shell
# <node-to-drain> 代表的是drain的工作节点，此命令需要在master节点上执行
$ kubectl drain <node-to-drain> --ignore-daemonsetskube
```

3. 升级kubelet配置文件

```shell
# 此命令需要在master节点上执行
$ sudo kubeadm upgrade node config --kubelet-version v1.17.2
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
...
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

`kubeadm upgrade node`在工作节点中执行以下操作

- 从集群中获取kubeadm的`ClusterConfiguration`
- 升级工作节点的kubelet配置

4. 在所有工作节点上升级kubelet和kubectl(root权限)

```shell
# replace x in 1.17.x-00 with the latest patch version
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.17.2-00 kubectl=1.17.2-00 && \
apt-mark hold kubelet kubectl
```

5. 重启kubelet

```shell
$ sudo systemctl restart kubelet
```

6. 升级后，将此工作节点标记为可调度来使其重新加入集群

```shell
# <node-to-drain> 代表的是工作节点的名称，此命令需要在master节点上执行
$ kubectl uncordon <node-to-drain>
```

### 验证集群的状态

检查各个节点的状态，是否处于正常的状态（Ready）

```shell
$ kubectl get nodes
NAME              STATUS   ROLES    AGE   VERSION
k8s-node1         Ready    <none>   67d   v1.17.2
yuxiaobo-master   Ready    master   68d   v1.17.2
```

若所有节点均显示Ready，并且版本号已升级为v1.17.2版本，即整个集群升级完毕。

### 从故障状态中恢复

如果`kubeadm upgrade`失败并且不回滚（例如由于执行期间意外关闭），则可以`kubeadm upgrade`再次运行。此命令是幂等的，并最终确保实际状态是您声明的所需状态。

要从不良状态中恢复，您也可以在`kubeadm upgrade apply --force`不更改集群运行版本的情况下运行。

在升级期间，kubeadm在以下位置写入备份文件夹`/etc/kubernetes/tmp`: - `kubeadm-backup-etcd-<date>-<time>` - `kubeadm-backup-manifests-<date>-<time>`

`kubeadm-backup-etcd`包含此控制平面节点的本地etcd成员数据的备份。如果etcd升级失败并且自动回滚不起作用，则可以在中手动还原此文件夹的内容`/var/lib/etcd`。如果使用外部etcd，则此备份文件夹将为空。

`kubeadm-backup-manifests`包含此控制平面节点的静态Pod清单文件的备份。如果升级失败并且自动回滚不起作用，则可以手动还原此文件夹的内容`/etc/kubernetes/manifests`。如果某个组件的升级前清单文件和升级后清单文件之间没有区别，则不会写入该文件的备份文件。

## 问题总结

1. 镜像问题：

- 集群中各组件升级时需要grc.io的镜像，此时需要科学上网

- 如果是客户现场使用，没有外网的环境，需要我们提供现成的镜像

2. 在进行版本升级时，需要指定个各组件要升级的版本，使各个组件的版本保持一致性。

在低版本之间升级时（从v1.13.x升级到v1.14.x）ubuntu系统有可能会对组件进行自动升级，直接升级到当前的最新版本（示例是从v1.16.3升级到最新版本v1.17.2，不受影响）。如果发生这种情况，导致kubeadm和kubelet的版本不一致，可以先删除当前的kubeadm和kubelet，再次安装将要升级的组件对应版本，然后继续后面的升级步骤。
