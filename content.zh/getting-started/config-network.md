---
title: 五、设置基本的路由网络
weight: 15
bookToc: false
---


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