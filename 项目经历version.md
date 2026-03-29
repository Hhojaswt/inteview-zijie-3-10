客户是一家全球领先的金融机构，在 Azure 上运行其核心风控平台，采用 AKS + VM 混合架构：

前端微服务部署在 AKS 集群中（多可用区，节点池为 VMSS）；
后端依赖一个运行在独立虚拟机上的高性能 C++ 计算引擎（低延迟要求 <50ms）；
两者通过 VNet 对等互联，使用私有 IP 通信。
在一次版本发布后，客户发现：

AKS 中多个关键 Pod 出现周期性 CrashLoopBackOff；
同时，VM 上的计算服务日志显示与 AKS 的连接频繁超时；

快速定位 AKS Pod 崩溃的根本原因；
分析 VM 与 AKS 之间的网络延迟是否异常；
协调 Azure Network、AKS Platform、Storage 等内部团队，推动根因解决；

A - Action（行动）

🔹 第一步：排除应用层 & 配置问题

检查 Pod 日志、Events、Liveness Probe 配置：发现 Liveness 探针失败，但应用本身启动正常；
查看节点资源使用情况（通过 Azure Monitor）：CPU/内存充足，无 OOMKilled；
检查镜像版本和 Helm Chart 配置：确认无配置错误。
初步判断：健康检查机制可能被干扰，但原因不明。
🔹 第二步：深入节点层分析（Node-Level）

登录受影响的 AKS 节点（SSH），使用 dmesg 和 journalctl 查看系统日志；
发现大量如下内核日志：
TCP: request_sock_TCP: Possible SYN flooding on port 8080. Sending cookies.

进一步使用 netstat -s | grep -i "listen overflows" 发现监听队列溢出计数持续增长。
判断：TCP SYN Flood 导致 accept queue 溢出，导致 kubelet 无法响应健康检查 HTTP 请求。
🔹 第三步：分析流量来源与 VM 关联性

使用 tcpdump 抓包分析 8080 端口流量，发现：
来源 IP 主要来自那台 后端计算 VM；
VM 上运行的应用每 100ms 发起一次短连接（未复用连接池），且未设置合理的超时；
原因浮现：新版本发布后，VM 上的应用逻辑变更，导致对 AKS 接口的调用频率上升 10 倍，且使用短连接模式。
这导致：

AKS 侧短时间内收到大量 TCP SYN；
内核 backlog 队列满，健康检查请求被丢弃；
kubelet 标记 Pod 不健康 → 重启 → 再次崩溃，形成恶性循环。
🔹 第四步：协同解决与优化

短期缓解：
建议客户临时调大内核参数：

1net.core.somaxconn = 65535
2net.ipv4.tcp_max_syn_backlog = 65535
在 AKS 节点池上通过 DaemonSet 应用配置；
同时延长 Liveness Probe 的 initialDelaySeconds 和 timeoutsecond

长期优化：
推动客户开发团队修改 VM 应用代码：引入 HTTP 连接池（Keep-Alive）和指数退避重试机制；
建议将关键服务间调用迁移至 Service Mesh（如 Istio）以实现熔断、限流；


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

1TCP: request_sock_TCP: Possible SYN flooding on port 8080. Sending cookies.
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
