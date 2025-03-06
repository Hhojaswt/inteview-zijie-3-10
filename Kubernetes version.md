- Docker 是一种容器化技术，用于打包、分发和运行应用程序。它允许开发者在一个独立的环境中运行应用，不受操作系统差异的影响。负责创建、运行容器，但不擅长管理大量的容器。Docker 解决的是“如何打包和运行单个应用”。
- Kubernetes（简称 K8s）是一个容器编排（管理）工具，用于自动化容器的部署、扩展和管理。它主要解决 Docker 容器在生产环境中的调度和管理问题。负责管理 Docker 容器，包括调度、负载均衡、故障恢复等。Kubernetes 解决的是“如何管理和调度成百上千个 Docker 容器”。
- 镜像可以用来创建 Docker 容器，容器就是镜像的运行实例。
- Node 是物理机或虚拟机：每个 Node 提供 CPU、内存、存储和网络等计算资源，是 Kubernetes 集群中运行 Pod 的基础单位。
- Pod 是部署在 Node 上的最小单元：Pod 封装了一个或多个紧密耦合的容器，这些容器共享同一个网络命名空间和存储卷。Pod 是应用部署的基本单位，Kubernetes 调度器会将 Pod 分配到合适的 Node 上运行。

| **对比项	Docker** | **镜像（Image）**     | **容器（container）**  |
|----------|------------------------------|------------------------|
| 定义   | 运行应用的模板      | 镜像的运行实例 |
| 存储   | 只读，不能修改      | 读写，运行时可更改 |
| 用途   | 供容器使用          | 实际运行的应用 |


#### **一、Docker 核心问题**
**1. Docker 镜像与容器的区别是什么？**  
- **镜像**：只读模板，包含应用代码、依赖和配置，通过 Dockerfile 定义。  
- **容器**：镜像的运行实例，具有可写层（存储运行时数据），生命周期短暂。  

**2. 如何优化 Docker 镜像大小？**  
- **方法**：  
  1. **多阶段构建**：编译阶段和生产阶段分离。  
     ```dockerfile
     FROM golang:1.18 AS builder  #使用 golang:1.18 作为基础镜像，提供了 Go 语言的编译环境。
     WORKDIR /app   #将工作目录设置为 /app
     COPY . .       #将当前目录下的所有文件复制到容器中的 /app 目录。
     RUN go build -o myapp .  #将 Go 应用编译成二进制文件，输出文件命名为 myapp。

     FROM alpine:latest
     COPY --from=builder /app/myapp .
     CMD ["./myapp"]
     ```
  - 如果安装了 Python、Node.js、Java 等，镜像体积会变大。例如 python:3.9（≈100MB） vs. python:3.9-alpine（≈23MB）。
  - 镜像越小，下载速度越快，运行时占用的资源更少
    
  2. **选择轻量级基础镜像**：如 Alpine（5MB）替代 Ubuntu（70MB）
    
  3. **合并指令减少层数**：  
     ```dockerfile
     RUN apt-get update && apt-get install -y \
         package1 \
         package2 \
         && rm -rf /var/lib/apt/lists/*
     ```
     apt update 生成了 apt 缓存文件，apt install -y curl 依赖 apt update 的缓存，rm -rf /var/lib/apt/lists/* 只是删除了当前层的文件，但前面的层仍然保留这些缓存
     Docker 镜像使用分层存储（Layered Filesystem），每个层是只读的文件系统，多个层叠加形成最终的镜像。多个镜像可以共享相同的层，节省磁盘空间。Docker 可以复用已有的层，每个层都会增加镜像体积

**3. Docker 的网络模式有哪些？**  
- **bridge**： 默认模式，容器之间可以通信，但不能直接访问主机网络。但可以通过 docker network connect 连接到同一个网络。宿主机可以通过 http://localhost:8080 访问 Nginx。
  ```dockerfile
  docker run -d --name web -p 8080:80 nginx
  ```
- **host**：让容器使用宿主机的网络，性能高，但端口冲突风险高。
  ```dockerfile
  docker run -d --network host nginx
  ```
- **overlay**：用于 Kubernetes 或 Swarm，支持跨主机通信
  ```dockerfile
  docker network create -d overlay my_overlay
  docker service create --network my_overlay nginx
  ```
- **none**：容器没有任何网络，只能手动配置
  ```dockerfile
  docker run -d --network none nginx
  ```
- **Container**：让多个容器共享一个网络（如 Pod 内的容器）
  ```dockerfile
  docker run -d --name app1 nginx
  docker run -d --name app2 --network container:app1 busybox top
  ```  
- -d 让容器在后台运行，而不会占据当前终端。
  
**4. 如何实现 Docker 数据持久化？**  
- **Volume**：  
  ```bash
  docker volume create mydata
  docker run -v /host/path:/container/path nginx
  docker volume inspect mydata  #查看真实路径
  docker volume rm mydata   #删除
  ```
  Volume（数据卷） 是 Docker 提供的一种 持久化存储 机制。默认情况下，Docker 容器的数据存储在容器内部，如果容器被删除，数据就会丢失。Volume 允许数据在容器重启或删除后仍然保留。
  - /host/path：宿主机上的目录（本机路径）。
  - /container/path：容器内的目录（挂载路径）。
  - -v 代表 挂载（mount）一个数据卷（volume），用于在宿主机（host）和容器（container）之间共享数据。
  
- **Bind Mount**：直接挂载宿主机目录。
  ```bash
  docker run -d -v /data:/app nginx
  ```
  -  -d表示容器会在后台运行，但不会在终端显示任何输出。
  
- **tmpfs**：内存临时存储。  
- tmpfs（Temporary Filesystem，临时文件系统）是一种 基于内存的文件系统，它存储数据在 RAM（内存） 中，而不是磁盘上,比 SSD 还快,减少磁盘 IO，适用于高频读写但无需持久化的数据
  ```bash
  docker run -d --tmpfs /app/tmp:rw,size=100m,uid=1000 nginx
  ```
  - uid=1000：指定用户 ID，限制访问权限。
  - size=100m：最大 100MB，防止占满内存。
  - --tmpfs /app/tmp：在 /app/tmp 目录创建 tmpfs（数据存放在内存中）。 

**5. Docker Compose vs Docker Swarm**  
- **Compose**：单机多容器编排（通过 YAML 定义服务）。 Docker Compose 是一个 多容器管理工具，用于在单机上定义和运行多个 Docker 容器。
  ```bash
  version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    depends_on:
      - db

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
   运行一个 Web + MySQL 应用，docker-compose.yml 可能是这样的    
  docker-compose up -d
   运行 Compose
  ```
- **Swarm**：原生集群管理工具（已逐渐被 Kubernetes 取代）。 Docker Swarm 是 Docker 内置的 集群管理工具，用于在多台服务器上部署和管理容器。
  ```dockerfile
  docker swarm init --advertise-addr 192.168.1.100 #把当前机器设置为 Swarm 管理节点。
  docker swarm join --token <TOKEN> 192.168.1.100:2377 # 让其他服务器加入 Swarm 集群。
  docker service create --replicas 3 -p 8080:80 --name web nginx #在集群中 创建 3 个 Nginx 容器，并在 8080 端口提供服务。
  ```  
---

#### **二、Kubernetes 核心问题**
**1. Pod 是什么？为什么需要 Pod？**  
- **Pod**：Kubernetes 最小调度单元，包含一个或多个共享网络/存储的容器。 Kubernetes 不会直接管理容器，而是管理 Pod，每个 Pod 里可以运行一个或多个容器。Pod 内的容器共享 IP 地址、存储卷（Volume），可以通过 localhost 互相通信。
- **设计目的**：  
  - 支持紧密耦合的容器（如日志收集 Sidecar）。  
  - 共享资源（如 Volume、IP 地址）。  

**2. 服务发现与负载均衡如何实现？**  
- **Service**：Kubernetes 中的 Service 是一种抽象，用于定义一组 Pod 的访问策略和访问方式，并提供负载均衡功能，将流量分发到后端多个 Pod 上，Kubernetes 使用 kube-proxy 组件管理 Service 的虚拟 IP（ClusterIP）。kube-proxy 监听 API Server 的 Service 和 Endpoints 变化，并维护 iptables 或 IPVS 规则。
  - ClusterIP：只在集群内部暴露一个虚拟 IP 地址。流量会通过 kube-proxy 分发到后端 Pod。适合集群内部服务通信，不暴露外部访问。
  - NodePort：通过节点端口暴露服务。  外部访问通过 NodeIP:NodePort 实现，但不支持自动负载均衡
  - LoadBalancer：云厂商提供的负载均衡器。  外部流量先进入云负载均衡器，然后转发到集群中的节点，再由 kube-proxy 分发到 Pod。
- **Ingress**：HTTP/HTTPS 路由规则（如 Nginx Ingress Controller）。  


**3. Kubernetes 调度器如何工作？**  
- **流程**：  评估节点资源：调度器检查各个节点的 CPU、内存、存储等资源情况，确保分配的 Pod 不会导致节点资源耗尽。过将 Pod 分布到不同节点上，调度器帮助实现集群的负载均衡，调度器尽可能地提高集群资源利用率
  1. **过滤（Predicates）**：排除不满足条件的节点（如资源不足）。  
  2. **评分（Priorities）**：为剩余节点打分（如资源利用率最低者优先）。  
  3. **绑定（Bind）**：将 Pod 分配到最佳节点。  
- **自定义调度**：通过 `nodeSelector`、`affinity/anti-affinity` 或自定义调度器。  

**4. 如何配置持久化存储？**  
- **PersistentVolume (PV)**：集群存储资源（如 NFS、云盘）。  
- **PersistentVolumeClaim (PVC)**：用户对存储资源的请求。  
- **示例**：  
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  ```  

**5. 如何监控 Kubernetes 集群？**  
- **工具栈**：  
  - Prometheus + Grafana：采集指标并可视化。  
  - kubectl top：实时查看资源使用。  
  - EFK（Elasticsearch+Fluentd+Kibana）：日志收集与分析。  

---

#### **三、大数据组件与 Kubernetes 集成**
**1. 如何在 Kubernetes 上部署 Hadoop？**  
- **方案**：  
  1. **StatefulSet 管理 NameNode/DataNode**：确保持久化存储和稳定网络标识。  
  2. **ConfigMap 存储配置文件**：动态更新 `core-site.xml` 和 `hdfs-site.xml`。  
  3. **Service 暴露端口**：通过 ClusterIP 或 NodePort 访问 HDFS。  

**2. Kafka 在 Kubernetes 中的最佳实践**  
- **部署要点**：  
  1. **StatefulSet + Headless Service**：确保 Pod 稳定的网络标识（如 `kafka-0.kafka-headless.default.svc.cluster.local`）。  
  2. **持久化存储**：每个 Broker 绑定独立的 PV。  
  3. **反亲和性（anti-affinity）**：分散 Broker 到不同节点，避免单点故障。  
- **配置示例**：  
  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: kafka
  spec:
    serviceName: kafka-headless
    replicas: 3
    template:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app: kafka
                topologyKey: kubernetes.io/hostname
    volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 100Gi
  ```  

**3. 优化 Spark 作业在 Kubernetes 上的资源利用率**  
- **策略**：  
  1. **动态资源分配**：启用 `spark.dynamicAllocation.enabled=true`。使 Executor 数量根据作业负载自动扩展或缩减，从而避免资源闲置或不足。
 ```properties
       spark.dynamicAllocation.minExecutors=1
       spark.dynamicAllocation.maxExecutors=10
       spark.dynamicAllocation.initialExecutors=2
 ```
  2. **资源请求限制**：设置 `spec.containers.resources.limits/requests`. Spark Executor 以 Pod 的形式运行。合理设置 Pod 的 CPU 和内存的 requests 与 limits，可以帮助 Kubernetes 调度器更好地安排资源
 ```properties
--conf spark.kubernetes.executor.request.cores=0.5
--conf spark.kubernetes.executor.limit.cores=1
--conf spark.executor.memory=2g
--conf spark.executor.memoryOverhead=512m
 ```
  3. **本地存储缓存**：使用 `emptyDir` 或 HostPath 加速 Shuffle 过程。  

- 在 YARN 中：
- 资源管理主要依赖于 Container 资源（CPU、内存）的配置，通过 Capacity Scheduler 或 Fair Scheduler 分配资源。
- Spark on YARN 也支持动态资源分配，但调度和资源隔离主要由 YARN 控制。
- 通常需要合理设置 spark.yarn.executor.memoryOverhead 和 container 的资源请求/限制。
  
- 实际操作步骤和建议
- 1. 调优 spark-submit 参数： 根据作业负载和集群配置，调整 Spark 动态分配、Executor 数量、核心数、内存和内存开销参数。
- 2.配置 Kubernetes Executor Pod 在提交 Spark 作业时，通过参数设置（例如 spark.kubernetes.executor.request.cores、spark.kubernetes.executor.limit.cores、spark.executor.memory 等）为 Executor Pod 配置合适的资源请求和限制。
- 3. 使用 Pod Affinity/Anti-affinity： 如果你的集群中有其他重要服务，利用调度策略使 Spark Executor Pod 尽量不与这些服务共用同一节点，减少资源争抢。
- 4. 监控与调整：使用 kubectl top pods 和监控工具（如 Prometheus、Grafana）观察资源利用率，根据实际情况调整作业参数和集群配置。
- 5. 集群自动伸缩{auto scale}：如果在云环境下，启用集群自动伸缩功能，在负载高峰时自动扩容节点，从而为 Spark 作业提供更多资源，负载降低后缩减节点以节省成本。
---

#### **四、故障排查与安全**
**1. Pod 处于 Pending 状态的常见原因？**  
- **原因**：  
  1. 资源不足（CPU/内存）。Pod 定义中设置了特定的节点选择条件，但集群中没有符合条件的节点。
  2. 无可用节点满足亲和性规则。 集群中没有 Node 满足 Pod 所请求的 CPU、内存或其他资源需求，调度器找不到合适的节点来运行该 Pod。 Pod 配置中指定的资源请求超出节点可用资源。
  3. PVC 未绑定到可用 PV。
     - PersistentVolume (PV) 这是集群管理员预先配置的存储资源，提供了一定容量和访问模式，类似于独立的存储卷。
     - PersistentVolumeClaim (PVC) 这是用户或应用对存储资源的请求。用户定义所需的存储容量、访问模式等，系统会自动将 PVC 绑定到满足条件的 PV 上。如果有符合条件的 PV，PVC 会绑定到该 PV 上。如果没有符合条件的 PV，PVC 会处于等待状态，
     - PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 主要用于在 Kubernetes 集群中提供和管理持久存储，它们解决了容器化环境中数据持久化的问题。
     - StorageClass 用于定义存储提供者的类型（如 SSD、HDD、云盘等），PVC 可以引用一个 StorageClass，系统根据 StorageClass 动态分配 PV。
  4. CNI 插件未安装或配置错误：Pod 网络依赖于 CNI 插件，如果 CNI 插件未正确部署，Pod 可能无法获取 IP 地址，从而导致 Pending。
  5. 调度器未运行或异常：如果 Kubernetes 调度器出现问题，可能导致 Pod 无法被正常调度。
- **排查命令**：  
  ```bash
  kubectl describe pod <pod-name>   # 查看事件详情
  kubectl get events --sort-by=.metadata.creationTimestamp
  ```  

**2. 如何配置 Kubernetes RBAC？**  
- **步骤**：  
  1. **定义 Role**：  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: Role
     metadata:
       namespace: default
       name: pod-reader
     rules:
     - apiGroups: [""]
       resources: ["pods"]
       verbs: ["get", "watch", "list"]
     ```  
  2. **绑定 Role 到用户/ServiceAccount**：  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: RoleBinding
     metadata:
       name: read-pods
       namespace: default
     subjects:
     - kind: User
       name: alice
       apiGroup: rbac.authorization.k8s.io
     roleRef:
       kind: Role
       name: pod-reader
       apiGroup: rbac.authorization.k8s.io
     ```  

**3. 如何排查容器启动失败问题？**  
- **步骤**：  
  1. 查看 Pod 状态：`kubectl get pods`。  
  2. 查看日志：`kubectl logs <pod-name> [-c <container-name>]`。  
  3. 进入容器调试：`kubectl exec -it <pod-name> -- sh`。  
  4. 检查事件：`kubectl describe pod <pod-name>`。  

---

#### **五、开放性问题**
**1. 设计一个高可用的 Kubernetes 集群架构**  
- **关键点**：  
  1. **多 Master 节点**：使用 `kubeadm` 部署 3 个 Master，启用 `etcd` 集群。  
  2. **负载均衡**：Master 节点前部署负载均衡器（如 HAProxy）。  
  3. **Worker 节点跨可用区**：避免单机房故障。  
  4. **持久化存储**：使用云厂商的跨可用区存储（如 AWS EBS Multi-AZ）。  

**2. 如何实现 Kubernetes 集群的自动扩缩容？**  
- **方案**：  
  1. **Horizontal Pod Autoscaler (HPA)**：基于 CPU/内存或自定义指标扩缩 Pod。  
  2. **Cluster Autoscaler**：自动增减节点（需云厂商支持）。  
  3. **Vertical Pod Autoscaler (VPA)**：自动调整 Pod 资源请求。  

---

### **总结**
字节跳动大数据运维专家的 Kubernetes 和 Docker 面试重点包括：  
1. **核心概念**：Pod、Service、Volume、调度机制。  
2. **运维实践**：故障排查、性能优化、持久化存储配置。  
3. **集成能力**：Hadoop/Kafka/Spark 在 Kubernetes 上的部署与管理。  
4. **安全与治理**：RBAC、网络策略、镜像安全扫描。  
5. **架构设计**：高可用集群、自动扩缩容、跨云多集群管理。  

**加分项**：  
- 熟悉 Operator 模式（如 Spark Operator、Kafka Operator）。  
- 有大规模集群（1000+节点）的运维经验。  
- 掌握 Service Mesh（如 Istio）在大数据场景的应用。
