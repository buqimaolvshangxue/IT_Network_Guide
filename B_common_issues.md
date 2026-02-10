# 附录 B — 常见故障速查

本附录按故障频率排序，提供快速诊断和解决方案。

---

## 1. 上不了网（Wi-Fi 连着但没网）

| 项目 | 内容 |
|------|------|
| **现象** | 设备显示已连接 Wi-Fi，但无法访问互联网 |
| **可能原因** | ① 路由器未联网 ② DNS 故障 ③ IP 地址获取失败 |
| **排查步骤** | 1. 检查其他设备能否上网<br>2. ping 网关：`ping 192.168.1.1`<br>3. ping 外网：`ping 8.8.8.8`<br>4. ping 域名：`ping baidu.com` |
| **解决方法** | **原因1（路由器问题）**：重启路由器，检查 WAN 口指示灯<br>**原因2（DNS 故障）**：手动设置 DNS 为 `8.8.8.8` 和 `114.114.114.114`<br>**原因3（IP 问题）**：`sudo dhclient -r && sudo dhclient` 或重新连接 Wi-Fi |

---

## 2. 网速很慢

| 项目 | 内容 |
|------|------|
| **现象** | 网页加载缓慢，下载速度远低于带宽 |
| **可能原因** | ① 带宽被占满 ② Wi-Fi 信号差 ③ DNS 解析慢 |
| **排查步骤** | 1. 测速：访问 speedtest.net<br>2. 检查连接设备数：登录路由器管理页面<br>3. 查看信号强度（Wi-Fi）<br>4. 测试 DNS：`nslookup baidu.com` |
| **解决方法** | **原因1（带宽占满）**：路由器 QoS 限速，或关闭下载任务<br>**原因2（信号差）**：靠近路由器，或切换到 5GHz 频段，或使用有线连接<br>**原因3（DNS 慢）**：更换 DNS 为 `223.5.5.5`（阿里）或 `1.1.1.1`（Cloudflare） |

---

## 3. 服务起了但外面访问不了

| 项目 | 内容 |
|------|------|
| **现象** | 本地 `localhost:8080` 可访问，但外网或局域网其他设备无法访问 |
| **可能原因** | ① 服务只监听 127.0.0.1 ② 防火墙拦截 ③ 端口未映射 |
| **排查步骤** | 1. 检查监听地址：`netstat -tlnp \| grep 8080`<br>2. 检查防火墙：`sudo ufw status`<br>3. 本机测试：`curl http://本机IP:8080`<br>4. 检查路由器端口转发设置 |
| **解决方法** | **原因1（监听地址）**：修改配置，监听 `0.0.0.0:8080` 而非 `127.0.0.1:8080`<br>**原因2（防火墙）**：`sudo ufw allow 8080` 或 `sudo firewall-cmd --add-port=8080/tcp --permanent`<br>**原因3（端口映射）**：路由器管理页面添加端口转发规则：外网端口 → 内网 IP:端口 |

---

## 4. 域名打不开但 IP 可以访问

| 项目 | 内容 |
|------|------|
| **现象** | `http://192.168.1.100` 可访问，但 `http://myserver.local` 无法访问 |
| **可能原因** | ① DNS 未配置 ② DNS 缓存未刷新 ③ hosts 文件未生效 |
| **排查步骤** | 1. 测试解析：`nslookup myserver.local`<br>2. 测试 ping：`ping myserver.local`<br>3. 检查 hosts：`cat /etc/hosts` |
| **解决方法** | **原因1（DNS 未配置）**：在 DNS 服务器添加 A 记录，或在本机 `/etc/hosts` 添加：`192.168.1.100 myserver.local`<br>**原因2（缓存）**：刷新 DNS 缓存：`sudo systemd-resolve --flush-caches` 或 `sudo dscacheutil -flushcache`（macOS）<br>**原因3（hosts 权限）**：检查文件权限：`sudo chmod 644 /etc/hosts` |

---

## 5. SSH 连不上

| 项目 | 内容 |
|------|------|
| **现象** | `ssh user@server` 超时或拒绝连接 |
| **可能原因** | ① SSH 服务未启动 ② 防火墙拦截 ③ 密钥权限错误 |
| **排查步骤** | 1. 测试端口：`telnet server 22` 或 `nc -zv server 22`<br>2. 服务器检查：`sudo systemctl status sshd`<br>3. 查看日志：`sudo tail -f /var/log/auth.log` |
| **解决方法** | **原因1（服务未启动）**：`sudo systemctl start sshd && sudo systemctl enable sshd`<br>**原因2（防火墙）**：`sudo ufw allow 22` 或 `sudo firewall-cmd --add-service=ssh --permanent`<br>**原因3（密钥权限）**：`chmod 600 ~/.ssh/id_ed25519`（或 `id_rsa`）和 `chmod 700 ~/.ssh` |

---

## 6. 端口转发不生效

| 项目 | 内容 |
|------|------|
| **现象** | 配置了 SSH 端口转发 `-L 8080:localhost:80`，但 `localhost:8080` 无法访问 |
| **可能原因** | ① 远程服务未运行 ② 转发命令错误 ③ 本地端口被占用 |
| **排查步骤** | 1. 检查远程服务：SSH 登录后 `curl localhost:80`<br>2. 检查本地端口：`lsof -i :8080`<br>3. SSH 详细输出：`ssh -v -L 8080:localhost:80 user@server` |
| **解决方法** | **原因1（远程服务）**：在远程服务器启动服务<br>**原因2（命令错误）**：确认格式：`-L 本地端口:目标主机:目标端口`<br>**原因3（端口占用）**：更换本地端口：`-L 8081:localhost:80` 或关闭占用进程 |

---

## 7. VPN 连不上

| 项目 | 内容 |
|------|------|
| **现象** | WireGuard/OpenVPN 连接失败或超时 |
| **可能原因** | ① 服务器防火墙拦截 ② 配置文件错误 ③ UDP 端口被封 |
| **排查步骤** | 1. 测试端口：`nc -u -zv server 51820`（WireGuard）<br>2. 检查服务：`sudo systemctl status wg-quick@wg0`<br>3. 查看日志：`sudo journalctl -u wg-quick@wg0 -f` |
| **解决方法** | **原因1（防火墙）**：`sudo ufw allow 51820/udp`<br>**原因2（配置错误）**：检查 Endpoint、PublicKey、AllowedIPs 是否正确<br>**原因3（UDP 被封）**：更换端口或使用 TCP 模式（需服务端支持） |

---

## 8. Docker 容器网络不通

| 项目 | 内容 |
|------|------|
| **现象** | 容器内无法访问外网，或宿主机无法访问容器端口 |
| **可能原因** | ① 端口未映射 ② 网络模式错误 ③ Docker 网络故障 |
| **排查步骤** | 1. 检查端口映射：`docker ps` 查看 PORTS 列<br>2. 容器内测试：`docker exec -it 容器名 ping 8.8.8.8`<br>3. 检查网络：`docker network ls` |
| **解决方法** | **原因1（端口未映射）**：重新运行容器：`docker run -p 8080:80 镜像名`<br>**原因2（网络模式）**：使用 `--network host` 或 `--network bridge`<br>**原因3（网络故障）**：重启 Docker：`sudo systemctl restart docker` 或重建网络：`docker network prune` |

---

## 9. 证书报错（浏览器提示不安全）

| 项目 | 内容 |
|------|------|
| **现象** | 访问 HTTPS 网站时浏览器显示"您的连接不是私密连接" |
| **可能原因** | ① 证书过期 ② 域名不匹配 ③ 自签名证书未信任 |
| **排查步骤** | 1. 检查证书：`openssl s_client -connect domain.com:443`<br>2. 查看有效期：`echo \| openssl s_client -connect domain.com:443 2>/dev/null \| openssl x509 -noout -dates`<br>3. 检查域名：证书中的 CN 或 SAN 是否匹配访问域名 |
| **解决方法** | **原因1（过期）**：续期证书：`sudo certbot renew`（Let's Encrypt）<br>**原因2（域名不匹配）**：重新申请包含正确域名的证书<br>**原因3（自签名）**：将 CA 证书导入系统信任列表，或使用 Let's Encrypt 免费证书 |

---

## 10. IP 冲突

| 项目 | 内容 |
|------|------|
| **现象** | 设备提示"IP 地址冲突"，网络时断时续 |
| **可能原因** | ① 手动设置了重复 IP ② DHCP 地址池范围过小 ③ 设备 MAC 地址冲突（罕见） |
| **排查步骤** | 1. 查看本机 IP：`ip addr` 或 `ifconfig`<br>2. 扫描局域网：`nmap -sn 192.168.1.0/24`<br>3. 检查 ARP 表：`arp -a` |
| **解决方法** | **原因1（手动 IP）**：改用 DHCP 自动获取，或更换静态 IP<br>**原因2（地址池小）**：路由器扩大 DHCP 地址池范围<br>**原因3（MAC 冲突）**：更换网卡或修改 MAC 地址（虚拟机场景） |

---

## 11. DNS 解析不生效（改了但没变化）

| 项目 | 内容 |
|------|------|
| **现象** | 修改了 DNS 记录，但访问域名仍指向旧 IP |
| **可能原因** | ① DNS 缓存未刷新 ② TTL 未过期 ③ DNS 传播延迟 |
| **排查步骤** | 1. 本地测试：`nslookup domain.com`<br>2. 指定 DNS 测试：`nslookup domain.com 8.8.8.8`<br>3. 检查 TTL：`dig domain.com` 查看 TTL 值 |
| **解决方法** | **原因1（本地缓存）**：刷新缓存：`sudo systemd-resolve --flush-caches` 或重启网络<br>**原因2（TTL）**：等待 TTL 时间过期（通常几分钟到几小时）<br>**原因3（传播延迟）**：等待 24-48 小时，或使用 `https://dnschecker.org` 检查全球传播状态 |

---

## 12. Wi-Fi 连不上（密码正确但连接失败）

| 项目 | 内容 |
|------|------|
| **现象** | 输入正确密码后仍无法连接，或一直显示"正在连接" |
| **可能原因** | ① MAC 地址过滤 ② 路由器连接数已满 ③ 信道拥挤 |
| **排查步骤** | 1. 检查路由器 MAC 过滤设置<br>2. 查看已连接设备数量<br>3. 尝试连接其他 Wi-Fi 网络 |
| **解决方法** | **原因1（MAC 过滤）**：路由器管理页面添加设备 MAC 地址到白名单<br>**原因2（连接数满）**：断开不用的设备，或升级路由器<br>**原因3（信道拥挤）**：路由器切换到不拥挤的信道（如 1、6、11），或使用 5GHz 频段 |

---

## 13. ping 不通但服务正常

| 项目 | 内容 |
|------|------|
| **现象** | `ping server` 超时，但 `curl http://server` 或 SSH 可以正常访问 |
| **可能原因** | ① 防火墙禁用 ICMP ② 服务器禁 ping ③ 路由器设置 |
| **排查步骤** | 1. 测试其他协议：`curl -I http://server`<br>2. 检查防火墙：`sudo iptables -L \| grep icmp`<br>3. 尝试从其他网络 ping |
| **解决方法** | **原因1（防火墙）**：允许 ICMP：`sudo ufw allow proto icmp` 或 `sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT`<br>**原因2（系统禁 ping）**：`sudo sysctl -w net.ipv4.icmp_echo_ignore_all=0`<br>**原因3（路由器）**：这是正常安全设置，不影响实际服务使用 |

---

## 14. 内网穿透连不上

| 项目 | 内容 |
|------|------|
| **现象** | frp/Tailscale 配置后无法通过公网访问内网服务 |
| **可能原因** | ① 服务端未启动 ② 配置文件错误 ③ 端口未开放 |
| **排查步骤** | 1. 检查服务端：`sudo systemctl status frps`<br>2. 检查客户端：`sudo systemctl status frpc`<br>3. 查看日志：`sudo journalctl -u frpc -f`<br>4. 测试服务端端口：`telnet 服务器IP 7000` |
| **解决方法** | **原因1（服务端）**：启动服务端：`sudo systemctl start frps`<br>**原因2（配置错误）**：检查 `frpc.ini` 中的 `server_addr`、`server_port`、`local_port`、`remote_port` 是否正确<br>**原因3（端口）**：服务器开放 frps 端口（默认 7000）和映射端口：`sudo ufw allow 7000,8080/tcp` |

---

## 快速诊断流程

```
网络故障
    ↓
能 ping 通网关？
    ├─ 否 → 本地网络问题（检查网卡、路由器）
    └─ 是 → 能 ping 通 8.8.8.8？
            ├─ 否 → 路由器/ISP 问题
            └─ 是 → 能 ping 通域名？
                    ├─ 否 → DNS 问题
                    └─ 是 → 应用层问题（防火墙、服务配置）
```

---

## 常用诊断命令速查

| 命令 | 用途 |
|------|------|
| `ping 8.8.8.8` | 测试网络连通性 |
| `nslookup domain.com` | 测试 DNS 解析 |
| `traceroute domain.com` | 追踪网络路径 |
| `netstat -tlnp` | 查看监听端口 |
| `ss -tlnp` | 查看监听端口（更快） |
| `ip addr` | 查看网络接口和 IP |
| `ip route` | 查看路由表 |
| `sudo ufw status` | 查看防火墙状态 |
| `curl -I http://domain.com` | 测试 HTTP 连接 |
| `telnet host port` | 测试 TCP 端口 |
| `nc -zv host port` | 测试端口连通性 |
| `dig domain.com` | 详细 DNS 查询 |
| `mtr domain.com` | 持续追踪网络路径 |
| `iftop` | 实时流量监控 |
| `tcpdump -i eth0` | 抓包分析 |

---

## 📚 导航

- [← 返回目录](README.md)
- [附录 A — 命令速查表](A_command_cheatsheet.md)
- [附录 B — 常见故障速查](B_common_issues.md)
- [附录 C — 术语表](C_glossary.md)
- [附录 D — 推荐资源](D_resources.md)
