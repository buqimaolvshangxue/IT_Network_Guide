# IT 网络知识完备指南

> 从零开始，让你成为一个胸有成竹的 IT 人。

## 这是什么

一份面向网络初学者和开发者的完整教程。读完之后你能：

- 理解网络中每个设备、每条线、每个配置项的含义
- 独立完成路由器配置、VPN 搭建、代理搭建、内网穿透、域名管理、服务部署
- 搞清楚 Clash/V2Ray 等代理工具的原理，解决"开了代理但公司内网打不开"的问题
- 知道你日常敲的每个命令（ssh/curl/git/docker）底层走的是什么协议
- 网络出问题时，有系统的排查思路

## 怎么读

**如果你是零基础**：从第 0 章开始，按顺序读。每章只依赖前面的知识。

**如果你有一些基础**：看目录，跳到你不懂的章节。

**如果你在解决具体问题**：翻附录 B（常见故障速查）或直接去第 11 章（排障场景实战）。

## 新手最短路径（推荐）

如果你是第一次系统学网络，建议按这条最短路径走：

1. [第 0 章](00_packet_journey.md)（先建立全局图：数据包从 A 到 B 的完整旅程）
2. [第 2 章](02_addresses_and_identity.md) + [第 3 章](03_auto_config_and_discovery.md)（IP/DHCP/DNS 基础）
3. [第 4 章](04_transport_routing_and_protocols.md)（端口 + TCP/UDP + 路由表 + 命令速查）
4. **[第 7 章](07_vpn_proxy_and_traffic.md) ⭐（全书最重要！）**（VPN vs 代理 vs TUN 模式，解决"开代理后公司内网挂了"）
5. [第 8 章](08_home_network.md)（把家里网络配好）
6. [第 9 章](09_remote_access.md)（SSH + 代理服务搭建 + 代理配置）
7. [第 11 章](11_troubleshooting.md)（建立排障习惯 + 高频场景）

> 💡 每章开头都有 `🚦` 快速入口（30 秒先懂 / 最小可用 / 阅读顺序），先看这个再读正文。

## 扩展区域（打好基础后按需读）

- [第 1 章](01_network_devices.md)：交换机/路由器/AP 的物理原理
- [第 5 章](05_boundaries_and_isolation.md)：NAT 细节 / 防火墙规则 / VLAN / DMZ
- [第 6 章](06_security_and_encryption.md)：HTTPS/TLS/SSH 密钥的加密原理
- [第 10 章](10_self_hosting.md)：VPS/域名/反向代理/Docker 自建服务
- [第 12 章](12_enterprise_network.md)：企业网络架构 / 域控 / 802.1X

## 目录

### 第一部分：基础与概念 — 让你"懂"

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第 0 章](00_packet_journey.md) | 数据包之旅 | 从电脑到 Google，数据包经过了什么？（全局地图）|
| [第 1 章](01_network_devices.md) | 网络设备 | 路由器、交换机、AP、光猫，每个盒子干什么？ |
| [第 2 章](02_addresses_and_identity.md) | 地址与标识 | IP/MAC/子网掩码，怎么区分"谁是谁"？ |
| [第 3 章](03_auto_config_and_discovery.md) | 自动配置与发现 | DHCP/ARP/DNS，设备怎么自动联网？ |
| [第 4 章](04_transport_routing_and_protocols.md) | 传输、路由与协议 | 端口/TCP/UDP/路由表，你日常敲的命令底层走的是什么？ |
| [第 5 章](05_boundaries_and_isolation.md) | 网络边界与隔离 | NAT/防火墙/VLAN/DMZ，谁能访问谁？ |
| [第 6 章](06_security_and_encryption.md) | 安全与加密 | HTTPS/TLS/SSH 密钥，怎么保证通信不被偷看？ |
| ⭐ [第 7 章](07_vpn_proxy_and_traffic.md) | **VPN、代理与流量控制** | **VPN vs 代理 vs TUN 的本质区别？为什么开了代理公司内网挂了？** |

### 第二部分：实战手册 — 让你"会做"

| 章节 | 主题 | 你能做什么 |
|------|------|-----------|
| [第 8 章](08_home_network.md) | 家庭网络配置 | 路由器设置、Wi-Fi 优化、**外网访问内网**（端口转发/内网穿透）|
| [第 9 章](09_remote_access.md) | SSH 与远程连接 | SSH 登录、VPN/代理服务器搭建、代理客户端配置 |
| [第 10 章](10_self_hosting.md) | 自建服务 | VPS、域名、反向代理、HTTPS、Docker 网络 |
| [第 11 章](11_troubleshooting.md) | 网络排查 | 分层排查法 + 11个高频故障场景（含代理/Docker/WSL2）|
| [第 12 章](12_enterprise_network.md) | 企业网络 | 企业网络架构、域控、802.1X、上网行为管理 |

### 附录 — 随时查

| 附录 | 内容 |
|------|------|
| [附录 A](A_command_cheatsheet.md) | 命令速查表（按场景分类，含协议底层对照）|
| [附录 B](B_common_issues.md) | 常见故障速查（现象→原因→解决）|
| [附录 C](C_glossary.md) | 术语表（所有术语一句话解释）|
| [附录 D](D_resources.md) | 推荐资源（每个主题 1-2 个高质量资源）|

## 图例约定

- 🔑 **关键概念**：必须记住的核心知识
- ⚠️ **易错点**：常见的坑和误区
- 💻 **动手验证**：在你自己的电脑上敲命令看效果
- 📖 **章节引用**：当前内容详细原理在哪章
- 📦 **扩展区域**：进阶内容，第一遍可跳过

命令示例同时给出 Mac 和 Linux 版本（有差异时标注）。
