---
title: 在 Bitwarden 中使用 SSH Agent
published: 2026-05-20
description: 'Bitwarden 的 SSH Agent 功能允许你将 SSH 私钥安全地存储在密码库中，无需在本地磁盘保留明文私钥文件，同时支持多设备同步。本文将介绍如何从创建密钥到客户端配置的完整流程。'
tags: [密码管理]
category: '工具'
draft: false
---

## 简介

Bitwarden 的 SSH Agent 可以将你的 SSH 私钥保存在加密的密码库中，通过桌面应用提供代理服务，使终端和工具（如 Git、ssh 客户端）直接使用这些密钥，而无需在本地磁盘存放未加密的私钥文件。密钥会随 Bitwarden 账户同步，实现多端一致性。

## 前置条件

- 安装 [Bitwarden 桌面应用](https://bitwarden.com/download/#downloads-desktop-applications)（Windows、macOS 或 Linux 版本）。
- 已登录 Bitwarden 账户并解锁密码库。

## 创建新的 SSH 密钥

1. 在 Bitwarden 桌面应用中，点击**新建**，选择**SSH 密钥**类型。

   ![新建 SSH 密钥](https://bitwarden.com/assets/1XYC3HwXOTMAPvyW1GS3Mk/5c7833241844eded5fa119d44d6efebd/new-ssh.png?w=1400&fm=avif)

2. 按提示填写名称、用户名、主机等信息，然后保存。密钥对会自动生成并存储在库中。

## 导入已有的 SSH 密钥

1. 同样新建一个**SSH 密钥**类型的条目，**暂时输入名称信息**，保存。
2. 回到编辑器，打开刚刚创建的条目，将你的现有**私钥**完整粘贴到私钥字段中，补全公钥（如果可用）和其他信息，保存。

> **提示**：私钥内容包括以 `-----BEGIN OPENSSH PRIVATE KEY-----` 开头和 `-----END OPENSSH PRIVATE KEY-----` 结尾的完整文本。

## 配置 Bitwarden SSH Agent

要让 SSH 客户端识别 Bitwarden 提供的代理，必须设置环境变量 `SSH_AUTH_SOCK`，指向相应的 Unix 套接字文件。以下按平台说明。

### Windows 平台

Windows 上需**先禁用系统自带的 OpenSSH 代理服务**，避免端口冲突。

1. 打开“服务”（搜索 `services.msc` 或通过开始菜单查找），找到 **OpenSSH Authentication Agent**。

   ![OpenSSH 服务](https://bitwarden.com/assets/77fTJpxIBH5ikJYQW1KFL7/0c6fa3b9f68f7a85569ad6ede489979e/openSSH_agent.png?w=700&fm=avif)

2. 双击该服务，将**启动类型**改为**禁用**，点击**应用**后再点击**确定**。

   ![禁用服务](https://bitwarden.com/assets/6Ghl3WkroGhoyy4fUMDbCg/800780f201fff9d6dc2de3d6577587ac/Screenshot_2024-12-04_142322.png?w=400&fm=avif)

3. 这样 Bitwarden SSH Agent 就会自动接管，无需额外设置环境变量（Windows 版处理方式与 Unix 不同）。

### macOS 平台

根据安装方式确定套接字路径：

- **App Store 版本**：
  ```bash
  export SSH_AUTH_SOCK="/Users/<user>/Library/Containers/com.bitwarden.desktop/Data/.bitwarden-ssh-agent.sock"
  ```
- **.dmg 安装版本**：
  ```bash
  export SSH_AUTH_SOCK="/Users/<user>/.bitwarden-ssh-agent.sock"
  ```

> **全局生效**：若希望所有终端（包括 GUI 应用）都能使用该代理，可使用 `launchctl setenv` 设置：
> ```bash
> launchctl setenv SSH_AUTH_SOCK "/Users/<user>/.bitwarden-ssh-agent.sock"
> ```
> 之后需重新登录或重启相关应用。

**持久化**：将上述 `export` 命令添加到 `~/.zshrc` 或 `~/.bashrc` 文件中，以便每次打开终端自动生效。

### Linux 平台

设置环境变量并指向用户目录下的套接字：

```bash
export SSH_AUTH_SOCK="/home/<user>/.bitwarden-ssh-agent.sock"
```

同样，建议将该行加入 `~/.bashrc` 或 `~/.zshrc` 实现持久化。

> **注意**：务必使用正确的用户名替换 `<user>`，并且路径中的 `home` 前缀与实际家目录一致。

## 启用 Bitwarden SSH Agent

配置好环境变量后，还需要在 Bitwarden 桌面应用中开启代理功能：

1. 打开 Bitwarden 桌面应用，进入**设置**（Preferences/Options）。
2. 勾选**启用 SSH 代理**（Enable SSH Agent）。
3. 可以根据需要调整**使用 SSH 代理时要求身份验证**的策略：
   - **始终**：每次使用密钥都要求验证；
   - **记住直到密码库锁定**：解锁密码库后一段时间内不再询问；
   - **从不**：不单独验证（仅依赖密码库解锁状态）。

## 验证 SSH Agent

保持 Bitwarden 桌面应用**解锁**状态，打开已配置好环境变量的终端，执行：

```bash
ssh-add -L
```

如果一切正常，应该列出你之前添加到 Bitwarden 中的所有公钥。现在可以直接使用 `ssh` 连接远程服务器，或通过 Git 等工具进行免密操作了。