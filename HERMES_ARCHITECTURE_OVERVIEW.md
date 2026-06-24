# Hermes Agent 架构概览

> 基于 Nous Research [hermes-agent](https://github.com/NousResearch/hermes-agent) 代码库的结构分析
> 梳理日期：2025-07

---

## 一、项目定位

**Hermes Agent** 是由 **Nous Research** 构建的**自改进型 AI 智能体（Agent）**。其核心差异化在于：它不是一次性的 LLM 调用封装，而是一个具有**闭环学习能力**的长期运行 Agent，能够在真实世界中持续工作、学习和进化。

---

## 二、解决什么场景的问题

| 场景 | 问题 | Hermes 的答案 |
|---|---|---|
| **日常编程/运维** | 每次手动查文档、敲命令、反复上下文切换 | CLI 终端内自然语言驱动，Agent 自动调用工具完成任务 |
| **跨平台消息接入** | 想从 Telegram/Discord/Slack 等聊天工具操控 Agent，但每个平台一套集成 | **Gateway 网关**统一接入，一个 Agent 后端支撑 20+ 个平台 |
| **长期记忆遗忘** | 每次对话 Agent 不记得你是谁、做过什么 | **FTS5 会话搜索** + **Memory Manager**（多 provider）实现跨会话记忆 |
| **技能重复造轮子** | 完成复杂任务后下次遇到同样场景又要重头教 | **自主 Skill 创建** — Agent 完成复杂任务后自动提炼为可复用的 Slash Command |
| **定时自动化** | 想 Agent 每天自动出日报、巡检、备份 | 内置 **Cron 调度器**，自然语言定义定时任务，结果投递到任意平台 |
| **环境不持久** | Agent 跑在本地笔记本，关机就丢 | **6 种终端后端**（Local/Docker/SSH/Singularity/Modal/Daytona），云上 serverless 休眠计费 |
| **研究训练** | 需要批量采集 Agent 轨迹来训练工具调用模型 | **batch_runner.py** + 轨迹压缩，生产训练数据 |
| **IDE 集成** | 不想脱离 IDE 切换终端操作 Agent | **ACP Adapter** 对接 VS Code / Zed / JetBrains |

---

## 三、核心架构与流程链路

```
┌─────────────────────────────────────────────────────────────┐
│                     用户入口                                 │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐         │
│  │  CLI     │  │  Gateway  │  │  ACP Adapter     │         │
│  │ (cli.py) │  │ (run.py)  │  │ (VS Code/Zed)    │         │
│  └────┬─────┘  └─────┬─────┘  └────────┬─────────┘         │
│       │              │                  │                   │
│       └──────────────┴──────────────────┘                   │
│                          │                                   │
└──────────────────────────┼───────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│               AIAgent 核心循环                                │
│             (run_agent.py / conversation_loop.py)             │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 1. 构建 System Prompt                                │     │
│  │    ├─ 身份指令 (agent/prompt_builder.py)             │     │
│  │    ├─ 记忆上下文 (memory_manager.py)                 │     │
│  │    ├─ 技能列表 (skill_commands.py)                   │     │
│  │    ├─ 工具 Schema (model_tools.py → registry)        │     │
│  │    └─ 环境提示 (context_files, subdirectory_hints)   │     │
│  └──────────────────────┬──────────────────────────────┘     │
│                         ▼                                    │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 2. while 循环 (双重预算控制)                         │     │
│  │    max_iterations=90 + IterationBudget               │     │
│  │                                                      │     │
│  │    ┌──────────────┐                                  │     │
│  │    │ LLM API 调用  │  ← client.chat.completions.    │     │
│  │    │ (非流式/流式) │     create(model, messages,    │     │
│  │    └──────┬───────┘      tools=tool_schemas)        │     │
│  │           ▼                                         │     │
│  │    ┌──────────────────┐                              │     │
│  │    │ 有 tool_calls?   │                              │     │
│  │    └──┬───────┬───────┘                              │     │
│  │   Yes│       │No                                    │     │
│  │       ▼       ▼                                     │     │
│  │    ┌────────┐ ┌──────────────┐                       │     │
│  │    │ 执行工具│ │ 返回最终回复 │                       │     │
│  │    │dispatch│ │ final_resp. │                       │     │
│  │    └───┬────┘ └──────────────┘                       │     │
│  │        │                                              │     │
│  │    ┌───▼──────────┐                                   │     │
│  │    │ 结果追加消息 │                                   │     │
│  │    │ → 继续循环   │                                   │     │
│  │    └──────────────┘                                   │     │
│  └─────────────────────────────────────────────────────┘     │
│                         ▼                                    │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 3. 后处理（每次用户消息完成后）                      │     │
│  │    ├─ 记忆同步 (MemoryManager.sync_all)              │     │
│  │    ├─ 技能创建评估 (自动生成 skill 的条件检查)        │     │
│  │    ├─ 背景审查 (background_review_callback)          │     │
│  │    ├─ 轨迹保存 (trajectory.py)                       │     │
│  │    └─ 上下文压缩 (context_compressor.py)             │     │
│  └─────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 核心循环伪代码

```python
# conversation_loop.py:774
while (api_call_count < max_iterations and budget.remaining > 0) or grace_call:
    # 中断检查
    if interrupt_requested: break

    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas
    )

    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

---

## 四、关键子系统详解

### 4.1 工具系统（tools/）

**文件**：[tools/registry.py](tools/registry.py)、[toolsets.py](toolsets.py)、[model_tools.py](model_tools.py)

- **注册式架构**：每个工具文件在模块级别调用 `registry.register()` 自注册（`ToolEntry`：schema + handler + toolset + check_fn）
- **自动发现**：`discover_builtin_tools()` 扫描 `tools/*.py`，AST 检测 `registry.register()` 调用后导入
- **Toolset 分组**：`_HERMES_CORE_TOOLS` 定义核心工具集，支持组合/按需启用
- 80+ 工具涵盖类别：

| 类别 | 工具示例 |
|---|---|
| 文件操作 | `read_file`, `write_file`, `patch`, `search_files` |
| 终端 | `terminal`, `process`, `execute_code` |
| Web | `web_search`, `web_extract` |
| 浏览器自动化 | `browser_navigate`, `browser_click`, `browser_snapshot`, `browser_vision` |
| 视觉/图像 | `vision_analyze`, `image_generate`, `video_generation` |
| 记忆/技能 | `memory`, `session_search`, `skills_list`, `skill_view` |
| Kanban | `kanban_show`, `kanban_list`, `kanban_complete` — 多 Agent 协调 |
| Cron | `cronjob` |
| 智能家居 | `ha_list_entities`, `ha_get_state`, `ha_call_service` |
| 消息 | `send_message` — 跨平台消息发送（Gateway 运行时启用） |
| 计算机使用 | `computer_use` — macOS CUA 驱动 |
| 工具链 | `delegate_task`, `todo`, `clarify`, `mixture_of_agents` |

### 4.2 Gateway 消息网关（gateway/）

**入口**：[gateway/run.py](gateway/run.py)

- 统一接入层，一个 Agent 后端服务于**所有平台**
- 每个平台一个 adapter 文件（[gateway/platforms/](gateway/platforms/)）

| 平台 | adapter 文件 |
|---|---|
| Telegram | `telegram.py` |
| Discord | `discord_tool.py` (工具侧) |
| Slack | `slack.py` |
| WhatsApp | `whatsapp.py` |
| Signal | `signal.py` |
| Email (IMAP/SMTP) | `email.py` |
| Matrix | `matrix.py` |
| Mattermost | — |
| 飞书 | `feishu.py` |
| 钉钉 | `dingtalk.py` |
| 企业微信 | `wecom.py` |
| 微信 | `weixin.py` |
| QQ 机器人 | `qqbot/` |
| BlueBubbles (iMessage) | `bluebubbles.py` |
| Home Assistant | `homeassistant.py` |
| Webhook | `webhook.py` |
| API Server | `api_server.py` |
| 腾讯元宝 | `yuanbao.py` |

**关键模块**：
- `gateway/session.py` — 会话管理（上下文追踪、持久化、重置策略）
- `gateway/config.py` — 平台配置（Platform dataclass）
- `gateway/hooks.py` — 事件钩子系统（`agent:step`, `agent:interim` 等）
- `gateway/slash_access.py` — Slash 命令访问控制

### 4.3 记忆系统

| 组件 | 文件 | 职责 |
|---|---|---|
| MemoryManager | `agent/memory_manager.py` | 统一编排多 provider（prefetch/sync/queue） |
| MemoryProvider | `agent/memory_provider.py` | Provider 抽象基类 |
| 会话搜索 (FTS5) | `hermes_state.py` | SQLite FTS5 全文索引，跨会话历史搜索 |
| 上下文压缩 | `agent/context_compressor.py` | 防止 token 溢出 |
| 上下文清理 | `agent/memory_manager.py` | `sanitize_context()` / `StreamingContextScrubber` |

**支持的外部记忆 Provider（插件）**：

| Provider | 路径 |
|---|---|
| Honcho | `plugins/memory/honcho/` |
| Mem0 | `plugins/memory/mem0/` |
| SuperMemory | `plugins/memory/supermemory/` |

### 4.4 Skill 技能系统

**文件**：[agent/skill_commands.py](agent/skill_commands.py)、[agent/skill_preprocessing.py](agent/skill_preprocessing.py)、[agent/skill_utils.py](agent/skill_utils.py)

- **自动创建**：Agent 完成复杂任务后，评估是否提炼为可复用的 skill
- **Slash Command 形式**：`/skill-name`，CLI 和 Gateway 双端可用
- **技能目录结构**：

| 目录 | 用途 |
|---|---|
| `skills/` | 内置技能（约 20+ 类目） |
| `~/.hermes/skills/` | 用户自定义技能 |
| `optional-skills/` | 较重/小众技能，默认不激活 |

- 支持 [agentskills.io](https://agentskills.io) 开放标准
- 技能注入方式：通过 **user message** 注入（非 system prompt），保留 prompt caching 收益

### 4.5 终端环境抽象（tools/environments/）

**文件**：[tools/environments/](tools/environments/)

统一 `TerminalEnvironment` 接口，6 种后端实现：

| 后端 | 文件 | 适用场景 |
|---|---|---|
| Local | `local.py` | 本地机器直接执行 |
| Docker | `docker.py` | 容器化隔离环境 |
| SSH | `ssh.py` | 远程服务器 |
| Singularity | `singularity.py` | HPC/容器平台 |
| Modal | `modal.py` + `managed_modal.py` | Serverless 云（休眠计费） |
| Daytona | `daytona.py` | 云开发环境 |

### 4.6 Cron 调度器（cron/）

**文件**：[cron/scheduler.py](cron/scheduler.py)、[cron/jobs.py](cron/jobs.py)

- 自然语言定义定时任务，如：`"每天早 9 点出日报发送到 Slack"`
- 任务结果投递到任意 gateway 平台
- 支持任务超时/重试/依赖管理

### 4.7 TUI 终端 UI（ui-tui/）

- React Ink 实现的终端界面，`hermes --tui` 启动
- 组件结构：`ui-tui/src/app/components/`
- Python 后端 `tui_gateway/` 提供 JSON-RPC 协议

### 4.8 ACP Adapter（acp_adapter/）

- 实现 [Agent Communication Protocol](https://github.com/nousresearch/hermes-agent/tree/main/acp_adapter)
- 对接 VS Code / Zed / JetBrains IDE

### 4.9 研究工具

| 组件 | 文件 | 用途 |
|---|---|---|
| batch_runner | `batch_runner.py` | 批量轨迹生成 |
| 轨迹压缩 | `agent/trajectory.py` | 轨迹格式转换、保存 |
| 轨迹格式 | `run_agent.py` | `_convert_to_trajectory_format()` |

---

## 五、依赖关系链

```
tools/registry.py          ← 基石，无外部依赖
      │
      ▼
tools/*.py                 ← 各工具文件，import 时调用 registry.register()
      │
      ▼
model_tools.py             ← 导入 registry + 导入所有工具模块触发发现
      │
      ├──────────────────────────────────────┐
      ▼                                      ▼
run_agent.py  (AIAgent ~12k LOC)     cli.py (HermesCLI ~15k LOC)
      │
      ▼
agent/conversation_loop.py  ← 核心循环 (~4.6k LOC)
      │
      ├── agent/prompt_builder.py        — 系统提示词构建
      ├── agent/memory_manager.py        — 记忆编排
      ├── agent/context_compressor.py    — 上下文压缩
      ├── agent/retry_utils.py           — 失败重试
      ├── agent/error_classifier.py      — 错误分类 + 故障转移
      ├── agent/model_metadata.py        — 模型元数据/上下文长度探测
      ├── agent/display.py              — KawaiiSpinner 动画
      ├── agent/think_scrubber.py        — 推理内容清洗
      ├── agent/tool_guardrails.py       — 工具调用安全护栏
      ├── agent/tool_dispatch_helpers.py — 工具并行调度/冲突检测
      ├── agent/redact.py               — 敏感信息脱敏
      └── agent/prompt_caching.py        — Anthropic prompt caching
```

**Gateway 侧依赖**：

```
gateway/run.py
      │
      ├── gateway/session.py        — 会话管理
      ├── gateway/config.py         — 平台配置（Platform dataclass）
      ├── gateway/hooks.py          — 事件钩子（step/interim 等）
      ├── gateway/slash_access.py   — 命令访问控制
      ├── gateway/delivery.py       — 消息投递
      ├── gateway/mirror.py         — 消息镜像
      ├── gateway/platforms/        — 20+ 平台 adapter
      └── gateway/builtin_hooks/    — 内置钩子扩展点
              │
              ▼ (内部创建 AIAgent 实例)
         run_agent.py
```

---

## 六、核心概念一览

| 概念 | 说明 | 关键文件 |
|---|---|---|
| **AIAgent** | 核心 Agent 类，管理会话循环、工具路由、记忆、模型调用 | `run_agent.py` |
| **Tool Registry** | 自注册工具中心，所有工具统一注册 schema + handler | `tools/registry.py` |
| **Toolset** | 工具分组，预定义组合（research/development/full_stack 等） | `toolsets.py` |
| **Gateway** | 消息网关，多平台统一接入 | `gateway/run.py` |
| **MemoryManager** | 记忆编排，管理多 provider 的 prefetch/sync | `agent/memory_manager.py` |
| **SessionDB** | SQLite FTS5 会话存储，支持全文搜索 | `hermes_state.py` |
| **Skill** | 可复用技能，自动创建或手动编写，Slash Command 形式 | `agent/skill_commands.py` |
| **TerminalEnvironment** | 终端抽象层，6 种后端统一接口 | `tools/environments/` |
| **IterationBudget** | 双重预算控制（调用次数 + token 预算） | `agent/iteration_budget.py` |
| **ContextCompressor** | 上下文压缩，防止 token 溢出 | `agent/context_compressor.py` |
| **Cron** | 自然语言定时任务调度 | `cron/` |
| **TUI** | React Ink 终端 UI | `ui-tui/` |
| **ACP** | IDE 集成协议 (VS Code/Zed/JetBrains) | `acp_adapter/` |

---

## 七、项目目录结构关键索引

```
hermes-agent/
├── run_agent.py                  # AIAgent 核心 (~12k LOC)
├── model_tools.py                # 工具编排枢纽 (~923 LOC)
├── toolsets.py                   # Toolset 定义 (~882 LOC)
├── cli.py                        # CLI 交互入口 (~15k LOC)
├── hermes_state.py               # SQLite 会话存储 (~3.3k LOC)
├── hermes_constants.py           # 配置文件路径
├── hermes_logging.py             # 日志配置
├── batch_runner.py               # 批量轨迹生成
│
├── agent/                        # Agent 内部模块
│   ├── conversation_loop.py      #   核心循环 (~4.6k LOC)
│   ├── prompt_builder.py         #   提示词构建
│   ├── memory_manager.py         #   记忆编排 (~640 LOC)
│   ├── skill_commands.py         #   Skill 命令 (~523 LOC)
│   ├── context_compressor.py     #   上下文压缩
│   ├── tool_guardrails.py        #   工具安全护栏
│   ├── tool_dispatch_helpers.py  #   并行调度/冲突检测
│   └── ...                       #   显示/重试/错误分类/脱敏等
│
├── tools/                        # 工具实现
│   ├── registry.py               #   注册中心 (~589 LOC)
│   ├── terminal_tool.py          #   终端执行
│   ├── file_tools.py             #   文件操作
│   ├── web_tools.py              #   Web 搜索
│   ├── browser_tool.py           #   浏览器自动化
│   ├── environments/             #   6 种终端后端
│   └── ...                       #   80+ 工具
│
├── gateway/                      # 消息网关
│   ├── run.py                    #   网关入口
│   ├── session.py                #   会话管理 (~1.3k LOC)
│   ├── config.py                 #   平台配置
│   ├── hooks.py                  #   事件钩子
│   ├── delivery.py               #   消息投递
│   └── platforms/                #   20+ 平台 adapter
│
├── plugins/                      # 插件系统
│   ├── memory/                   #   记忆 provider 插件
│   ├── model-providers/          #   模型 provider 插件
│   ├── kanban/                   #   多 Agent 看板
│   └── ...                       #   观察性/图像生成/平台等
│
├── cron/                         # Cron 调度
├── skills/                       # 内置技能 (~20+ 类目)
├── optional-skills/              # 可选技能
├── ui-tui/                       # React Ink TUI
├── tui_gateway/                  # TUI JSON-RPC 后端
├── acp_adapter/                  # IDE 集成协议
├── website/                      # Docusaurus 文档站
└── tests/                        # ~17k 测试用例
```

---

## 八、部署方式

```bash
# 一键安装 (Linux/macOS/WSL2/Termux)
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 或 PowerShell (Windows)
iex (irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1)
```

**常用命令**：

| 命令 | 用途 |
|---|---|
| `hermes` | 启动交互式 CLI |
| `hermes model` | 选择 LLM 提供商和模型 |
| `hermes tools` | 配置启用哪些工具集 |
| `hermes gateway` | 启动消息网关 |
| `hermes setup` | 完整设置向导 |
| `hermes update` | 更新到最新版 |
| `hermes doctor` | 诊断问题 |

**支持的 LLM 提供商**：OpenRouter（200+ 模型）、Nous Portal、NVIDIA NIM、Xiaomi MiMo、z.ai/GLM、Kimi、MiniMax、Hugging Face、OpenAI、Anthropic，或任何兼容端点。
