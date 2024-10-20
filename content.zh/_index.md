---
title: 首页
bookToc: false
---

# 二进制安装 K8s：中文版

## 目录

- 前言
- 一、基本信息和服务器准备
- 二、K8s 的证书
- 三、安装 Master 节点
- 四、安装 Worker 节点
- 五、设置基本的路由网络
- 六、运行Pod

## 前言

这是我在学习二进制安装 K8s 时候做的笔记，我装的是 1 个 Master 和 2 个 Node。安装的时候没用脚本，也没有 HA。

这个笔记适合：

- 用来培养自己对于 K8s 各个组件的感性理解
- 用来安装一个极简环境

不适合：

- 不适合用来部署生产环境
- 学习一些比较进阶的概念

如无意外，这份笔记每年会更新两次。

### 版本说明

现在这份文档主要编写于 2024 年 9 到 10 月份，下面是主要使用的软件版本。

OS: Rocky Linux 8.10

组件的版本:

| 组件       | 版本    |
| ---------- | ------- |
| Kubernetes | v1.31.1 |
| etcd       | v3.4.34 |
| containerd | v1.7.22 |

### 参考

这份笔记参考了下面的材料：

- [《Kubernetes 权威指南》](https://book.douban.com/subject/35458432/)
- [kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)


## 三、安装 Master 节点

在 Master 节点上，我们要安装 `etcd`，`kube-apiserver`，`kube-controller-manager` 和 `kube-scheduler` 四个服务。

另外，还要安装 `etcdctl` 和 `kubectl` 这两个工具，分别用来管理 etcd 和 K8s。

这些全是二进制可执行的文件，我们把他们都放在 `/usr/local/bin` 里面。

如果前面还没做过的话，现在可以在 `~/.bash_profile` 文件中把 `/usr/local/bin` 加到 PATH 里。

```bash
echo "export PATH=$PATH:/usr/local/bin" >> ~/.bash_profile
source ~/.bash_profile
```

### 下载

**etcd**

etcd 可以在这里下载：

[https://github.com/etcd-io/etcd/releases/](https://github.com/etcd-io/etcd/releases/)

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.34/etcd-v3.4.34-linux-amd64.tar.gz
```

解压出来，里面有 `etcd` 和 `etcdctl`。把他们都放在 `/usr/local/bin` 里。

**Kubernetes 的组件**

Kubernetes 的二进制文件可以在这里下载：

[https://kubernetes.io/releases/download/#binaries](https://kubernetes.io/releases/download/#binaries)

我们把 `kube-apiserver`，`kube-controller-manager`，`kube-scheduler` 和 `kubectl` 下载下来，也都放在 `/usr/local/bin` 里面。

至此，在 `/usr/local/bin` 里面一共有 8 个文件（前一个章节还有 2 个 cfssl 的工具放在这里了）。调整一下他们的权限。

```bash
chown root.root /usr/local/bin/*
chmod +x /usr/local/bin/*
```

### 配置 etcd

给 etcd 创建一个存放数据的文件夹。

```bash
mkdir /var/lib/etcd
```

创建 /etc/systemd/system/etcd.service，写下面的配置

```ini
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --name master \
--data-dir /var/lib/etcd \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://0.0.0.0:2379 \
--listen-peer-urls http://0.0.0.0:2380 \
--initial-advertise-peer-urls http://0.0.0.0:2380 \
--initial-cluster master=http://0.0.0.0:2380 \
--initial-cluster-token etcd-cluster-0 \
--initial-cluster-state new
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

上面这个配置启动了一个单节点的 etcd，开放了所有端口，并且没有设置身份认证。**不要在生产环境这样做。**

启动 etcd，并且将其设置为随机器启动。

```bash
systemctl start etcd
systemctl enable etcd
```

### 配置 kube-apiserver

创建 `/etc/systemd/system/kube-apiserver.service`，写下面的配置

```ini
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/etc/kubernetes/kube-apiserver.env
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_ARGS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

创建 `/etc/kubernetes/kube-apiserver.env`，写下面这些启动参数。

```ini
KUBE_APISERVER_ARGS="--allow-privileged=true \
--apiserver-count=1 \
--authorization-mode=Node,RBAC \
--bind-address=192.168.56.10 \
--client-ca-file=/etc/kubernetes/pki/ca.pem \
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
--etcd-servers=http://127.0.0.1:2379 \
--event-ttl=1h \
--runtime-config='api/all=true' \
--service-account-key-file=/etc/kubernetes/pki/sa-pub.pem \
--service-account-signing-key-file=/etc/kubernetes/pki/sa-key.pem \
--service-account-issuer=https://master:6443 \
--service-cluster-ip-range=10.96.0.0/24 \
--tls-cert-file=/etc/kubernetes/pki/kube-apiserver.pem \
--tls-private-key-file=/etc/kubernetes/pki/kube-apiserver-key.pem \
--v=2"
```

这些配置指定了 `kube-apiserver` 的认证方式、各种证书的位置、使用的 etcd 地址，还有 Service 网段（service-cluster-ip-range）等等信息。

启动并设置开机自动启动。

```bash
systemctl start kube-apiserver
systemctl enable kube-apiserver
```

### 配置 kubeconfig

接下来我们要准备一份 `kubeconfig`，如前所述，这个文件是打开 Kubernetes API 的钥匙。

创建 `/etc/kubernetes/admin.kubeconfig`。

```yaml
apiVersion: v1
kind: Config
clusters:
  - cluster:
      certificate-authority: /etc/kubernetes/pki/ca.pem
      server: https://master:6443
    name: kubernetes
users:
  - name: admin
    user:
      client-certificate: /etc/kubernetes/pki/client.pem
      client-key: /etc/kubernetes/pki/client-key.pem
contexts:
  - context:
      cluster: kubernetes
      user: admin
      namespace: default
    name: admin@kubernetes
current-context: admin@kubernetes
```

这里面指定了 API 的访问地址、CA 证书，以及我们做为客户端要使用的 client 证书/Key。

在 `~/.bash_profile` 里面添加上这个 `kubeconfig` 的环境变量，这样一来，我们的 `kubectl` 就知道要用这个了 config 了。

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.kubeconfig" >> ~/.bash_profile
source ~/.bash_profile
```

现在我们可以测试一下 `kubectl` 和 `kube-apiserver` 的连通性，执行下面的命令：

```bash
kubectl version
```

如果能看到 `Server Version`，则说明 `kubectl` 到 `kube-apiserver` 都通了，前面的证书生成和配置都没什么问题。

如果没有看到 `Server Version`，应该停下来，把前面的步骤再检查一下。

### 配置 kube-controller-manager

创建 `/etc/systemd/system/kube-controller-manager.service`，写下面的配置。

```ini
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/etc/kubernetes/kube-controller-manager.env
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

创建 `/etc/kubernetes/kube-controller-manager.env`，写下面这些启动参数。

```ini
KUBE_CONTROLLER_MANAGER_ARGS="--bind-address=192.168.56.10 \
--cluster-cidr=10.244.0.0/16 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
--cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
--kubeconfig=/etc/kubernetes/admin.kubeconfig \
--root-ca-file=/etc/kubernetes/pki/ca.pem \
--service-account-private-key-file=/etc/kubernetes/pki/sa-key.pem \
--service-cluster-ip-range=10.96.0.0/24 \
--use-service-account-credentials=true \
--v=2"
```

这些配置指定了集群的 Pod 的地址段、Service 的地址段，证书位置，kubeconfig 位置等等。

启动并设置开机自动启动。

```bash
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```

### 配置 kube-scheduler

创建 `/etc/systemd/system/kube-scheduler.service`。

```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/etc/kubernetes/kube-scheduler.env
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_ARGS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

创建 `/etc/kubernetes/kube-scheduler.env`。

```ini
KUBE_SCHEDULER_ARGS="--kubeconfig=/etc/kubernetes/admin.kubeconfig \
--leader-elect=true \
--v=0"
```

scheduler 基本上只要能连接到 API Server 就可以，所以这里的配置比较简单。

```bash
systemctl start kube-scheduler
systemctl enable kube-scheduler
```

接下来，我们[安装 Worker 节点](/cn/04-安装Worker节点.md)。

## 四、安装 Worker 节点

接下来我们安装 Worker 节点，主要是安装 `kubelet`，容器的 Runtime，还有准备网络工具，也就是所谓的 CNI Plugins。

这部分的操作要在两个 Node 服务器上分别执行。

Worker 的多数二进制的文件，我们也都都放在 `/usr/local/bin` 里面，并且可以在 `~/.bash_profile` 里面把 `/usr/local/bin` 加到 `PATH` 里。

```bash
mkdir /usr/local/bin
echo "export PATH=$PATH:/usr/local/bin" >> ~/.bash_profile
source ~/.bash_profile
```

### 安装容器 Runtime

在 Worker 节点上，调用的链条主要是下面这样：kubelet 指挥 containerd，containerd 指挥 runc，其中 containerd 和 runc 共同构成容器的运行环境。

```
┌───────────┐       ┌──────────────┐       ┌────────┐
│           │       │              │       │        │
│  kubelet  ┼──────►│  containerd  ┼──────►│  runc  │
│           │       │              │       │        │
└───────────┘       └──────────────┘       └────────┘
```

我们第一步安装 containerd。

> [!NOTE]
> containerd 官方提供了一个非常好的[安装说明](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)，推荐阅读。

首先下载 containerd 的文件包，把里面东西都解压到 `/usr/local/bin`。

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.22/containerd-1.7.22-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.7.22-linux-amd64.tar.gz
```

然后可以下载 containerd.service 文件，官方帮忙准备好了。

```bash
cd /etc/systemd/system/
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

创建 containerd 的配置文件 `/etc/containerd/config.toml`。

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"
  conf_dir = "/etc/cni/net.d"
```

启动 containerd，设置开机启动。

```bash
systemctl daemon-reload
systemctl start containerd
systemctl enable containerd
```

然后下载 runc，放在 `/usr/local/bin`。

```bash
cd /usr/local/bin
wget https://github.com/opencontainers/runc/releases/download/v1.1.14/runc.amd64
mv runc.amd64 runc
chmod +x runc
```

runc 不用启动，放在这里就行。

然后下载网络插件 CNI Plugins，放在 `/opt/cni/bin`，注意这个路径和我们前面用的都不一样，是约定俗成放在这里。

网络的配置在后面的章节。现在做到这些就可以。

```bash
mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz
```

到这里，容器 Runtime 的部分就算安装完成了。

### 安装 Kubelet

下载 `kubelet`，放在 `/usr/local/bin`。

```
cd /usr/local/bin
wget https://dl.k8s.io/v1.31.1/bin/linux/amd64/kubelet
chmod +x kubelet
```

Node 节点的 Kubernetes 配置文件和证书，我们也都放在 `/etc/kubernetes` 文件夹。

```bash
mkdir /etc/kubernetes
mkdir /etc/kubernetes/pki
```

把 Master 节点的 kubeconfig、CA 证书和 Key，以及 client 的证书和 Key 都复制到 Node 节点。下面这个在 Master 节点执行。

```bash
# node01
scp /etc/kubernetes/admin.kubeconfig root@node01:/etc/kubernetes
scp /etc/kubernetes/pki/ca* root@node01:/etc/kubernetes/pki
scp /etc/kubernetes/pki/client* root@node01:/etc/kubernetes/pki

# node02
scp /etc/kubernetes/admin.kubeconfig root@node02:/etc/kubernetes
scp /etc/kubernetes/pki/ca* root@node02:/etc/kubernetes/pki
scp /etc/kubernetes/pki/client* root@node02:/etc/kubernetes/pki
```

准备 kubelet 的配置文件 `/etc/kubernetes/kubelet.yaml`

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.pem"
serverTLSBootstrap: true
port: 10250

clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"

authorization:
  mode: Webhook

cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"

resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"

# 两个节点的这个配置不一样，node01 是 10.244.1.0/24，node02 是 10.244.2.0/24
podCIDR: "10.244.1.0/24"
```

准备 kubelet 的 Systemd service 文件 `/etc/systemd/system/kubelet.service`，请注意 --node-ip 的参数，这个地方要分别改为 Node 的主 IP，我们这个实验当中分别是 56.11 和 56.12。

```ini
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kubelet --kubeconfig=/etc/kubernetes/admin.kubeconfig \
--node-ip=192.168.56.11 \ # 这个地方要分别改为 Node 的主 IP，我们这个实验当中分别是 56.11 和 56.12
--config=/etc/kubernetes/kubelet.yaml \
--v=1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动 kubelet，设置开机启动。

```bash
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```

到这里，Worker 的安装工作就结束了。

接下来我们为集群[设置网络](/cn/05-设置网络.md)。

## 五、设置基本的路由网络

绝大多数场景下，Kubernetes 网络是倚靠网络插件来完成的，典型的插件包括 `flannel`，`calico`，`weaver` 等等。

为了演示基本的 Pod 网络，我们在这里先不使用上面的经典插件，通过自己设置路由来完成；网络插件的安装会在后面的章节补充。

如下图，每个 Node 都会有一个独立的 Pod 网段。在 Master 和 Node 节点，我们会手动加上到 Pod 的路由。

![](/imgs/05-routes.png)

### 添加 CNI 的配置

创建 CNI 的配置文件夹

```bash
mkdir /etc/cni/net.d/
```

创建子网的配置 `/etc/cni/net.d/10-bridge.conf`。两个 node 上的配置稍有区别：node01 上分配了 `10.244.1.0/24`，我们写成下面的样子；如果是 node02 的，就把 `subnet` 参数的网段配置改为 `10.244.2.0/24`。

```json
{
  "cniVersion": "1.0.0",
  "name": "bridge-network",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "10.244.1.0/24"
        }
      ]
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ]
  }
}
```

创建回环配置 `/etc/cni/net.d/99-loopback.conf`，两个 node 上的配置一样。

```json
{
  "cniVersion": "1.1.0",
  "name": "loopback",
  "type": "loopback"
}
```

### 添加路由

最后，在三台机器上分别设置路由。

master 上：

```bash
ip route add 10.244.1.0/24 via 192.168.56.11
ip route add 10.244.2.0/24 via 192.168.56.12
```

node01 上：

```bash
ip route add 10.244.2.0/24 via 192.168.56.12
```

node02 上：

```bash
ip route add 10.244.1.0/24 via 192.168.56.11
```

接下来我们就可以[运行 Pod](/cn/06-运行Pod.md)了。

## 六、运行 Pod

### 运行 Pod

在 master 机器上，执行下面的命令，创建一个 Deployment。

```bash
kubectl create deployment nginx --image=nginx:latest
```

执行下面的命令查看 Pod 状态。

```bash
kubectl get pods -o wide
```

当 Status 变成了 Running，这个 Pod 就启动成功了，同时还能看到 Pod 的 IP 地址。

从 master 节点上可以尝试 ping 和 wget 这个 IP。

```bash
ping -c 2 <POD_IP>
wget <POD_IP>
```

如果能 ping 通，wget 也能获取到 index.html 文件，说明网络正常。

## 七、安装 Flannel 网络插件和 kube-proxy

在前面的网络设置中我们使用系统间的路由表来做流量导向，这个方案可以在集群里访问 Pod 地址

这个章节我们来安装 Flannel，通过它实现 Pod 网络的联通。

并且还会介绍 kube-proxy 的安装，让集群支持 Service 网络。

### 安装和 Flannel

首先在 Node 节点下载 flannel 的守护进程 `flanneld`。

```bash
wget https://github.com/flannel-io/flannel/releases/download/v0.25.7/flannel-v0.25.7-linux-amd64.tar.gz
```

解压，将里面的 `flanneld` 文件挪到 `/usr/local/bin`。

```bash
tar xzvf flannel-v0.25.7-linux-amd64.tar.gz
mv flanneld /usr/local/bin
```

然后还要安装 flannel 的 CNI 插件。

```bash
wget https://github.com/flannel-io/cni-plugin/releases/download/v1.5.1-flannel3/cni-plugin-flannel-linux-amd64-v1.5.1-flannel3.tgz
```

解压，将里面的 `flannel-amd64` 文件挪到 `/usr/local/bin` 并改名字。

```bash
tar xzvf cni-plugin-flannel-linux-amd64-v1.5.1-flannel3.tgz
mv flannel-amd64 /opt/cni/bin/flannel
```

### 配置 flannel

切换成 flannel 之前，需要把我们前面配置的路由 CNI 配置清理掉。

> [!NOTE]
>
> 如果在前面创建了 Pod，在调整网络之前，需要把它们先都清理掉。否则会出问题。
>
> 可以执行 `kubectl delete deployment nginx` 将前面的 Pod 都删掉。

```bash
rm -f /etc/cni/net.d/*
```

然后创建 `/etc/cni/net.d/10-flannel.conflist`，添加以下内容。这个配置文件是给容器 Runtime 看的，为的是告诉 Runtime 我们要用 flannel。

```json
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

接下来准备一份给 flanneld 看的配置文件。创建 `/etc/kube-flannel/net-conf.json`，添加下面的内容，两个 Node 内容一样。

```json
{
  "Network": "10.244.0.0/16",
  "EnableNFTables": false,
  "Backend": {
    "Type": "vxlan"
  }
}
```

然后准备 flanneld 的 service 文件 `/etc/systemd/system/flanneld.service`。

```ini
[Unit]
Description=Flannel
Documentation=https://github.com/flannel-io/flannel/

[Service]
EnvironmentFile=/etc/kubernetes/flanneld.env
ExecStart=/usr/local/bin/flanneld $FLANNELD_ARGS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

并且还要准备启动的参数和环境变量。

```ini
NODE_NAME="node01"
FLANNELD_ARGS="-kube-subnet-mgr \
-kubeconfig-file=/etc/kubernetes/admin.kubeconfig \
-ip-masq \
-public-ip=192.168.56.11 \
-iface=eth1"
```

Flannel 可以独立运行，也可以作为 DaemonSet 以容器化方式运行。前者一般从 `etcd` 当中存储网络信息，后者一般通过 Kubernetes API 存储网络信息。但这里我们做了一些取巧：通过设置 NODE_NAME、kube-subnet-mgr、kubeconfig-file 等参数，让 flanneld 在独立运行的情况下也通过 Kubernetes API 存储网络信息。

这些配置参数当中 `NODE_NAME` 要根据具体的 node 进行修改，第一台就写 `node01`，第二台就写 `node02`；另外 public-ip 也要做相应调整，分别改成主要用来互联的那个 IP。

改完了 `flanneld` 的相关配置文件，我们还要去 `kube-controller-manager` 那边做一点调整。

在第 5 篇，我们是自己在每台 Node 上面安排了网段：Node01 用 10.224.1.0/24，Node02 用 10.224.2.0/24，使用了 flannel 之后，给 Node 分配网段的任务就可以交给 K8s 集群了。

在 master 的 kube-controller-manager 启动配置中，添加一条参数

```
KUBE_CONTROLLER_MANAGER_ARGS="... \
... \
--allocate-node-cidrs \
...
```

这句话的意思是，各个 Node 的网段由 `kube-controller-manager` 来分配。

保存，在 master 上重启 `kube-controller-manager`，并且在 node 上重启 `flanneld`。

### 安装 kube-proxy

[TODO]

