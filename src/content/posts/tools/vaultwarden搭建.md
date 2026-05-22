---
title: Vaultwarden 搭建与自动备份
published: 2026-05-20
description: '本文详细介绍如何自托管部署 Vaultwarden（轻量级 Bitwarden 实现），并使用 vaultwarden-backup 工具将数据加密后自动备份到阿里云对象存储，适用于个人或小团队的密码管理方案。'
tags: [密码管理]
category: '工具'
draft: false
---

## 简介

**Vaultwarden** 是一个用 Rust 编写的非官方 Bitwarden 服务器实现，完全兼容官方 Bitwarden 客户端，特别适合资源有限的自托管场景。

::github{repo="dani-garcia/vaultwarden"}

**vaultwarden-backup** 是一个专门为 Vaultwarden 设计的备份工具，可将数据加密后通过 [Rclone](https://rclone.org/) 同步到多种远程存储（如阿里云 OSS、AWS S3 等）。

::github{repo="ttionya/vaultwarden-backup"}

本文的重点是**分别部署** Vaultwarden 和 vaultwarden-backup 服务，并实现自动备份。如果你希望一键快速体验，也可以直接使用项目自带的 [docker-compose.yml](https://github.com/ttionya/vaultwarden-backup/blob/master/README_zh.md) 进行部署（具体参考官方仓库）。接下来我们将详细讲解手动部署的整个过程。

## 分别部署 Vaultwarden 与备份服务

### 1. 部署 Vaultwarden

创建一个工作目录，并在其中新建 `compose.yml`，内容如下：

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "<domain name>"       # 替换为你的域名
      SIGNUPS_ALLOWED: "true"       # 初次启动时允许注册，稍后关闭
    volumes:
      - ./vw-data:/data             # 数据持久化目录
    ports:
      - 11001:80                    # 可根据需要修改宿主机端口
```

启动容器：

```bash
docker compose up -d
```

#### 配置反向代理（以 Caddy 为例）

在 Caddyfile 中添加如下站点配置（根据实际路径调整）：

```nginx
<domain name> {
    log {
        level INFO
        output file /var/www/vaultwarden/access.log {
            roll_size 10MB
            roll_keep 10
        }
    }
    encode zstd gzip
    reverse_proxy localhost:11001 {
        header_up X-Real-IP {remote_host}
    }
}
```

重载 Caddy 使配置生效：

```bash
sudo caddy reload --config /etc/caddy/Caddyfile
```

> [!NOTE] Note
> 请确保日志目录（如 `/var/www/vaultwarden`）存在且有写入权限。

现在通过域名访问你的 Vaultwarden 实例，注册管理员账号并登录。

#### 关闭公开注册

注册完成后，停止容器，将 `SIGNUPS_ALLOWED` 改为 `false`，再重新启动，以防止他人注册。

```bash
docker compose down
# 编辑 compose.yml，将 SIGNUPS_ALLOWED 改为 "false"
docker compose up -d
```

### 2. 部署 vaultwarden-backup 并同步至阿里云 OSS

#### 2.1 创建对象存储 Bucket

登录阿里云控制台，创建一个 OSS Bucket，并获取 AccessKey ID 和 AccessKey Secret。具体步骤可参考阿里云官方文档，此处不再赘述。

#### 2.2 配置 Rclone

运行以下命令进入 Rclone 交互式配置界面，挂载配置目录 `vaultwarden-rclone-data`：

```bash
docker run --rm -it \
  -v ./vaultwarden-rclone-data:/config \
  ttionya/vaultwarden-backup:latest \
  rclone config
```

[Alibaba OSS官方配置文档](https://rclone.org/s3/#alibaba-oss)

按照提示创建新远程（remote），建议将名称设置为 `BitwardenBackup`。配置过程会要求填写云服务商（选择 Amazon S3、Provider 选 Alibaba）、AccessKey、SecretKey、Endpoint（如 `oss-cn-shenzhen.aliyuncs.com`）等信息。

完成后，可通过以下命令验证配置是否正确：

```bash
docker run --rm -it \
  -v ./vaultwarden-rclone-data:/config \
  ttionya/vaultwarden-backup:latest \
  rclone config show

# 示例输出：
# [BitwardenBackup]
# type = s3
# provider = Alibaba
# access_key_id = <access_key_id>
# secret_access_key = <secret_access_key>
# endpoint = oss-cn-shenzhen.aliyuncs.com
# acl = private
# storage_class = STANDARD
# bucket_acl = private
```

#### 2.3 添加备份服务到 compose.yml

在之前创建的 `compose.yml` 中添加 `backup` 服务，完整示例如下：

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "<domain name>"
      SIGNUPS_ALLOWED: "false"   # 生产环境建议关闭注册
    volumes:
      - ./vw-data:/data
    ports:
      - 11001:80

  backup:
    image: ttionya/vaultwarden-backup:latest
    container_name: vaultwarden-backup
    restart: always
    environment:
      RCLONE_REMOTE_DIR: '/<bucket-name>/'     # 替换为你的 Bucket 名称
      CRON: '15 1 * * *'                       # 备份计划：每天 01:15 执行
      ZIP_ENABLE: 'TRUE'                       # 启用压缩加密
      ZIP_PASSWORD: '<password>'               # 设置加密密码，务必妥善保管
      ZIP_TYPE: 'zip'
      TIMEZONE: 'Asia/Shanghai'
      DATA_DIR: "/data"
    volumes:
      - ./vw-data:/data:ro                     # 以只读方式挂载数据目录
      - ./vaultwarden-rclone-data:/config      # Rclone 配置目录
```

> [!IMPORTANT] IMPORTANT
> 通过以上配置，备份工具将每天早上 01:15 自动将 `/data` 下的 Vaultwarden 数据打包为加密 ZIP 文件，然后通过 Rclone 上传到阿里云 OSS 的指定路径。更多高级选项（如邮件通知、健康检查等）请参阅 [官方文档](https://github.com/ttionya/vaultwarden-backup/blob/master/README_zh.md)。

启动全部服务：

```bash
docker compose up -d
```