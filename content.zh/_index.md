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

