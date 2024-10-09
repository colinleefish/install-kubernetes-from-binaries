# 安装 Worker 节点

接下来我们安装 Worker 节点，主要是安装 `kubelet`，`kube-proxy`，容器的 Runtime，还有准备网络工具，也就是所谓的 CNI Plugins。

这部分的操作要在两个 Node 服务器上分别执行。

Worker 的多数二进制的文件，我们也都都放在 `/usr/local/bin` 里面，并且可以在 `~/.bash_profile` 里面把 `/usr/local/bin` 加到 `PATH` 里。

```bash
mkdir /usr/local/bin
echo "export PATH=$PATH:/usr/local/bin" >> ~/.bash_profile
source ~/.bash_profile
```

## 安装容器 Runtime

在 Worker 节点上，调用的链条主要是下面这样：kubelet 指挥 containerd，containerd 指挥 runc，其中 containerd 和 runc 共同构成容器的运行环境。

```
┌───────────┐       ┌──────────────┐       ┌────────┐
│           │       │              │       │        │
│  kubelet  ┼──────►│  containerd  ┼──────►│  runc  │
│           │       │              │       │        │
└───────────┘       └──────────────┘       └────────┘
```

我们第一步安装 containerd。

containerd 本身提供了一个非常好的安装说明：[https://github.com/containerd/containerd/blob/main/docs/getting-started.md](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

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

## 安装 Kubelet

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

podCIDR: "10.244.0.0/16"
# WIP: 这两个后面再写
# tlsCertFile: "/etc/kubernetes/pki/client.crt"
# tlsPrivateKeyFile: "/etc/kubernetes/pki/client.key"
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

