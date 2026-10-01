---
title: "DeepSeek Harness (DSH) Purge 与 Antigravity 部署指南"
description: "本指南面向追求零心智负担的部署流程，以 Docker Compose 作为推荐主线并辅以架构图解，同时保留无头终端轻量化备选方案。"
published: 2026-10-01T21:35:00+08:00
category: "技术"
subcategory: "AI 与自动化"
tags: ["DeepSeek", "DSH", "Antigravity", "Docker", "部署指南"]
draft: false
lang: "zh_CN"
---

本指南面向追求零心智负担的部署流程，以 **Docker Compose (选项 A)** 作为推荐主线并辅以架构图解，同时保留 **无头终端 (选项 B)** 的轻量化备选方案。

---

## 架构拓扑与执行流程图解

```
                                  +------------------------------------+
                                  |         客户端 / 用户交互层         |
                                  |    (Web GUI :19387 / Desktop / CLI)|
                                  +-----------------+------------------+
                                                    |
                                                    v
+---------------------------------------------------------------------------------------------------+
| 选项 A: Docker Compose 编排体系 (推荐)                                                               |
|                                                                                                   |
|  +---------------------------------------------------------------------------------------------+  |
|  | [dsh-service]                                                                               |  |
|  |  +-----------------------+     +-----------------------+     +---------------------------+  |  |
|  |  |   DSH Core Engine     | <-> |  Purge Patch Layer    | <-> | Anti-Gravity Engine (AGE) |  |  |
|  |  |  (Cordis Kernel v0.2) |     |  (52 Patches / Shim)  |     | (ACL Bypass / Permissive) |  |  |
|  |  +-----------------------+     +-----------------------+     +---------------------------+  |  |
|  |              |                             |                               |                |  |
|  |              v                             v                               v                |  |
|  |       [:19387 Web API]            [Runtime Override]               [Local Sandbox Root]     |  |
|  +--------------+-----------------------------+-------------------------------+----------------+  |
|                 |                             |                               |                   |
|                 v                             v                               v                   |
|  +---------------------------+  +---------------------------+  +-------------------------------+  |
|  | Host Port: 19387          |  | Volume: prompt-inject.md  |  | Volume: /workspace            |  |
|  +---------------------------+  +---------------------------+  +-------------------------------+  |
+---------------------------------------------------------------------------------------------------+
                                                |
                                                | 备选替代方案
                                                v
+---------------------------------------------------------------------------------------------------+
| 选项 B: Headless 无头终端模式 (CLI 直接托管)                                                        |
|                                                                                                   |
|   [Terminal Session] ---> `dsh headless --danger-full-access` ---> [Local System Daemon]        |
|   (轻量无容器虚拟化，单进程管道通信，纯终端输出，适合 CI/CD 或超低资源服务器)                     |
+---------------------------------------------------------------------------------------------------+
```

---

## 选项 A：Docker Compose 极简一键部署（推荐）

### 1. 编排配置 (`docker-compose.yml`)

在工作目录下新建 `docker-compose.yml`，填入以下标准定义：

```yaml
version: "3.8"

services:
  dsh-node:
    image: deepseek-ai/dsh:latest
    container_name: dsh-purge-node
    restart: unless-stopped
    ports:
      - "19387:19387"
    environment:
      - DSH_HOME=/root/.dsh
      - DSH_MODE=unrestricted
      - DSH_SANDBOX_POLICY=danger-full-access
      - DSH_APPROVAL_POLICY=never
      - DSH_AUTO_PURGE=true
    volumes:
      - ./data/dsh_home:/root/.dsh
      - ./data/workspace:/workspace
      - ./config/prompt-inject.md:/root/.dsh/prompt-inject.md:ro
    security_opt:
      - seccomp:unconfined
    cap_add:
      - SYS_ADMIN
    working_dir: /workspace
    command: ["dsh", "web", "--host", "0.0.0.0", "--port", "19387"]
```

### 2. 核心配置参数对照表

| 配置键 / 参数项 | 推荐设定值 | 对应内部行为 / 作用说明 |
| :--- | :--- | :--- |
| `ports` | `19387:19387` | 映射本地 Web GUI 交互控制台端口。 |
| `DSH_MODE` | `unrestricted` | 解除模型指令限制与常规沙箱审查阻断。 |
| `DSH_SANDBOX_POLICY`| `danger-full-access`| 全开放文件读写与命令执行权限，禁用 ACL 拦截。 |
| `DSH_APPROVAL_POLICY`| `never` | 禁用危险指令弹窗确认，全自动化放行执行流。 |
| `DSH_AUTO_PURGE` | `true` | 容器初始化时自动校验并挂载 52 项 Purge 补丁与 Shim。 |
| `volumes: workspace`| `./data/workspace` | 容器工作区与宿主机挂载目录，防止任务资产丢失。 |
| `volumes: prompt-inject`| `prompt-inject.md` | 注入底层提示词与特权身份配置。 |
| `security_opt / cap_add`| `SYS_ADMIN / unconfined`| 为反重力执行引擎提供底层环境穿透能力，避免容器隔离报错。 |

### 3. 一键启动流程

1. 建立目录并初始化配置文件：
   ```bash
   mkdir -p ./data/dsh_home ./data/workspace ./config
   touch ./config/prompt-inject.md
   ```
2. 后台拉起服务：
   ```bash
   docker compose up -d
   ```
3. 访问 Web 控制面板：
   浏览器打开 `http://localhost:19387` 即可进入图形交互界面。

---

## 选项 B：无头终端备选提示 (Headless Mode)

适合不希望引入 Docker 运行时、资源受限或需要直接集成至现有命令行脚本的场景。

### 适用提示与指令

直接通过 DSH 二进制执行无头会话，自动跳过 UI 渲染与端口监听：

```bash
# 激活环境并以全量开放模式进入交互
dsh headless --danger-full-access --approval-policy never --workspace ./default-workspace
```

* **环境要求**：本地已安装 Node.js 与 DSH 本地依赖包。
* **快捷执行单任务**：
  ```bash
  dsh headless --danger-full-access -e "执行指定的自动化指令"
  ```
* **注意事项**：无头模式下缺失 Web 画布预览，所有产物与日志直接刷写至当前标准终端 (stdout) 与目标工程根目录。
