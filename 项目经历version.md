客户是一家全球领先的金融机构，在 Azure 上运行其核心风控平台，采用 AKS + VM 混合架构：

前端微服务部署在 AKS 集群中（多可用区，节点池为 VMSS）；
后端依赖一个运行在独立虚拟机上的高性能 C++ 计算引擎（低延迟要求 <50ms）；
两者通过 VNet 对等互联，使用私有 IP 通信。
在一次版本发布后，客户发现：

AKS 中多个关键 Pod 出现周期性 CrashLoopBackOff；

A - Action（行动）

检查 Pod 与容器状态：
kubectl describe pod 显示：Liveness probe failed: HTTP probe failed with statuscode 503；
但 kubectl logs 显示应用已成功启动，且能处理部分请求；
排除应用未启动或端口冲突问题。
进入节点层排查：
SSH 登录对应 AKS 节点（Worker Node）；
使用 journalctl -u kubelet 查看 kubelet 日志，发现：

1Failed to probe http://<pod-ip>:8080/health: context deadline exceeded
但手动 curl http://<pod-ip>:8080/health 返回 200，健康检查正常。
抓包分析健康检查行为：
使用 tcpdump 监听健康检查端口：
tcpdump -i any -n port 8080 and host <pod-ip>

发现 kubelet 发起健康检查时，TCP SYN 包发出后长时间未收到 ACK；
进一步检查节点内核日志：
1dmesg | grep "syn backlog"
输出：
TCP: request_sock_TCP: Possible SYN flooding on port 8080. Sending cookies.
根因定位：
该节点上运行了大量短连接客户端（来自外部系统），导致 TCP 半连接队列（syn backlog）溢出；
当 kubelet 发起健康检查时，TCP 握手无法完成，probe 超时失败；
Kubelet 标记 Pod 不健康 → 重启 → 新 Pod 启动后再次陷入相同循环。
解决方案：
临时缓解：
调大内核参数：
1net.core.somaxconn = 65535
2net.ipv4.tcp_max_syn_backlog = 65535
3net.ipv4.tcp_abort_on_overflow = 1
通过 DaemonSet 在所有节点上应用；
延长 Liveness Probe 的 timeoutSeconds 从 1s 到 3s。
长期建议：
客户优化外部调用方连接行为（启用连接池）；
建议使用 readiness/liveness 分离探针，避免健康检查受业务流量干扰；
推动客户启用 AKS 的 Network Policies 限制非必要流量。


<img width="1126" height="1066" alt="fae67fa7f964e06314b8ea2e46b45622" src="https://github.com/user-attachments/assets/7293192e-8cc5-4fb8-9b04-e2ad9c2f4156" />
<img width="1201" height="947" alt="edc4758aaedb178c65f342b8435b388e" src="https://github.com/user-attachments/assets/16d8401e-4f6b-487b-a96a-a17a61d787fe" />
<img width="1236" height="429" alt="d3271b86e28cefc4a43ebc45d9b0d0ad" src="https://github.com/user-attachments/assets/e9076f5b-2697-4d90-8053-52ab4e9c9f52" />
<img width="1379" height="691" alt="b3bf5c1f25b50a5910d4a5d0d12e34d9" src="https://github.com/user-attachments/assets/c1651f46-7e9e-425a-b8ec-8cd1b232c1b6" />



