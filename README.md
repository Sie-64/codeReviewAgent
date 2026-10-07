# Code Review Agent（代码审查 Agent）

> 一个用「原生手写 ReAct 循环 + OpenAI 兼容接口」实现的命令行代码审查助手：读取代码、动手验证、输出结构化审查报告。

## 1. 项目简介

本项目是 **Code Agent 1** 作业的实现，方向为 **代码审查 Agent**。
它把 LLM 当作「推理引擎」，把文件读取与代码执行当作「动手能力」，用 ReAct（Thought → Action → Observation）模式驱动二者协作。

功能列表：

- 🔎 **代码审查**：分析代码质量、发现潜在 Bug、评估健壮性与可维护性。
- 📄 **文件读取工具**：`read_file` 读取带行号的源码，超大文件会被拒绝并提示缩小范围。
- ▶️ **代码执行工具**：`execute_code` 在 `node:vm` 沙箱中运行一小段 JS，用来**复现问题**（例如验证空数组求平均会得到 `NaN`）。
- 🧠 **多轮记忆**：数组式对话记忆 + 滑动窗口裁剪，防止上下文无限增长。
- 🛡️ **错误处理与重试**：超时 / 429 限流自动指数退避重试，工具错误以 `ERROR:` 字符串回灌给模型自我修正。
- 💬 **CLI 交互**：支持单次审查（`--file` / `--code`）与持续对话两种模式。
- 🔌 **模型可切换**：任何 OpenAI 兼容接口都可接入（OpenAI / DeepSeek / 通义千问 / Kimi …）。

## 2. 环境安装

要求 **Node.js >= 18**（需要原生 `fetch`、ESM 与 `node:vm`）。

```bash
node --version   # 需 >= 18
npm install
cp .env.example .env
```

> Windows（PowerShell）下复制命令为：`copy .env.example .env`

> ⚠️ **`cp .env.example .env` 只做一次。** 如果 `.env` 已经配置好，再执行这条命令会把你的 Key 覆盖回模板占位符
> `your_api_key_here`，之后运行会收到 401（报错长这样：`401 Authentication Fails, Your api key: ****here is invalid`）。
> 遇到这个报错先打开 `.env` 确认 Key 是不是被覆盖了。

## 3. 配置 API Key

编辑项目根目录的 `.env`（本项目当前默认接入 **DeepSeek**）：

```dotenv
OPENAI_API_KEY=sk-你的密钥
OPENAI_BASE_URL=https://api.deepseek.com/v1
OPENAI_MODEL=deepseek-chat
```

### 切换模型

只需改 `OPENAI_BASE_URL` 与 `OPENAI_MODEL` 两项，代码无需任何改动：

| 服务 | OPENAI_BASE_URL | OPENAI_MODEL 示例 |
|---|---|---|
| **DeepSeek（当前默认）** | `https://api.deepseek.com/v1` | `deepseek-chat` |
| OpenAI | `https://api.openai.com/v1` | `gpt-4o-mini` |
| 通义千问（DashScope 兼容模式） | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-plus` |
| Kimi（Moonshot） | `https://api.moonshot.cn/v1` | `moonshot-v1-8k` |

> 关于 DeepSeek 模型选择：推荐 `deepseek-chat`（DeepSeek-V3），它支持 `temperature`，格式遵循稳定。
> `deepseek-reasoner`（R1）会输出思维链且对 `temperature` 等采样参数的支持有限，本项目的 ReAct
> 格式依赖较低温度来稳定输出，因此默认用 `deepseek-chat`。

其余可调参数（同样位于 `.env`）：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `LLM_TIMEOUT` | `60000` | 单次 LLM 请求超时（毫秒） |
| `LLM_MAX_RETRIES` | `3` | 单次请求最大重试次数 |
| `AGENT_MAX_STEPS` | `8` | 单轮审查最多 ReAct 步数 |
| `AGENT_MAX_MEMORY_TURNS` | `10` | 记忆保留的最大对话轮数 |

## 4. 如何运行

```bash
# 1) 审查一个文件（Agent 会先调用 read_file）
npm start -- --file examples/sampleBuggy.js
# 等价快捷脚本
npm run review:file

# 2) 直接审查一段代码
npm start -- --code "function add(a,b){return a+b}"

# 3) 交互模式（多轮对话，可持续追问）
npm start

# 4) 打开调试日志，观察每一步 Thought / Action（日志走 stderr 友好提示）
npm start -- --file examples/sampleBuggy.js --verbose
```

退出交互模式：输入 `exit`（也支持 `quit` / `q`）。

## 5. 架构说明

核心数据流（四层分离，推理与执行解耦）：

```
用户输入（文件路径 / 代码片段）
        │
        ▼
[Memory] 记录 user 消息 + ReAct 提示
        │
        ▼
[llmClient] 调用 LLM 获取输出（带重试）
        │
        ▼
[Agent] 解析输出
        ├── 是 Final Answer ──▶ 返回审查报告（结束）
        └── 是 Action ──▶ [registry] 调用工具
                              │
                              ▼
                        得到 Observation
                              │
                              ▼
                        [Memory] 记录 Observation
                              │
                              └──▶ 回到 LLM 调用（下一轮）
```

分层职责：

| 层 | 文件 | 职责 |
|---|---|---|
| 入口层 | `src/index.js` | CLI 参数解析、交互循环、展示结果 |
| 核心层 | `src/core/agent.js` | ReAct 主循环、输出解析、工具调度 |
| 核心层 | `src/core/llmClient.js` | LLM 调用封装、超时、重试与退避 |
| 核心层 | `src/core/memory.js` | 对话记忆与裁剪 |
| 核心层 | `src/core/prompts.js` | System Prompt 与 ReAct 提示语 |
| 工具层 | `src/tools/registry.js` | 工具注册表与统一调度 |
| 工具层 | `src/tools/fileTool.js` | `read_file` 实现 |
| 工具层 | `src/tools/codeTool.js` | `execute_code` 实现 |
| 配置层 | `src/config.js` | 环境变量读取与校验 |

## 6. 项目结构

```
code-review-agent/
│
├── package.json                 # 项目清单：依赖、脚本、ESM 声明
├── .env.example                 # 环境变量模板
├── .gitignore                   # 忽略 node_modules、.env
│
├── src/
│   ├── index.js                 # 【入口层】CLI 入口 + 交互循环
│   ├── config.js                # 【配置层】读取环境变量
│   │
│   ├── core/
│   │   ├── agent.js             # Agent 主循环（ReAct）
│   │   ├── llmClient.js         # LLM 调用封装（重试/超时）
│   │   ├── memory.js            # 对话记忆（含裁剪）
│   │   └── prompts.js           # System Prompt + ReAct 模板
│   │
│   └── tools/
│       ├── registry.js          # 工具注册表 + 统一调度
│       ├── fileTool.js          # read_file 工具
│       └── codeTool.js          # execute_code 工具
│
├── examples/
│   └── sampleBuggy.js           # 含典型 Bug 的示例代码
│
├── README.md                    # 项目说明文档
└── Design.md                    # 设计文档
```

## 7. 工具说明

所有工具都遵循同一协议：**接收一个对象参数，字段名与 Prompt 中声明一致；失败时返回以 `ERROR:` 开头的字符串而不是抛异常**。

| 工具 | 参数 | 限制 | 返回 |
|---|---|---|---|
| `read_file` | `{ file_path: string }` | 必须是普通文件；大小 ≤ 200 KB（`MAX_FILE_BYTES`） | 带行号的全文（格式 `  12 \| const a = 1;`），或 `ERROR: ...` |
| `execute_code` | `{ code_snippet: string }` | 长度 ≤ 20000 字符（`MAX_CODE_CHARS`）；执行超时 5s；命中黑名单直接拒绝 | 标准输出 / 标准错误 / 异常堆栈 / `返回值`，或 `ERROR: ...` |

`execute_code` 黑名单关键词：`require(`、`process.exit`、`child_process`、`fs.unlink`、`fs.rmdir`、`fs.rm(`。

调用链示例：

```
Action: read_file
Action Input: {"file_path": "examples/sampleBuggy.js"}

      ↓ Agent 解析

callTool("read_file", { file_path: "examples/sampleBuggy.js" })
```

## 8. 注意事项

- ⚠️ **`execute_code` 不是安全沙箱**。`node:vm` 只做上下文隔离，理论上仍可被逃逸利用。本项目用黑名单、长度限制、5 秒超时、关闭动态代码生成来降低风险，但**生产环境请改用 `worker_threads` + 超时或独立容器隔离**。
- ⚠️ `examples/sampleBuggy.js` 中的 `module.exports` 是**刻意的**：它用于演示 `require(` 黑名单会被命中。由于项目为 ESM（`"type": "module"`），该文件仅作为「被审查的文本」，请勿直接 `node` 运行它。被审查对象本身可以是任意语言的代码。
- 🔧 **模型输出格式不稳时**：项目已把 `temperature` 固定为 `0.2` 并在 System Prompt 中给出严格格式示例；若仍频繁输出非法格式，建议换用指令遵循能力更强的模型，或考虑改用 function calling（见 `Design.md` 的可扩展方向）。
- 💰 **长对话要注意 Token 成本**：记忆只做滑动窗口裁剪（不做摘要压缩），`AGENT_MAX_MEMORY_TURNS` 越大成本越高。
- 🚪 **API Key 为空会直接退出**：CLI 会以红字提示并在退出码 `1` 结束，避免误以为在正常请求。
