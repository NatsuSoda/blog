---
title: 从零开始搭建自己的 VPS 服务器
description: 手把手教你购买、配置并搭建一台属于自己的 VPS 服务器，适合新手入门。
date: 2026-07-15
tags:
  - VPS
  - 服务器
---

# 从零开始搭建自己的 VPS 服务器

拥有一台属于自己的 VPS（Virtual Private Server，虚拟专用服务器），意味着你拥有了一个 24 小时在线的云端计算机。你可以用它来搭建个人博客、部署后端服务、搭建代理工具，或者纯粹用来学习 Linux。

## 第一步：选择 VPS 提供商

市面上主流的 VPS 提供商各有优缺点，以海外的服务提供商为例，以下是一些常见选择：

| 提供商 | 特点 | 最低价格 |
|--------|------|----------|
| **Vultr** | 全球多机房，按小时计费 | $2.5/月 |
| **BandwagonHost** | 线路优化好，适合国内用户 | $49.99/年 |
| **DigitalOcean** | 开发者友好，文档丰富 | $4/月 |
| **Linode** | 老牌厂商，稳定可靠 | $5/月 |

对于新手来说，**Vultr** 和 **DigitalOcean** 都是不错的入门选择。

## 第二步：选择操作系统

大多数 VPS 提供商都会提供以下几种主流 Linux 发行版：

- **Ubuntu**：社区活跃，文档丰富，新手首选
- **Debian**：稳定性极高，适合长期运行
- **CentOS Stream**：红帽系，企业环境常用

推荐新手选择 **Ubuntu 22.04 LTS** 或 **Ubuntu 24.04 LTS**，长期支持版本更加稳定。

## 第三步：连接到你的 VPS

购买完成后，你会收到 VPS 的相关信息：
- IP 地址
- root 用户名
- root 密码（或 SSH 密钥）

使用 SSH 连接到你的服务器：

```bash
ssh root@你的服务器IP
```

首次登录后，建议立即执行以下操作：

### 更新系统

```bash
apt update && apt upgrade -y
```

### 创建普通用户

直接使用 root 用户操作并不安全，建议创建一个普通用户：

```bash
adduser 你的用户名
usermod -aG sudo 你的用户名
```

## 第四步：基础安全配置

### 配置 SSH 密钥登录

使用密钥登录比密码登录更加安全：

```bash
# 在本地生成 SSH 密钥（如果没有的话）
ssh-keygen -t ed25519 -C "你的邮箱"

# 将公钥复制到服务器
ssh-copy-id 你的用户名@你的服务器IP
```

### 配置防火墙

推荐使用 `ufw` 来管理防火墙规则：

```bash
ufw allow OpenSSH
ufw enable
ufw status
```

如果你要搭建 Web 服务，还需开放 80 和 443 端口：

```bash
ufw allow 'Nginx Full'
```

## 第五步：安装常用工具

以下是一些在 VPS 上常用的工具：

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 安装 Node.js（使用 nvm）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install --lts

# 安装 Nginx
apt install nginx -y
```

## 第六步：搭建一个简单的 Web 服务

使用 Nginx 快速搭建一个静态页面：

```bash
# 创建一个简单的 HTML 页面
echo '<h1>你好，世界！我的 VPS 已上线 🎉</h1>' > /var/www/html/index.html

# 启动 Nginx
systemctl start nginx
systemctl enable nginx
```

现在在浏览器中访问你的 VPS IP 地址，你应该能看到刚刚创建的页面。

## 进阶推荐

当你熟悉了基础操作后，可以尝试以下进阶玩法：

1. **使用 Docker Compose 部署多容器应用**
2. **配置 Cloudflare Tunnel 隐藏服务器真实 IP**
3. **搭建个人图床（Chevereto 或 Lsky Pro）**
4. **部署 Uptime Kuma 监控服务状态**
5. **使用 acme.sh 自动申请 SSL 证书**

## 小结

拥有一台 VPS 是每个开发者成长路上很有意义的一步。从最初的命令行恐惧，到后来能够熟练部署各种服务，这个过程本身就是很好的学习经历。

接下来的文章里，我会分享更多关于 VPS 的实用技巧，敬请期待。
