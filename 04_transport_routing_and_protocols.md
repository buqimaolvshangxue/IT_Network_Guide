# 第 4 章 · 传输、路由与协议

> **本章目标：** 搞清楚数据包怎么选路（路由表）、怎么可靠传输（TCP/UDP）、说什么语言（HTTP/SSH/DNS），以及你日常敲的每个命令底层走的是什么协议。

---

## 🚦 新手入口

### 30 秒先懂

- **IP 地址**决定"送到哪台机器"，**端口号**决定"送给这台机器上的哪个程序"
- **路由表**决定数据包从哪个网卡出去——每台电脑都有，不只是路由器
- **TCP** = 可靠（三次握手，保证送达）；**UDP** = 快速（直发，可能丢）
- 你日常敲的 `ssh/curl/git/docker` 底层都是 TCP 数据包走某个端口

### 最小可用

1. 先看 §4.2（端口号）+ §4.3（TCP/UDP）——够解决 80% 问题
2. 再看 §4.4（路由表）——理解第 7 章 VPN/代理的基础
3. §4.6 速查表——日后遇到连接问题直接来查

---

## 🎯 学习目标

学完本章，你应该能：

- [ ] 用 `netstat -rn` / `ip route` 查看路由表，解释每一行的含义
- [ ] 说出 TCP 三次握手和四次挥手的步骤，解释 TIME_WAIT 状态
- [ ] 查表说出 `ssh/scp/curl/git clone/docker pull/ping/dig` 各自底层的协议、传输层和端口
- [ ] 解释浏览器输入 URL 按回车后的完整协议层流程
- [ ] 解释 HTTP/3 为什么从 TCP 换成了 UDP

---

## 4.1 这章解决什么问题

前面章节搞清楚了设备、地址、DNS——知道"送到哪台机器（IP）"。但现在还有几个问题：

- 一台服务器同时跑着 Web、数据库、SSH，数据包到了，**给哪个程序**？ → 端口号
- 数据包出发前，**走哪个网卡、往哪个方向**？ → 路由表
- 发邮件要"必须送到"，看视频要"快点就行"，**能用不同方式传**？ → TCP vs UDP
- `curl`/`ssh`/`git clone` 底层到底是什么协议？ → 速查表

---

## 4.2 端口号：区分同一台电脑上的不同程序

### IP 解决了"找到哪台电脑"，端口解决"找到哪个程序"

```
IP 地址 = 公司地址（找到公司楼）
端口号 = 部门分机号（找到具体部门）
```

你的电脑上同时跑着浏览器、SSH 服务器、MySQL，一个数据包到了：
- 目标端口 443 → 给浏览器（或 Nginx）
- 目标端口 22 → 给 SSH 服务
- 目标端口 3306 → 给 MySQL

### 一个网络连接 = 四元组

唯一标识一个连接需要四个数字：

```
源 IP: 192.168.1.100   源端口: 52341（操作系统随机分配）
目标 IP: 142.250.80.46  目标端口: 443（HTTPS，服务端固定）
```

- **目标端口**：服务端固定，你连 HTTPS 就写 443，不能改别人的
- **源端口**：操作系统自动分配临时端口（49152-65535），用完释放

**为什么三个标签页同时访问 Google 不混乱？**

```
标签页1 → 源端口 52341 → Google:443
标签页2 → 源端口 52342 → Google:443
标签页3 → 源端口 52343 → Google:443

Google 回复时带上对应的目标端口，操作系统按端口号分发给对应标签页。
```

### 常见端口速查

| 端口 | 服务 | 备注 |
|-----|------|------|
| **22** | SSH / SFTP / SCP | 安全远程登录和文件传输 |
| **53** | DNS | UDP（默认）/ TCP（大查询）|
| **80** | HTTP | 明文网页 |
| **443** | HTTPS / HTTP/3 | 加密网页；HTTP/3 用 UDP:443 |
| **3306** | MySQL | 数据库 |
| **5432** | PostgreSQL | 数据库 |
| **6379** | Redis | 缓存 |
| **27017** | MongoDB | 数据库 |
| **8080** | HTTP 开发备用 | 非 root 用户常用 |

### 端口的三个范围

| 范围 | 名称 | 谁用 | 说明 |
|-----|------|------|------|
| 0-1023 | 知名端口 | 系统服务 | 需要 root/管理员权限监听 |
| 1024-49151 | 注册端口 | 第三方软件 | MySQL/Redis/MongoDB 在这里 |
| 49152-65535 | 动态端口 | 操作系统自动分配 | 源端口从这里取 |

```bash
# 查看哪些端口在监听
sudo lsof -iTCP -sTCP:LISTEN -n -P     # Mac/Linux
netstat -ano | findstr LISTENING        # Windows

# 查看某个端口被谁占用
sudo lsof -i :8080                      # Mac/Linux
netstat -ano | findstr :8080            # Windows
```

---

## 4.3 TCP vs UDP：两种送货方式

### 核心区别

| | TCP | UDP |
|--|-----|-----|
| **类比** | 📞 打电话（先接通，再说话）| 📮 寄明信片（直接发，不管收没收到）|
| **连接** | 需要三次握手建立连接 | 直接发送，无连接 |
| **可靠性** | 保证送达、保证顺序 | 可能丢包、可能乱序 |
| **速度** | 慢一些（有确认机制）| 快（无额外开销）|
| **典型应用** | HTTP/HTTPS、SSH、数据库 | DNS、视频直播、游戏、HTTP/3 |

### TCP 三次握手（建立连接）

```
你的电脑                           服务器
    │                                │
    │──── SYN（"我要连你"）──────────→│  第1次握手
    │                                │
    │←─── SYN-ACK（"好，我准备好了"）──│  第2次握手
    │                                │
    │──── ACK（"收到，开始通信"）─────→│  第3次握手
    │                                │
    │           连接建立               │
```

**为什么要三次？** 两次不够——服务器不知道客户端收到了自己的 SYN-ACK；四次浪费——三次已经足够确认双方都能收发。

### TCP 四次挥手（断开连接）

```
主动关闭方                        被动关闭方
    │                                │
    │──── FIN（"我说完了，想关"）─────→│
    │←─── ACK（"知道了"）─────────────│
    │                                │
    │←─── FIN（"我也说完了"）──────────│  服务器可能还有数据要发，所以分两步
    │──── ACK（"收到，关了"）─────────→│
    │                                │
    │ TIME_WAIT（等待 2MSL ≈ 60秒）    │  ← 见下方说明
```

### TIME_WAIT 状态：为什么 "Address already in use"？

四次挥手后，主动关闭方会进入 **TIME_WAIT** 状态，**等待 2MSL（约 60 秒）才真正释放端口**。

**为什么要等这么久？**
- 确保最后的 ACK 送到对方（如果 ACK 丢了，对方会重发 FIN）
- 防止旧连接的"迷路包"影响新连接

**这就是 `Address already in use` 的常见原因：**

```bash
# 你的 Node.js 服务刚关掉，立刻重启：
Error: listen EADDRINUSE: address already in use :::3000

# 原因：端口 3000 的 TCP 连接还在 TIME_WAIT 状态，没有释放
```

**解决方法：**
```bash
# 方法1：等 60 秒后重试
# 方法2：找到并杀掉占用进程
lsof -i :3000
kill -9 <PID>

# 方法3（开发环境）：设置 SO_REUSEADDR 选项，跳过 TIME_WAIT
# Node.js 示例：
server.listen({ port: 3000, host: '0.0.0.0' }, () => {})
// 配置中开启 reusePort: true
```

> 📖 完整排查流程 → 第 11 章 §11.4（场景9：Address already in use）

### 选 TCP 还是 UDP？

| 场景 | 选择 | 原因 |
|-----|------|------|
| 网页/API 请求 | TCP | 数据必须完整 |
| 文件传输 | TCP | 不能少一个字节 |
| 数据库连接 | TCP | 查询结果不能出错 |
| DNS 查询 | UDP | 数据量小，速度重要 |
| 视频直播 | UDP | 丢帧比卡顿好 |
| 在线游戏 | UDP | 低延迟最重要 |
| HTTP/3 | UDP（QUIC）| 见 §4.5.2 |

---

## 4.4 🔑 路由：数据包的"问路"过程（重要！）

> 路由表是理解 VPN/代理本质的关键——第 7 章会反复用到。

### 4.4.1 每台电脑都有路由表

路由表不是路由器专属的。你的 Mac/Windows/Linux **也有路由表**，每个数据包离开前都要查这张表：

```
路由表 = 一组规则：
"如果目标地址是 X，就从 Y 网卡出去，下一跳发给 Z"
```

**查看路由表：**

```bash
# Mac
netstat -rn
# 或者看默认路由
route -n get default

# Linux
ip route
# 或
route -n

# Windows
route print
```

### 4.4.2 读懂路由表输出

**Mac 上 `netstat -rn` 的输出：**

```
Routing tables

Internet:
Destination        Gateway            Flags  Netif
default            192.168.1.1        UGSc   en0    ← ★ 默认路由（最重要的一行）
127.0.0.1          127.0.0.1          UH     lo0    ← 回环地址
192.168.1.0/24     link#7             UCS    en0    ← 局域网，直接送达
192.168.1.1        link#7             UHLW   en0    ← 网关地址
```

**Linux 上 `ip route` 的输出：**

```
default via 192.168.1.1 dev eth0        ← ★ 默认路由
192.168.1.0/24 dev eth0 proto kernel    ← 局域网
127.0.0.0/8 dev lo scope host           ← 回环
```

**逐行解读：**

| 字段 | 含义 | 示例 |
|-----|------|------|
| Destination | 目标网段 | `192.168.1.0/24` = 去这个网段 |
| Gateway | 下一跳 | `192.168.1.1` = 发给这个 IP |
| Netif | 出口网卡 | `en0` = 从 Wi-Fi 网卡出去 |
| `default` | 兜底规则 | 不认识的目标，发给默认网关 |

### 4.4.3 路由匹配规则

**最长前缀匹配**：越精确的规则优先。

```
数据包目标: 192.168.1.100

路由表有两条规则：
1. 192.168.0.0/16  → 网关A（匹配，但不精确）
2. 192.168.1.0/24  → 直接送达（更精确，优先）

结论：选规则2，直接送达
```

**默认路由（`default` / `0.0.0.0/0`）**：

```
匹配所有 IP，但优先级最低（最不精确）
作用：兜底——"不知道往哪走就发给默认网关"
默认网关 = 你家路由器的 IP（通常是 192.168.1.1）
```

### 4.4.4 🔑 路由表决定了你的流量走向

```
正常情况（无 VPN/代理）：
  默认路由 → Gateway: 192.168.1.1（路由器）→ 出门上互联网

连接 VPN 后：
  默认路由 → Gateway: 10.8.0.1（VPN 虚拟网卡）→ 进 VPN 隧道

Clash TUN 模式开启后：
  默认路由 → Gateway: 198.18.0.1（Clash 虚拟网卡）→ 进 Clash 处理
```

**这就是 VPN 和 TUN 模式能接管所有流量的秘密——改了默认路由。**

> 📖 VPN/代理对路由表的具体改变 → 第 7 章 §7.4.2（有真实 netstat -rn 对比）

### 4.4.5 每一跳都查路由表（traceroute 原理）

数据包从你电脑到 Google，**每经过一个路由器，都查一次路由表**：

```
你的电脑 → 路由器 → 运营商节点 → 骨干网 → ... → Google
         ↑查表      ↑查表         ↑查表
```

traceroute 的原理（TTL 递增法）：

```
发送 TTL=1 的包 → 第1跳路由器收到，TTL减为0 → 返回 ICMP 超时消息 → 我们得到第1跳 IP
发送 TTL=2 的包 → 到达第2跳路由器，TTL减为0 → 返回 ICMP 超时消息 → 得到第2跳 IP
...以此类推
```

```bash
traceroute google.com      # Mac/Linux
tracert google.com         # Windows
mtr google.com             # 更强大（需要安装）
```

---

## 4.5 常见应用层协议

### HTTP/HTTPS：网页浏览

**URL 结构解析：**

```
https://www.example.com:443/path/page?id=123#section
│       │              │   │          │      │
│       │              │   │          │      └─ 锚点
│       │              │   │          └──────── 查询参数
│       │              │   └─────────────────── 路径
│       │              └─────────────────────── 端口（HTTPS 默认 443，可省略）
│       └────────────────────────────────────── 域名
└────────────────────────────────────────────── 协议
```

**HTTP 常用状态码：**

| 状态码 | 含义 | 常见原因 |
|--------|------|---------|
| 200 | 成功 | 正常 |
| 301 | 永久重定向 | 域名/路径变了 |
| 302 | 临时重定向 | 短链接跳转 |
| 401 | 未授权 | 需要登录 |
| 403 | 禁止访问 | 无权限 |
| 404 | 找不到 | URL 错误，页面删除 |
| 500 | 服务器内部错误 | 代码 bug |
| 502 | 网关错误 | 后端服务挂了 |
| 503 | 服务不可用 | 服务器过载 |

### 4.5.1 🔑 浏览器输入 URL 按回车——到底发生了什么？

这是第 0 章 "访问 Google" 在**协议层面**的精确解读：

```
Step 1: 解析 URL
  https://google.com/search?q=hello
  → 协议: https
  → 域名: google.com
  → 路径: /search
  → 参数: q=hello
  → 端口: 443（https 默认，省略了）

Step 2: DNS 查询（UDP:53）
  → 查询 google.com → 得到 IP: 142.250.80.46
  → DNS 查询本身是 UDP 数据包，也要查路由表！（详见第 3 章）

Step 3: TCP 三次握手（目标端口 443）
  → SYN → SYN-ACK → ACK
  → 建立可靠连接

Step 4: TLS 握手
  → 交换证书，验证 google.com 的身份
  → 协商加密密钥
  → 建立加密通道（详见第 6 章）

Step 5: 发送 HTTP 请求
  GET /search?q=hello HTTP/2
  Host: google.com
  User-Agent: Mozilla/5.0...

Step 6: 接收 HTTP 响应
  HTTP/2 200 OK
  Content-Type: text/html
  [HTML 内容...]

Step 7: 浏览器解析 HTML
  → 发现 CSS/JS/图片等资源
  → 并行重复 Step 2-6（很多个请求同时进行）
  → 渲染页面

关键认知：浏览器本质上是"自动化 HTTP 客户端 + HTML/CSS/JS 渲染引擎"
```

### 4.5.2 🔑 HTTP 协议演进——为什么 HTTP/3 用 UDP？

| 版本 | 传输层 | 核心改进 | 现状 |
|------|--------|---------|------|
| HTTP/1.0 | TCP | 每个请求一个 TCP 连接，用完关闭 | 已淘汰 |
| HTTP/1.1 | TCP | 连接复用（keep-alive），但有队头阻塞 | 仍广泛使用 |
| HTTP/2 | TCP | **多路复用**：一个 TCP 连接并行多个请求，头部压缩 | 主流 |
| **HTTP/3** | **UDP（QUIC）** | 解决队头阻塞，连接建立更快 | 快速普及中 |

**为什么 HTTP/3 抛弃了 TCP，改用 UDP？**

TCP 有一个根本性的问题——**队头阻塞（Head-of-line blocking）**：

```
HTTP/2（TCP 多路复用）：
  请求1 ─────────────→ 响应1
  请求2 ───→             → 响应2
  请求3 ──────→               → 响应3
  
  如果某个 TCP 数据包丢了：
  ████ 丢包 ████
  所有后续数据都得等这个包重传！
  ↑↑↑ 这就是 TCP 层的队头阻塞
```

**HTTP/3（QUIC）的解决方案：**

```
QUIC 基于 UDP，自己实现了：
  ✅ 可靠传输（比 TCP 更灵活的重传机制）
  ✅ 多路复用（不同流互不阻塞）
  ✅ 内置 TLS 加密（比 TCP+TLS 握手更快）
  ✅ 连接迁移（Wi-Fi 切 4G 不断连）

结果：即使丢包，只影响对应的流，不影响其他请求。
端口：还是 443，只是传输层从 TCP 换成了 UDP。
```

### 4.5.3 🔑 文件传输协议对比

| 工具 | 底层协议 | 端口 | 加密 | 特点 | 推荐场景 |
|------|---------|------|------|------|---------|
| **ftp** | FTP | 21（控制）+20（数据）| ❌ 明文 | 古老，双端口设计复杂 | **不推荐** |
| **sftp** | SSH | 22 | ✅ | 交互式文件浏览（类似 FTP 界面）| GUI 工具管理远程文件 |
| **scp** | SSH | 22 | ✅ | 单次复制，简单 | 快速传一个文件 |
| **rsync** | SSH（默认）| 22 | ✅ | **增量传输**，只传变化部分 | 大目录同步/备份 |

**关键认知：**
- SFTP 和 SCP 都走 SSH 协议（端口 22），**能 SSH 就能 SFTP/SCP**
- FTP 是完全独立的协议（端口 21），需要单独的 FTP 服务器软件
- rsync 最强大：只同步差异，特别适合备份和大目录同步

```bash
# 各工具使用示例

# scp：复制文件到远程
scp local_file.txt user@server:/remote/path/

# scp：从远程复制文件到本地
scp user@server:/remote/file.txt ./local/

# sftp：交互式会话
sftp user@server
> ls           # 列出远程文件
> put file.txt # 上传
> get file.txt # 下载
> exit

# rsync：同步目录（只传改变的部分）
rsync -avz ./local_dir/ user@server:/remote/dir/
# -a: 保留权限/时间戳  -v: 显示进度  -z: 压缩传输
```

---

## 4.6 🔑 程序员常用命令的协议底层速查表

> ⚠️ 几乎所有你日常敲的命令，底层都是 TCP 或 UDP 数据包走某个端口。  
> 搞懂这张表，遇到连接问题时你就知道该查什么。

### 远程登录与文件传输

| 命令 | 底层协议 | 传输层 | 端口 | 一句话原理 |
|-----|---------|--------|------|---------|
| `ssh user@host` | SSH | TCP | 22 | 加密的远程终端，所有内容加密传输 |
| `scp file user@host:path` | SSH | TCP | 22 | 通过 SSH 通道复制文件（一次性传输）|
| `sftp user@host` | SSH | TCP | 22 | 通过 SSH 通道的交互式文件管理 |
| `rsync -avz src user@host:dst` | SSH（默认）| TCP | 22 | 增量同步，只传有差异的部分 |
| `ftp host` | FTP | TCP | 21+20 | 明文传输，**不安全，不推荐** |

### HTTP 客户端工具

| 命令 | 底层协议 | 传输层 | 端口 | 一句话原理 |
|-----|---------|--------|------|---------|
| `curl URL` | HTTP/HTTPS/FTP/... | TCP | 80/443/... | 万能 URL 工具，支持几十种协议 |
| `wget URL` | HTTP/HTTPS/FTP | TCP | 80/443/21 | 批量下载，支持断点续传 |
| 浏览器 | HTTP/HTTPS | TCP（HTTP/1,2）/ UDP（HTTP/3）| 80/443 | 自动化 HTTP 客户端 + 渲染引擎 |

### 网络诊断

| 命令 | 底层协议 | 传输层 | 端口 | 一句话原理 |
|-----|---------|--------|------|---------|
| `ping host` | ICMP | **不是 TCP/UDP** | 无端口 | IP 层协议，测"通不通" |
| `traceroute host` | ICMP / UDP | 取决于实现 | 无固定 | TTL 递增，逐跳探测路径 |
| `dig domain` / `nslookup` | DNS | UDP（默认）/ TCP | 53 | 手动 DNS 查询 |
| `nc -zv host port` | 无固定 | TCP 或 UDP | 自定义 | 端口连通性测试（瑞士军刀）|
| `telnet host port` | Telnet | TCP | 23/自定义 | 端口测试（古老但常用）|
| `ss -tlnp` / `netstat -tlnp` | 系统调用 | — | — | 查本机端口监听状态 |
| `tcpdump` | 系统调用 | — | — | 抓包分析（原始网络流量）|
| `lsof -i :端口` | 系统调用 | — | — | 查看哪个进程占用了端口 |

### 开发工具

| 命令 | 底层协议 | 传输层 | 端口 | 一句话原理 |
|-----|---------|--------|------|---------|
| `git clone git@host:repo` | SSH | TCP | 22 | Git 通过 SSH 通道传输（需要 SSH key）|
| `git clone https://host/repo` | HTTPS | TCP | 443 | Git 通过 HTTPS 传输（用密码/token）|
| `git push/pull`（SSH）| SSH | TCP | 22 | 同上 |
| `docker pull image` | HTTPS | TCP | 443 | 从 Docker Registry 下载镜像 |
| `docker run -p 8080:80 nginx` | — | — | — | 端口映射，不是网络请求 |
| `npm install` / `yarn` | HTTPS | TCP | 443 | 从 npm registry 下载包 |
| `pip install` | HTTPS | TCP | 443 | 从 PyPI 下载包 |
| `brew install` | HTTPS | TCP | 443 | 从 GitHub/Homebrew CDN 下载 |

### 数据库连接

| 命令 | 底层协议 | 传输层 | 端口 | 一句话原理 |
|-----|---------|--------|------|---------|
| `mysql -h host -u user` | MySQL 协议 | TCP | 3306 | MySQL 客户端连接 |
| `psql -h host -U user` | PostgreSQL 协议 | TCP | 5432 | PostgreSQL 客户端连接 |
| `redis-cli -h host` | RESP 协议 | TCP | 6379 | Redis 客户端连接 |
| `mongo --host host` | MongoDB 协议 | TCP | 27017 | MongoDB 客户端连接 |

### 关键分类认知

```
1. 几乎所有"可靠连接"都走 TCP（需要保证数据完整）

2. 走 UDP 的主要有：
   - DNS（53）：查询数据量小，速度优先
   - HTTP/3（443）：QUIC 协议，自己实现可靠性
   - 视频/游戏流量：实时性优先

3. ping/traceroute 比较特殊：
   - ICMP 协议直接在 IP 层，根本没有端口概念
   - "ping 不通"≠ 网络断了（防火墙可能只禁了 ICMP）

4. SSH 是"寄生"能手，很多工具借用 SSH 通道：
   - SCP、SFTP、rsync（默认）、git（SSH 模式）
   - 所以 SSH 端口 22 是最重要的端口之一

5. 所有包管理器（npm/pip/docker/brew）底层都是 HTTPS:443
   → 遇到包下载慢/失败，代理配置是关键（见第 9 章 §9.5）
```

### curl 为什么是"万能工具"？

```bash
# curl 支持的协议远不止 HTTP：
curl https://google.com          # HTTPS
curl ftp://ftp.example.com/file  # FTP 下载
curl sftp://user@host/file       # SFTP 下载
curl -X POST https://api.example.com/data -d '{"key":"value"}'  # API 请求

# curl -v 能看到完整的协议过程：
curl -v https://google.com
# 输出包含：
# * TCP 连接过程
# * TLS 握手详情
# * HTTP 请求头
# * HTTP 响应头
# → 调试网络问题最有用的工具
```

**curl vs 浏览器的关键区别：**

```
curl：只发 HTTP 请求，返回原始响应内容
浏览器：HTTP 请求 + 执行 JS + 渲染 CSS + 遵守 CORS + Cookie 管理 + 证书验证...

这就是为什么 curl 能访问但浏览器报错（或反过来）
→ 详见第 11 章 §11.4 场景8
```

---

## 4.7 协议分层协作总结

用 §4.6 的速查表对应四层模型：

```
┌────────────────────────────────────────────────────────────────┐
│  应用层   │ 你"看到"的协议：HTTP/SSH/DNS/MySQL/FTP...            │
│          │ 你敲的命令：curl/ssh/git/docker/npm/dig...           │
├────────────────────────────────────────────────────────────────┤
│  传输层   │ TCP（可靠）或 UDP（快速）                             │
│          │ 加上端口号：22/443/53/3306...                        │
├────────────────────────────────────────────────────────────────┤
│  网络层   │ IP 地址，路由表决定走哪条路                            │
│          │ ICMP（ping/traceroute）在这层，没有端口               │
├────────────────────────────────────────────────────────────────┤
│  链路层   │ 以太网帧，MAC 地址，在局域网内传输                      │
│          │ ARP 协议获取 MAC 地址                                │
└────────────────────────────────────────────────────────────────┘
```

**遇到连接问题，分层排查：**

```
第一步：ping（ICMP/网络层）→ 能 ping 通吗？
  不通 → 网络层有问题（路由/防火墙禁 ICMP）

第二步：nc -zv host port（TCP/传输层）→ 端口通吗？
  不通 → 防火墙/安全组问题，或服务没启动

第三步：curl -v URL（应用层）→ HTTP 能通吗？
  不通 → 应用层问题（域名配置/证书/代码 bug）
```

---

## 4.8 🔬 动手验证

### 实验1：查看路由表并找到默认网关

```bash
# Mac
netstat -rn | head -20
# 找 "default" 那一行，Gateway 列就是你的默认网关（路由器 IP）

# Linux
ip route | grep default
# 输出示例：default via 192.168.1.1 dev eth0

# 验证：ping 默认网关，应该 <1ms
ping 192.168.1.1
```

### 实验2：SSH 握手过程（-v 详细模式）

```bash
ssh -v user@your-server.com

# 输出会显示：
# * SSH 协议协商
# * 密钥交换
# * 认证方式
# 理解 SSH 握手全过程
```

### 实验3：curl -v 看 TCP+TLS+HTTP 完整过程

```bash
curl -v https://example.com 2>&1 | head -50

# 观察输出：
# * Connected to example.com ... ← TCP 连接
# * TLSv1.3 ... ← TLS 握手
# > GET / HTTP/1.1 ← HTTP 请求
# < HTTP/1.1 200 OK ← HTTP 响应
```

### 实验4：dig 看 DNS 查询（注意是 UDP）

```bash
dig google.com

# 输出示例：
# ;; Query time: 8 msec
# ;; SERVER: 192.168.1.1#53(192.168.1.1) ← DNS 服务器地址:53端口
# ;; WHEN: ...
# ;; MSG SIZE  rcvd: 55 ← UDP 包大小

# 查看用的是 UDP 还是 TCP：
dig +tcp google.com    # 强制用 TCP
dig google.com         # 默认 UDP
```

### 实验5：ss 查看端口监听，找到 TIME_WAIT 状态

```bash
# 查看所有 TCP 连接（含状态）
ss -tn | head -30

# 查看 TIME_WAIT 状态的连接
ss -tn state time-wait

# 查看本机监听的端口
ss -tlnp
```

### 实验6：验证包管理器都走 HTTPS:443

```bash
# 抓包看 npm install 的流量
sudo tcpdump -i any port 443 -n &
npm install lodash

# 观察：所有 npm 流量都是目标端口 443 的 HTTPS 连接
# 这就是为什么代理配置里 HTTPS_PROXY 对 npm 有效
```

---

## 本章小结

| 概念 | 一句话 | 详见 |
|-----|-------|------|
| 端口号 | 区分同一台电脑上的不同程序，0-65535 | §4.2 |
| TCP | 三次握手，可靠传输，适合网页/文件/数据库 | §4.3 |
| UDP | 无连接，快速传输，适合 DNS/直播/游戏 | §4.3 |
| TIME_WAIT | TCP 断连后端口要等 60 秒释放，是 "Address already in use" 的原因 | §4.3 |
| 路由表 | 每台电脑都有，决定数据包从哪个网卡出去，default 行最重要 | §4.4 |
| VPN 改路由 | VPN 连接后修改路由表，把默认路由指向 VPN 虚拟网卡 | §4.4.4 |
| 浏览器 URL 流程 | 解析URL → DNS → TCP → TLS → HTTP → 渲染 | §4.5.1 |
| HTTP/3 用 UDP | QUIC 协议自己实现可靠传输，解决 TCP 队头阻塞 | §4.5.2 |
| SFTP/SCP = SSH | 能 SSH 就能 SFTP/SCP，走同一个端口 22 | §4.5.3 |
| 命令速查表 | 几乎所有命令底层都是 TCP:端口，ping 是 ICMP 没有端口 | §4.6 |

---

## 📦 扩展区域

> 以下为进阶内容，第一遍可跳过。

### 端口扫描（nmap）

```bash
# 扫描目标的常用端口（只扫自己的服务器！）
nmap -F your-server.com

# 扫描指定端口范围
nmap -p 1-1024 your-server.com

# 识别服务版本
nmap -sV -p 22,80,443 your-server.com
```

⚠️ 只扫描自己的服务器，扫描别人的服务器在大多数地区违法。

### WebSocket：持久双向连接

HTTP 是"一问一答"，WebSocket 是"持久对话"：

```
HTTP:        客户端问 → 服务器答 → 连接关闭 → 再问再答
WebSocket:   客户端 ←────────持久连接────────→ 服务器
                         双方随时可以发消息
```

WebSocket 升级过程：先发一个 HTTP 请求，包含 `Upgrade: websocket` 头，服务器返回 101 Switching Protocols，然后连接切换为 WebSocket 协议。

应用：在线聊天、实时协作（飞书文档）、股票行情推送、在线游戏。

### gRPC 协议

Google 开发的高性能 RPC 框架：
- 基于 HTTP/2（多路复用）
- 使用 Protobuf 序列化（比 JSON 更小更快）
- 支持双向流式通信
- 常用于微服务间通信

```bash
# gRPC 请求示例（使用 grpcurl）
grpcurl -plaintext localhost:50051 helloworld.Greeter/SayHello
```

### ICMP：ping 和 traceroute 的底层

ICMP（Internet Control Message Protocol）是 IP 层的控制协议，没有端口概念：

- **Echo Request / Echo Reply**：ping 用的
- **Time Exceeded**：TTL 超时，traceroute 用的
- **Destination Unreachable**：目标不可达，端口不通时服务器发回的

```bash
# 发 ICMP Echo Request
ping 8.8.8.8

# 查看 ICMP 包内容
sudo tcpdump -i any icmp
```

---

> **上一章 ←** [第 3 章 · 自动配置与发现](03_auto_config_and_discovery.md)
>
> **下一章 →** [第 5 章 · 网络边界与隔离](05_boundaries_and_isolation.md)

---

## 📚 导航

- [← 返回目录](README.md)
- [← 第 3 章](03_auto_config_and_discovery.md)
- [第 5 章 →](05_boundaries_and_isolation.md)
