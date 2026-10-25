---
title: "DeepSeek Harness (dsh-purge) 与 CliRelay 联动部署实战：搭建全自动 AI Agent 与多模型中继架构"
description: "深度整合 dsh-purge（DeepSeek Harness 策略清洗与执行引擎）与 CliRelay（统一 AI CLI 代理中继网关），通过 Docker Compose 极简一键编排，打通多模型负载均衡与无约束自动化工作流。"
published: 2026-10-01T22:20:00+08:00
updated: 2026-10-17T18:40:00+08:00
category: "技术"
subcategory: "AI 与自动化"
tags: ["DeepSeek", "DSH", "dsh-purge", "CliRelay", "AI Agent", "Docker", "部署指南"]
draft: false
lang: "zh_CN"
---

在实际用大语言模型（LLM）跑自动化 Agent 或写代码的流水线时，很多开发者都会卡在两个地方：

1. **上游模型管理太碎**：日常开发经常要在 DeepSeek、Claude、Google Gemini、OpenAI Codex、Antigravity 等不同平台之间切换。各家的 API Key 分散在不同配置文件里，一旦遇到单渠道限流（Rate Limits）或余额见底，任务就会直接报错中断，平时想统一算算 Token 消耗和管管配额也很麻烦。
2. **本地沙箱弹窗不断**：官方的 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`@deepseek-ai/dsh`）在本地改文件、跑编译或执行测试脚本时，默认带有严格的沙箱隔离和审批门（Approval Gate）。稍微涉及到底层读写就会弹窗要确认，还会强制覆盖部分系统提示词，导致原本想挂机跑通的自动化任务经常停下来等人工点同意。

为了解决这两个痛点，我们可以把社区里的两个开源项目——[CliRelay](https://github.com/kittors/CliRelay) 和 [dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 结合起来用：前端用 CliRelay 统一收口所有模型请求并负责故障自动切换，后端用加载了 dsh-purge 的容器节点放开执行权限，实现真正无人值守的自动化跑批。下面直接分享这套方案的架构逻辑和 Docker Compose 一键部署流程。

## 一、核心开源组件说明

### 1. CliRelay：统一 AI CLI 代理中继网关

[CliRelay](https://github.com/kittors/CliRelay) 是基于 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) 深度改造的 AI CLI 代理服务端。它的核心作用就是把你手头零散的 AI 订阅账号、OAuth 凭据和 API Key 拢到一个统一入口里，集中做路由转发和额度管控。

```text
[AI 编程工具 / Agent]                [统一中继网关]               [上游模型服务商]
  - dsh / Harness        ──────▶                      ──────▶   - Google Gemini
  - Claude Code          ──────▶       CliRelay       ──────▶   - OpenAI / Codex
  - Gemini CLI / Codex   ──────▶     (端口 :8317)      ──────▶   - Anthropic Claude
  - OpenAI 兼容客户端     ──────▶                      ──────▶   - DeepSeek / Qwen / Kimi
```

**技术底座与实用功能**：

* **底层架构**：采用 Go 1.26 + Gin 编写，使用 PostgreSQL 15+（配合 Ent ORM）持久化存储配置与账单数据，搭配 Redis 7+ 处理缓存、分布式锁、限流和会话队列。
* **多协议与会话兼容**：原生支持 Google Gemini、Anthropic Claude、OpenAI Codex、Qwen、iFlow、Kimi、Antigravity、xAI/Grok、Vertex、AWS Bedrock 等上游接口，并支持维持 OAuth 登录态。
* **核心路由与管理能力**：
  * **单一入口接入**：对外暴露统一的 OpenAI 兼容端点（如 `http://localhost:8317/v1`），客户端只需对接这一个地址。
  * **故障自动切换（Failover）与负载均衡**：当某个上游渠道触发限频（HTTP 429）或超时报错时，网关会自动切到备用渠道重试，不会直接把错误抛给客户端。
  * **细粒度配额控制**：支持多租户权限划分（RBAC），可以给每把分发出去的客户端 Key 单独设额度上限、定时重置周期和请求QPS限制。
  * **双端可视化监控**：内置 Web 管理后台（`/manage`）和基于 Bubble Tea 的终端 TUI 界面，能直接查看实时请求日志、检索上下文正文，并支持 OTA 在线更新。

### 2. dsh-purge：DeepSeek Harness 策略解限与自动化插件

[dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 是专门针对官方 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`@deepseek-ai/dsh`）开发的插件，主要用于红队安全测试和受控容器环境下的全自动任务执行。

官方 DSH 默认在模型和操作系统之间加了一层保护壳，包含提示词重写、高危操作审批弹窗和文件读写沙箱。平时手动写代码时这层保护很有用，但在跑安全靶场演练、大规模代码重构或者容器内自动化脚本时，频繁的拦截会让任务根本没法自动跑完。

**主要改造机制**：

* **绕过审查与文件读写拦截**：内置 52 项 Purge 补丁与底层 Shim，在受控环境内旁路掉默认的模型指令过滤和常规文件 ACL 读写限制。
* **全程免确认执行**：将宿主机的审批策略直接设为 `never`，无论是覆写代码文件、拉起子进程还是执行终端命令，全部静默放行，不再弹窗卡住流水线。
* **持久化注入自定义提示词**：支持通过 `prompt-inject.md` 挂载全局系统提示词和预设角色，且不会因为官方 DSH 升级而被覆盖。
* **多端形态通用**：同时支持官方 Web 模式（`dsh web`）、桌面客户端（如 [DSH Desktop](https://github.com/dataelement/dsh-desktop) 及官方可执行程序）以及无头命令行模式（`dsh headless`）。
* **支持一键回滚**：打补丁前会自动把官方原文件备份为 `.dshpurge.bak`，随时可以通过 `/purge revert` 无损恢复原状。

## 二、联动架构与任务流转过程

把 CliRelay 和集成了 `dsh-purge` 的容器节点（`dsh-node`）串起来之后，整个自动化任务的执行链路非常清晰：

```text
[用户 / 自动化脚本]
       │ 1. 下发任务 (Web :19387 或 CLI)
       ▼
[dsh-node 容器节点 (已加载 dsh-purge)]
       │ 2. 请求模型推理 (http://cli-proxy-api:8317/v1)
       ▼
[CliRelay 网关] ──(监控/记账: PostgreSQL & Redis)
       │ 3. 负载均衡 / 故障自动切换 (429/502 自动切备用源)
       ▼
[DeepSeek / Claude / Gemini / OpenAI 等上游模型]
       │ 4. 返回推理结果与执行计划
       ▼
[dsh-node 容器节点] ──▶ 5. 免弹窗直接读写 /workspace 并执行 Shell 验证
```

1. **下发任务**：开发者在浏览器的 Web 控制台（`:19387`）或者通过 CI 脚本向 `dsh-node` 提交重构代码、跑测试或复现漏洞的任务。
2. **模型路由与容灾**：`dsh-node` 将推理请求发给 CliRelay 网关。CliRelay 负责统计 Token 消耗并挑选可用渠道转发；如果遇到主渠道限流或偶发抽风，网关会在底层自动切到备用模型拿到回复，保证 `dsh-node` 侧的思考链路不断。
3. **无障碍落地执行**：`dsh-node` 收到模型给出的执行步骤后，直接在挂载的 `/workspace` 目录里增删文件、跑编译或执行单元测试，中间无需人工点击确认，直到最终完成目标。

## 三、Docker Compose 一键部署 dsh-node（推荐）

由于开启 `dsh-purge` 后 Agent 拥有很高的文件读写和命令执行权限，**强烈建议将其放在 Docker 容器里运行**。这样既能让 Agent 在容器内畅通无阻地干活，又能把破坏半径严格限制在挂载的工作目录内，保护宿主机安全。

### 1. 编写 `docker-compose.yml`

在本地新建一个工作目录并创建 `docker-compose.yml`：

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

### 2. 关键配置项说明

| 配置项 / 环境变量 | 设定值 | 实际作用说明 |
| :--- | :--- | :--- |
| `ports` | `19387:19387` | 暴露容器内的 Web GUI 控制台端口到宿主机。 |
| `DSH_MODE` | `unrestricted` | 关闭默认的模型指令审查与常规沙箱拦截。 |
| `DSH_SANDBOX_POLICY` | `danger-full-access` | 开放容器内完整的文件读写与命令执行权限，关闭 ACL 限制。 |
| `DSH_APPROVAL_POLICY` | `never` | 关闭敏感操作确认弹窗，所有读写和命令直接放行。 |
| `DSH_AUTO_PURGE` | `true` | 容器启动时自动校验并打上 52 项 Purge 补丁与底层 Shim。 |
| `volumes: workspace` | `./data/workspace` | 将宿主机目录挂载为容器工作区，防止容器销毁导致代码丢失。 |
| `volumes: prompt-inject` | `prompt-inject.md` | 以只读方式挂载自定义系统提示词配置文件。 |
| `security_opt` / `cap_add` | `unconfined` / `SYS_ADMIN` | 放宽容器默认的系统调用限制，避免内部调试工具或沙箱嵌套报错。 |

### 3. 启动容器服务

依次在终端执行以下命令初始化目录并启动容器：

```bash
# 1. 创建持久化目录与自定义提示词空文件
mkdir -p ./data/dsh_home ./data/workspace ./config
touch ./config/prompt-inject.md

# 2. 后台启动容器
docker compose up -d
```

启动完成后，直接用浏览器访问 `http://localhost:19387` 就能打开图形化操作界面。

## 四、将 dsh-node 接入 CliRelay 网关

### 1. 部署并配置 CliRelay

如果你还没跑起 CliRelay，可以直接按官方推荐的 Compose 方式拉起完整服务链（包含主程序、PostgreSQL 15、Redis 7 和自动更新组件）：

```bash
git clone https://github.com/kittors/CliRelay.git
cd CliRelay
mkdir -p auths logs data

# 启动 CliRelay 及配套数据库与缓存服务
docker compose up -d
```

服务跑起来之后，按以下三步完成渠道配置：

* 浏览器打开 `http://localhost:8317/manage` 进入管理后台。
* 在 **AI 供应商（AI Providers）** 页面填入你自己的 DeepSeek、Claude、Gemini 或 OpenAI 密钥，配置好上游账号池。
* 在 **API 密钥（API Keys）** 页面新建一把分发给本地工具用的客户端 Key（格式如 `cr-live-xxxxxx`），并勾选允许调用的模型分组。此时你的本地统一接入地址就是 `http://localhost:8317/v1`。

### 2. 修改 dsh-node 路由配置

打开 `dsh-node` 挂载出来的配置文件 `./data/dsh_home/config.json`（或通过环境变量注入），把模型接口地址指向刚刚配好的 CliRelay 网关：

```json
{
  "model_provider": "openai_compatible",
  "base_url": "http://host.docker.internal:8317/v1",
  "api_key": "cr-live-xxxxxx",
  "model": "deepseek-reasoner"
}
```

> **网络配置小贴士**：如果你把 `dsh-node` 和 `CliRelay` 放在同一个 Docker Compose 网络（Network）下，可以把 `base_url` 直接改成容器服务名 `http://cli-proxy-api:8317/v1`，走 Docker 内部网桥通信更稳定。

### 3. 运行监控与自动容灾效果

打通链路后，当你在 `dsh-node` 里跑大工作量的长任务时，CliRelay 会在背后帮你兜底两件事：

* **算清每一笔 Token 账**：在 `/manage` 面板里可以直观看到每次代码生成消耗了多少 Prompt Tokens 和 Completion Tokens，以及各个上游接口的响应延迟走势。
* **遇错自动切线路**：长任务最怕跑到一半遇到上游报 `HTTP 429`（请求过快）或 `HTTP 502`（网关超时）。接入 CliRelay 后，一旦主渠道报错，网关会在毫秒级自动把请求切到备用 API Key 或同级别替代模型上，让 `dsh-node` 的自动化流水线稳稳跑完。

## 五、轻量替代方案：宿主机无头模式 (Headless Mode)

如果你不想折腾 Docker，或者打算直接在 CI/CD 流水线（如 GitHub Actions）、无桌面服务器里调用 DSH，可以直接在宿主机装好 Node.js（建议 Node ≥ 22）和 `@deepseek-ai/dsh`，使用无头模式运行：

```bash
# 进入全开放权限的交互式无头终端
dsh headless --danger-full-access --approval-policy never --workspace ./default-workspace

# 或者直接通过 -e 参数单次执行指定任务（适合写进自动化脚本）
dsh headless --danger-full-access -e "请运行当前项目的全部单元测试，并自动修复所有报错"
```

无头模式不会启动 Web 端口，也不渲染前端界面，所有执行过程和结果直接输出到终端标准输出（`stdout`），非常适合放进定时任务（Cron）或 CI Runner 里直接调用。

## 六、安全注意事项与免责说明

* **必须在隔离环境中运行**：开启 `danger-full-access` 和 `dsh-purge` 后，Agent 具备直接修改文件和执行系统命令的能力。请务必在 Docker 容器、虚拟机或专用的离线测试靶场中使用，切勿直接在存有重要生产数据或个人隐私文件的宿主机根目录下裸跑。
* **合规使用边界**：本文介绍的补丁与自动化配置仅用于本地研发提效与合法授权的网络安全测试，请勿用于任何未授权的渗透测试或违反各模型服务商条款的场景。
* **项目独立性**：文中提到的 [dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 与 [CliRelay](https://github.com/kittors/CliRelay) 均为开源社区独立维护项目，与各商业大模型官方团队无隶属关系。

## 参考链接

1. **dsh-purge 项目仓库**：[GitHub - YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)
2. **DeepSeek Harness 官方仓库**：[GitHub - deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（[官方文档](https://deepseek-harness.github.io/deepseek-harness/)）
3. **CliRelay 项目仓库**：[GitHub - kittors/CliRelay](https://github.com/kittors/CliRelay)（[中文说明文档](https://github.com/kittors/CliRelay/blob/main/README_CN.md)）
4. **CLIProxyAPI 上游项目**：[GitHub - router-for-me/CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)
5. **DSH 桌面端社区项目**：[GitHub - dataelement/dsh-desktop](https://github.com/dataelement/dsh-desktop)
