# 第 9 章 · 自建服务 —— VPS、域名、反向代理、Docker

> **本章目标：** 从零开始搭建一个完整的线上服务——买 VPS、安全配置、买域名、配 DNS、装反向代理、部署应用、自动 HTTPS。走完这章，你就能把自己的网站/服务放到互联网上。

---

## 9.1 这章解决什么问题？

- 想搭个博客/网盘/API 服务，放到互联网上
- 买了 VPS 不知道怎么初始化
- 听说过 Nginx/Caddy 但不知道怎么配
- Docker 的网络搞不明白

---

## 🚦 两条阅读路线（先选一条）

### 30 秒先懂

- 你上线一个服务，最小闭环只有 5 件事：`VPS`、`SSH 安全`、`域名 DNS`、`反向代理`、`应用进程`
- 新手第一次先追求"能访问 + 不裸奔"，不要同时追求全功能
- 跑通后再加 Docker、监控、自动更新等增强项

### 路线 A：快速上线路线（先跑通）

1. 按 `9.2` 买一台 Ubuntu VPS（1C1G 就够起步）。
2. 按 `9.3` 做最小安全：密钥登录 + 禁密码 + ufw 放行 22/80/443。
3. 按 `9.4` 配域名 DNS 到 VPS。
4. 按 `9.5` 用 Caddy 反向代理一个服务并自动 HTTPS。
5. 用浏览器验证：`https://你的域名` 能访问。

### 路线 B：完整稳妥路线（长期维护）

- 路线 A 全部完成后，再做：
- 读 `9.6`（Nginx 版本，理解主流生产配置）
- 读 `9.7`（Docker 网络与端口暴露边界）
- 读 `9.8`（完整流程核对，补监控和运维习惯）

---

## 9.2 VPS 购买指南

### 什么是 VPS？

🔑 **VPS（Virtual Private Server）= 云上的一台虚拟电脑。** 你可以远程登录、装软件、跑服务，24 小时在线。

### 国内 vs 国外

| 特性 | 国内（阿里云/腾讯云） | 国外（Vultr/DigitalOcean/Bandwagon） |
|---|---|---|
| **备案** | ⚠️ 必须备案才能用域名 | 不需要备案 |
| **国内访问速度** | 快 | 看线路（香港/日本/新加坡较快） |
| **价格** | 轻量应用服务器约 50-100 元/月 | 约 $5-10/月 |
| **支付** | 支付宝/微信 | 信用卡/PayPal |

🔑 **建议：**
- 面向国内用户的服务 → 国内云（但要备案）
- 个人学习/测试/不想备案 → 国外 VPS（推荐香港/日本节点）

### 配置选择

| 用途 | CPU | 内存 | 硬盘 | 带宽 |
|---|---|---|---|---|
| 个人博客/轻量服务 | 1 核 | 1GB | 25GB SSD | 1-3Mbps |
| 多个 Docker 服务 | 2 核 | 2-4GB | 50GB SSD | 3-5Mbps |
| 数据库/重度应用 | 4 核 | 8GB+ | 100GB+ SSD | 5Mbps+ |

🔑 **新手推荐：** 1 核 1G 起步，不够再升级。系统选 **Ubuntu 22.04 LTS**（资料最多、最好找教程）。

---

## 9.3 VPS 初始安全配置

拿到 VPS 后，**第一件事不是装应用，而是做安全配置。** 裸奔的 VPS 几分钟内就会被全球扫描器盯上。

### 第 1 步：创建普通用户

```bash
# 用 root 登录后，创建普通用户
adduser deploy
# 设置密码，其他信息回车跳过

# 给 sudo 权限
usermod -aG sudo deploy

# 切换到新用户测试
su - deploy
sudo whoami   # 应该输出 root
```

### 第 2 步：部署 SSH 密钥

```bash
# 在你的 Mac 上（不是服务器上）
ssh-copy-id deploy@你的VPS_IP

# 测试密钥登录
ssh deploy@你的VPS_IP
# 不需要输密码就能登录 = 成功 ✅
```

### 第 3 步：禁用密码登录

⚠️ **先确认密钥登录成功后，再执行这一步！否则你会被锁在外面！**

```bash
# 在服务器上编辑 SSH 配置
sudo vim /etc/ssh/sshd_config

# 修改以下配置：
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes

# 重启 SSH
sudo systemctl restart sshd

# ⚠️ 不要关闭当前连接！新开终端测试能不能登录！
```

### 第 4 步：配置防火墙（ufw）

```bash
# 安装 ufw（Ubuntu 通常自带）
sudo apt install ufw -y

# 默认拒绝所有入站
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许 SSH（⚠️ 一定要先允许 SSH 再开防火墙！）
sudo ufw allow 22/tcp
# 如果改了 SSH 端口：
# sudo ufw allow 22222/tcp

# 允许 HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 开启防火墙
sudo ufw enable

# 查看规则
sudo ufw status
```

### 第 5 步：更新系统

```bash
sudo apt update && sudo apt upgrade -y

# 开启自动安全更新
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 安全配置清单

```
✅ 创建普通用户，不用 root 日常操作
✅ 部署 SSH 密钥
✅ 禁用密码登录
✅ 禁止 root SSH 登录
✅ 配置 ufw 防火墙
✅ 更新系统并开启自动更新
```

---

## 9.4 域名注册与 DNS 配置

### 买域名

| 注册商 | 特点 |
|---|---|
| **Cloudflare** | 成本价续费（不赚域名钱）、自带 DNS/CDN |
| **Namesilo** | 便宜、送隐私保护 |
| **阿里云/腾讯云** | 国内方便，但 .com 域名需要实名 |

🔑 **推荐：在 Cloudflare 买域名**，自动托管 DNS，后面配置最方便。

### DNS 配置

假设你买了 `example.com`，VPS IP 是 `43.128.10.20`：

```
在 Cloudflare DNS 管理面板添加记录：

类型    名称              内容              代理状态
A       example.com      43.128.10.20     🟠 已代理（推荐）
A       www              43.128.10.20     🟠 已代理
A       blog             43.128.10.20     🟠 已代理
CNAME   nas              example.com      🟠 已代理
```

**Cloudflare 代理模式（🟠 橙色云朵）：**
- 流量经过 Cloudflare → 隐藏你的真实 IP
- 自动提供 CDN 加速和 DDoS 防护
- 自动 HTTPS（Cloudflare 到用户这段）

**DNS only 模式（⚪ 灰色云朵）：**
- 只做 DNS 解析，不经过 Cloudflare
- 暴露你的真实 IP

### 验证 DNS

```bash
# 查看 DNS 是否生效
dig example.com
dig blog.example.com

# 如果用了 Cloudflare 代理，IP 会显示 Cloudflare 的 IP，不是你的 VPS IP
# 这是正常的
```

---

## 9.5 反向代理 —— Caddy（✅ 推荐新手）

### 什么是反向代理？

```
没有反向代理：
  用户 → blog.example.com:8080    （要记端口号，没有 HTTPS）

有反向代理：
  用户 → blog.example.com         （自动 HTTPS，自动转发到 8080）
        ↓
  Caddy/Nginx（监听 80/443）
        ↓
  实际服务（监听 8080）
```

🔑 **反向代理就是"前台接待"**——用户只和前台打交道，前台把请求转给后面对应的服务。

### 为什么推荐 Caddy？

- **自动 HTTPS** —— 自动申请和续期 Let's Encrypt 证书，零配置
- **配置超简单** —— 比 Nginx 简单 10 倍
- **适合新手** —— 不容易配错

### 安装 Caddy

```bash
# Ubuntu/Debian
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudflare.com/caddy/stable/deb/debian/setup.deb.sh' | sudo bash
sudo apt install caddy -y

# 或者直接下载二进制
# https://caddyserver.com/download
```

### Caddyfile 配置

```bash
sudo vim /etc/caddy/Caddyfile
```

```
# 最简单的配置：一个域名反向代理到本地服务
blog.example.com {
    reverse_proxy localhost:8080
}

# 多个域名多个服务
blog.example.com {
    reverse_proxy localhost:8080
}

nas.example.com {
    reverse_proxy localhost:5000
}

api.example.com {
    reverse_proxy localhost:3000
}

# 静态文件服务
docs.example.com {
    root * /var/www/docs
    file_server
}
```

```bash
# 重新加载配置
sudo systemctl reload caddy

# 查看状态
sudo systemctl status caddy

# 查看日志
sudo journalctl -u caddy -f
```

🔑 **就这么简单！** Caddy 会自动：
- 监听 80 和 443 端口
- 申请 Let's Encrypt SSL 证书
- HTTP 自动跳转 HTTPS
- 证书到期前自动续期

⚠️ **前提：域名的 DNS 已经指向你的 VPS IP，且 80/443 端口已开放。**

---

## 9.6 反向代理 —— Nginx（更主流）

Nginx 是目前最主流的反向代理/Web 服务器，配置比 Caddy 复杂但功能更强大。

### 安装

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
```

### 配置文件结构

```
/etc/nginx/
├── nginx.conf              # 主配置（一般不改）
├── sites-available/        # 所有站点配置
│   ├── default
│   └── blog.example.com
├── sites-enabled/          # 启用的站点（软链接）
│   └── blog.example.com -> ../sites-available/blog.example.com
└── conf.d/                 # 额外配置
```

### 配置示例

```bash
sudo vim /etc/nginx/sites-available/blog.example.com
```

```nginx
server {
    listen 80;
    server_name blog.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# 启用站点
sudo ln -s /etc/nginx/sites-available/blog.example.com /etc/nginx/sites-enabled/

# 测试配置语法
sudo nginx -t

# 重新加载
sudo systemctl reload nginx
```

### 用 Certbot 申请 HTTPS 证书

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx -y

# 一键申请证书并自动配置 Nginx
sudo certbot --nginx -d blog.example.com

# 测试自动续期
sudo certbot renew --dry-run
```

### Caddy vs Nginx 对比

| 特性 | Caddy | Nginx |
|---|---|---|
| HTTPS 证书 | ✅ 全自动 | 需要 Certbot |
| 配置复杂度 | 极简 | 较复杂 |
| 性能 | 好 | ✅ 更好（高并发） |
| 生态/资料 | 较少 | ✅ 非常丰富 |
| 适合 | 个人项目、新手 | 生产环境、高流量 |

🔑 **新手用 Caddy，生产环境或需要高级功能用 Nginx。**

---

## 9.7 Docker 基础网络

### 为什么要了解 Docker 网络？

现在大部分服务都用 Docker 部署。Docker 容器有自己的网络，理解它才能正确配置端口映射和服务互联。

### Docker 网络模式

| 模式 | 说明 | 用法 |
|---|---|---|
| **bridge**（默认） | 容器有独立 IP，通过端口映射访问 | `-p 8080:80` |
| **host** | 容器直接用宿主机网络 | `--network host` |
| **none** | 没有网络 | 极少用 |

### 端口映射（-p）

```bash
# 格式：-p 宿主机端口:容器端口
docker run -d -p 8080:80 nginx
# 访问 宿主机IP:8080 → 容器的 80 端口

# ⚠️ 关键：0.0.0.0 vs 127.0.0.1
docker run -d -p 8080:80 nginx           # 监听 0.0.0.0:8080（所有人可访问）
docker run -d -p 127.0.0.1:8080:80 nginx # 只监听本机（只有本机和反向代理能访问）
```

🔑 **安全建议：** 如果服务前面有反向代理（Caddy/Nginx），Docker 端口映射用 `127.0.0.1:端口`，不要暴露到公网。

```bash
# ✅ 推荐：只监听本地，通过 Caddy 反向代理
docker run -d -p 127.0.0.1:8080:80 myapp

# Caddyfile:
# blog.example.com {
#     reverse_proxy localhost:8080
# }

# ❌ 不推荐：直接暴露到公网
docker run -d -p 8080:80 myapp
```

### Docker Compose 网络

```yaml
# docker-compose.yml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "127.0.0.1:8080:80"    # 只监听本地
    networks:
      - mynet

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: secret
    # 不映射端口！只有同网络的容器能访问
    networks:
      - mynet

networks:
  mynet:
    driver: bridge
```

🔑 **同一个 Docker 网络内的容器可以用服务名互相访问：**

```bash
# web 容器内可以直接用 "db" 作为主机名连接数据库
mysql -h db -u root -p
# 不需要知道 db 容器的 IP
```

### 完整实战：Docker + Caddy 部署博客

```yaml
# docker-compose.yml
version: '3'
services:
  blog:
    image: ghost:5
    ports:
      - "127.0.0.1:2368:2368"
    environment:
      url: https://blog.example.com
    volumes:
      - ghost_data:/var/lib/ghost/content

volumes:
  ghost_data:
```

```bash
# Caddyfile
blog.example.com {
    reverse_proxy localhost:2368
}
```

```bash
# 启动
docker compose up -d
sudo systemctl reload caddy

# 访问 https://blog.example.com → 你的博客就上线了！
```

---

## 9.8 完整流程总结

```
第 1 步：买 VPS
    → 选配置、选系统（Ubuntu 22.04）
    ↓
第 2 步：安全配置
    → 创建用户 → SSH 密钥 → 禁密码 → 防火墙 → 更新系统
    ↓
第 3 步：买域名
    → Cloudflare 注册域名
    ↓
第 4 步：配 DNS
    → A 记录指向 VPS IP
    ↓
第 5 步：装反向代理
    → Caddy（新手）或 Nginx
    ↓
第 6 步：部署应用
    → Docker Compose 启动服务
    ↓
第 7 步：配置反向代理
    → Caddyfile 添加域名和转发规则
    ↓
第 8 步：自动 HTTPS
    → Caddy 自动搞定 / Nginx + Certbot
    ↓
✅ 完成！你的服务已经在互联网上了
```

---

## 本章小结

| 步骤 | 关键点 |
|---|---|
| VPS | Ubuntu 22.04，1 核 1G 起步 |
| 安全配置 | 密钥登录 → 禁密码 → 防火墙（顺序不能错！） |
| 域名 | Cloudflare 买 + 托管 DNS |
| 反向代理 | Caddy 自动 HTTPS，Nginx 更主流 |
| Docker | 端口映射用 127.0.0.1，同网络用服务名互联 |

**🔑 核心认知：** 自建服务就是"买台云电脑 + 买个域名 + 装个门卫（反向代理）+ 跑你的应用"。

---

## 📦 扩展区域

### Docker 网络进阶

- **macvlan**：给容器分配和宿主机同网段的 IP，像独立设备一样
- **overlay**：跨主机容器通信（Docker Swarm/Kubernetes 用）

### Let's Encrypt 通配符证书

```bash
# 申请 *.example.com 的通配符证书（需要 DNS 验证）
sudo certbot certonly --manual --preferred-challenges dns -d "*.example.com"
```

### Traefik —— 另一个反向代理

Traefik 是专为 Docker/Kubernetes 设计的反向代理，可以自动发现 Docker 容器并配置路由。适合跑很多 Docker 服务的场景。

### 监控你的服务

- **Uptime Kuma**：开源的服务监控面板，Docker 一键部署
- **Watchtower**：自动更新 Docker 容器

```bash
# Uptime Kuma 一键部署
docker run -d --restart=always -p 127.0.0.1:3001:3001 louislam/uptime-kuma
```

---

> **下一章 →** [第 10 章 · 网络排查](10_troubleshooting.md)：分层排查法、常用工具、常见故障
---

## 📚 导航

- [← 返回目录](README.md)
- [← 第8章](08_remote_access.md)
- [第10章 →](10_troubleshooting.md)
