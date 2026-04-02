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



1. 背景场景

某金融客户的交易前端部署在 Azure Kubernetes Service (AKS) 上。
现象：在每天上午 10:00 左右，Transaction-Service 会出现间歇性的 HTTP 500 错误，持续约 5-10 分钟。
初步观察：
Pod 没有重启（OOMKilled）。
节点（VM）的整体 CPU 使用率看起来并不高（平均 40%）。
数据库没有明显的 DTU 瓶颈。

第一步：指标异常检测 (Azure Monitor Metrics + AI)
动作：打开 Azure Monitor 的 Metrics 面板，查看 AKS 容器的 CPU 使用率。
发现：
虽然平均 CPU 不高，但开启 Smart Detection（智能检测） 后，发现 Transaction-Service 的 Pod 在报错期间，CPU 使用率有极其短暂的尖峰（Spike），随后迅速回落。
同时，Application Insights 的“故障”视图显示，错误类型主要是 System.TimeoutException。
假设：可能是某种资源争抢导致线程阻塞，或者发生了微突发（Micro-burst）导致的资源限制。
第二步：日志关联分析 (Log Analytics + KQL)
动作：进入 Log Analytics，使用 KQL 关联应用日志和系统日志。
// 查询应用异常和容器事件的关联

let ExceptionLogs = AppExceptions
| where TimeGenerated > ago(1h)
| where ServiceName == "transaction-service"
| project TimeGenerated, Message, Cloud_RoleInstance;
let ContainerEvents = KubeEvents
| where TimeGenerated > ago(1h)
| where Name == "transaction-service"
| project TimeGenerated, Reason, Message, NodeName;
ExceptionLogs


发现：
查询结果显示，在报错时间点，并没有 OOM 事件，也没有调度失败。
但是，通过 Application Insights 的“性能”视图，你发现依赖项（Dependency）中，调用下游 Inventory-API 的耗时从正常的 50ms 飙升到了 3000ms。
第三步：全链路追踪与网络分析 (Application Insights + Azure Monitor for Containers)
动作：在 App Insights 中找到一个失败的请求，点击 Transaction Diagnostics 查看端到端 Trace。
发现：
Trace 显示：Frontend -> Transaction-Service -> Inventory-API。
在 Transaction-Service 调用 Inventory-API 之间，出现了一个巨大的空白（Gap）。这说明请求发出了，但响应还没回来，或者连接建立卡住了。
检查 Azure Monitor for Containers 的网络指标，发现 Transaction-Service 所在的节点与 Inventory-API 所在的节点之间，TCP 重传率在报错期间激增。

第四步：底层根因挖掘 (VMSS + Azure Network Watcher)
动作：既然怀疑网络，检查底层的 VM Scale Set (VMSS) 和 CNI 插件。
分析：
通过 KQL 查询 VMSS 的 AzureDiagnostics，发现报错期间，部分节点的 azure-cni 插件日志中有 IP allocation failed 或 NetworkPolicy drop 的警告。
进一步检查 Azure Network Watcher 的流量日志，发现由于 Inventory-API 的某个新版本 Pod 配置了错误的 Network Policy（网络策略），导致它拒绝了来自特定子网（恰好是 Transaction-Service 所在的节点网段）的流量。
为什么是间歇性的？因为 K8s 的 Service 负载均衡是轮询的，当流量落到那个配置错误的 Pod 上时，连接就会超时，直到客户端重试落到正常的 Pod 上。
3. 最终根因与解决

根因：Inventory-API 的新版本 Helm Chart 中，Network Policy 的标签选择器写错了，导致它错误地拦截了来自 Transaction-Service 的部分流量。
解决：
临时：使用 kubectl 或 Azure Portal 直接回滚 Inventory-API 的部署。
自动化（专家操作）：创建一个 Azure Automation Runbook。

触发器：当 Azure Monitor 检测到 HTTP 5xx 错误率 > 5% 且持续时间 > 2分钟。
动作：Runbook 自动调用 Azure Resource Graph 查询当前版本的 Revision，并自动执行 kubectl rollout undo 进行回滚，同时发送 Teams 通知给运维团队。
| join kind=inner (ContainerEvents) on $left.TimeGenerated == $right.TimeGenerated
