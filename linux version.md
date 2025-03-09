### **一、基础知识 & 系统管理**

#### **1. 文件系统与权限**
**Q1: 如何理解 `inode`？如何查看文件的 inode 信息？**  
- **inode** 是文件系统中的一个数据结构，存储文件的元信息（如权限、大小、时间戳、数据块位置等），但不包含文件名。  
- **查看 inode**:  
  ```bash
  df -i               #查看 inode 使用情况
  ls -i filename      # 查看文件的 inode 号
  stat filename       # 显示 inode 的详细信息
  ```
即使文件名改变，inode 仍然不变，只需修改目录项，而数据不会受到影响。


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
- **创建硬链接**: ，就是使用 ln 命令，不加 -s 参数。例如：
  ```bash
  ln source_file hard_link_name
  ```
ln -s 命令用于创建符号链接（软链接）
硬链接为文件创建了一个新的目录项，这个新目录项和原文件共享同一个 inode，即文件在磁盘上的实际数据。
如果原文件被删除，只要硬链接存在，文件数据仍然可以访问。

#### **2. 进程管理**
**Q1: 如何查看进程的父子关系？**  
```bash
pstree -p          # 显示进程树结构
ps -ef --forest    # 显示进程层级
```
- 进程的父子关系指的是当一个进程（父进程）创建另一个进程（子进程）时形成的一种层次关系。
- 父进程通常通过系统调用（如 Unix/Linux 下的 fork()）创建子进程。每个进程都有一个唯一的进程标识符（PID）。子进程会记录父进程的标识符（PPID），从而表明它的父进程是谁。
- 父进程通常负责监控或等待子进程的执行结束，使用诸如 wait() 或 waitpid() 等系统调用来回收子进程退出后的资源。如果父进程没有妥善处理，子进程结束后可能会成为僵尸进程（Zombie Process）。
- 这种父子关系可以形成一棵进程树，初始进程（如 Unix 下的 init 或 systemd）位于树的根部，而所有其他进程都是它的后代。

**Q2: `kill -9` 和 `kill -15` 的区别？强制终止进程的后果？**  
- `kill -15`（默认）: 发送 SIGTERM，允许进程清理资源后退出。  
- `kill -9`: 发送 SIGKILL，强制终止进程，可能导致资源未释放（如临时文件、锁）。  
```bash
kill -9 <PID>   # 立即强制终止进程
kill -15 <PID>  # 尝试优雅地终止进程（默认）
killall -9 nginx   # 杀死所有名为 nginx 的进程,多个进程
```
killall / pkill：按进程名终止多个进程。

**Q3: 如何让进程在后台运行并脱离终端？**  
```bash
nohup command &            # 忽略 SIGHUP 信号
nohup python my_script.py & #适用于让进程在后台运行并避免终端关闭后进程被杀死。
disown                      # 将当前作业移出终端会话
```

---

#### **3. 存储管理**
**Q1: 如何查看磁盘 I/O 使用情况？**  
```bash
iostat -x 1          # 查看磁盘 I/O 详细统计（包括 %util 磁盘 I/O 利用率，接近 100% 说明磁盘负载高。 、await I/O 请求的平均等待时间，越高表示磁盘繁忙。）
iotop                # 实时监控进程的 I/O 使用
```
- 类型	特点	适用场景
- 同步 I/O	进程等待 I/O 完成	传统文件读写
- 异步 I/O	进程不阻塞，I/O 完成后通知	高性能数据库、Web 服务器
- 磁盘 I/O 直接影响数据库查询性能，*SSD* 比传统 HDD 在 IOPS（每秒输入输出操作）上更快。
- 服务器上的 高磁盘 I/O 可能意味着数据库查询过多、日志写入频繁或 Swap 使用率过高。
- 交换空间（Swap）：当 RAM 不足时，系统会将数据移动到 Swap 分区（磁盘上的交换空间），但 Swap 访问速度比 RAM 慢，会影响性能。当系统运行的进程占满 RAM 时，Linux 会将长期不活跃的内存数据移动到 Swap，腾出 RAM 供活跃进程使用。可以增加物理内存（最佳方案），优化 Swap 使用策略（调整 swappiness），使用 SSD 作为 Swap（提升读写速度，但影响寿命）
- 如果没有 Swap，RAM 被占满时，系统可能会直接杀死进程（OOM Killer 机制）。
- 启用 Swap 可以让系统在内存不足时有缓冲空间，避免突然终止关键进程。swamping值越低越依赖 RAM
  
**Q2: 如何快速定位占用磁盘空间最大的文件或目录？**  
```bash
du -sh /*               # 查看根目录下各目录大小
du -ah /path | sort -rh | head -n 10  # 显示前 10 大文件
```
df -h	显示整个磁盘的使用情况	检查磁盘是否满了； du -sh /path	显示目录/文件的大小	找出哪些文件/目录占用空间


**Q3: LVM 的工作原理？如何扩展逻辑卷？**  
- **LVM**：逻辑卷管理，通过 PV（物理卷）物理存储设备，如硬盘 /dev/sdb、VG（卷组）由多个 PV 组成的存储池 、LV（逻辑卷）实现动态扩容。  
- **扩展逻辑卷步骤**:  
  ```bash
  lvextend -L +10G /dev/vgname/lvname  # 扩展 LV
  resize2fs /dev/vgname/lvname         # 调整 ext4 文件系统（xfs 用 xfs_growfs）
  xfs_growfs /dev/my_vg/my_lv          #如果是xfs系统
  pvcreate /dev/sdc
  vgreduce my_vg /dev/sdb              # 从卷组中移除旧磁盘
  ```
  LVM（逻ical Volume Manager，逻辑卷管理器）是 Linux 中的一种 磁盘管理方式，LVM 可以在多个磁盘上创建一个 VG，从而突破单个磁盘的容量限制。

---

#### **4. 服务管理**
**Q1: Systemd 和 SysVinit 的区别？**  
- **Systemd**：并行启动服务、依赖管理、日志统一（journalctl）。 systemctl 是 Systemd 的核心工具，用于管理 Linux 服务器上的服务（如启动、停止、重启、查看状态等）。 
- **常用命令**:  
  ```bash
  systemctl start/stop/status servicename
  ```
Systemd：更标准化，所有服务用 systemctl 统一管理。journald，二进制日志，使用 target（如 multi-user.target），BIOS → Bootloader（GRUB） → 加载内核 → `systemd`（PID=1） → 目标（target） → 登录
SysVinit：依赖手动编写 init.d 脚本，管理方式不统一。传统 syslog，文本日志，运行级别（runlevel），BIOS → Bootloader（GRUB） → 加载内核（kernel） → `init`（PID=1） → 运行 `/etc/rc.d/` 脚本 → 登录

**Q2: 如何查看服务的启动日志？**  
```bash
journalctl -u servicename       # 查看特定服务的日志
```
nginx 是一个高性能 Web 服务器，适用于反向代理、负载均衡等。

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
- sync作用：强制将所有 脏数据（dirty data） 从内存同步（flush）到磁盘。 原因：避免数据丢失，确保数据已经写入磁盘，而不是仅停留在缓存中。
- 为什么要先 sync？
Linux 使用 写缓存（write buffer），数据可能还在内存中，尚未真正写入磁盘。如果 drop_caches 直接清理缓存，数据可能会丢失。sync 先确保数据写入磁盘，再清理缓存，防止数据丢失。
- echo 3 设置该参数为 3，表示清理所有缓存, echo 1：只清理 PageCache（通常最安全）。echo 2：只清理 inode 和目录缓存。
---

#### **2. 工具使用**
**Q1: 使用 `vmstat` 分析性能瓶颈**  
```bash
vmstat 1          # 每 1 秒输出一次（关注 si/so 交换、us/sy CPU 使用）
```
- si	Swap In（从 Swap 交换到内存，单位 KB/s）。如果 si > 0，表示系统正在从 Swap 取回数据，可能是内存不足。
- so	Swap Out（从内存交换到 Swap，单位 KB/s）。如果 so > 0，表示系统在把数据写入 Swap，可能需要增加 RAM。
- bi	Block In（从磁盘读取的数据量，单位 KB/s）。
- bo	Block Out（写入磁盘的数据量，单位 KB/s）。
- us	用户态 CPU 使用率（User Time）。表示用户进程（应用程序） 占用 CPU 的比例。
- sy	内核态 CPU 使用率（System Time）。表示内核进程和系统调用 占用 CPU 的比例。
- id	空闲 CPU 时间（Idle Time）。如果 id 很低（接近 0%），说明 CPU 被大量占用。
- wa	等待 I/O（IO Wait）。如果 wa 很高，表示磁盘 I/O 可能是性能瓶颈。
- st	Steal Time（仅在虚拟化环境中出现）。表示 VM 被 Hypervisor 抢占 CPU 资源的时间（通常出现在 VPS/云服务器上）。
- us + sy ≈ 100%：CPU 过载，可能需要优化应用。wa 高（如 wa > 20%）：可能是磁盘 I/O 瓶颈，需要优化磁盘读写。id 低（如 id < 5%）：CPU 可能是瓶颈，需优化程序或增加 CPU 核心数。

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
net.core.somaxconn	最大 TCP 连接排队数（提高高并发能力）	128


**Q2: 调整文件系统挂载参数**  
在 `/etc/fstab` 中添加 `noatime,nobarrier`（减少磁盘写入）。  
/etc/fstab 是 Linux 和 Unix 系统中的 文件系统表（Filesystem Table），它用于定义 磁盘分区、网络存储（NFS）、交换分区（swap） 等文件系统的自动挂载规则。

---

### **三、网络与安全**

#### **1. 网络配置**
**Q1: 如何查看 TCP 连接状态？**  
```bash
ss -antp         # 显示所有 TCP 连接及进程信息
```
-a	显示所有连接，包括监听（LISTEN） 和 已建立（ESTABLISHED）
-n	以 数值格式 显示 IP 地址和端口（避免解析为域名，加快查询速度）
-t	仅显示 TCP 连接（默认 ss 也可以显示 UDP，但这里只看 TCP）
-p	显示进程（PID），查看是哪个进程在使用端口

**Q2: 用 `tcpdump` 抓取特定端口的流量**  
```bash
tcpdump -i eth0 port 80 -w capture.pcap
```
-i eth0	指定网络接口（这里是 eth0，可换成 ens33、wlan0 等）
-w capture.pcap	将抓到的数据包保存到 capture.pcap 文件，用于后续分析

---

#### **2. 防火墙与安全**
**Q1: 用 `iptables` 限制 IP 访问**  
```bash
iptables -A INPUT -s 192.168.1.100 -j DROP
```
- 指定源 IP（-s = --source），即来自 192.168.1.100 的流量，-j DROP	拒绝（丢弃）该 IP 发送的所有数据包，不返回任何响应

**Q2: 排查 SSH 连接缓慢**  
检查服务端配置 `/etc/ssh/sshd_config`:  
```bash
UseDNS no          # 禁用 DNS 反向解析
GSSAPIAuthentication no  # 禁用 GSSAPI 认证
```
SSH（Secure Shell，安全外壳协议）是一种用于 远程管理服务器和设备 的加密网络协议，主要作用是提供安全的远程登录、命令执行和数据传输。
SSH 提供 SCP 和 SFTP，用于在本地和远程服务器之间加密传输文件。
- 本地端口转发
```bash
ssh -L 8080:localhost:3306 user@192.168.1.100
```
- 远程端口转发
```bash
ssh -R 9000:localhost:80 user@192.168.1.100
```
- scp（Secure Copy）用法
从本地复制文件到远程服务器
```bash
scp myfile.txt user@192.168.1.100:/home/user/
```
指定端口（非默认 22 端口）
```bash
scp -P 2222 myfile.txt user@192.168.1.100:/home/user/
```
从远程服务器下载文件到本地
```bash
scp user@192.168.1.100:/home/user/myfile.txt .
```
连接远程服务器
```bash
sftp user@192.168.1.100
```
下载文件
```bash
sftp> get myfile.txt
```
上传文件
```bash
sftp> put myfile.txt
```
断点续传
```bash
sftp> reget largefile.iso
```
断点续传，scp 不支持，但 sftp 可以 reget
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
### **指令：**
更改系统语言：
```bash
[ray@CentOSTest ~]$ locale
LANG=zh_CN.UTF-8
[ray@CentOSTest ~]$ sudo localectl set-locale LANG=en_US.UTF-8
[ray@CentOSTest ~]$ localectl status
   System Locale: LANG=en_US.UTF-8
       VC Keymap: us
      X11 Layout: us
```
who：查看谁在线
netstat -a：查看网路连接状态
```bash
roto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 localhost:10248         0.0.0.0:*               LISTEN     
tcp        0      0 localhost:2381          0.0.0.0:*               LISTEN  
```
ps -aux：查看背景执行状态
```bash
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.0 167484 12536 ?        Ss   07:04   0:04 /sbin/init
root           2  0.0  0.0      0     0 ?        S    07:04   0:00 [kthreadd]
```
### **文件：**
- usr/share/doc: 大部分指令或者软件制作者，都会将说明文档存放到这里面
