

# Install Kubernetes from Binaries

This is a notebook I made while learning to install Kubernetes from binaries. I set up one Master and two Nodes. No scripts were used during the installation, and there's no HA setup.

This notebook is suitable for:

- Developing an intuitive understanding of the various Kubernetes components
- Setting up a minimal environment

Not suitable for:

- Deploying in a production environment
- Learning advanced concepts

Barring any surprises, this notebook will be updated twice a year.

## Notes

OS: Rocky Linux 8.10

Component versions:

| Components | Version |
| ---------- | ------- |
| Kubernetes | v1.31.1 |
| etcd       | v3.4.34 |
| containerd | v1.7.22 |

## Steps

1. Basic information and server preparation
2. Kubernetes certificates
3. Installing Master node
4. Installing Worker nodes
5. Setting up the network
6. Starting a Pod

## References

This notebook refers to the following materials:

- ["Kubernetes Authority Guide"](https://book.douban.com/subject/35458432/)
- [kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

## Disclaimer

This repository and its content are created solely for educational purposes. I, the owner of this repository, am not affiliated with, endorsed by, or in any way officially connected with Kubernetes, the Kubernetes project, or the Cloud Native Computing Foundation (CNCF). All trademarks, logos, and brand names are the property of their respective owners. The use of "Kubernetes" in this repository is purely for descriptive purposes and does not imply any relationship, sponsorship, or endorsement by Kubernetes or CNCF.

For official information about Kubernetes, please visit kubernetes.io.

---

# 二进制安装 K8s

这是我在学习二进制安装 K8s 时候做的笔记，我装的是 1 个 Master 和 2 个 Node。安装的时候没用脚本，也没有 HA。

这个笔记适合：

- 用来培养自己对于 K8s 各个组件的感性理解
- 用来安装一个极简环境

不适合：

- 不适合用来部署生产环境
- 学习一些比较进阶的概念

如无意外，这份笔记每年会更新两次。

## 说明

OS: Rocky Linux 8.10

组件的版本:

| 组件       | 版本    |
| ---------- | ------- |
| Kubernetes | v1.31.1 |
| etcd       | v3.4.34 |
| containerd | v1.7.22 |

## 步骤

1. 基本信息和服务器准备
2. K8s 的证书
3. 安装 Master 节点
4. 安装 Worker 节点
5. 设置网络
6. 启动一个 Pod

## 参考

这份笔记参考了下面的材料：

- [《Kubernetes 权威指南》](https://book.douban.com/subject/35458432/)
- [kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
