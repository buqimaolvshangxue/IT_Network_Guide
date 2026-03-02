# 第 11 章 · 网络排查 —— 分层排查法与常见场景

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

## 11.5 程序员高频场景（扩展版）

> 以下场景针对开发者日常遇到的网络问题，覆盖代理、Docker、WSL2 等。

### ⚠️ 排障误区提醒

**误区1：ping 不通 = 网断了**
```
错误：ping 失败就重启路由器
正确：
  - 很多服务器/防火墙禁用 ICMP（ping）
  - ping 不通但 curl 可以 → 服务是通的，只是禁了 ping
  - 应该用 nc -zv host 443 测试具体端口
```

**误区2：关了代理/VPN 后，立刻就能访问内网域名**
```
错误：关 Clash 后，公司 OA 域名立刻解析正确
正确：
  - DNS 有缓存！关闭 Clash 后，操作系统 DNS 缓存可能还保存着错误的 IP
  - 需要手动刷新 DNS 缓存：
    Mac: sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
    Linux: sudo systemd-resolve --flush-caches
    Windows: ipconfig /flushdns
  - 浏览器也有自己的 DNS 缓存（chrome://net-internals/#dns）
```

**误区3：ping 通但服务不通**
```
场景：ping 192.168.1.100 ✅，但 curl http://192.168.1.100:8080 失败

可能原因：
  1. 服务没有启动（ss -tlnp 检查）
  2. 服务监听 127.0.0.1 而不是 0.0.0.0（ss -tlnp 检查监听地址）
  3. 防火墙拦截了 8080 端口（ufw/iptables/安全组）
  4. Docker 容器端口没有映射

ping 测的是 IP 层（ICMP），服务问题在传输层/应用层。
```

---

### 场景 5：代理开了但 npm/git 仍然超时

```bash
# 症状：浏览器能访问 Google，但 npm install 卡住
# 原因：系统代理不覆盖终端工具

# 排查步骤：

# 1. 确认 Clash 系统代理端口
# Clash 默认：HTTP 7890，SOCKS5 7891

# 2. 检查终端是否设置了代理
echo $ALL_PROXY
echo $HTTP_PROXY
echo $HTTPS_PROXY
# 如果为空 → 终端没代理 → 这就是原因

# 3. 临时设置终端代理
export ALL_PROXY=socks5://127.0.0.1:7891
# 再试 npm install

# 4. 如果 Clash 开的是 TUN 模式
# TUN 模式会拦截所有流量，包括终端 → 不需要设置环境变量
# 检查 Clash 当前是否是 TUN 模式：看系统网卡是否有 utun/clash-tun

# 5. 各工具的代理配置（详见第 9 章 §9.4）
# npm: npm config set proxy http://127.0.0.1:7890
# git: git config --global http.proxy socks5://127.0.0.1:7891
```

---

### 场景 6：公司内网域名在代理工具下解析失败

```bash
# 症状：开了 Clash TUN 模式后，git.company.com 无法访问
# 错误：nslookup 返回 NXDOMAIN 或 timeout

# 完整排查：
# 1. 确认是代理/TUN 导致的（关 Clash 后能访问？）

# 2. 用 dig 查看当前 DNS 解析
dig git.company.com
# 如果返回 NXDOMAIN → DNS 不认识这个域名
# 看 "SERVER:" 行 → 显示的是哪个 DNS 服务器在解析？

# 3. 问题定位
# 如果 SERVER 是代理服务器的 DNS（如 8.8.8.8）→ TUN 接管了 DNS
# 代理 DNS 不知道公司内网域名 → NXDOMAIN

# 4. 解决：在 Clash 规则中添加 bypass
# 打开 Clash 规则文件，在最前面加：
#   - DOMAIN-SUFFIX,company.com,DIRECT
#   - IP-CIDR,10.0.0.0/8,DIRECT

# 5. 验证
dig git.company.com
# 现在 SERVER 应该是公司 DNS（如 10.0.0.53）
# 能解析出正确的内网 IP
```

> 📖 原理详解 → 第 7 章 §7.6.2

---

### 场景 7：curl 能访问但浏览器报错（或反过来）

```bash
# 场景A：curl 成功，浏览器报 ERR_CERT_AUTHORITY_INVALID
# 原因：
#   - 网站证书有问题（自签名证书）
#   - 公司有 HTTPS MITM 检查，安装了公司根证书，但 curl 不信任它
# 排查：
curl -v https://target.com
# 看 TLS 握手输出，检查证书链

# 公司 MITM 检查的特征：
# curl 返回 "SSL certificate problem: unable to get local issuer certificate"
# 但浏览器（已安装公司证书）可以正常访问
# 解决（临时，不安全）：curl -k https://target.com（跳过证书验证）

# 场景B：浏览器能访问，curl 超时
# 可能原因：
# 1. 浏览器走了代理（系统代理），curl 没有
export ALL_PROXY=socks5://127.0.0.1:7891
curl https://target.com

# 2. 网站需要 JavaScript / Cookie / 特定 User-Agent
# curl 默认不执行 JS，不带 Cookie
curl -H "User-Agent: Mozilla/5.0" -b "session=xxx" https://target.com

# 3. 浏览器有缓存，curl 是新请求
```

---

### 场景 8：Docker 容器内无法访问宿主机的服务

```bash
# 症状：容器内 curl http://localhost:3000 失败
# 原因：容器内的 localhost 是容器自己！不是宿主机！

# 正确做法：
# 方法1：用 host.docker.internal（Docker Desktop / macOS / Windows）
curl http://host.docker.internal:3000

# 方法2：用宿主机在 docker0 网桥上的 IP
# 在宿主机上查看
ip addr show docker0
# 通常是 172.17.0.1
curl http://172.17.0.1:3000

# 方法3：容器用 host 网络模式（Linux 专用）
docker run --network host nginx
# 这样容器和宿主机共享网络栈，localhost 就是宿主机
# ⚠️ macOS/Windows Docker Desktop 不支持 host 模式

# 方法4：宿主机服务监听 0.0.0.0（不只是 127.0.0.1）
# 检查宿主机服务的监听地址：
ss -tlnp | grep 3000
# 如果显示 127.0.0.1:3000 → 换成 0.0.0.0:3000
```

---

### 场景 9：Address already in use（端口被占用）

```bash
# 症状：启动服务报错
# Error: listen EADDRINUSE: address already in use :::3000
# 或
# Bind for 0.0.0.0:80 failed: port is already allocated

# 排查步骤：

# 1. 找出占用端口的进程
# Mac / Linux
lsof -i :3000
ss -tlnp | grep 3000

# 输出示例：
# node    12345  user  23u  IPv6 0xabc  0t0  TCP *:3000 (LISTEN)
# ↑进程名  ↑PID

# 2. 杀掉进程
kill -9 12345

# 3. 如果还是失败（TIME_WAIT 状态）
# TCP 断连后端口需要等待约 60 秒（TIME_WAIT），详见第 4 章 §4.3
# 查看 TIME_WAIT 状态的连接：
ss -tn state time-wait | grep 3000

# 解决方案：
# a) 等 60 秒后重试
# b) 换一个端口
# c) 程序里开启 SO_REUSEADDR（让新连接可以复用 TIME_WAIT 的端口）

# Docker 容器的端口占用
docker ps | grep 3000          # 查看哪个容器映射了 3000
docker stop <container_id>     # 停止容器
```

---

### 场景 10：HTTPS 代理后出现证书错误

```bash
# 场景：开了 Clash 或公司代理后，浏览器显示 "您的连接不是私密连接"
# 原因：代理工具在做 HTTPS MITM（中间人）解密

# 两种情况：

# 情况1：Clash 等代理工具一般不做 MITM，报错可能是其他原因
# 检查：curl -v https://target.com 看证书是谁颁发的

# 情况2：公司网络的 SSL 检查（SSL inspection）
# 公司防火墙解密 HTTPS 流量做安全检查，用公司 CA 重新签证书
# 症状：证书颁发者变成了公司的 CA（如 "Company Security Root CA"）
# 解决：
# a) 在浏览器/系统中信任公司根证书（IT 部门会提供）
# b) 在 curl 中指定 CA：curl --cacert /path/to/company-ca.crt https://target.com

# 情况3：自签名证书（开发环境）
# 临时绕过（仅开发用！）：curl -k / git config --global http.sslVerify false
```

---

### 场景 11：代理模式下 DNS 泄漏导致被墙网站访问失败

```bash
# 症状：系统代理模式下，Clash 规则设置了 google.com → PROXY
# 但 google.com 仍然访问失败

# 排查 DNS 泄漏：

# 1. 检查 DNS 是否泄漏
# 访问 https://dnsleaktest.com（如果能访问的话）
# 或用命令检查 DNS 查询走了哪里
dig google.com
# 看 SERVER 字段：是代理服务器 IP 还是国内 DNS？

# 2. 系统代理模式下 DNS 泄漏的原理
# 系统代理只拦截 HTTP/HTTPS 流量
# DNS 查询是 UDP:53，绕过了系统代理
# 国内 DNS 对 google.com 返回污染的 IP
# 虽然 HTTP 走了代理，但 DNS 解析出的是错误 IP

# 3. 解决方案：切换到 TUN 模式
# Clash 的 TUN 模式会接管所有流量，包括 DNS（fake-ip 机制）
# DNS 查询不再泄漏

# 4. 或者手动配置 DoH（DNS over HTTPS）
# 在系统设置中启用加密 DNS
# macOS: 系统设置 → 网络 → DNS → 设置 DoH 服务器
```

---

## 11.6 排障速查表（扩展版）

| 症状 | 快速排查命令 | 可能原因 | 详细场景 |
|-----|------------|---------|---------|
| 完全没网 | `ip addr` / `ifconfig` | 没有 IP，DHCP 失败 | — |
| 有 IP 但 ping 不通网关 | `ping 192.168.1.1` | 网线/Wi-Fi 问题 | — |
| ping 通网关但 ping 不通外网 | `ping 8.8.8.8` | 路由器/运营商问题 | — |
| ping 通 IP 但 ping 不通域名 | `nslookup baidu.com` | DNS 问题 | 场景 3 |
| 域名解析正常但网页打不开 | `curl -v https://xxx` | 端口/防火墙/服务问题 | — |
| 端口不通 | `nc -zv IP 端口` | 防火墙/服务未启动 | — |
| 服务启动了但外部访问不了 | `ss -tlnp` | 监听 127.0.0.1 / 防火墙 | 场景 1 |
| npm/docker 超时，浏览器正常 | `echo $ALL_PROXY` | 终端没有代理环境变量 | 场景 5 |
| 公司内网域名无法解析 | `dig domain`，看 SERVER | TUN 模式 DNS 接管 | 场景 6 |
| curl 成功但浏览器报错 | `curl -v URL`，看证书 | 证书/MITM/CORS | 场景 7 |
| 容器访问宿主机服务失败 | `curl host.docker.internal:port` | localhost != 宿主机 | 场景 8 |
| Address already in use | `lsof -i :端口` / `ss -tlnp` | 端口冲突/TIME_WAIT | 场景 9 |
| 代理下 HTTPS 证书错误 | `curl -v URL`，看证书颁发者 | 公司 SSL 检查/MITM | 场景 10 |
| 系统代理但被墙网站仍失败 | `dig 域名`，看 SERVER | DNS 泄漏 | 场景 11 |

---

> **上一章 ←** [第 10 章 · 自建服务](10_self_hosting.md)
>
> **下一章 →** [第 12 章 · 企业网络](12_enterprise_network.md)

---

## 📚 导航

- [← 返回目录](README.md)
- [← 第 10 章](10_self_hosting.md)
- [第 12 章 →](12_enterprise_network.md)
