### **一、基础知识 & 系统管理**

#### **1. 文件系统与权限**
**Q1: 如何理解 `inode`？如何查看文件的 inode 信息？**  
- **inode** 是文件系统中的一个数据结构，存储文件的元信息（如权限、大小、时间戳、数据块位置等），但不包含文件名。  
- **查看 inode**:  
  ```bash
  ls -i filename      # 查看文件的 inode 号
  stat filename       # 显示 inode 的详细信息
  ```

**Q2: `chmod 755` 和 `chmod u=rwx,g=rx,o=rx` 的区别？**  
- 两者等价，均表示：  
  - 所有者（u）：读、写、执行（7 = 4+2+1）  
  - 组（g）和其他用户（o）：读、执行（5 = 4+1）。  

**Q3: 如何查找系统中所有权限为 777 的文件？**  
```bash
find / -type f -perm 0777 -print  # 查找所有权限为 777 的普通文件
```
-type f 表示普通文件（regular file）。除了 f 之外，find 命令还可以根据不同的文件类型进行过滤，常见的文件类型标识包括：
d：目录（directory）
l：符号链接（symbolic link）
c：字符设备文件（character device file）
b：块设备文件（block device file）
p：命名管道（FIFO，也称为“有名管道”）


**Q4: 软链接和硬链接的区别？如何创建软链接？**  
- **软链接**：类似快捷方式，指向文件名，可跨文件系统，删除源文件后失效。  
- **硬链接**：直接指向 inode，不可跨文件系统，删除源文件不影响硬链接。  
- **创建软链接**:  
  ```bash
  ln -s /path/to/source /path/to/link
  ```
ln -s 命令用于创建符号链接（软链接）

---

#### **2. 进程管理**
**Q1: 如何查看进程的父子关系？**  
```bash
pstree -p          # 显示进程树结构
ps -ef --forest    # 显示进程层级
```

**Q2: `kill -9` 和 `kill -15` 的区别？强制终止进程的后果？**  
- `kill -15`（默认）: 发送 SIGTERM，允许进程清理资源后退出。  
- `kill -9`: 发送 SIGKILL，强制终止进程，可能导致资源未释放（如临时文件、锁）。  

**Q3: 如何让进程在后台运行并脱离终端？**  
```bash
nohup command &            # 忽略 SIGHUP 信号
disown                      # 将当前作业移出终端会话
```

---

#### **3. 存储管理**
**Q1: 如何查看磁盘 I/O 使用情况？**  
```bash
iostat -x 1          # 查看磁盘 I/O 详细统计（包括 %util、await）
iotop                # 实时监控进程的 I/O 使用
```

**Q2: 如何快速定位占用磁盘空间最大的文件或目录？**  
```bash
du -sh /*               # 查看根目录下各目录大小
du -ah /path | sort -rh | head -n 10  # 显示前 10 大文件
```

**Q3: LVM 的工作原理？如何扩展逻辑卷？**  
- **LVM**：逻辑卷管理，通过 PV（物理卷）、VG（卷组）、LV（逻辑卷）实现动态扩容。  
- **扩展逻辑卷步骤**:  
  ```bash
  lvextend -L +10G /dev/vgname/lvname  # 扩展 LV
  resize2fs /dev/vgname/lvname         # 调整 ext4 文件系统（xfs 用 xfs_growfs）
  ```

---

#### **4. 服务管理**
**Q1: Systemd 和 SysVinit 的区别？**  
- **Systemd**：并行启动服务、依赖管理、日志统一（journalctl）。  
- **常用命令**:  
  ```bash
  systemctl start/stop/status servicename
  ```

**Q2: 如何查看服务的启动日志？**  
```bash
journalctl -u servicename       # 查看特定服务的日志
```

---

### **二、性能优化与监控**

#### **1. 性能指标分析**
**Q1: 如何分析系统负载？**  
- `uptime`: 显示 1/5/15 分钟的平均负载。  
- `top`/`htop`: 查看实时 CPU、内存、进程占用。

**Q2: 如何排查 CPU 使用率过高？**  
1. `top` 找到高 CPU 进程。  
2. `perf top` 分析函数级热点。  
3. `strace -p PID` 追踪系统调用。

**Q3: 内存中的 `buffer` 和 `cache` 区别？如何释放缓存？**  
- **buffer**: 内核缓存磁盘块数据（写操作）。  
- **cache**: 缓存文件内容（读操作）。  
- **释放缓存**:  
  ```bash
  sync && echo 3 > /proc/sys/vm/drop_caches  # 释放 pagecache、dentries、inodes
  ```

---

#### **2. 工具使用**
**Q1: 使用 `vmstat` 分析性能瓶颈**  
```bash
vmstat 1          # 每 1 秒输出一次（关注 si/so 交换、us/sy CPU 使用）
```

**Q2: 如何用 `strace` 追踪进程的系统调用？**  
```bash
strace -p PID -e trace=open,read,write  # 追踪指定进程的特定系统调用
```

---

#### **3. 调优实战**
**Q1: 优化内核参数**  
编辑 `/etc/sysctl.conf`，调整：  
```bash
net.core.somaxconn = 65535     # 提高 TCP 连接队列长度
vm.swappiness = 10             # 减少交换分区使用
```

**Q2: 调整文件系统挂载参数**  
在 `/etc/fstab` 中添加 `noatime,nobarrier`（减少磁盘写入）。  

---

### **三、网络与安全**

#### **1. 网络配置**
**Q1: 如何查看 TCP 连接状态？**  
```bash
ss -antp         # 显示所有 TCP 连接及进程信息
```

**Q2: 用 `tcpdump` 抓取特定端口的流量**  
```bash
tcpdump -i eth0 port 80 -w capture.pcap
```

---

#### **2. 防火墙与安全**
**Q1: 用 `iptables` 限制 IP 访问**  
```bash
iptables -A INPUT -s 192.168.1.100 -j DROP
```

**Q2: 排查 SSH 连接缓慢**  
检查服务端配置 `/etc/ssh/sshd_config`:  
```bash
UseDNS no          # 禁用 DNS 反向解析
GSSAPIAuthentication no  # 禁用 GSSAPI 认证
```

---

### **四、脚本开发与自动化**

#### **1. Shell 脚本**
**Q1: 批量重命名日志文件**  
```bash
for file in *.log; do
  mv "$file" "$(date +%Y%m%d)-${file}"
done
```

**Q2: 用 `awk` 提取文本字段**  
```bash
awk '{print $1, $3}' access.log  # 输出第 1 和第 3 列
```

---

### **五、大数据场景问题**

#### **1. Hadoop/Spark**
**Q1: 排查 HDFS 写入慢**  
- 检查 DataNode 磁盘 I/O（`iostat -x 1`）。  
- 网络带宽（`iftop`）。  

**Q2: 调整 JVM 参数**  
```bash
export JAVA_OPTS="-Xmx4g -XX:+UseG1GC"  # 堆内存 4G，使用 G1 垃圾回收
```

---

### **六、故障排查实战**

#### **1. 磁盘空间未满但报错 "No space left on device"**  
检查 inode 使用：  
```bash
df -i          # 查看 inode 使用率
```

---

### **七、附加考点**

#### **1. 零拷贝（Zero-Copy）技术**  
- 通过 `sendfile()` 系统调用实现，减少数据在用户态和内核态之间的拷贝，常用于 Kafka、Nginx 等高性能场景。  
