# Installing K8s from Binaries

This is a notebook I made while learning to install Kubernetes from binaries. I set up one Master and two Nodes. No scripts were used during the installation, and there's no HA setup.

If Kubernetes were like a set of Lego bricks, I hope this notebook could serve as a simple (unofficial) manual.

Barring any surprises, this notebook will be updated twice a year.

## Notes

This notebook is suitable for:

- Developing an intuitive understanding of the various Kubernetes components
- Setting up a minimal environment

Not suitable for:

- Deploying in a production environment
- Learning advanced concepts

## Steps

1. Basic information and server preparation
2. Kubernetes certificates
3. Installing etcd and Master node
4. Installing Worker nodes
5. Understanding and setting up networking
6. Starting a Pod

## References

This notebook refers to the following materials:

- ["Kubernetes Authority Guide"](https://book.douban.com/subject/35458432/)
- [kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
## 

--------

# 二进制安装 K8s

这是我在学习二进制安装 K8s 时候做的笔记，我装的是 1 个 Master 和 2 个 Node。安装的时候没用脚本，也没有 HA。

如果 K8s 是个乐高拼图，希望这份笔记能当作一份极简的（非官方的）说明书。

如无意外，这份笔记每年会更新两次。

## 说明

这个笔记适合：

- 用来培养自己对于 K8s 各个组件的感性理解
- 用来安装一个极简环境

不适合：

- 不适合用来部署生产环境
- 学习一些比较进阶的概念

## 步骤

1. 基本信息和服务器准备
2. K8s 的证书
3. 安装 etcd 和 Master 节点
4. 安装 Worker 节点
5. 理解和设置网络
6. 启动一个 Pod

## 参考

这份笔记参考了下面的材料：

- [《Kubernetes 权威指南》](https://book.douban.com/subject/35458432/)
- [kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

