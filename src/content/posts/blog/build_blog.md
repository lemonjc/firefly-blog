---
title: 基于 Firefly 搭建博客并部署到服务器
published: 2026-05-13
description: '记录使用 Firefly 搭建个人博客，并通过 Caddy 将静态站点部署到 Debian 服务器的完整流程。'
image: ''
tags: [blog]
category: 'blog'
draft: false
lang: 'zh-CN'
---

## Firefly 简介

**Firefly** 是一款基于 Astro 框架和 Fuwari 模板开发的清新美观且现代化个人博客主题模板，专为技术爱好者和内容创作者设计。该主题融合了现代 Web 技术栈，提供了丰富的功能模块和高度可定制的界面，让您能够轻松打造出专业且美观的个人博客网站。

**📝Firefly使用文档： [https://docs-firefly.cuteleaf.cn](https://docs-firefly.cuteleaf.cn/)**

**⭐Firefly开源地址：[https://github.com/CuteLeaf/Firefly](https://github.com/CuteLeaf/Firefly)** 

## 准备工作

开始之前，需要先准备好以下环境：

- [Node.js](https://nodejs.org/) 22.0 或更高版本
- [pnpm](https://pnpm.io/) 包管理器
- [Git](https://git-scm.com/)
- 一台用于部署博客的服务器

下文的安装命令以 Debian 系统为例，Ubuntu 等 Debian 系发行版也可以参考。

## 安装运行环境

### 安装 Node.js

Debian / Ubuntu 官方仓库中的 Node.js 版本通常比较旧，因此不建议直接使用系统默认源安装。这里使用 [NodeSource](https://nodesource.com/products/distributions) 提供的软件源。

在 NodeSource 页面中选择对应的系统版本和 Node.js 版本。下面以 `Debian 12` 和 `Node.js 24` 为例：

```bash
sudo apt-get install -y curl
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
```

### 安装 pnpm

```bash
sudo npm install -g pnpm
```

### 安装 Git

```bash
sudo apt update -y && sudo apt install git -y
```

## 搭建 Firefly 博客

### 克隆项目

```bash
git clone https://github.com/CuteLeaf/Firefly.git
cd Firefly
```

### 安装依赖

```bash
pnpm install
```

### 启动开发服务器

```bash
pnpm dev
```

该命令适合在发布前本地预览。开发服务器启动后，修改文件时页面会自动刷新，方便实时查看效果。

### 构建生产版本

```bash
pnpm build
```

构建完成后，静态网页文件会生成在 `dist` 目录中。

### 预览构建结果

```bash
pnpm preview
```

## 配置站点与编写文章

### 修改站点配置

站点名称、作者信息、导航栏、社交链接等配置可以参考官方文档中的[站点配置](https://docs-firefly.cuteleaf.cn/zh/guide/site.html)。

### 编写文章

文章默认存放在 `src/content/posts` 目录中，支持直接放置 `.md` / `.mdx` 文件。也可以创建子目录，用于组织文章和相关资源。

如果希望通过命令创建文章，可以使用：

```bash
pnpm new-post <filename>
```

更多写作方式可以参考官方文档中的[文章写作指南](https://docs-firefly.cuteleaf.cn/zh/guide/writing.html)。

## 部署到服务器

这里使用 Caddy 作为静态文件服务器。Caddy 配置简单，并且可以自动申请和续期 HTTPS 证书，适合个人博客部署。

### 安装 Caddy

官方安装文档可以参考：[https://caddyserver.com/docs/install](https://caddyserver.com/docs/install)

本文以 Debian 安装 Stable releases 版本为例：

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### 配置 Caddy

编辑 Caddy 配置文件：

```bash
sudo nano /etc/caddy/Caddyfile
```

删除或注释掉默认配置，然后写入以下内容：

```caddyfile
<your-domain-name> {
    root * <site-path>
    file_server
}
```

其中：

- `<your-domain-name>` 是你的域名，例如 `example.com`
- `<site-path>` 是站点部署目录，例如 `/var/www/blog`

如果同一个站点需要绑定多个域名，可以写成：

```caddyfile
<your-domain-name-1>, <your-domain-name-2> {
    root * <site-path>
    file_server
}
```

### 验证并重启 Caddy

```bash
sudo caddy validate -c /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

## 同步静态资源

部署的核心步骤是：先构建出静态网页，再将构建产物同步到 Caddy 指向的站点目录。

如果在服务器或本地机器上构建，静态资源目录通常是：

```text
<blog-path>/dist
```

如果服务器性能较弱，导致无法在服务器上完成构建，也可以先将代码推送到 GitHub。当前项目会通过 GitHub Actions 进行构建，并将构建结果推送到 `pages` 分支。这种情况下，可以使用 `pages` 分支中的内容作为静态资源。

下文使用以下占位符：

- `<static-resources-path>`：静态资源目录。本地构建时通常是 `<blog-path>/dist`；使用 `pages` 分支时通常是仓库根目录。
- `<site-path>`：服务器上的站点目录，例如 `/var/www/blog`。

使用 `rsync` 同步静态资源：

```bash
sudo rsync -av --delete <static-resources-path>/ <site-path>/
```

参数说明：

- `-a`：归档模式，保留目录结构、权限和时间等信息。
- `-v`：显示详细输出。
- `--delete`：删除目标目录中源目录已经不存在的文件，保证两边内容一致。
- 末尾的 `/` 不能省略。带 `/` 表示同步目录中的文件；不带 `/` 则会把整个目录同步到目标目录下。

完成以上步骤后，就可以通过配置好的域名访问博客了。
