** 计算
✅ 案例一：VM 层面网络延迟导致关键服务响应劣化（纯 VM 网络类 TS 案例）
- S - Situation（情境）
客户是一家全球性银行，其核心风险计算引擎运行在 Azure 上的一组高性能 Linux 虚拟机（VM） 上，该服务通过私有 VNet 为多个前端应用提供低延迟 API 接口（SLA 要求 P99 < 50ms）。

某交易日早盘（市场波动剧烈），多个前端系统报告 风险限额更新延迟，但：

Azure Monitor 仪表盘显示 P99 延迟稳定在 45ms
无任何告警触发
SRE 团队初步判断“网络或客户端问题”

🔍 深度排查：发现“看不见的延迟”
步骤 1：绕过仪表盘，直接分析原始时间戳
SRE 团队执行以下 KQL 查询，检查数据是否“准时到达”：
关键发现：

TimeGenerated (UTC)	ingestion_time() (UTC)	IngestionDelaySec	RiskCalcProcessingLatencyMs
2025-11-23T08:12:03Z	2025-11-23T08:18:47Z	404 秒	182 ms
2025-11-23T08:12:15Z	2025-11-23T08:18:49Z	394	176 ms
💥 真相：风险引擎在 08:12 已严重超时（182ms >> 50ms SLA），但数据直到 08:18 才进入 workspace！

步骤 2：根因定位 —— 监控代理资源争抢
登录问题 VM，检查系统状态：

CPU 使用率持续 98%+（由突发市场数据流引发）
top 显示 azuremonitoragent 进程处于 低优先级睡眠状态
AMA 日志 (/var/log/azuremonitoragent/ama.log) 出现：

[WARN] Buffer full (512MB). Dropping oldest telemetry.
[ERROR] HTTP POST to ods.opinsights.azure.com failed: context deadline exceeded

同时，ss -tuln | grep :443 显示大量连接处于 SYN-SENT，表明 内核网络栈因 CPU 无法及时处理出站连接。

🔍 根本原因：

在高负载下，风险计算进程占满 CPU，导致 Azure Monitor Agent 无法及时调度，日志积压在本地缓冲区；

当 CPU 短暂回落时，AMA 批量上传历史数据，造成 TimeGenerated 与 ingestion_time() 严重脱节；

结果：监控系统“回放”过去的问题，而非实时反映当前状态。

🛠️ 解决方案与优化
资源隔离
为 azuremonitoragent 设置 CPU 亲和性与 cgroup 限制，预留 2 vCPU 专用资源：

systemd-run --scope --slice=monitoring.slice --property=CPUQuota=200% /opt/microsoft/azuremonitoragent/bin/ama

调整 DCR 缓冲策略

增大缓冲区并启用压缩（通过 ARM 模板）：

"dataSources": {
  "extensions": [{
    "name": "RiskMetrics",
    "streams": ["Microsoft-Perf"],
    "settings": {
      "bufferSizeMB": 1024,
      "maxSendBatchSize": 1048576
    }
  }]
}
新增“可观测性健康度”监控

创建 KQL 告警规则，监控摄入延迟：
kql
编辑
RiskEngineMetrics_CL
| where TimeGenerated > ago(10m)
| extend delay_sec = datetime_diff('second', ingestion_time(), TimeGenerated)
| summarize max_delay = max(delay_sec) by bin(TimeGenerated, 1m)
| where max_delay > 60
→ 触发 “Monitoring Data Stale” 告警，优先级高于业务指标。
应用层增强
在风险引擎内部实现 本地延迟直方图 + Prometheus 指标，通过 Azure Managed Service for Prometheus 独立上报，避免依赖 AMA 单一通道。
✅ 成果
摄入延迟从 >400 秒降至 <8 秒
下一次市场波动期间，P99 超限告警在 15 秒内触发

✅ 案例二：AKS 集群因健康检查失败导致 Pod 频繁重启（纯 K8S 层面 TS 案例）
- S - Situation（情境）
客户在 Azure 上运行一个微服务架构的交易系统，部署在 AKS（Azure Kubernetes Service）集群 中。

某日凌晨，多个关键服务的 Pod 开始出现周期性崩溃（CrashLoopBackOff），自动重启频率高达每 5 分钟一次，影响交易处理能力。

客户已检查应用日志、资源配额、镜像版本，均无异常，请求高级技术支持介入。

- T - Task（任务）
定位 Pod 频繁重启的根本原因；
判断是否为 AKS 平台缺陷或配置问题；
快速恢复服务，并防止复发。

- A - Action（行动）
检查 Pod 与容器状态：
kubectl describe pod 显示：Liveness probe failed: HTTP probe failed with statuscode 503；
但 kubectl logs 显示应用已成功启动，且能处理部分请求；
排除应用未启动或端口冲突问题。

进入节点层排查：
SSH 登录对应 AKS 节点（Worker Node）；

使用 journalctl -u kubelet 查看 kubelet 日志，发现：
Failed to probe http://<pod-ip>:8080/health: context deadline exceeded

但手动 curl http://<pod-ip>:8080/health 返回 200，健康检查正常。

抓包分析健康检查行为：
使用 tcpdump 监听健康检查端口：
tcpdump -i any -n port 8080 and host <pod-ip>

发现 kubelet 发起健康检查时，TCP SYN 包发出后长时间未收到 ACK；

进一步检查节点内核日志：

dmesg | grep "syn backlog"

输出：

TCP: request_sock_TCP: Possible SYN flooding on port 8080. Sending cookies.

根因定位：
该节点上运行了大量短连接客户端（来自外部系统），导致 TCP 半连接队列（syn backlog）溢出；
如果 TIME_WAIT 数量远高于 ESTABLISHED → 短连接为主；
如果 ESTABLISHED 稳定且数量合理 → 长连接为主。

当 kubelet 发起健康检查时，TCP 握手无法完成，probe 超时失败；
Kubelet 标记 Pod 不健康 → 重启 → 新 Pod 启动后再次陷入相同循环。

解决方案：
临时缓解：
调大内核参数：
net.core.somaxconn = 65535

net.ipv4.tcp_max_syn_backlog = 65535

net.ipv4.tcp_abort_on_overflow = 1

通过 DaemonSet 在所有节点上应用；

延长 Liveness Probe 的 timeoutSeconds 从 1s 到 3s。

长期建议：
客户优化外部调用方连接行为（启用连接池）；
建议使用 readiness/liveness 分离探针，避免健康检查受业务流量干扰；

推动客户启用 AKS 的 Network Policies 限制非必要流量。
R - Result（结果）
所有 Pod 在参数调整后 10 分钟内恢复正常，不再重启；
MTTR 缩短至 4 小时以内；
客户架构团队采纳建议，后续在生产集群中实施了“健康检查隔离”策略；
该案例成为 Azure 内部培训材料中“Kubelet Probe 与 TCP 栈交互的经典案例”。
