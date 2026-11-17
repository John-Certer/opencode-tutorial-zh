# opencode-tutorial-zh

[![npm](https://img.shields.io/npm/v/opencode-ai)](https://www.npmjs.com/package/opencode-ai)
[![platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-blue)](https://opencode.ai)
[![license](https://img.shields.io/badge/docs-CC%20BY%204.0-green)](./README.md)

> 中文教程：把 OpenCode（开源终端 AI 编程 Agent）从零装好、配好、跑起来——含免费模型用法、自定义 Agent、无人值守自动化实战，以及实际踩过的坑。

[English projects welcome · 官方仓库](https://github.com/sst/opencode) · [官方文档](https://opencode.ai/docs)

## 目录

- [OpenCode 是什么](#opencode-是什么)
- [安装](#安装三选一)
- [第一次对话：免费模型](#第一次对话选个免费模型就能用)
- [配置文件 opencode.jsonc](#配置文件opencodejsonc)
- [自定义 Agent](#自定义-agent一个-markdown-文件就是一个角色)
- [非交互模式 opencode run](#非交互模式opencode-run自动化的灵魂)
- [实战：无人值守自动化](#实战把-opencode-当无人值守自动化引擎)
- [常见坑速查表](#常见坑速查表)
- [一页速查](#一页速查)

## OpenCode 是什么

OpenCode 是一个运行在**终端**里的 AI 编程智能体（Agent）。和网页版聊天助手最大的区别是：它不只"说"，还能"做"——

- 直接读写你磁盘上的文件、执行 shell 命令；
- 通过自定义 Agent（一个 Markdown 文件）塑造专属角色和行为约束；
- 提供 `opencode run` 非交互模式，可以嵌进脚本、CI、自动化流水线里无人值守跑；
- 模型随便换：OpenAI、Anthropic、智谱、火山方舟、DeepSeek……甚至内置了一批**免登录的免费模型**。

## 安装（三选一）

**npm（Windows / macOS / Linux 通用）**

```bash
npm i -g opencode-ai
opencode --version    # 输出 1.19.6 之类即成功
```

前提是装过 Node.js（`node -v` 能出版本号就行）。

**一键脚本（macOS / Linux）**

```bash
curl -fsSL https://opencode.ai/install | bash
```

**Windows 安装包**：去 [GitHub Releases](https://github.com/sst/opencode/releases) 下载对应平台的安装包。

装完在任意终端敲 `opencode` 回车进入 TUI；`Ctrl+C` 两次退出。

## 第一次对话：选个免费模型就能用

进入 TUI 后按提示选择模型。**模型列表里标 `Free` 的模型不需要任何 API key**，选中直接聊：

- `GLM-5.3-Flash Free`（智谱 GLM 免费档）
- `Big Pickle`（官方匿名模型）
- `Qwen3 Coder Free`、`Kimi K2 Free` 等

<img width="1730" height="924" alt="glm-free-tui" src="https://github.com/user-attachments/assets/87629e5a-d65f-4252-a4c0-9dae73197db6" />


> 实测：TUI 里选中 GLM-5.3-Flash Free 直接开聊，无 API key、无登录。

正式干活再接自己的 API key（`opencode auth login`）。

## 配置文件：opencode.jsonc

配置文件放在**项目根目录**（只对该项目生效）或 `~/.config/opencode/`（全局生效）。实用模板——放行文件读写和命令执行、禁止联网、接入智谱/火山：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "my-agent",
  "permission": {
    "edit": "allow",
    "bash": "allow",
    "webfetch": "deny",
    "websearch": "deny"
  },
  "provider": {
    "zhipu": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "智谱 GLM",
      "options": { "baseURL": "https://open.bigmodel.cn/api/paas/v4" },
      "models": { "glm-5.3-flash": { "name": "GLM-5.3-Flash" } }
    },
    "volcano": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "火山方舟 Doubao",
      "options": { "baseURL": "https://ark.cn-beijing.volces.com/api/v3" },
      "models": { "doubao-seed-2-1-pro": { "name": "Doubao-Seed-2.1-pro" } }
    }
  }
}
```

API key 用环境变量传入（`ZHIPU_API_KEY` / `ARK_API_KEY`），不要写进配置文件。

> ⚠️ **必踩的坑**：OpenCode 1.x 用的是 `"permission"`（单数），写成 `"permissions"`（复数，V2 语法）会直接报错 `Configuration is invalid`，且报错信息不会指明是哪个键错了。

## 自定义 Agent：一个 Markdown 文件就是一个角色

在项目下建 `.opencode/agents/` 目录，放一个 Markdown 文件，**文件名就是角色名**：

```markdown
<!-- .opencode/agents/my-agent.md -->
---
description: 一个严谨的代码审查员
mode: primary
temperature: 0.1
---

你是资深代码审查员。审查时按严重程度排序输出问题清单，
只报告问题，不要擅自修改文件。
```

规则三条：

1. **frontmatter** 是运行配置：`description` 简介、`mode: primary` 可作主角色直接运行（`subagent` 为子角色）；
2. **正文**就是 system prompt；
3. 保存后验证：`opencode agent list` 输出 `my-agent (primary)` 即成功。

<img width="1713" height="591" alt="agent-demo" src="https://github.com/user-attachments/assets/a31bc4a7-461f-48bd-af73-07cee43e51e3" />


> 实测：写好 `code-reviewer.md` 后一条命令调用（GLM-5.3-Flash Free），模型严格按人设输出——按严重度排序、带行号、一句总评，行为完全被 Markdown 定义。

权限精细化（frontmatter 内，禁止角色改文件）：

```yaml
permissions:
  - action: edit
    resource: "*"
    effect: deny
```

## 非交互模式：opencode run（自动化的灵魂）

TUI 给人用，`opencode run` 给**脚本**用：

```bash
# 一句话任务
opencode run --model opencode/big-pickle "统计 src 下各语言代码行数"

# 指定自定义角色
opencode run --model zhipu/glm-5.3-flash --agent my-agent "审查最近的改动"

# JSON 事件流（方便程序解析）
opencode run --agent my-agent "生成周报" --format json > events.jsonl
```

三个必须知道的特性：

1. **无人值守下 `ask` 权限自动被拒绝**——自动化场景要么 `allow`、要么 `deny`，别用 `ask`；
2. **退出码不可全信**——即使任务没完成也可能 exit 0，脚本里要检查产物；
3. **角色文件自动加载**——`.opencode/agents/` 下的文件在 run 模式下 `--agent 角色名` 直接生效。

## 实战：把 OpenCode 当"无人值守自动化引擎"

一个真实场景：参加 AI 攻防比赛，提交 Agent 任务包，主办方把包挂进容器里的 OpenCode，由大模型驱动自动执行渗透任务。本地复刻这套环境做赛前演练：

```
my-project/
├── opencode.jsonc                  # 配置：放行 bash/edit、禁网
├── .opencode/agents/
│   └── awd-attack-agent.md         # 角色：攻击策略 + 行为约束（system prompt）
├── agent/                          # 任务包（对应容器里的挂载点）
└── output/                         # 产出目录
```

环境变量注入任务参数，一条命令跑 30 分钟演练：

```bash
export TARGET_ENDPOINT=http://127.0.0.1:8899   # 模拟平台下发的目标
timeout 30m opencode run \
  --model zhipu/glm-5.3-flash \
  --agent awd-attack-agent \
  "$(python3 -c "import yaml;print(yaml.safe_load(open('agent/run.yaml'))['prompt'])")"
```

**配置文件管权限 + Markdown 管角色 + 环境变量管输入 + run 管执行**——稍加改造就是通用自动化任务执行器：定时数据清洗、批量代码迁移、无人值守运维脚本，同一个套路。

## 常见坑速查表

| 坑 | 现象 | 解决 |
| --- | --- | --- |
| 配置写成 `permissions` | `Configuration is invalid` | 1.x 用单数 `permission` |
| 角色文件没生效 | `agent list` 里找不到 | 目录是 `.opencode/agents/`（复数）且文件名即角色名 |
| run 模式卡在某一步 | `ask` 权限被自动拒绝 | 自动化场景改 `allow` 或 `deny` |
| Windows 下 `python3` 静默退出 | 敲了没反应 | Windows 用 `python`，`python3` 常是商店占位符 |
| 免费模型偶尔排队慢 | 响应等很久 | Free 档共享额度，错峰用或换自己的 key |

## 一页速查

```bash
npm i -g opencode-ai          # 安装
opencode                      # TUI 交互模式
opencode agent list           # 查看已加载的角色
opencode auth login           # 配置 API key（免费模型可跳过）
opencode run "任务"            # 非交互执行
opencode run -m 模型 --agent 角色 "任务"   # 指定模型与角色
opencode run --format json "任务" > out.jsonl  # JSON 事件流
```

核心心智模型：**TUI 给人用，run 给脚本用；配置文件定权限，Markdown 定人格，环境变量定输入**。

## Contributing

发现新坑或新技巧欢迎提 PR / Issue；教程随 OpenCode 版本更新（当前基于 v1.19.6 实测）。

## License

本文档以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布，转载请保留署名与链接。
