---
title: 三、安装 Master 节点
weight: 13
bookToc: false
---

# 三、安装 Master 节点

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
