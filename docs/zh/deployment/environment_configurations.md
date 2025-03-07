---
displayed_sidebar: docs
---

# 检查环境配置

本文列出了在部署 StarRocks 之前需要检查并配置的所有环境和系统配置项。正确设置这些配置项可以确保集群的高可用并提升性能。

## 端口

StarRocks 为不同的服务使用特定的端口。如果您在这些实例上部署了其他服务，请检查这些端口是否被占用。

### FE 端口

在用于 FE 部署的实例上，您需要检查以下端口：

- `8030`：FE HTTP Server 端口（`http_port`）
- `9020`：FE Thrift Server 端口（`rpc_port`）
- `9030`：FE MySQL Server 端口（`query_port`）
- `9010`：FE 内部通讯端口（`edit_log_port`）
- `6090`：FE 云原生元数据服务 RPC 监听端口（`cloud_native_meta_port`）

在 FE 实例上执行如下命令查看这些端口是否被占用：

```Bash
netstat -tunlp | grep 8030
netstat -tunlp | grep 9020
netstat -tunlp | grep 9030
netstat -tunlp | grep 9010
netstat -tunlp | grep 6090
```

如果上述任何端口被占用，您必须在部署 FE 节点时指定可用于替换的端口。详细说明参见 [手动部署 StarRocks - 启动 Leader FE 节点](../deployment/deploy_manually.md#第一步启动-leader-fe-节点)。

### BE 端口

在用于 BE 部署的实例上，您需要检查以下端口：

- `9060`：BE Thrift Server 端口（`be_port`）
- `8040`：BE HTTP Server 端口（`be_http_port`）
- `9050`：BE 心跳服务端口（`heartbeat_service_port`）
- `8060`：BE bRPC 端口（`brpc_port`）
- `9070`：BE 和 CN 的额外 Agent 服务端口。（`starlet_port`）

在 BE 实例上执行如下命令查看这些端口是否被占用：

```Bash
netstat -tunlp | grep 9060
netstat -tunlp | grep 8040
netstat -tunlp | grep 9050
netstat -tunlp | grep 8060
netstat -tunlp | grep 9070
```

如果上述任何端口被占用，您必须在部署 BE 节点时指定可用于替换的端口。详细说明参见 [部署 StarRocks - 启动 BE 服务](../deployment/deploy_manually.md#第二步启动-be-服务)。

### CN 端口

在用于 CN 部署的实例上，您需要检查以下端口：

- `9060`：CN Thrift Server 端口（`be_port`）
- `8040`：CN HTTP Server 端口（`be_http_port`）
- `9050`：CN 心跳服务端口（`heartbeat_service_port`）
- `8060`：CN bRPC 端口（`brpc_port`）
- `9070`：BE 和 CN 的额外 Agent 服务端口。（`starlet_port`）

在 CN 实例上执行如下命令查看这些端口是否被占用：

```Bash
netstat -tunlp | grep 9060
netstat -tunlp | grep 8040
netstat -tunlp | grep 9050
netstat -tunlp | grep 8060
netstat -tunlp | grep 9070
```

如果上述任何端口被占用，您必须在部署 CN 节点时指定可用于替换的端口。详细说明参见 [部署 StarRocks - 启动 CN 服务](../deployment/deploy_manually.md#第三步可选启动-cn-服务)。

## 主机名

如需为您的 StarRocks 集群 [启用 FQDN 访问](../administration/management/enable_fqdn.md)，您必须为每个实例设置一个主机名。

在每个实例的 **/etc/hosts** 文件中，您必须指定集群中其他实例的 IP 地址和相应的主机名。

> **注意**
>
> **/etc/hosts** 文件中的所有 IP 地址都必须是唯一。

## JDK 设置

StarRocks 依靠环境变量 `JAVA_HOME` 定位实例上的 Java 依赖项。

运行以下命令查看环境变量 `JAVA_HOME`：

```Bash
echo $JAVA_HOME
```

按照以下步骤设置 `JAVA_HOME`：

1. 在 **/etc/profile** 文件中设置 `JAVA_HOME`：

   ```Bash
   sudo  vi /etc/profile
   # 将 <path_to_JDK> 替换为 JDK 的安装路径。
   export JAVA_HOME=<path_to_JDK>
   export PATH=$PATH:$JAVA_HOME/bin
   ```

2. 使变更生效：

   ```Bash
   source /etc/profile
   ```

运行以下命令验证变更是否成功：

```Bash
java -version
```

## CPU Scaling Governor

该配置项为**可选配置项**。如果您的 CPU 不支持 Scaling Governor，则可以跳过该项。

CPU Scaling Governor 用于控制 CPU 能耗模式。如果您的 CPU 支持该配置项，建议您将其设置为 `performance` 以获得更好的 CPU 性能：

```Bash
echo 'performance' | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## 内存设置

### Memory Overcommit

Memory Overcommit 允许操作系统将额外的内存资源分配给进程。建议您启用 Memory Overcommit。

```Bash
# 修改配置文件。
cat >> /etc/sysctl.conf << EOF
vm.overcommit_memory=1
EOF
# 使修改生效。
sysctl -p
```

### Transparent Huge Pages

Transparent Huge Pages 默认启用。因其会干扰内存分配，进而导致性能下降，建议您禁用此功能。

```Bash
# 临时变更。
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
# 永久变更。
cat >> /etc/rc.d/rc.local << EOF
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
   echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
chmod +x /etc/rc.d/rc.local
```

### Swap Space

建议您禁用 Swap Space。

检查并禁用 Swap Space 操作步骤如下：

1. 关闭 Swap Space。

   ```SQL
   swapoff /<path_to_swap_space>
   swapoff -a
   ```

2. 从 **/etc/fstab** 文件中删除 Swap Space 信息。

   ```Bash
   /<path_to_swap_space> swap swap defaults 0 0
   ```

3. 确认 Swap Space 已关闭。

   ```Bash
   free -m
   ```

### Swappiness

Swappiness 会对性能造成影响，因此建议您禁用 Swappiness。

```Bash
# 修改配置文件。
cat >> /etc/sysctl.conf << EOF
vm.swappiness=0
EOF
# 使修改生效。
sysctl -p
```

## 存储设置

建议您根据所选用的存储介质来确定合适的调度算法。

您可以使用以下命令检查您当前使用的调度算法：

```Bash
cat /sys/block/${disk}/queue/scheduler
# 例如，运行 cat /sys/block/vdb/queue/scheduler
```

推荐您为 SATA 磁盘使用 mq-deadline 调度算法，为 NVMe 或 SSD 磁盘使用 kyber 调度算法。

### SATA

mq-deadline 调度算法适合 SATA 磁盘。

```Bash
# 临时变更。
echo mq-deadline | sudo tee /sys/block/${disk}/queue/scheduler
# 永久变更。
cat >> /etc/rc.d/rc.local << EOF
echo mq-deadline | sudo tee /sys/block/${disk}/queue/scheduler
EOF
chmod +x /etc/rc.d/rc.local
```

### SSD 和 NVMe

- 如果您的 NVMe 或 SSD 磁盘支持 kyber 调度算法。

   ```Bash
   # 临时变更。
   echo kyber | sudo tee /sys/block/${disk}/queue/scheduler
   # 永久变更。
   cat >> /etc/rc.d/rc.local << EOF
   echo kyber | sudo tee /sys/block/${disk}/queue/scheduler
   EOF
   chmod +x /etc/rc.d/rc.local
   ```

- 如果您的系统不支持 SSD 和 NVMe 的 kyber 调度算法，建议您使用 none（或 noop）调度算法。

   ```Bash
   # 临时变更。
   echo none | sudo tee /sys/block/vdb/queue/scheduler
   # 永久变更。
   cat >> /etc/rc.d/rc.local << EOF
   echo none | sudo tee /sys/block/${disk}/queue/scheduler
   EOF
   chmod +x /etc/rc.d/rc.local
   ```

## SELinux

建议您禁用 SELinux。

```Bash
# 临时变更。
setenforce 0
# 永久变更。
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
sed -i 's/SELINUXTYPE/#SELINUXTYPE/' /etc/selinux/config
```

## 防火墙

如果您启用了防火墙，请为 FE、BE 和 Broker 开启内部端口。

```Bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

## LANG 变量

您需要使用以下命令手动检查和配置 LANG 变量：

```Bash
# 修改配置文件。
echo "export LANG=en_US.UTF8" >> /etc/profile
# 使修改生效。
source /etc/profile
```

## 时区

请根据您所在的实际时区设置此项。

以下示例将时区设置为 `/Asia/Shanghai`。

```Bash
cp -f /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock
```

## ulimit 设置

如果**最大文件描述符**和**最大用户进程**的值设置得过小，StarRocks 运行可能会出现问题。建议您将系统资源上限调大。

```Bash
cat >> /etc/security/limits.conf << EOF
* soft nproc 65535
* hard nproc 65535
* soft nofile 655350
* hard nofile 655350
* soft stack unlimited
* hard stack unlimited
* hard memlock unlimited
* soft memlock unlimited
EOF

cat >> /etc/security/limits.d/20-nproc.conf << EOF 
*          soft    nproc     65535
root       soft    nproc     65535
EOF
```

## 文件系统配置

建议您使用 ext4 或 xfs 日志文件系统。您可以运行以下命令来检查挂载类型：

```Bash
df -Th
```

## 网络配置

### tcp_abort_on_overflow

如果系统当前因后台进程无法处理的新连接而溢出，则允许系统重置新连接：

```Bash
# 修改配置文件。
cat >> /etc/sysctl.conf << EOF
net.ipv4.tcp_abort_on_overflow=1
EOF
# 使修改生效。
sysctl -p
```

### somaxconn

设置监听 Socket 队列的最大连接请求数为 `1024`：

```Bash
# 修改配置文件。
cat >> /etc/sysctl.conf << EOF
net.core.somaxconn=1024
EOF
# 使修改生效。
sysctl -p
```

## NTP 设置

需要在 StarRocks 集群各节点之间配置时间同步，从而保证事务的线性一致性。您可以使用 pool.ntp.org 提供的互联网时间服务，也可以使用离线环境内置的 NTP 服务。例如，您可以使用云服务提供商提供的 NTP 服务。

1. 查看 NTP 时间服务器或 Chrony 服务是否存在。

   ```Bash
   rpm -qa | grep ntp
   systemctl status chronyd
   ```

2. 如不存在，运行以下命令安装 NTP 时间服务器。

   ```Bash
   sudo yum install ntp ntpdate && \
   sudo systemctl start ntpd.service && \
   sudo systemctl enable ntpd.service
   ```

3. 检查 NTP 服务。

   ```Bash
   systemctl list-unit-files | grep ntp
   ```

4. 检查 NTP 服务连接和监控状态。

   ```Bash
   netstat -tunlp | grep ntp
   ```

5. 检查服务是否与 NTP 服务器同步。

   ```Bash
   ntpstat
   ```

6. 检查网络中的 NTP 服务器。

   ```Bash
   ntpq -p
   ```

## 高并发配置

如果您的 StarRocks 集群负载并发较高，建议您进行如下配置.

### max_map_count

进程可以拥有的 VMA（虚拟内存区域）的数量。将该值调整为 `262144`：

```bash
# 修改配置文件。
cat >> /etc/sysctl.conf << EOF
vm.max_map_count = 262144
EOF
# 使修改生效。
sysctl -p
```

### 其他

```Bash
echo 120000 > /proc/sys/kernel/threads-max
echo 200000 > /proc/sys/kernel/pid_max
```
