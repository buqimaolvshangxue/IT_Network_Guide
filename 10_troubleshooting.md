# 第 10 章 · 网络排查 —— 分层排查法与常用工具

> **本章目标：** 掌握一套系统化的网络排查方法论——遇到"网不通"不再慌，按层排查，快速定位问题。同时熟练使用 ping、traceroute、curl、tcpdump 等核心工具。

---

## 10.1 这章解决什么问题？

- 网不通了，不知道从哪查起
- ping 得通但网页打不开
- 服务部署了但外面访问不了
- 网速突然变慢

---

## 🚦 排障快速入口（先定位再深挖）

### 30 秒先懂

- 排障不是猜，而是按层缩小范围
- 口诀：`线 → IP → 端口 → 服务`
- 一次只验证一个假设，别同时改很多配置

### 最小可用（先执行）

1. 先跑 `ping 网关` 和 `ping 8.8.8.8`，判断是内网问题还是外网问题。
2. 再用 `nc`/`curl` 判断是端口层问题还是应用层问题。
3. 最后看日志做定点修复，不要盲目重装。

### 阅读顺序（建议）

- 先看 `10.2 核心方法论`
- 再看 `10.3 工具详解`
- 最后按你的故障类型看 `10.4 场景实战`

---

## 10.2 🔑 核心方法论：分层排查法

网络问题千变万化，但排查思路只有一个：**从下往上，逐层排查。**

```
第 1 层：物理层    → 线插好了吗？Wi-Fi 连上了吗？
    ↓
第 2 层：网络层    → 有 IP 吗？能 ping 通网关吗？
    ↓
第 3 层：传输层    → 端口通吗？防火墙挡了吗？
    ↓
第 4 层：应用层    → 服务跑了吗？配置对吗？DNS 对吗？
```

🔑 **口诀：线 → IP → 端口 → 服务**

### 完整排查流程图

```
网不通了！
  │
  ├─ 第 1 步：检查物理连接
  │   ├─ 网线插好了？灯亮了？
  │   ├─ Wi-Fi 连上了？信号强度？
  │   └─ 不行 → 换网线/重启路由器/靠近路由器
  │
  ├─ 第 2 步：检查 IP 配置
  │   ├─ ip addr / ifconfig → 有 IP 吗？
  │   ├─ IP 是 169.254.x.x？→ DHCP 失败
  │   ├─ ping 网关（192.168.1.1）→ 通吗？
  │   └─ 不通 → 重启网络/重启路由器/检查 DHCP
  │
  ├─ 第 3 步：检查外网连通性
  │   ├─ ping 8.8.8.8 → 通吗？
  │   ├─ 通 → 网络层没问题，往上查
  │   └─ 不通 → 路由器/运营商问题
  │
  ├─ 第 4 步：检查 DNS
  │   ├─ ping 8.8.8.8 通但 ping baidu.com 不通？
  │   ├─ nslookup baidu.com → DNS 能解析吗？
  │   └─ 不能 → DNS 服务器问题，换 DNS
  │
  ├─ 第 5 步：检查端口
  │   ├─ telnet/nc 目标IP 端口 → 端口通吗？
  │   ├─ 不通 → 防火墙？服务没启动？端口没监听？
  │   └─ 通 → 应用层问题
  │
  └─ 第 6 步：检查应用
      ├─ curl 目标URL → 返回什么？
      ├─ 查看服务日志
      └─ 检查配置文件
```

---

## 10.3 排查工具详解

### 工具 1：ping —— 最基础的连通性测试

```bash
# 基本用法
ping 192.168.1.1        # ping 网关
ping 8.8.8.8            # ping 外网 IP
ping baidu.com          # ping 域名（同时测试 DNS）

# 指定次数
ping -c 4 8.8.8.8       # 只 ping 4 次

# 输出解读
# PING 8.8.8.8: 64 bytes from 8.8.8.8: icmp_seq=0 ttl=116 time=25.3 ms
#                                                    ↑TTL    ↑延迟
# TTL：数据包还能经过多少跳（初始通常 64 或 128）
# time：往返延迟（越小越好）
```

**ping 的结果怎么看：**

| 结果 | 说明 |
|---|---|
| 正常回复 | 网络通 |
| `Request timeout` | 对方不可达或禁 ping |
| `Destination host unreachable` | 路由不可达（网关找不到目标） |
| `Network is unreachable` | 本机没有路由（可能没有网关） |
| 延迟很高（>100ms 局域网） | 网络拥堵或线路问题 |
| 丢包率高 | 网络不稳定 |

⚠️ **ping 不通不一定是网络不通！** 很多服务器禁用了 ICMP（ping），比如部分云服务器。这时候用 `telnet` 或 `curl` 测试。

### 工具 2：traceroute / mtr —— 路径追踪

```bash
# traceroute：显示数据包经过的每一跳
traceroute baidu.com
# Mac 上也可以用
traceroute -I baidu.com    # 用 ICMP 模式（更容易通过防火墙）

# 输出示例：
#  1  192.168.1.1      1.2 ms    ← 你的路由器
#  2  10.0.0.1         5.3 ms    ← 光猫/运营商
#  3  * * *                      ← 这一跳不回复（正常，不代表不通）
#  4  202.97.x.x       15.1 ms   ← 骨干网
#  5  39.156.66.10     25.4 ms   ← 目标服务器

# mtr：traceroute 的增强版（实时刷新，显示丢包率）
# 安装
brew install mtr          # Mac
sudo apt install mtr -y   # Linux

mtr baidu.com
# 实时显示每一跳的延迟和丢包率，非常直观
```

**怎么看 traceroute 结果：**
- 某一跳开始全部超时 → 那一跳之后的网络有问题
- 中间某一跳超时但后面正常 → 那一跳只是不回复 ICMP，不影响
- 某一跳延迟突然增大 → 那一段链路可能拥堵

### 工具 3：telnet / nc —— 端口连通性测试

```bash
# telnet 测试端口（Mac/Linux）
telnet 43.128.10.20 22
# 连上了 → 端口开放 ✅
# Connection refused → 端口没监听
# 超时 → 防火墙挡了

# nc（netcat）更好用
nc -zv 43.128.10.20 22
# Connection to 43.128.10.20 22 port [tcp/ssh] succeeded!  ← 端口开放 ✅

# 扫描端口范围
nc -zv 43.128.10.20 80-443

# ⚠️ Mac 上 telnet 可能没装，用 nc 代替
```

### 工具 4：curl —— HTTP 层测试

```bash
# 基本请求
curl https://baidu.com

# 只看响应头（不下载内容）
curl -I https://baidu.com
# HTTP/1.1 200 OK        ← 状态码
# Content-Type: text/html
# ...

# 显示详细过程（DNS、连接、TLS 握手）
curl -v https://baidu.com

# 测量各阶段耗时
curl -o /dev/null -s -w "\
DNS解析:     %{time_namelookup}s\n\
TCP连接:     %{time_connect}s\n\
TLS握手:     %{time_appconnect}s\n\
首字节:      %{time_starttransfer}s\n\
总耗时:      %{time_total}s\n" https://baidu.com

# 输出示例：
# DNS解析:     0.012s
# TCP连接:     0.025s
# TLS握手:     0.089s
# 首字节:      0.120s
# 总耗时:      0.135s
# → 如果 DNS 解析很慢，说明 DNS 有问题
# → 如果 TCP 连接很慢，说明网络延迟高
# → 如果首字节很慢，说明服务器处理慢

# 指定 DNS 解析（绕过本地 DNS）
curl --resolve blog.example.com:443:43.128.10.20 https://blog.example.com
```

### 工具 5：dig / nslookup —— DNS 排查

```bash
# 基本查询
dig baidu.com
dig +short baidu.com          # 只看结果

# 指定 DNS 服务器查询
dig @8.8.8.8 baidu.com        # 用 Google DNS
dig @223.5.5.5 baidu.com      # 用阿里 DNS

# 查看完整解析链
dig +trace baidu.com

# 查看特定记录类型
dig baidu.com MX              # 邮件记录
dig baidu.com NS              # DNS 服务器
dig baidu.com TXT             # 文本记录

# nslookup（更简单）
nslookup baidu.com
nslookup baidu.com 8.8.8.8    # 指定 DNS 服务器
```

### 工具 6：ss / netstat —— 查看本机端口监听

```bash
# ss（推荐，更快）
ss -tlnp
# -t: TCP
# -l: 只看监听状态
# -n: 显示端口号（不解析服务名）
# -p: 显示进程名

# 输出示例：
# State   Local Address:Port   Process
# LISTEN  0.0.0.0:22           sshd
# LISTEN  127.0.0.1:8080       node
# LISTEN  0.0.0.0:80           nginx
#
# 🔑 注意 Local Address：
# 0.0.0.0:80   → 所有网卡都监听（外部可访问）
# 127.0.0.1:80 → 只监听本地（外部不可访问）

# netstat（老工具，功能类似）
netstat -tlnp

# 查看某个端口被谁占用
ss -tlnp | grep :8080
# 或
lsof -i :8080     # Mac 上更好用
```

### 工具 7：tcpdump —— 抓包分析（进阶）

```bash
# 抓取某个网卡的所有流量
sudo tcpdump -i eth0

# 只抓某个端口
sudo tcpdump -i eth0 port 80

# 只抓某个 IP
sudo tcpdump -i eth0 host 192.168.1.100

# 抓包保存到文件（用 Wireshark 打开分析）
sudo tcpdump -i eth0 -w capture.pcap

# 常用组合
sudo tcpdump -i eth0 port 443 and host 39.156.66.10 -c 100
# 抓 100 个发往百度 443 端口的包
```

🔑 **tcpdump 是终极排查工具**——当其他工具都找不到问题时，抓包看实际的网络数据。但日常很少用到，知道有这个工具就行。

---

## 10.4 常见场景排查实战

### 场景 1：部署了服务但外网访问不了

```bash
# 第 1 步：确认服务在跑
ss -tlnp | grep :8080
# 没有输出 → 服务没启动，检查服务

# 第 2 步：确认监听地址
# 如果显示 127.0.0.1:8080 → 只监听本地，外部访问不了
# 改成 0.0.0.0:8080 或通过反向代理

# 第 3 步：本地测试
curl http://127.0.0.1:8080
# 正常 → 服务本身没问题

# 第 4 步：检查防火墙
sudo ufw status
# 8080 没有 ALLOW → sudo ufw allow 8080/tcp
# 或者用反向代理，只开 80/443

# 第 5 步：检查云服务商安全组
# 阿里云/腾讯云/AWS 都有"安全组"，需要在控制台放行端口

# 第 6 步：从外部测试
nc -zv 你的IP 8080
curl http://你的IP:8080
```

**排查清单：**
```
✅ 服务在运行？（ss -tlnp）
✅ 监听 0.0.0.0 不是 127.0.0.1？
✅ 本地 curl 正常？
✅ ufw 防火墙放行了？
✅ 云服务商安全组放行了？
✅ 域名 DNS 指向正确？
✅ 反向代理配置正确？
```

### 场景 2：SSH 连不上服务器

```bash
# 第 1 步：ping 服务器
ping 43.128.10.20
# 不通 → 服务器宕机？IP 错了？

# 第 2 步：测试 SSH 端口
nc -zv 43.128.10.20 22
# Connection refused → SSH 服务没跑
# 超时 → 防火墙挡了

# 第 3 步：用 -v 看详细错误
ssh -v ubuntu@43.128.10.20
# 会显示详细的连接过程和错误信息

# 常见错误：
# "Permission denied" → 密钥/密码问题
# "Connection refused" → SSH 服务没启动或端口不对
# "No route to host" → 网络不通
# "Connection timed out" → 防火墙或 IP 错误
```

### 场景 3：DNS 解析异常

```bash
# 第 1 步：确认是 DNS 问题
ping 8.8.8.8          # 通 → 网络没问题
ping baidu.com        # 不通 → DNS 问题

# 第 2 步：测试 DNS 解析
nslookup baidu.com
dig baidu.com

# 第 3 步：换 DNS 测试
dig @8.8.8.8 baidu.com      # 用 Google DNS
dig @223.5.5.5 baidu.com    # 用阿里 DNS
# 能解析 → 你当前的 DNS 服务器有问题

# 第 4 步：检查 /etc/hosts
cat /etc/hosts
# 有没有错误的映射？

# 第 5 步：检查 /etc/resolv.conf
cat /etc/resolv.conf
# nameserver 后面是什么？改成 8.8.8.8 试试

# 第 6 步：清除 DNS 缓存
# Mac
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
# Linux
sudo systemd-resolve --flush-caches
```

### 场景 4：网速慢

```bash
# 第 1 步：测速
# 安装 speedtest-cli
pip install speedtest-cli
speedtest-cli
# 或用浏览器打开 fast.com / speedtest.net

# 第 2 步：有线 vs 无线
# 用网线直连路由器测速 → 如果有线快无线慢 → Wi-Fi 问题

# 第 3 步：查看网络延迟
mtr baidu.com
# 看哪一跳延迟突然增大

# 第 4 步：查看是否有设备占带宽
# 路由器后台 → 流量统计 → 看哪个设备在大量下载

# 第 5 步：DNS 慢？
curl -o /dev/null -s -w "DNS: %{time_namelookup}s\n" https://baidu.com
# DNS 解析超过 0.5s → 换 DNS
```

---

## 10.5 排查速查表

| 症状 | 快速排查命令 | 可能原因 |
|---|---|---|
| 完全没网 | `ip addr` / `ifconfig` | 没有 IP，DHCP 失败 |
| 有 IP 但 ping 不通网关 | `ping 192.168.1.1` | 网线/Wi-Fi 问题 |
| ping 通网关但 ping 不通外网 | `ping 8.8.8.8` | 路由器/运营商问题 |
| ping 通 IP 但 ping 不通域名 | `nslookup baidu.com` | DNS 问题 |
| 域名解析正常但网页打不开 | `curl -I https://xxx` | 端口/防火墙/服务问题 |
| 端口不通 | `nc -zv IP 端口` | 防火墙/服务未启动 |
| 服务启动了但外部访问不了 | `ss -tlnp` | 监听 127.0.0.1 / 防火墙 |
| 网速慢 | `mtr 目标` / `speedtest-cli` | 链路拥堵/Wi-Fi 干扰 |
| SSH 连不上 | `ssh -v user@ip` | 密钥/端口/防火墙 |

---

## 本章小结

| 概念 | 一句话 |
|---|---|
| 分层排查法 | 线 → IP → 端口 → 服务，从下往上查 |
| ping | 测试基本连通性 |
| traceroute/mtr | 看数据经过哪些节点，哪里卡了 |
| nc/telnet | 测试端口是否开放 |
| curl | 测试 HTTP 层，看响应和耗时 |
| dig/nslookup | 排查 DNS 问题 |
| ss/netstat | 查看本机端口监听状态 |
| tcpdump | 终极抓包工具 |

**🔑 核心认知：** 网络排查不靠猜，靠**逐层验证**。每一层用对应的工具确认"这一层没问题"，然后往上查。90% 的问题在前 4 步就能定位。

---

## 📦 扩展区域

### Wireshark —— 图形化抓包工具（推荐学习）

tcpdump 抓的包可以用 Wireshark 打开，图形界面更直观：
- 可以按协议过滤
- 可以看到完整的 TCP 握手过程
- 可以分析 HTTP 请求/响应内容

```bash
# Mac 安装
brew install --cask wireshark

# Linux 安装
sudo apt install wireshark

# 打开 tcpdump 抓的包
wireshark capture.pcap
```

#### 💻 动手实验：用 Wireshark 观察 TCP 三次握手

这是学习网络最直观的方式——亲眼看到 SYN、SYN-ACK、ACK 的交互过程：

**第 1 步：启动 Wireshark**
1. 打开 Wireshark
2. 选择你正在使用的网卡（如 `en0` 或 `eth0`）
3. 点击开始捕获

**第 2 步：过滤流量**
在捕获开始后，在过滤框输入：
```
tcp.port == 443
```
这样只显示 HTTPS 流量，减少噪音。

**第 3 步：触发一个连接**
在终端执行：
```bash
curl https://www.baidu.com
```

**第 4 步：观察三次握手**

在 Wireshark 中你会看到类似这样的包：

```
No.  Time      Source          Dest            Protocol  Info
1    0.000     192.168.1.100   110.242.68.66   TCP       54321 → 443 [SYN]
2    0.028     110.242.68.66   192.168.1.100   TCP       443 → 54321 [SYN, ACK]
3    0.029     192.168.1.100   110.242.68.66   TCP       54321 → 443 [ACK]
```

**解读：**
- 第 1 包 `[SYN]`：客户端说"我要建立连接"（Seq=0）
- 第 2 包 `[SYN, ACK]`：服务器说"好的，我也准备好了"（Seq=0, Ack=1）
- 第 3 包 `[ACK]`：客户端说"收到，连接建立"（Ack=1）

**第 5 步：点击单个包查看详情**

点击任意一个包，在下方详情面板可以看到：
- **Frame**：物理层信息
- **Ethernet**：数据链路层（源 MAC、目标 MAC）
- **Internet Protocol**：网络层（源 IP、目标 IP）
- **Transmission Control Protocol**：传输层（源端口、目标端口、Flags）

**常用过滤表达式：**

| 过滤需求 | 表达式 |
|---------|--------|
| 只看 HTTP 流量 | `tcp.port == 80` |
| 只看 DNS 查询 | `dns` |
| 只看某个 IP | `ip.addr == 192.168.1.100` |
| 只看 TCP SYN 包 | `tcp.flags.syn == 1` |
| 排除某 IP | `!ip.addr == 192.168.1.1` |
| 组合过滤 | `tcp.port == 443 && ip.addr == 1.2.3.4` |

> 💡 **学习建议**：抓包是理解网络的终极方式。建议抓一次完整的 HTTPS 访问（包括 TCP 握手、TLS 握手、HTTP 请求/响应、TCP 挥手），逐个包点开看，比看十遍文字说明都有效。

### iperf3 —— 网络带宽测试

测试两台机器之间的实际带宽（不依赖外部测速网站）：

```bash
# 安装
sudo apt install iperf3 -y    # 两台机器都要装

# 服务端
iperf3 -s

# 客户端
iperf3 -c 服务器IP
# 输出实际带宽，比如 940 Mbits/sec
```

### nmap —— 端口扫描

```bash
# 扫描目标的常用端口
nmap 43.128.10.20

# 扫描指定端口范围
nmap -p 1-1000 43.128.10.20

# 扫描 UDP 端口
sudo nmap -sU 43.128.10.20
```

⚠️ **不要对别人的服务器进行端口扫描，可能违法。只扫自己的。**

---

> **下一章 →** [第 11 章 · 办公网络](11_enterprise_network.md)：企业网络架构、VPN 接入、权限管理
---

## 📚 导航

- [← 返回目录](README.md)
- [← 第10章](09_self_hosting.md)
- [第11章 →](11_enterprise_network.md)
