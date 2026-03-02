# 附录 A — 命令速查表

本附录整理了网络学习和日常运维中最常用的命令，按使用场景分类。每条命令都包含：命令格式、含义说明、常用参数和示例输出。**Mac、Linux、Windows** 系统有差异的地方会特别标注。

---

## 1. 查看网络信息

### 1.1 查看 IP 地址

| 系统 | 命令 | 说明 | 示例输出 |
|------|------|------|----------|
| **Linux** | `ip addr` 或 `ip a` | 查看所有网络接口的 IP 地址 | `inet 192.168.1.100/24` |
| **Mac** | `ifconfig` | 查看所有网络接口信息 | `inet 192.168.1.100 netmask 0xffffff00` |
| **Windows** | `ipconfig` | 查看网络配置 | `IPv4 地址 . . . : 192.168.1.100` |
| **通用** | `hostname -I` (Linux) / `ipconfig getifaddr en0` (Mac) | 只显示主 IP 地址 | `192.168.1.100` |

**常用参数：**
```bash
# Linux - 只看某个网卡
ip addr show eth0

# Mac - 只看某个网卡
ifconfig en0

# 只显示 IPv4 地址
ip -4 addr  # Linux
```

**示例输出（Linux）：**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
    inet6 fe80::a00:27ff:fe4e:66a1/64 scope link
```

### 1.2 查看 MAC 地址

| 命令 | 说明 | 示例输出 |
|------|------|----------|
| `ip link show` (Linux) | 查看所有网卡的 MAC 地址 | `link/ether 08:00:27:4e:66:a1` |
| `ifconfig` (Mac) | 查看网卡信息（包含 MAC） | `ether 08:00:27:4e:66:a1` |
| `ipconfig /all` (Windows) | 查看完整网络配置（含 MAC） | `物理地址. . . . . . . . : 08-00-27-4E-66-A1` |
| `cat /sys/class/net/eth0/address` (Linux) | 直接读取某网卡 MAC | `08:00:27:4e:66:a1` |

### 1.3 查看网关

| 系统 | 命令 | 说明 | 示例输出 |
|------|------|------|----------|
| **Linux** | `ip route` 或 `ip r` | 查看路由表和默认网关 | `default via 192.168.1.1 dev eth0` |
| **Mac** | `netstat -nr` | 查看路由表 | `default 192.168.1.1 UGSc en0` |
| **Windows** | `route print` 或 `ipconfig` | 查看路由表/网关 | `0.0.0.0 0.0.0.0 192.168.1.1` |
| **通用** | `route -n` | 数字格式显示路由表 | `0.0.0.0 192.168.1.1 0.0.0.0 UG 0 0 0 eth0` |

**快速查看默认网关：**
```bash
# Linux
ip route | grep default

# Mac
netstat -nr | grep default
```

### 1.4 查看 DNS 配置

| 命令 | 说明 | 示例输出 |
|------|------|----------|
| `cat /etc/resolv.conf` | 查看系统 DNS 服务器配置 | `nameserver 8.8.8.8` |
| `ipconfig /all` (Windows) | 查看完整网络配置（含 DNS） | `DNS 服务器 . . . : 8.8.8.8` |
| `systemd-resolve --status` (Linux) | 查看详细 DNS 状态 | 显示每个网卡的 DNS 服务器 |
| `scutil --dns` (Mac) | 查看 DNS 配置详情 | 显示所有 DNS 解析器配置 |

**示例输出：**
```
nameserver 8.8.8.8
nameserver 8.8.4.4
search localdomain
```

### 1.5 查看 ARP 表

| 命令 | 说明 | 示例输出 |
|------|------|----------|
| `arp -a` | 查看 ARP 缓存表（IP 到 MAC 映射） | `192.168.1.1 at 00:11:22:33:44:55 on eth0` |
| `ip neigh` (Linux) | 查看邻居表（ARP 表） | `192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE` |
| `arp -n` | 不解析主机名，只显示 IP | 数字格式显示 |

### 1.6 查看路由表

| 命令 | 说明 | 适用系统 |
|------|------|----------|
| `ip route show` | 显示路由表 | Linux |
| `route -n` | 数字格式显示路由表 | Linux/Mac |
| `netstat -rn` | 显示路由表 | Mac/Linux |

**示例输出：**
```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

### 1.7 查看公网 IP

| 命令 | 说明 | 示例输出 |
|------|------|----------|
| `curl ifconfig.me` | 查询公网 IP | `203.0.113.45` |
| `curl ip.sb` | 查询公网 IP（备选） | `203.0.113.45` |
| `curl ipinfo.io/ip` | 查询公网 IP | `203.0.113.45` |
| `dig +short myip.opendns.com @resolver1.opendns.com` | 通过 DNS 查询公网 IP | `203.0.113.45` |

---

## 2. 测试连通性

### 2.1 ping 测试

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `ping IP/域名` | 测试网络连通性和延迟 | `-c 4` 发送 4 个包后停止 |
| `ping -c 10 baidu.com` | 发送 10 个 ICMP 包 | `-i 0.5` 间隔 0.5 秒 |
| `ping -s 1000 IP` | 指定包大小（字节） | `-W 2` 超时时间 2 秒 |

**示例输出：**
```
PING baidu.com (110.242.68.66): 56 data bytes
64 bytes from 110.242.68.66: icmp_seq=0 ttl=52 time=28.5 ms
64 bytes from 110.242.68.66: icmp_seq=1 ttl=52 time=27.8 ms

--- baidu.com ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 27.8/28.2/28.5/0.4 ms
```

### 2.2 traceroute 路由追踪

| 命令 | 说明 | 适用系统 |
|------|------|----------|
| `traceroute 域名/IP` | 追踪数据包到目标的路径 | Linux/Mac |
| `tracert 域名/IP` | 追踪路由（Windows） | Windows |
| `traceroute -n 域名` | 不解析主机名（更快） | Linux/Mac |
| `mtr 域名` | 持续追踪路由（结合 ping 和 traceroute） | Linux/Mac（需安装） |

**示例输出：**
```
traceroute to baidu.com (110.242.68.66), 30 hops max
 1  192.168.1.1  1.234 ms  0.987 ms  1.123 ms
 2  10.0.0.1  5.678 ms  5.432 ms  5.890 ms
 3  110.242.68.66  28.456 ms  27.890 ms  28.123 ms
```

### 2.3 telnet 端口测试

| 命令 | 说明 | 示例 |
|------|------|------|
| `telnet IP 端口` | 测试 TCP 端口是否开放 | `telnet 192.168.1.100 80` |

**示例输出：**
```
# 端口开放
Trying 192.168.1.100...
Connected to 192.168.1.100.

# 端口关闭
Trying 192.168.1.100...
telnet: Unable to connect to remote host: Connection refused
```

### 2.4 nc (netcat) 端口测试

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `nc -zv IP 端口` | 测试端口连通性 | `-z` 扫描模式，`-v` 详细输出 |
| `nc -zv IP 端口范围` | 扫描端口范围 | `nc -zv 192.168.1.1 20-80` |
| `nc -l 端口` | 监听端口（服务端） | `-l` 监听模式 |
| `Test-NetConnection IP -Port 端口` | PowerShell 测试端口 | Windows PowerShell |

**示例输出：**
```
Connection to 192.168.1.100 80 port [tcp/http] succeeded!
```

### 2.5 curl 测试 HTTP

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `curl -I URL` | 只获取 HTTP 响应头 | `-I` 仅头部 |
| `curl -v URL` | 显示详细请求过程 | `-v` 详细模式 |
| `curl -o /dev/null -s -w "%{http_code}\n" URL` | 只显示 HTTP 状态码 | `-s` 静默模式 |

**示例输出：**
```
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
Content-Length: 1234
```

---

## 3. DNS 相关

### 3.1 nslookup 域名查询

| 命令 | 说明 | 示例 |
|------|------|------|
| `nslookup 域名` | 查询域名的 IP 地址 | `nslookup baidu.com` |
| `nslookup 域名 DNS服务器` | 指定 DNS 服务器查询 | `nslookup baidu.com 8.8.8.8` |

**示例输出：**
```
Server:    8.8.8.8
Address:   8.8.8.8#53

Non-authoritative answer:
Name:   baidu.com
Address: 110.242.68.66
```

### 3.2 dig 域名查询（推荐）

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `dig 域名` | 详细查询域名信息 | 显示完整 DNS 响应 |
| `dig 域名 +short` | 只显示查询结果 IP | `+short` 简洁输出 |
| `dig @8.8.8.8 域名` | 指定 DNS 服务器查询 | `@DNS服务器` |
| `dig 域名 A` | 查询 A 记录（IPv4） | `A` / `AAAA` / `MX` / `NS` |
| `dig 域名 ANY` | 查询所有记录类型 | `ANY` 所有类型 |
| `dig -x IP` | 反向查询（IP 到域名） | `-x` 反向查询 |

**示例输出：**
```
; <<>> DiG 9.10.6 <<>> baidu.com
;; ANSWER SECTION:
baidu.com.    600    IN    A    110.242.68.66
baidu.com.    600    IN    A    39.156.66.10

;; Query time: 28 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

### 3.3 清除 DNS 缓存

| 系统 | 命令 | 说明 |
|------|------|------|
| **Windows** | `ipconfig /flushdns` | 清除 DNS 缓存 |
| **Mac** | `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` | 清除 DNS 缓存 |
| **Linux (systemd)** | `sudo systemd-resolve --flush-caches` | 清除 systemd-resolved 缓存 |
| **Linux (nscd)** | `sudo /etc/init.d/nscd restart` | 重启 nscd 服务 |

---

## 4. 端口和服务

### 4.1 查看监听端口

| 系统 | 命令 | 说明 | 常用参数 |
|------|------|------|----------|
| **Linux** | `ss -tlnp` | 查看 TCP 监听端口和进程 | `-t` TCP, `-l` 监听, `-n` 数字, `-p` 进程 |
| **Linux** | `netstat -tlnp` | 查看 TCP 监听端口（旧命令） | 同上 |
| **Mac** | `lsof -i -P` | 查看所有网络连接 | `-i` 网络, `-P` 数字端口 |
| **Mac** | `netstat -an \| grep LISTEN` | 查看监听端口 | `-a` 所有, `-n` 数字 |
| **Windows** | `netstat -ano \| findstr LISTENING` | 查看监听端口 | `-a` 所有, `-n` 数字, `-o` 进程ID |

**示例输出（ss -tlnp）：**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
LISTEN   0        128      0.0.0.0:22          0.0.0.0:*           users:(("sshd",pid=1234,fd=3))
LISTEN   0        128      0.0.0.0:80          0.0.0.0:*           users:(("nginx",pid=5678,fd=6))
```

### 4.2 查看某个端口被谁占用

| 命令 | 说明 | 示例 |
|------|------|------|
| `lsof -i :端口` | 查看指定端口的占用情况 | `lsof -i :80` |
| `ss -tlnp \| grep :端口` (Linux) | 通过 ss 查看端口 | `ss -tlnp \| grep :80` |
| `netstat -tlnp \| grep :端口` | 通过 netstat 查看端口 | `netstat -tlnp \| grep :80` |

**示例输出：**
```
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx    5678  root   6u   IPv4  12345      0t0  TCP *:80 (LISTEN)
```

### 4.3 查看所有网络连接

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `ss -anp` (Linux) | 查看所有网络连接 | `-a` 所有, `-n` 数字, `-p` 进程 |
| `netstat -anp` | 查看所有网络连接（旧命令） | 同上 |
| `ss -s` | 显示网络连接统计信息 | `-s` 统计 |

**示例输出：**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
ESTAB    0        0        192.168.1.100:22     192.168.1.200:54321 users:(("sshd",pid=9012))
```

---

## 5. 防火墙（Linux ufw）

### 5.1 UFW 基本命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `sudo ufw status` | 查看防火墙状态 | 显示规则列表 |
| `sudo ufw status verbose` | 查看详细状态 | 包含默认策略 |
| `sudo ufw enable` | 启用防火墙 | 开机自启 |
| `sudo ufw disable` | 禁用防火墙 | 关闭防火墙 |

### 5.2 UFW 规则管理

| 命令 | 说明 | 示例 |
|------|------|------|
| `sudo ufw allow 端口` | 允许某端口 | `sudo ufw allow 80` |
| `sudo ufw allow 端口/协议` | 指定协议 | `sudo ufw allow 53/udp` |
| `sudo ufw allow from IP` | 允许某 IP 的所有连接 | `sudo ufw allow from 192.168.1.100` |
| `sudo ufw allow from IP to any port 端口` | 允许某 IP 访问指定端口 | `sudo ufw allow from 192.168.1.100 to any port 22` |
| `sudo ufw deny 端口` | 拒绝某端口 | `sudo ufw deny 23` |
| `sudo ufw delete allow 端口` | 删除规则 | `sudo ufw delete allow 80` |
| `sudo ufw delete 规则编号` | 按编号删除规则 | `sudo ufw delete 3` |

**示例输出：**
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

---

## 6. SSH 相关

### 6.1 SSH 连接

| 命令 | 说明 | 常用参数 |
|------|------|----------|
| `ssh user@ip` | SSH 登录远程主机 | 基本连接 |
| `ssh -p 端口 user@ip` | 指定端口连接 | `-p` 端口号 |
| `ssh -i 私钥文件 user@ip` | 使用指定私钥 | `-i` 私钥路径 |
| `ssh -v user@ip` | 显示详细调试信息 | `-v` / `-vv` / `-vvv` |

### 6.2 SSH 密钥管理

| 命令 | 说明 | 示例 |
|------|------|------|
| `ssh-keygen -t ed25519` | 生成 ED25519 密钥对（推荐） | 更安全更快 |
| `ssh-keygen -t rsa -b 4096` | 生成 4096 位 RSA 密钥 | 传统方式 |
| `ssh-copy-id user@ip` | 复制公钥到远程主机 | 自动配置免密登录 |
| `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@ip` | 指定公钥文件 | `-i` 指定文件 |

**生成密钥示例：**
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# 会生成：
# ~/.ssh/id_ed25519 (私钥)
# ~/.ssh/id_ed25519.pub (公钥)
```

### 6.3 SSH 端口转发

| 命令 | 说明 | 使用场景 |
|------|------|----------|
| `ssh -L 本地端口:目标:目标端口 跳板机` | 本地端口转发 | 访问内网服务 |
| `ssh -R 远程端口:本地:本地端口 服务器` | 远程端口转发 | 内网穿透 |
| `ssh -D 1080 服务器` | 动态转发（SOCKS 代理） | 科学上网 |
| `ssh -N -f -L 本地端口:目标:目标端口 跳板机` | 后台运行转发 | `-N` 不执行命令, `-f` 后台 |

**本地转发示例：**
```bash
# 通过跳板机访问内网数据库
ssh -L 3306:db.internal:3306 user@jumpserver
# 然后本地访问 localhost:3306 就是访问 db.internal:3306
```

### 6.4 SCP 文件传输

| 命令 | 说明 | 示例 |
|------|------|------|
| `scp 本地文件 user@ip:远程路径` | 上传文件 | `scp file.txt user@192.168.1.100:/tmp/` |
| `scp user@ip:远程文件 本地路径` | 下载文件 | `scp user@192.168.1.100:/tmp/file.txt ./` |
| `scp -r 本地目录 user@ip:远程路径` | 上传目录 | `-r` 递归复制 |
| `scp -P 端口 文件 user@ip:路径` | 指定端口 | `-P` 大写 P |

---

## 7. 系统网络配置

### 7.1 修改主机名

| 系统 | 命令 | 说明 |
|------|------|------|
| **Linux** | `sudo hostnamectl set-hostname 新主机名` | 永久修改主机名 |
| **Linux** | `hostname 新主机名` | 临时修改（重启失效） |
| **Mac** | `sudo scutil --set HostName 新主机名` | 修改主机名 |

### 7.2 修改 /etc/hosts

| 命令 | 说明 | 示例 |
|------|------|------|
| `sudo vim /etc/hosts` | 编辑 hosts 文件 | 添加域名映射 |

**hosts 文件格式：**
```
127.0.0.1       localhost
192.168.1.100   myserver.local
192.168.1.200   db.internal
```

### 7.3 重启网络服务

| 系统 | 命令 | 说明 |
|------|------|------|
| **Ubuntu/Debian** | `sudo systemctl restart networking` | 重启网络服务 |
| **CentOS/RHEL** | `sudo systemctl restart network` | 重启网络服务 |
| **Ubuntu 18+** | `sudo netplan apply` | 应用 netplan 配置 |
| **Mac** | `sudo ifconfig en0 down && sudo ifconfig en0 up` | 重启网卡 |

### 7.4 开启 IP 转发

| 命令 | 说明 | 用途 |
|------|------|------|
| `sudo sysctl -w net.ipv4.ip_forward=1` | 临时开启 IP 转发 | 路由器/NAT |
| `echo "net.ipv4.ip_forward=1" \| sudo tee -a /etc/sysctl.conf` | 永久开启 | 写入配置文件 |
| `sudo sysctl -p` | 重新加载 sysctl 配置 | 使配置生效 |

---

## 8. Docker 网络

### 8.1 Docker 网络管理

| 命令 | 说明 | 示例 |
|------|------|------|
| `docker network ls` | 列出所有 Docker 网络 | 显示网络列表 |
| `docker network inspect bridge` | 查看网络详细信息 | 显示容器 IP 等 |
| `docker network create 网络名` | 创建自定义网络 | `docker network create mynet` |
| `docker network connect 网络名 容器名` | 将容器连接到网络 | 动态添加网络 |
| `docker network disconnect 网络名 容器名` | 断开容器网络连接 | 移除网络 |

**示例输出（docker network ls）：**
```
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local
def456ghi789   host      host      local
ghi789jkl012   none      null      local
```

### 8.2 Docker 端口映射

| 命令 | 说明 | 示例 |
|------|------|------|
| `docker port 容器名` | 查看容器端口映射 | 显示端口映射关系 |
| `docker run -p 宿主端口:容器端口 镜像` | 创建容器并映射端口 | `docker run -p 8080:80 nginx` |
| `docker run -P 镜像` | 随机映射所有暴露端口 | `-P` 大写 P |
| `docker ps` | 查看运行中容器（含端口） | 显示端口映射 |

**示例输出（docker port）：**
```
80/tcp -> 0.0.0.0:8080
443/tcp -> 0.0.0.0:8443
```

### 8.3 Docker 容器网络调试

| 命令 | 说明 | 示例 |
|------|------|------|
| `docker exec -it 容器名 bash` | 进入容器 Shell | 在容器内执行命令 |
| `docker exec 容器名 ping IP` | 在容器内执行 ping | 测试容器网络 |
| `docker inspect 容器名 \| grep IPAddress` | 查看容器 IP 地址 | 快速获取 IP |

---

## 附：常用网络端口速查

| 端口 | 服务 | 协议 | 说明 |
|------|------|------|------|
| 20/21 | FTP | TCP | 文件传输协议 |
| 22 | SSH | TCP | 安全远程登录 |
| 23 | Telnet | TCP | 远程登录（不安全） |
| 25 | SMTP | TCP | 邮件发送 |
| 53 | DNS | UDP/TCP | 域名解析 |
| 80 | HTTP | TCP | 网页服务 |
| 110 | POP3 | TCP | 邮件接收 |
| 143 | IMAP | TCP | 邮件接收 |
| 443 | HTTPS | TCP | 加密网页服务 |
| 3306 | MySQL | TCP | MySQL 数据库 |
| 5432 | PostgreSQL | TCP | PostgreSQL 数据库 |
| 6379 | Redis | TCP | Redis 缓存 |
| 27017 | MongoDB | TCP | MongoDB 数据库 |
| 3389 | RDP | TCP | Windows 远程桌面 |
| 8080 | HTTP-Alt | TCP | 备用 HTTP 端口 |

---

## 使用建议

1. **优先使用新命令**：Linux 上优先使用 `ip` 和 `ss` 命令，它们比 `ifconfig` 和 `netstat` 更强大且维护更好。

2. **善用管道和过滤**：结合 `grep`、`awk`、`head`、`tail` 等命令过滤输出，例如：
   ```bash
   ss -tlnp | grep :80
   ip addr | grep inet
   ```

3. **记录常用命令**：将常用命令保存为 Shell 别名或脚本，提高效率。

4. **查看帮助文档**：使用 `man 命令` 或 `命令 --help` 查看详细用法。

5. **注意权限**：很多网络命令需要 root 权限，记得加 `sudo`。

6. **系统差异**：Mac 和 Linux 命令有差异，注意区分使用。

---

**提示**：本速查表涵盖了日常网络运维 90% 的场景。建议打印或保存为书签，随时查阅。遇到问题时，先用这些命令排查，能快速定位大部分网络故障。

---

## 8. 程序员常用命令的协议底层速查

> 完整说明见第 4 章 §4.6。

### 远程登录与文件传输

| 命令 | 协议 | 传输层 | 端口 | 说明 |
|-----|------|--------|------|------|
| `ssh user@host` | SSH | TCP | 22 | 加密远程登录 |
| `scp file user@host:path` | SSH | TCP | 22 | 通过 SSH 复制文件 |
| `sftp user@host` | SSH | TCP | 22 | 交互式 SSH 文件管理 |
| `rsync -avz src user@host:dst` | SSH | TCP | 22 | 增量同步，只传变化 |
| `ftp host` | FTP | TCP | 21+20 | 明文（不推荐）|

### HTTP 与诊断

| 命令 | 协议 | 传输层 | 端口 | 说明 |
|-----|------|--------|------|------|
| `curl URL` | HTTP/HTTPS/... | TCP | 80/443/... | 万能 URL 工具 |
| `wget URL` | HTTP/HTTPS | TCP | 80/443 | 批量下载 |
| `ping host` | ICMP | **无** | **无端口** | IP 层连通性测试 |
| `traceroute host` | ICMP/UDP | 无/UDP | 无固定 | 路径追踪 |
| `dig domain` | DNS | UDP | 53 | DNS 查询 |
| `nslookup domain` | DNS | UDP | 53 | DNS 查询（简单版）|
| `nc -zv host port` | — | TCP | 自定义 | 端口连通性测试 |

### 开发工具

| 命令 | 协议 | 传输层 | 端口 | 说明 |
|-----|------|--------|------|------|
| `git clone git@host:repo` | SSH | TCP | 22 | SSH 方式 Clone |
| `git clone https://host/repo` | HTTPS | TCP | 443 | HTTPS 方式 Clone |
| `docker pull image` | HTTPS | TCP | 443 | 从 Registry 拉镜像 |
| `npm install` | HTTPS | TCP | 443 | npm 包下载 |
| `pip install` | HTTPS | TCP | 443 | Python 包下载 |

### 数据库

| 命令 | 协议 | 传输层 | 端口 | 说明 |
|-----|------|--------|------|------|
| `mysql -h host -u user` | MySQL 协议 | TCP | 3306 | MySQL 客户端 |
| `psql -h host -U user` | PostgreSQL 协议 | TCP | 5432 | PG 客户端 |
| `redis-cli -h host` | RESP | TCP | 6379 | Redis 客户端 |
| `mongo --host host` | MongoDB 协议 | TCP | 27017 | MongoDB 客户端 |

---

## 📚 导航

- [← 返回目录](README.md)
- [附录 A — 命令速查表](A_command_cheatsheet.md)
- [附录 B — 常见故障速查](B_common_issues.md)
- [附录 C — 术语表](C_glossary.md)
- [附录 D — 推荐资源](D_resources.md)
