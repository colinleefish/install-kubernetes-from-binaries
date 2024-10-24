---
title: 一、基本信息和服务器准备
weight: 11
bookToc: false
---

# 一、基本信息和服务器准备

这节记录了服务器准备阶段要做的事情。

## 基本信息

需要的准备内容：

- 3 台 Linux 服务器，我们这里使用了 Rocky Linux 8
- 3 个同网段的 IP，挂在这 3 台服务器上

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
{{% tabs %}}

{{% tab "master" %}}

```bash
hostnamectl set-hostname master
```

{{% /tab %}}

{{% tab "node01" %}}


```bash
hostnamectl set-hostname node01
```

{{% /tab %}}

{{% tab "node02" %}}
```bash
hostnamectl set-hostname node02
```
{{% /tab %}}

{{% /tabs %}}

把这三个 hostname 都写在 /etc/hosts 里面

```
# /etc/hosts

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
# 三台都要执行

vim /etc/fstab
vim /etc/selinux/config
```

允许 `ip_forward`（这样的话 Node 节点才知道要把数据包转发给 Pod）

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
