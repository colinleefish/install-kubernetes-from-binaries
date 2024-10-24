---
title: 二、K8s 的证书和 Key
weight: 12
bookToc: false
---

## 二、K8s 的证书

K8s 各个组件之间的交互都是通过 HTTPS 实现的：controller 访问 apiserver、kubelet 访问 apiserver，甚至我们使用 kubectl 工具操作时，都是通过 HTTPS 连接到 apiserver。

这与我们通过浏览器访问网站时使用的 HTTPS 类似，但 K8s 采用双向认证。不仅服务端需要证书和密钥，客户端也要准备自己的证书和密钥来证明身份。

为了简化，接下来我们只准备几个证书，能共享的组件就共用这些证书。

| 证书                 | 用到这个证书的组件 | 文件                                                                     | 角色                                 |
| :------------------- | ------------------ | ------------------------------------------------------------------------ | ------------------------------------ |
| CA                   | 所有               | <ul><li>ca-key.pem（Key）</li><li>ca.pem（证书）</li></ul>               | 根证书                               |
| API Server 证书      | kube-apiserver     | <ul><li>apiserver-key.pem（Key）</li><li>apiserver.pem（证书）</li></ul> | API Server 的服务端证书              |
| Client 证书          | 各种组件都会用     | <ul><li>client-key.pem（Key）</li><li>client.pem（证书）</li></ul>       | 用来访问 API Server 的客户端证书     |
| Service Account 证书 | kube-apiserver     | <ul><li>sa-key.pem（私钥）</li><li>sa-pub.pem（公钥）</li></ul>          | Service Account JWT Token 的签发证书 |

{{% hint info %}}
**提示**

Service Account 证书是一对公私钥，和前面说到的 HTTPS 双向认证证书说的不是一个东西。这个后面会再提到。
{{% /hint %}}

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
