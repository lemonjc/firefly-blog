---
title: EasyTier异地组网
published: 2026-05-19
description: '使用 EasyTier 实现去中心化异地组网，无需公网 IP，轻松打通多地点设备互联。'
tags: [异地组网, 虚拟局域网, EasyTier]
category: '网络'
draft: false 
---

## 简介

**EasyTier** 是一款简单、安全、去中心化的异地组网工具，适用于远程办公、异地设备互访、游戏加速等场景。它无需公网 IP，无需复杂配置，即可在不同网络中的设备之间建立安全隧道。

本文将带你从零开始搭建 EasyTier 网络：先在服务器上部署自建节点并生成网络配置，再让桌面客户端和 OpenWrt 路由器加入组网。

## 下载

[官方下载页面](https://easytier.cn/guide/download.html)

- **Linux 服务器**：下载 `CLI` 压缩包
- **桌面系统**：可直接下载图形客户端
- **OpenWrt**：使用适配的 LuCI 插件：https://github.com/EasyTier/luci-app-easytier

## 服务器端部署自建节点

本部分将在服务器上完成三项任务：获取 CLI 程序、通过临时 Web 控制台生成组网配置、将 EasyTier 安装为系统服务。

### 1. 下载并解压 CLI

根据你的系统架构下载对应的 Linux CLI 压缩包，例如：

```bash
wget https://github.com/EasyTier/EasyTier/releases/download/vx.x.x/easytier-linux-x86_64-vx.x.x.zip
```

解压后进入目录：

```bash
cd easytier-linux-x86_64
```

### 2. 通过 Web 控制台生成网络配置

借助 EasyTier 自带的 Web 控制台，可以可视化生成一份完整的网络配置文件，后续直接将这份配置交给服务端使用，免去手写配置的麻烦。

#### 2.1 启动 Web 控制台

在解压目录中运行：

```bash
./easytier-web-embed \
    --api-server-port 11211 \
    --api-host "http://127.0.0.1:11211" \
    --config-server-port 22020 \
    --config-server-protocol udp
```

命令执行后若无报错，说明 Web 控制台已在后台运行，可通过 `http://127.0.0.1:11211` 访问。

参数说明（常用）：

- `--api-server-port`：Web 前后端绑定的端口
- `--api-host`：前端访问 API 后端的地址，这样前端无需手动指定后端
- `--config-server-port`：配置下发端口，后续 easytier-core 会通过该端口获取配置
- `--config-server-protocol`：配置下发所用协议（支持 tcp、udp、ws）
- `--web-server-port`：额外监听 Web 前端的端口（不受 `--no-web` 影响）
- `--no-web`：禁用 Web 前端，仅保留 API 功能

#### 2.2 建立 SSH 隧道并登录控制台

因为 Web 控制台只监听了 `127.0.0.1`，需要在本地计算机上通过 SSH 隧道转发端口。在本地终端执行（请替换 `user` 和 `<your-server-ip>` 为实际值）：

```bash
ssh -N -L 11211:127.0.0.1:11211 user@<your-server-ip>
```

然后在浏览器打开 `http://127.0.0.1:11211`，根据提示注册并登录账号。

#### 2.3 临时运行 easytier-core 接入控制台

为了让 Web 控制台能“看到”一台设备并基于它创建网络，需要临时启动一次 easytier-core 并连接到配置下发端口（用户名填在 Web 控制台创建的用户名）：

```bash
sudo ./easytier-core -w udp://127.0.0.1:22020/<你的用户名>
```

#### 2.4 在 Web 界面创建网络

刷新 Web 控制台，设备列表中应当出现刚才接入的客户端。点击其右侧的齿轮图标并选择“创建网络”。

依次进行如下设置：

1. **取消 `DHCP` 勾选框**（后续手动为该节点指定固定虚拟 IP）
2. **输入虚拟 IPv4 地址**，例如 `10.10.10.1/24`，这样该服务器节点将固定使用 `10.10.10.1`
3. **填写 `网络名称` 和 `网络密码`**，其他客户端加入网络时需要凭借此名称与密码
4. **初始节点留空**：当前节点作为网络起点，等待其他客户端通过其地址加入
5. **高级设置**：建议勾选 `启用私用模式`（勾选后仅明确指定了节点的设备才能加入，安全性更高；不勾选则网络可被公开发现，但连接仍需密码）
6. **主机名**：可按需修改
7. 其他选项保持默认即可

完成设置后，点击 `编辑为文件`，即可看到生成的完整配置内容。将这段配置保存到本地文本文件（例如 `default.conf`），后续会使用。

#### 2.5 停止临时服务

配置文件到手后，可以安全停掉刚刚用于生成配置的 Web 控制台和 easytier-core 进程。

### 3. 安装为 systemd 服务

现在将 EasyTier 部署为长期运行的服务，并使用刚才生成的配置文件。

创建安装目录并复制 CLI 程序：

```bash
sudo mkdir -p /opt/easytier
sudo cp easytier-* /opt/easytier
```

创建配置目录并放入配置文件：

```bash
sudo mkdir /opt/easytier/config
sudo touch /opt/easytier/config/default.conf
```

用编辑器打开 `/opt/easytier/config/default.conf`，将之前保存的配置内容完整粘贴进去，保存退出。

创建 systemd 服务单元文件 `easytier@.service`（注意服务名拼写为 `easytier`）：

```bash
sudo touch /etc/systemd/system/easytier@.service
```

写入以下内容：

```toml
[Unit]
Description=EasyTier Service
Wants=network.target
After=network.target syslog.target
StartLimitIntervalSec=0

[Service]
Type=simple
WorkingDirectory=/opt/easytier
ExecStart=/opt/easytier/easytier-core -c /opt/easytier/config/%i.conf
Restart=always
RestartSec=1
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

启用并启动服务：

```bash
sudo systemctl enable easytier@default
sudo systemctl start easytier@default
```

检查服务状态，确认运行正常：

```bash
systemctl status easytier@default
```

此后 EasyTier 服务将开机自启。如需停止服务，执行：

```bash
sudo systemctl stop easytier@default
```

> **提示：获取服务器节点地址**
> 默认情况下，EasyTier 会监听 TCP/UDP 的 11010 端口作为节点间通信端口。客户端接入时，可使用 `tcp://<服务器公网IP>:11010` 作为初始节点地址。如果配置文件中自定义了 `listeners`，以实际配置的地址为准。

## 客户端接入网络

其他设备（如笔记本、PC）可以通过图形客户端加入刚才创建的网络。

打开 EasyTier 图形客户端，进行如下设置：

1. **勾选 `DHCP`**：自动从网络获取虚拟 IP，无需手动分配
2. **填入 `网络名称` 和 `网络密码`**：与服务器侧设置的一致
3. **在“初始节点”中添加服务器节点地址**：格式为 `tcp://<服务器IP>:11010`（若服务器端修改了监听端口，请相应调整）
4. 点击运行，客户端即可加入组网

此时，你的设备已经拥有一个虚拟局域网 IP（如 `10.10.10.x`），可以与服务器节点互通。

## OpenWrt 使用 EasyTier 并代理子网

将 EasyTier 部署在 OpenWrt 路由器上，可以让路由器所在的整个局域网设备无需安装客户端即可直接访问虚拟网络中的资源。

### 1. 安装 LuCI 插件与 CLI 程序

从 [luci-app-easytier 仓库](https://github.com/EasyTier/luci-app-easytier) 下载与你的 OpenWrt 版本匹配的插件包，并上传到路由器 `/tmp` 目录。

安装插件（根据软件包格式选择）：

```bash
# IPK 包（OpenWrt 22.03.x）
opkg install /tmp/luci-app-easytier_*.ipk

# APK 包（OpenWrt SNAPSHOT）
apk add --allow-untrusted /tmp/luci-app-easytier_*.apk
```

刷新 LuCI 界面，在“VPN”菜单下进入“EasyTier”。按照页面提示上传对应架构的 Linux CLI 二进制程序（需包含 `easytier-core`、`easytier-cli` 和 `easytier-web-embed`）。

### 2. 配置 EasyTier

在“EasyTier 配置”页面的“基本配置”中：

- **启用**：勾选
- **启动方式**：选择默认
- **网络名称 / 网络密码**：填入前面创建网络时使用的名称和密码
- **启用 DHCP**：勾选
- **节点服务器**：添加服务器节点地址，例如 `tcp://<服务器IP>:11010`
- **子网代理**：填写需要代理的本地子网，例如 `192.168.1.0/24`

其余选项保持默认即可。

### 3. 高级访问控制

进入“高级设置 - 访问控制”，勾选 **“允许从局域网 lan 到虚拟网络 EasyTier 的流量”**。这样，连接到该路由器的局域网设备（如手机、打印机）无需安装 EasyTier 即可直接访问虚拟网络中的设备。若不勾选，则只有加入 EasyTier 组网的设备能单向访问此路由器的局域网。

点击 **保存并应用**，OpenWrt 即加入组网，同时开始代理指定子网。

## 结语

通过以上步骤，你已经完成了 EasyTier 从服务器部署、客户端接入到 OpenWrt 子网代理的全流程。此时，不同物理位置的设备仿佛处于同一个局域网，可以安全、便捷地互访。如果你希望进一步调整网络结构（如增加中继、调整 MTU 等），请参考 [EasyTier 官方文档](https://easytier.cn/guide/introduction.html) 进行深入配置。