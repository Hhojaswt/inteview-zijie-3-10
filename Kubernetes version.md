- Docker 是一种容器化技术，用于打包、分发和运行应用程序。它允许开发者在一个独立的环境中运行应用，不受操作系统差异的影响。负责创建、运行容器，但不擅长管理大量的容器。Docker 解决的是“如何打包和运行单个应用”。
- Kubernetes（简称 K8s）是一个容器编排（管理）工具，用于自动化容器的部署、扩展和管理。它主要解决 Docker 容器在生产环境中的调度和管理问题。负责管理 Docker 容器，包括调度、负载均衡、故障恢复等。Kubernetes 解决的是“如何管理和调度成百上千个 Docker 容器”。
- 镜像可以用来创建 Docker 容器，容器就是镜像的运行实例。

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
     FROM golang:1.18 AS builder
     WORKDIR /app
     COPY . .
     RUN go build -o myapp .

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
- **Service**：  
  - ClusterIP：内部虚拟 IP，自动负载均衡到 Pod。  
  - NodePort：通过节点端口暴露服务。  
  - LoadBalancer：云厂商提供的负载均衡器。  
- **Ingress**：HTTP/HTTPS 路由规则（如 Nginx Ingress Controller）。  

**3. Kubernetes 调度器如何工作？**  
- **流程**：  
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
  1. **动态资源分配**：启用 `spark.dynamicAllocation.enabled=true`。  
  2. **资源请求限制**：设置 `spec.containers.resources.limits/requests`。  
  3. **本地存储缓存**：使用 `emptyDir` 或 HostPath 加速 Shuffle 过程。  

---

#### **四、故障排查与安全**
**1. Pod 处于 Pending 状态的常见原因？**  
- **原因**：  
  1. 资源不足（CPU/内存）。  
  2. 无可用节点满足亲和性规则。  
  3. PVC 未绑定到可用 PV。  
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
