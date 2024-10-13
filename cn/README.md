# 二进制安装 K8s：中文版

## 目录

- 前言
- 一、基本信息和服务器准备
- 二、K8s 的证书
- 三、安装 Master 节点
- 四、安装 Worker 节点
- 五、设置网络

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

## 一、基本信息和服务器准备

这部分记录了服务器准备信息。

### 基本信息

需要的准备内容：

1. 3 台能上网的 Linux 服务器，这次我用的是 Rocky Linux 8
2. 3 个同网段的 IP，挂在这 3 台服务器上

计划表如下：

| 角色   | IP            | 要在上面安装的组件                                                                                    |
| ------ | ------------- | ----------------------------------------------------------------------------------------------------- |
| master | 192.168.56.10 | <ul><li>etcd</li><li>kube-apiserver</li> <li>kube-controller-manager</li><li>kube-scheduler</li></ul> |
| node01 | 192.168.56.11 | <ul><li>containerd</li><li>kubelet</li> <li>kube-proxy</li> </ul>                                     |
| node02 | 192.168.56.12 | <ul><li>containerd</li><li>kubelet</li> <li>kube-proxy</li> </ul>                                     |

> [!NOTE]  
> 项目中包含了一个给 Virtualbox 用的 Vagrantfile，用来快速启动 3 个服务器，知道怎么用的话可以用那个

### 服务器准备

设置这 3 台服务器的 hostname

```bash
hostnamectl set-hostname (master|node01|node01)
```

把这三个 hostname 都写在 /etc/hosts 里面

```/etc/hosts
192.168.56.10 master
192.168.56.11 node01
192.168.56.12 node02
```

设置好时间和时区（根据自己的实际情况进行设置）

```bash
timedatectl set-timezone <你的时区>
```

在 3 台服务器上安装一些基础工具

```bash
dnf install wget vim
```

master 节点还要安装 OpenSSL，一会儿我们在这台机器上创建证书

```bash
dnf install openssl
```

node 节点还要安装一些后面要用到的工具

```bash
dnf install socat conntrack ipset
```

关闭 swap 和 selinux

```bash
vim /etc/fstab
vim /etc/selinux/config
```

允许 ip_forward（这样的话 Node 节点才知道要把数据包转发给 Pod）

编辑 `/etc/sysctl.conf`，添加下面这句话

```ini
net.ipv4.ip_forward = 1
```

保存，然后执行 `sysctl -p` 让我们的设置生效。

创建 K8s 各个组件的配置文件夹，我们后面还要创建其他文件夹

```bash
mkdir /etc/kubernetes       # 这里放配置
mkdir /etc/kubernetes/pki   # 这里放证书
```

### 有关网络的特别说明

执行 `ip addr` 后，

如果你的机器上只显示 `lo` 和 `eth0`，那么在本次配置中，你的主网卡就是 `eth0`。

如果你使用了 VirtualBox，可能会有一个 `NAT` 网络和一个 `Host-only` 网络：`NAT` 网络用于联网下载资源，`Host-only` 网络用于虚拟机之间的互联。在这种情况下，主网卡是 `Host-only` 的那个网卡，K8s 各 Node 之间的通信会通过这张网卡进行。

建议在每台服务器上相互 ping 一下，确保节点间可以互通。

接下来准备 [K8s 的证书](/cn/02-K8s的证书.md)。

## 二、K8s 的证书

K8s 各个组件之间的交互都是通过 HTTPS 实现的：controller 访问 apiserver、kubelet 访问 apiserver，甚至我们使用 kubectl 工具操作时，都是通过 HTTPS 连接到 apiserver。

这与我们通过浏览器访问网站时使用的 HTTPS 类似，但 K8s 采用双向认证。不仅服务端需要证书和密钥，客户端也要准备自己的证书和密钥来证明身份。

为了简化，接下来我们只准备几个证书，能共享的组件就共用这些证书。

| 证书                 | 用到这个证书的组件 | 文件                                                      | 角色                                 |
| :------------------- | ------------------ | --------------------------------------------------------- | ------------------------------------ |
| CA                   | 所有               | <ul><li>ca-key.pem（Key）</li><li>ca.pem（证书）</li></ul>    | 根证书                               |
| API Server 证书      | kube-apiserver     | <ul><li>apiserver-key.pem（Key）</li><li>apiserver.pem（证书）</li></ul> | API Server 的服务端证书              |
| Client 证书          | 各种组件都会用     | <ul><li>client-key.pem（Key）</li><li>client.pem（证书）</li></ul>       | 用来访问 API Server 的客户端证书               |
| Service Account 证书 | kube-apiserver     | <ul><li>sa-key.pem（私钥）</li><li>sa-pub.pem（公钥）</li></ul>           | Service Account JWT Token 的签发证书 |

> [!NOTE]  
> Service Account 证书是一对公私钥，和前面说到的 HTTPS 双向认证证书说的不是一个东西。这个后面会再提到。

### 工具准备

这个部分我们用 Cloudflare 的 SSL 工具 `cfssl`。

下载地址在这里：[https://github.com/cloudflare/cfssl/releases/](https://github.com/cloudflare/cfssl/releases/)

我们主要用到里面的 `cfssl` 和 `cfssljson` 这两个二进制。

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64

mv cfssl_1.6.5_linux_amd64 /usr/local/bin/cfssl
mv cfssljson_1.6.5_linux_amd64 /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl
chmod +x /usr/local/bin/cfssljson
```

我们趁这个机会把 `/usr/local/bin` 放在 `PATH` 里，后面其他的 bin 也都会放在这里。

```bash
echo "export PATH=$PATH:/usr/local/bin" >> ~/.bash_profile
source ~/.bash_profile
```

你可能听说 OpenSSL 也能生成证书的工具。那个用起来有一些难受，所以我们在创建证书的时候就不用那个了，创建 Key 的时候会简单用用。

### 生成 CA 证书

证书是一个树状的结构，CA 证书是这个树状结构的根，K8s 其他证书的生成过程中都要带上 CA 的证书，仿佛是声明了一种父子关系：其他证书都是这张根证书的“儿子”，他们有一个共同的“父亲”。

我们先来生成 CA 这个“父亲”。

```bash
cd /etc/kubernetes/pki/
```

创建 `ca-csr.json` 文件，写下面的内容。

```json
{
  "CN": "IKFB",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
```

然后用 `cfssl` 工具生成 `ca-key.pem` 密钥和 `ca.pem` 证书。

```sh
cfssl genkey -initca ca-csr.json | cfssljson -bare ca
```

### 生成 API Server 的证书

接下来我们要生成 API Server 的证书，这个证书类似于常见的 HTTPS 证书，主要用于在服务端证明它确实是 API Server。

需要注意的是，客户端访问 API Server 的地址可能不同：有的通过 Service IP（10.96.0.1），有的使用 https://master:6443 ，或者直接通过 IP 地址（192.168.56.10）。因此，证书必须涵盖这些不同的访问地址。在签发证书时，我们需要将这些地址信息一并包含，就像给 API Server 发一张‘身份证’，上面列出了它的多个合法名称。

在证书技术中，指定多个地址的方式叫做 Subject Alternative Names (SANs)，我们稍后会在证书的配置文件中添加这些 SAN 信息。

复制下面的内容到 `kube-apiserver-csr.json`。

```json
{
  "CN": "kube-apiserver",
  "hosts": [
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster.local",
    "master",
    "10.96.0.1",
    "192.168.56.10"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
```

然后我们生成 API Server 的证书和 Key。

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -profile=kube-apiserver kube-apiserver-csr.json | cfssljson -bare kube-apiserver
```

这样就得到了 `kube-apiserver-key.pem` 和 `kube-apiserver.pem`。

### 生成 Client 证书

接下来我们生成一张 Client 证书，供访问 API Server 的组件使用。

这次安装的 K8s 通过证书认证身份。我们创建的证书相当于 K8s 的‘超级管理员’身份。

这个证书会配置到 controller、scheduler 和 kubelet 的 kubeconfig 中，用于访问 API Server。

> [!NOTE]
> 其实这些组件应该各自生成专属的证书和 Kubeconfig，但为了简化流程，这里统一使用一张证书。
>
> 请勿在生产环境中这样操作。

首先创建 `client-csr.json`。

```json
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
```

然后执行创建证书的命令。

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem client-csr.json | cfssljson -bare client
```

这样就获得了 `client-key.pem` 和 `client.pem`。

命令执行后，你可能会看到提示‘[WARNING] This certificate lacks a "hosts" field...’，意思是因为没有指定 hosts，这个证书不适合用来标识服务器身份。但我们并不打算将它作为服务端证书使用，所以可以忽略这个警告。

我们指定了 `CN=admin,O=system:masters`，相当于声明这个用户叫 admin，属于 `system:masters` 组。这对应 K8s 的一个预设规则：属于 `system:masters` 组的用户都是超级管理员。

### 生成 Service Account 公私钥

和前面的证书 / Key 不同，Service Account 证书是一对公私钥。

有时候 Pod 也要和 Kubernetes 的 API 交互，因此在 Pod 创建的时候，Kubernetes 会为每个 Pod 创建一个 JWT Token 做为交互凭证，这个 Service Account 的公私钥就是用来生成和验证 JWT Token 的。

这个比较简单，我们直接用 OpenSSL 生成。

生成私钥和公钥

```bash
# 生成私钥
openssl genrsa -out sa-key.pem 2048
# 用私钥生成公钥
openssl rsa -in sa-key.pem -pubout -out sa-pub.pem
```

到这里为止，如果证书文件夹包含了下面这些文件，那基本说明这步安装完成。

```bash
[root@master pki]# ls -alh
total 60K
drwxr-xr-x. 2 root root 4.0K Oct 13 13:51 .
drwxr-xr-x. 3 root root   17 Oct 13 13:38 ..
-rw-r--r--. 1 root root 1009 Oct 13 13:48 ca.csr
-rw-r--r--. 1 root root  214 Oct 13 13:48 ca-csr.json
-rw-------. 1 root root 1.7K Oct 13 13:48 ca-key.pem
-rw-r--r--. 1 root root 1.3K Oct 13 13:48 ca.pem
-rw-r--r--. 1 root root  920 Oct 13 13:51 client.csr
-rw-r--r--. 1 root root  130 Oct 13 13:51 client-csr.json
-rw-------. 1 root root 1.7K Oct 13 13:51 client-key.pem
-rw-r--r--. 1 root root 1.3K Oct 13 13:51 client.pem
-rw-r--r--. 1 root root 1.2K Oct 13 13:50 kube-apiserver.csr
-rw-r--r--. 1 root root  411 Oct 13 13:49 kube-apiserver-csr.json
-rw-------. 1 root root 1.7K Oct 13 13:50 kube-apiserver-key.pem
-rw-r--r--. 1 root root 1.6K Oct 13 13:50 kube-apiserver.pem
-rw-------. 1 root root 1.7K Oct 13 13:51 sa-key.pem
-rw-r--r--. 1 root root  451 Oct 13 13:51 sa-pub.pem
```

接下来开始[安装 Master 节点](/cn/03-安装Master节点.md)。

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

## 五、设置网络

绝大多数场景下，Kubernetes 网络是倚靠网络插件来完成的，典型的插件包括 `flannel`，`calico`，`weaver` 等等。

为了演示基本的 Pod 网络，我们在这里不使用上面的经典插件，通过自己设置路由来完成。

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

````bash

```bash
ip route add 10.244.1.0/24 via 192.168.56.11
````

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
