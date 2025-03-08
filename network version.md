#### **一、网络基础与协议**
**1. TCP三次握手和四次挥手的详细过程？为什么需要TIME_WAIT状态？**  
- **三次握手**：  
  1. **SYN**：客户端发送SYN包（seq=x）到服务端，进入SYN_SENT状态。  
  2. **SYN-ACK**：服务端回复SYN-ACK包（seq=y, ack=x+1），进入SYN_RCVD状态。  
  3. **ACK**：客户端发送ACK包（ack=y+1），双方进入ESTABLISHED状态，连接建立。  
- **四次挥手**：  
  1. **FIN**：主动关闭方（如客户端）发送FIN包（seq=u），进入FIN_WAIT_1状态。  
  2. **ACK**：被动关闭方（如服务端）回复ACK包（ack=u+1），进入CLOSE_WAIT状态。  
  3. **FIN**：被动关闭方发送FIN包（seq=v），进入LAST_ACK状态。  
  4. **ACK**：主动关闭方回复ACK包（ack=v+1），进入TIME_WAIT状态，等待2MSL后关闭。  
- **TIME_WAIT作用**：  
  确保最后一个ACK包被对方接收，防止旧连接的延迟报文干扰新连接。

**2. TCP和UDP的区别？举一个大数据场景中使用UDP的例子。**  
- **TCP**：面向三次握手连接、可靠传输（重传机制）、流量控制（滑动窗口）、拥塞控制（慢启动）。  
- **UDP**：无连接、不可靠、低延迟、无流量控制。  
- **大数据场景示例**：  
  - Kafka的Metrics上报（轻量级统计信息，允许少量丢包）。  
  - Flink的TaskManager心跳通信（低延迟优先）。
  - 在线游戏：对实时性要求高，偶尔丢包不会影响游戏体验。
  - DNS 查询：传输数据量小、延迟低且对偶尔重传容忍。
  - TCP 则适用于需要高可靠性、数据完整性保障的应用，如网页浏览、文件传输和电子邮件。

---

#### **二、网络工具与排查**
**3. 如何用`tcpdump`抓取Kafka Broker的9092端口流量并保存到文件？**  
```bash
tcpdump -i eth0 port 9092 -w kafka_traffic.pcap
```  
- **分析**：用Wireshark打开`kafka_traffic.pcap`，过滤Kafka协议（如`kafka`关键字）。ip.dst_host contains "cloudapp.azure.com”


**4. 如何快速判断服务器是否监听某端口（如HDFS NameNode的8020）？**  
```bash
nc -zv <NameNode_IP> 8020   # 成功显示"succeeded"，失败显示"Connection refused" nc：这是 netcat 工具，-z：启用零 I/O 模式，-v：启用详细（verbose）模式
ss -tln | grep 8020         # 查看本地监听的TCP端口
```

**5. 如何分析网络延迟问题？**  
- **工具组合**：  
  1. `ping`：检查基础连通性和RTT（往返延迟）。  
  2. `mtr`：动态追踪路由路径，识别中间节点丢包或延迟。  mtr -r -c 10 google.com
  3. `traceroute`：静态路由追踪。  
  4. `tcpdump`抓包分析TCP重传（`tcp.analysis.retransmission`）。  

---

#### **三、性能调优**
**6. 如何通过内核参数优化TCP性能？**  
修改`/etc/sysctl.conf`，调整以下参数：  
```bash
net.core.somaxconn = 65535       # 增大TCP连接队列长度（防止SYN Flood）
net.ipv4.tcp_tw_reuse = 1        # 允许复用TIME_WAIT连接（需开启时间戳，允许内核重用处于 TIME_WAIT 状态的 TCP 连接。
net.ipv4.tcp_fin_timeout = 30    # 减少FIN_WAIT_2状态超时时间,设置 TCP 连接在 FIN_WAIT_2 状态下等待的时间
net.ipv4.tcp_max_syn_backlog = 8192  # 等待完成三次握手的连接请求，增大SYN队列容量
```  
执行`sysctl -p`生效。

**7. 如何解决Kafka Producer发送消息的高延迟问题？**  
- **网络层排查**：  
  1. 检查Broker网络带宽：`sar -n DEV 1`（观察`rxkB/s`和`txkB/s`）。 sar：全称 System Activity Reporter，用于收集和显示系统性能数据（如 CPU、内存、I/O、网络等）的工具。
-n DEV：指定显示网络设备（DEVice）的统计信息，比如每个网卡的收发包数、传输速率、错误数等。
  2. 抓包分析TCP窗口大小和重传率：  
     ```bash
     tcpdump -i eth0 host <Broker_IP> and port 9092 -w producer.pcap
     ```  
  3. 调整Producer参数：  
     - `linger.ms=100`（增大批量发送延迟）。  
     - `compression.type=snappy`（减少网络传输量）。  
这两个参数通常出现在 Kafka Producer 的配置中，目的是提高传输效率和吞吐量。
---

#### **四、安全与防火墙**
**8. 如何用iptables限制HDFS DataNode的流量仅允许来自集群内网？**  
```bash
iptables -A INPUT -p tcp --dport 50010 -s 10.0.0.0/24 -j ACCEPT  # DataNode数据传输端口
iptables -A INPUT -p tcp --dport 50010 -j DROP                   # 拒绝其他IP
```

**9. 如何防止SSH暴力破解？**  
- **方法**：  
  1. 禁用密码登录，强制密钥认证：  
     ```bash
     # /etc/ssh/sshd_config
     PasswordAuthentication no
     ```  
  2. 使用`fail2ban`自动封禁恶意IP：  
     ```bash
     fail2ban-client status sshd   # 查看封禁状态
     ```  
  3. 修改默认SSH端口：  
     ```bash
     Port 2222
     ```  

---

#### **五、大数据组件网络问题**
**10. HDFS写入速度慢，如何区分是网络还是磁盘问题？**  
- **排查步骤**：  
  1. **磁盘检查**：  
     ```bash
     iostat -x 1        # 观察%util（>80%表示磁盘瓶颈）
     ```  
  2. **网络检查**：  
     ```bash
     sar -n DEV 1       # 检查网卡吞吐量（rxkB/s、txkB/s）
     mtr <DataNode_IP>  # 分析跨节点延迟和丢包
     ```  
  3. **HDFS配置优化**：  
     - 调整`dfs.datanode.data.dir`分散磁盘IO。  
     - 启用HDFS短路读（避免跨节点读取）。  

**11. Kafka Consumer出现频繁Rebalance，可能是什么网络问题？**  
- **原因**：  
  1. 网络抖动导致Consumer心跳超时（`session.timeout.ms`默认10秒）。  
  2. 跨机房消费导致网络延迟高。  
- **解决方案**：  
  1. 增大`session.timeout.ms`和`heartbeat.interval.ms`。  
  2. 部署Consumer尽量靠近Broker（同机房/可用区）。  

---

#### **六、云原生与容器网络**
**12. Docker容器网络模式有哪些？如何实现容器跨主机通信？**  
- **网络模式**：  
  1. `bridge`：默认网桥模式，容器通过虚拟网桥通信。  
  2. `host`：共享宿主机网络命名空间。  
  3. `overlay`：跨主机的容器网络（需Swarm/Kubernetes支持）。  
- **跨主机通信方案**：  
  - 使用Calico、Flannel等CNI插件。  
  - 配置VXLAN隧道或BGP路由。  

**13. Kubernetes中Service的ClusterIP和NodePort有什么区别？**  
- **ClusterIP**：集群内部虚拟IP，仅限Pod间通信。  
- **NodePort**：通过节点IP+端口暴露服务到外部（范围30000-32767）。  
- **示例**：  
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    type: NodePort
    ports:
      - port: 80
        targetPort: 9376
        nodePort: 30080
    selector:
      app: my-app
  ```

---

#### **七、开放性问题**
**14. 如果某机房网络中断，如何保障HDFS和Kafka的高可用？**  
- **HDFS**：  
  1. 确保数据跨机房多副本（`dfs.replication=3`且副本分布在多个机房）。  dfs.replication=3 是 Hadoop HDFS（Hadoop Distributed File System） 的一个配置参数
  2. 手动切换客户端到备用机房NameNode。 NameNode HA 配置： 部署多个 NameNode（主/备模式），借助 ZooKeeper 进行故障检测和自动切换，确保 NameNode 故障时服务仍能继续提供。
- **Kafka**：  
  1. 使用MirrorMaker工具同步Topic到备用机房。 将 Kafka 数据在不同地点之间进行复制
  2. 生产端配置多Broker列表（`bootstrap.servers=primary:9092,backup:9092`）。  

**15. 设计一个支持百万级QPS的Kafka集群网络架构。**  
- **关键点**：  
  1. **Broker横向扩展**：部署至少10个Broker，分散Partition Leader。  
  2. **网络硬件**：万兆网卡，启用多队列（`ethtool -L eth0 combined 16`）。  
  3. **分区与副本**：  
     - 每个Topic设置100+分区（`num.partitions=100`）。Kafka 中的每个主题（Topic）会被划分为多个分区。  
     - 副本数`replication.factor=3`，保证跨机架分布。  为了保证数据的高可用性和容错性，每个分区可以配置多个副本。
Leader 副本： 负责处理读写请求。
Follower 副本： 从 Leader 副本中同步数据，不直接处理外部请求。
  4. **客户端优化**：  
     - Producer启用`acks=1`（平衡可靠性和延迟）。  
     - Consumer增加并发度（`num.stream.threads=CPU核心数`）。  

---

### **总结**
大数据运维网络面试核心考点：  
1. **协议深度**：TCP/IP、HTTP/2、QUIC（结合业务场景）。  
2. **工具链实战**：tcpdump、Wireshark、ss、iptables。  
3. **调优能力**：内核参数、组件配置（如Kafka的`num.network.threads`）。  
4. **架构设计**：跨机房容灾、高吞吐网络方案。  
5. **云原生网络**：Kubernetes CNI、Service Mesh（如Istio）。  

**加分项**：  
- 熟悉DPDK、RDMA等高性能网络技术。  
- 有大规模集群网络故障排查经验（如全网卡丢包、交换机故障）。
