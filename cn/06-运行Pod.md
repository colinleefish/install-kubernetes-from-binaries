# 运行 Pod

## 运行 Pod

在 master 机器上，执行下面的命令，创建一个 Deployment。

```bash
kubectl create deployment nginx   --image=nginx:latest
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
