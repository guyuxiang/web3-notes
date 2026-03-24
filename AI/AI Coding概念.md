# AI Coding 概念

## Anthropic 官方课程汇总

课程列表（共13门），学完有证书

1. 访问 https://anthropic.skilljar.com/

| #    | 课程名称                                    | 简介                               |
| ---- | ------------------------------------------- | ---------------------------------- |
| 1    | **Claude 101**                              | 了解 Claude 日常使用方法和核心功能 |
| 2    | **Claude Code in Action**                   | 将 Claude Code 集成到开发工作流    |
| 3    | **AI Fluency: Framework & Foundations**     | 与 AI 系统有效协作的fluency技能    |
| 4    | **Building with the Claude API**            | 使用 Claude API 构建应用（全面）   |
| 5    | **Introduction to Model Context Protocol**  | 用 Python 构建 MCP 服务器和客户端  |
| 6    | **Model Context Protocol: Advanced Topics** | MCP 高级主题（采样、通知、传输）   |
| 7    | **Introduction to agent skills**            | 在 Claude Code 中构建和共享 Skills |
| 8    | **Claude with Amazon Bedrock**              | 通过 AWS Bedrock 使用 Claude       |
| 9    | **Claude with Google Cloud's Vertex AI**    | 通过 Google Vertex AI 使用 Claude  |
| 10   | **AI Fluency for educators**                | 教育工作者 AI fluency              |
| 11   | **AI Fluency for students**                 | 学生 AI fluency                    |
| 12   | **Teaching AI Fluency**                     | 教授 AI fluency                    |
| 13   | **AI Fluency for nonprofits**               | 非营利组织 AI fluency              |



## 1. MCP (Model Context Protocol)

#### 定义

**Model Context Protocol（MCP）** 是一种 **AI 模型与外部工具 / 数据源之间的标准通信协议**。
 它由 **Anthropic** 提出，目标是让 **LLM（大模型）可以像调用 API 一样调用各种工具、数据库、系统能力**，并且形成统一规范。

官方描述：[Model Context Protocol](https://modelcontextprotocol.io/introduction)

> Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems.

（可以把 MCP 想象成Web开发的 RPC 接口。正如 JSON-RPC 为连接以太坊节点提供标准化方式，MCP 为连接 AI 应用与外部系统提供标准化方式。）

```
+------------------+
|      AI Model     |
| (Claude / GPT)    |
+---------+---------+
          |
          | MCP
          |
+---------v---------+
|     MCP Client     |
| (Agent / CLI)      |
+---------+---------+
          |
          | MCP
          |
+---------v---------+
|     MCP Server     |
|  (Tool Provider)   |
+---------+---------+
          |
      Tools / Data
```

#### MCP 核心概念

根据 [MCP 官方文档](https://modelcontextprotocol.io/docs/getting-started/intro.md)：

| 概念 | 说明 |
|------|------|
| **MCP Client** | AI Agent（如 Claude Code、OpenAI） |
| **MCP Server** | 提供工具、数据、资源的服务端       |
| **Tools**      | AI 可以调用的函数                  |
| **Resources**  | 可读取的数据文件                   |
| **Prompts**    | 预定义的提示模板                   |



#### Coding案例

#### **Solana MCP**

https://mcp.solana.com/

是一个专门为 Solana 开发者设计的服务端程序。它让 AI 能够直接读取 Solana 的官方文档、Anchor开发框架、协助编写和调用智能合约（Programs）、查询链上数据。

**使用示例**

```bash
# 添加 Solana MCP 服务器
claude mcp add --transport http solana-mcp-server https://mcp.solana.com/mcp

# codex
codex mcp add solana-mcp-server --url https://mcp.solana.com/mcp
```

**示例查询：**

- “Anchor 0.31 中是如何实现 CPI 事件的？”
- “构建一个支持 token-2022 及更早版本代币的 AMM。”
- “如何实现带有时间锁定奖励的质押机制？”
- “在 Solana 程序中处理十进制值的最佳实践是什么？”

![image-20260303114115259](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303114115259.png)



#### OpenZeppelin Contracts MCP

https://mcp.openzeppelin.com/

OpenZeppelin 推出 **Contracts MCP**，这是一个基于服务器的引擎，可将 OpenZeppelin 的 Contracts 经过验证的安全性以及样式规则直接引入任何 AI 驱动的开发工作流程。**自动执行 OpenZeppelin 标准** 你生成的每一行智能合约代码都会根据驱动我们 Contracts Wizard 的相同规则集进行验证。 导入、修饰符、命名约定、安全检查

**使用示例**

```bash
# 添加 Solidity 合约 MCP 服务器
# claude code
claude mcp add -t http OpenZeppelinSolidityContracts https://mcp.openzeppelin.com/contracts/solidity/mcp
# codex
codex mcp add OpenZeppelinSolidityContracts --url https://mcp.openzeppelin.com/contracts/solidity/mcp

# Uniswap Hooks
# 基于OpenZeppelin模板生成Uniswap Hooks安全智能合约
claude mcp add -t http OpenZeppelinUniswapHooks https://mcp.openzeppelin.com/contracts/uniswap-hooks/mcp
```

**示例查询：**

![image-20260303115500720](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303115500720.png)



#### MCP 生态

MCP 官方维护着**数百个预构建服务器**：

- **GitHub**: 代码管理、PR、Issues
- **Notion**: 文档和知识库
- **Slack**: 团队通讯
- **PostgreSQL/MySQL**: 数据库查询
- **Figma**: 设计文件访问
- **Sentry**: 错误监控
- 等等...

- [MCP Registry](https://github.com/modelcontextprotocol/servers)
- [Anthropic MCP Registry](https://www.anthropic.com/mcp-registry)



**CLI VS MCP **

大语言模型在命令行工具的使用上表现得极为出色。大模型自己可以使用 CLI 工具智能地交互、采样数据、处理

-  LLM 天生擅长在shell中使用 CLI，大语言模型在 **命令行工具（CLI）交互** 上表现非常好，LLM 根本不需要一个MCP这种特殊协议
-  CLI 本质上和 MCP类似，也有自描述（--help）、可组合性(多参数)，并且人类和机器都可读可用

今年的趋势是提供CLI，Google刚刚开源了Workspace CLI项目，把Gmail、Drive、Docs、Sheets、Calendar、Chat等全套办公工具装进了命令行。

![image-20260305141840404](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260305141840404.png)



## 2. Skill (技能)

#### 定义

**Skill** 是封装好的**可复用能力单元**，让 AI 能够执行特定类型的任务。

#### OpenAI Skills

根据 [OpenAI 官方文档](https://platform.openai.com/docs/guides/tools-skills)：

> Skills allow you to upload and reuse versioned skill bundles in hosted shell environments.

（Skills 允许你在托管的 shell 环境中上传和复用版本化的技能包。）

#### 

从本质上讲，技能是一个包含`SKILL.md`文件的文件夹。

```
my-skill/
├── SKILL.md          # 技能定义和说明(最少包含一个 SKILL.md 文件)
├── scripts/          # 可选：自动化脚本
├── references/       # 可选：参考文档
├── assets/           # 可选：模板、资源
```

实际就是以下的打包

- prompt
- workflow
- 工具调用
- 参考文档

#### Skill 示例

以 OpenClaw 的 `weather` skill 为例：

```markdown
# SKILL.md
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents.
---

# PDF Processing

## When to use this skill
Use this skill when the user needs to work with PDF files...

## How to extract text
1. Use pdfplumber for text extraction...

## How to fill forms
...
```

#### Skill构建指南

https://priv-sdn-001.mowen.cn/mo/file/meta/10/36/46/2026580254842736641.pdf?Expires=1772095759&OSSAccessKeyId=LTAI5tE16jzdfWCPVBmyB5Nn&Signature=44erL1lzQrJOB8A4Rw1zaXlyuSA%3D&response-expires=Thu%2C%2026%20Feb%202026%2008%3A49%3A19%20GMT

**Agent Skills 工作流程**

（1）阶段一：发现（Discovery）

　　AI 启动时，只读取所有 Skill 的 name 和 description 

（2）阶段二：激活（Activation）

　　当你的任务匹配某个 Skill 的描述时，智能体会将完整的`SKILL.md`指令解读到上下文中。

（3）阶段三：执行（Execution）

　　按照 Skill 中的步骤执行任务。如果需要，调用 scripts/ 里的脚本或读取 references/ 里的文档



#### Coding案例

#### 测试







#### 部署

#### vercel-labs/agent-skills

让你的 AI Agent **可以直接操作 Vercel 平台**，实现自动部署、管理项目、操作环境变量等 DevOps 能力。

```
# 安装
npx skills add vercel-labs/agent-skills
```

- 创建 Vercel 项目
- 连接 GitHub 仓库
- 管理环境变量
- 触发部署
- 获取部署 URL



#### solidity-gas-optimization Skill

智能合约Gas优化的skill，这是一份全面的 Solidity 智能合约 Gas 优化指南，包含 80 多种技术，涵盖 8 个类别。

指南基于 RareSkills 出版的《Gas 优化手册》。规则的优先级排序依据是影响和安全性。

https://skills.sh/pseudoyu/agent-skills/solidity-gas-optimization



#### Uniswap Skills

Uniswap 面向开发者的 AI 工具，提供最新的、特定于 Uniswap 协议、API 和智能合约的指导，帮助集成互换、构建 v4 钩子、提供流动性，并从您的编辑器内与 EVM 进行交互。

```
# Claude Code Marketplace
/plugin marketplace add uniswap/uniswap-ai

# Install individual plugins
/plugin install uniswap-hooks      # v4 hook development
/plugin install uniswap-trading    # Swap integration
/plugin install uniswap-cca        # CCA auctions
/plugin install uniswap-driver     # Swap & liquidity planning
/plugin install uniswap-viem       # EVM integration (viem/wagmi)
```

| Plugin              | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| **uniswap-trading** | 通过[交易 API](https://developers.uniswap.org/dashboard)、通用路由器 SDK 或直接合约调用集成互换交易。 |
| **uniswap-hooks**   | 构建 Uniswap v4 hooks 的安全优先指南。                       |
| **uniswap-viem**    | viem和wagmi集成。                                            |
| **uniswap-driver**  | 利用深度链接进行代币发现和兑换/流动性规划。                  |
| **uniswap-cca**     | 配置并部署用于代币分发的 CCA 合约。                          |

**另外Uniswap 提供两个上下文文件：**

预先提供相关文档有助于模型给出更准确的答案，避免出现错误信息。

- **[llms.txt](https://docs.uniswap.org/v4-llms.txt)**：包含指向文档章节链接的简洁摘要。适用于大多数模型（10万+ token 上下文窗口）。
- **[llms-full.txt](https://docs.uniswap.org/v4-llms-full.txt)**：包含更多内联内容的详细版本。如果您的模型具有较大的上下文窗口，或者您希望在不点击链接的情况下查看更多详细信息，请使用此版本。



## 3. Memory (记忆)

#### 定义

**Memory** 是 AI Agent**存储和检索信息**的机制，使其能够跨越多轮对话保持上下文。

#### 记忆类型

| 类型 | 持续时间 | 存储位置 | 示例 |
|------|----------|----------|------|
| **Working Memory** | 当前会话 | 上下文窗口 | 对话历史 |
| **Short-term** | 短期 | 会话存储 | 用户偏好 |
| **Long-term** | 长期 | 外部数据库 | 累计知识 |

#### OpenClaw 记忆机制

```markdown
## 记忆文件结构
memory/
├── YYYY-MM-DD.md    # 每日日志
└── MEMORY.md        # 长期记忆
```



## 4. Rules

#### 定义

**Rules** 是定义 AI 行为边界、性格和能力的**显式指令**。

- 全量加载：启动时全部加入上下文
- 始终生效：每次对话都遵守
- 单纯的 Markdown 文件

**适合用 Rules 的场景：**特征规则 - 短小、通用、始终需要

#### Rules 设计原则

1. **明确性**: 清晰定义行为边界
2. **优先级**: 重要规则放在前面
3. **一致性**: 避免规则冲突
4. **可观测**: 便于调试和优化

#### Coding案例

**案例1：前端开发规则**
```markdown
instructions = """
你是一个 Vue3 开发专家。

## 代码规范
1. 使用 Composition API + `<script setup>`
2. 使用 TypeScript，类型要完整
3. 组件名使用 PascalCase
4. Props 使用 defineProps<{}>()

## 目录结构
- components/ - 公共组件
- views/ - 页面组件
- composables/ - 组合式函数
- utils/ - 工具函数
- api/ - 接口定义

## 禁止
- 不使用 Options API
- 不使用 any 类型
- 不直接操作 DOM
"""
```



## 5. Subagents(子代理)

#### 定义

Subagents 是负责处理特定类型任务的专用 AI 代理。

每个 subagent 都在独立的上下文窗口中运行，拥有自定义系统提示、专属工具调用权限与独立操作权限。

官方描述

> 在内部研究评估中，由 Claude Opus 4 作为主代理、Claude Sonnet 4 作为子代理的多代理系统比单代理 Claude Opus 4 的表现高出 90.2%。
>
> — Anthropic，*How we built our multi-agent research system*



在使用 Claude Code 处理复杂任务时，你可能遇到过这样的困境：主对话的上下文越来越长，AI 开始"忘记"之前的重要信息，响应质量逐渐下降。

Subagent（子代理）就是为解决这个问题而生的。

- 每个 subagent 拥有自己的**独立上下文**。关联关系是主任务 Agent 委托给 subagent 执行，各个 subagent 间无法直接通信。
- 每个 subagent 都有自己的**单一职责**，是专门应对某个事项的定制化行为。
- 每个 subagent 能够将任务路由到更合适（更快、更实惠）的模型，能够达到**控制成本**的诉求。
- 可以通过限制 subagent 可以使用的工具，来达到**强制约束**的诉求。
- 用户级 subagents，达到**跨项目重用配置**的诉求。

![image-20260303174246733](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303174246733.png)

Claude Code 自带三个常用的**内置 Subagent**

| 类型                | 模型   | 定位                     |
| ------------------- | ------ | ------------------------ |
| **Explore**         | Haiku  | 快速只读探索代码库       |
| **Plan**            | Sonnet | 研究代码库，准备实施计划 |
| **General-purpose** | Sonnet | 处理复杂的多步任务       |



#### **创建一个自己的subagent**

https://code.claude.com/docs/zh-CN/sub-agents

| 位置       | 路径                | 作用域                   |
| ---------- | ------------------- | ------------------------ |
| **用户级** | `~/.claude/agents/` | 全局，跨所有项目         |
| **项目级** | `.claude/agents/`   | 仅当前项目（优先级更高） |

填写 agent 的功能描述，选择可用的工具(包括 MCP 工具)，可使用的模型等：

![image-20260303152259200](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303152259200.png)



#### 资源

**wshobson/agents** - 优质 Subagent 模板库，包含代码审查、测试运行等常用代理：

**Claude Plugin Hub** - 插件市场，可以找到各种 agent 和插件：



#### Coding案例

#### code-reviewer agent

当主任务完成后，可以主动调用 code-reviewer agent 进行代码审查任务。

在调用过程中，主任务会将新编写/新修改的代码传递给 agent 去执行任务。其上下文是独立获取的。

```
---
name: code-reviewer
description: Reviews code for security and style issues. Use PROACTIVELY after code changes.
tools: Read, Grep, Glob
model: sonnet
---

You are a senior code reviewer focusing on:
- Security vulnerability detection
- Code style and best practices
- Performance optimization

## Review Output
1. **Critical Issues**: Security, correctness
2. **Warnings**: Style, performance
3. **Suggestions**: Refactoring opportunities
```



#### debug agent

```
---
name: debugger
description: 专门处理错误、测试失败和异常行为的调试专家。遇到任何问题时主动使用。
tools: Read, Edit, Bash, Grep, Glob
---

你是一名专注于根本原因分析的资深调试专家。

调用时执行：
1. 捕获错误信息和堆栈跟踪
2. 明确问题复现步骤
3. 定位故障位置
4. 实施最简修复方案
5. 验证解决方案有效

调试流程：
- 分析错误信息和日志
- 检查近期代码变更
- 提出并测试假设
- 添加针对性调试日志
- 检查变量状态

针对每个问题，请提供：
- 根本原因说明
- 支持诊断的证据
- 具体代码修复方案
- 测试方法
- 预防建议

重点修复根本问题，而非表面症状。
```



## 6. Plugin(插件)

#### 定义

**插件（Plugin）** 是用于扩展 Claude Code 功能的可共享组件，包含 skills、Subagents、hooks 和 MCP servers。

> 来源：https://docs.anthropic.com/en/docs/claude-code/plugins
>
> 插件参考：https://docs.anthropic.com/en/docs/claude-code/plugins-reference

官方描述：

> "Plugins let you extend Claude Code with custom functionality that can be shared across projects and teams."

---

---

#### 插件结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # 插件清单
├── skills/                # Agent Skills
│   └── code-review/
│       └── SKILL.md
├── agents/               # 自定义 Agents
├── hooks/                # 事件处理
├── commands/             # Slash 命令
├── .mcp.json            # MCP 服务器配置
├── .lsp.json            # LSP 服务器配置
├── settings.json         # 默认设置
└── README.md            # 文档
```

| 目录              | 位置        | 用途                          |
| ----------------- | ----------- | ----------------------------- |
| `.claude-plugin/` | Plugin root | 包含 `plugin.json` 清单       |
| `commands/`       | Plugin root | Markdown 文件形式的 Skills    |
| `agents/`         | Plugin root | 自定义 agent 定义             |
| `skills/`         | Plugin root | 带 `SKILL.md` 的 Agent Skills |
| `hooks/`          | Plugin root | `hooks.json` 事件处理         |
| `.mcp.json`       | Plugin root | MCP 服务器配置                |
| `.lsp.json`       | Plugin root | 代码智能 LSP 服务器配置       |
| `settings.json`   | Plugin root | 插件启用时的默认设置          |

---

#### 插件与 Skill 、MCP、Sub-agent的关系

```
┌─────────────────────────────────────────────────────────────┐
│                        插件 Plugin                          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  plugin.json (清单)                                  │  │
│  │  - name: my-plugin                                  │  │
│  │  - version: 1.0.0                                   │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Skills (Agent Skills)                               │  │
│  │  skills/hello/SKILL.md     → /my-plugin:hello      │  │
│  │  skills/code-review/SKILL.md → /my-plugin:code-review│  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Agents (自定义 Subagents)                           │  │
│  │  agents/security-agent/                            │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  MCP Servers                                        │  │
│  │  .mcp.json                                         │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---



## 7. Multi-agents / Agent teams

| **维度**     | **Claude Code: Agent Teams**                                 | **OpenAI Codex: Multi-agents**                               |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **协作模型** | **强耦合协作**：Team Lead 负责拆解、分配任务，成员实时通信、辩论。 | **弱耦合并行**：基于任务委派，各 Agent 在独立沙箱运行，最后汇总。 |
| **通信机制** | **实时信箱 (Inbox)**：Agent 之间可以直接 `sendMessage` 或广播。 | **基于 PR/任务流**：通常通过代码合并请求或状态更新进行异步沟通。 |
| **运行环境** | **本地优先**：直接在你的终端运行，通过 `tmux` 展现多窗口协作。 | **云端优先**：在独立的云端隔离环境中运行，处理重型并行任务。 |
| **任务管理** | **共享任务列表**：使用 JSON 维护全局 TODO，状态实时同步。    | **流水线模式**：更接近传统的 CI/CD 自动化任务流。            |
| **适用场景** | 深度 Debug、跨层重构（前端+后端+测试同时调整）。             | 大规模样板代码生成、自动化漏洞修复、云端异步长任务。         |

![image-20260303174306904](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303174306904.png)

Agent teams 目前是一个实验性功能，在 Claude Code 中还是缺省禁用。

如果要使用，要在 `~/.claude/settings.json` 或在环境变量中增加 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 的配置。

与 subagent 还是有根本性的差异的，最核心的差异点：subagent 只能在单一会话内进行，且各个 subagent 只能和主 Agent 沟通，无法与其他 subagent 直接沟通。



现阶段 agent teams 支持两种 UI 显示模式：

- In-process：所有队友都在主终端内运行，使用 Shift+Up/Down 选择队友并操作。
- Split panes：每个队友都有在自己的终端界面，需要配合 tmux 或 iTerm2 使用。



#### Coding案例

```
我正在做一个Uniswap V3 Market Making Bot，optimized for stablecoin trading pairs with price maintained near 1.0.
组建一个团队从不同角度探讨这个问题：一名成员负责用户体验（UX），一名负责技术架构，还有一名担任质疑者角色。
```

![企业微信截图_17725321355204](C:\Users\顾宇翔\Desktop\企业微信截图_17725321355204.png)



## 附录1：测试

目前市场上的 AI 测试 Agent 主要分为“集成开发辅助型”和“端到端自主测试型”两类。

#### 1. 自主端到端测试

这类 Agent 能够像人类测试员一样理解需求，自主编写并执行测试。

- **mabl Agentic Tester**: [AI-Powered Testing for the Next Generation of Software | mabl](https://www.mabl.com/)
- 提供端到端的自主性。AI可以根据需求或自然语言生成 E2E 测试。它能阅读用户故事（User Stories），自动生成测试大纲、结构化流程，并直接转化为稳定的测试脚本。
- 执行 Web 测试、执行 Mobile 测试，像真实用户一样操作系统，点击 UI、调用 API、执行业务流程
- 感知页面状态、DOM变化、接口返回，记录应用行为，判断当前状态是否符合预期，分析异常情况

#### 2. 开发侧测试(IDE-Native)

这类 Agent  嵌入在开发流程中，缩短了修复周期。

- **web3-testing Skill**: 专注于智能合约测试的skills,掌握使用 Hardhat、Foundry 和高级测试模式对智能合约进行全面测试和问题修复。
- **solidity-security Skill**: 专注于Solidity智能合约代码审计的 skills，能够自动发现潜在的逻辑漏洞，输出报告，修复漏洞。



#### web3-testing Skill

https://skills.sh/wshobson/agents/web3-testing

- 为智能合约编写单元测试
- 搭建集成测试套件
- 进行Gas优化测试
- 针对边缘情况的模糊测试
- 自动化测试覆盖率报告
- 在 Etherscan 上验证合约

![image-20260305160706244](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260305160706244.png)

![image-20260305160805579](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260305160805579.png)



#### solidity-security Skill

https://skills.sh/wshobson/agents/solidity-security

- 编写安全的智能合约
- 审计现有合约是否存在漏洞
- 在确保安全的前提下优化Gas使用
- 准备专业审计合同
- 了解常见的攻击途径

**Resources**

- **references/reentrancy.md**：全面的再入预防
- **references/access-control.md**：基于角色的访问模式
- **references/overflow-underflow.md**：SafeMath 和整数安全性
- **references/gas-optimization.md**：Gas优化技术
- **references/vulnerability-patterns.md**：通用漏洞目录
- **assets/solidity-contracts-templates.sol**：安全合约模板
- **assets/security-checklist.md**：预审计清单
- **scripts/analyze-contract.sh**：静态分析工具

**Tools**

- **Slither**：静态分析工具
- **Mythril**：安全分析工具
- **Echidna**：模糊测试工具
- **Manticore**：符号化执行
- **Securify**：自动化安全扫描器

实际过程Agent会先用自己的知识人工审计一遍代码，然后在用Tools跑一下安全工具的扫描

审计报告： [SECURITY_AUDIT_REPORT_2026-03-05_ZH.pdf](..\..\SECURITY_AUDIT_REPORT_2026-03-05_ZH.pdf) 

![image-20260305134720970](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260305134720970.png)

![image-20260305115317559](C:\Users\顾宇翔\Desktop\image-20260305115317559.png)





## 附录2：Cobo WaaS对接案例

数字资产托管及钱包解决方案提供商 Cobo 宣布推出 Cobo WaaS Skill，

作为开发者，可以使用 Cobo WaaS Skill 与 Claude Code、Cursor 或其他熟悉的 AI 编码助手，通过自然语言与[Cobo WaaS 2.0 API](https://www.cobo.com/developers/v2/guides/overview/introduction)进行交互，或者为您的应用程序生成 SDK 调用代码、调试与故障排查，可实现在 30 分钟内构建可用的 Web 钱包应用。

### 主要功能

Cobo WaaS Skill 是一系列命令、参考文档、代码示例和最佳实践的集合，可以实现：

- **智能 SDK 代码生成** - 用自然语言描述业务需求，自动生成生产级 Python、Node.js、Go 或 Java SDK 代码
- **调试与故障排查** - 基于日志和错误信息，快速定位 API 调用问题，自动生成详细的调试日志和错误分析
- **完整的 CLI 工具集** - 执行所有 WaaS API 操作（钱包、交易、地址管理等），以及进行 API Key 管理、Webhook 测试、GraphQL 查询



**Vibe Coding：**

- **“生成 Python 代码来创建钱包并轮询交易完成状态”**
- **“编写 Node.js 代码用于处理交易事件的 webhook 处理程序”**
- **“为交易所应用生成 Python 代码，为每个用户创建充值地址”**



## 附录3：找Skills

#### 技能市场

[skills.sh](https://skills.sh/)，这是一个技能包目录和排行榜

#### find-skills

```
# find-skills 提供技能可以帮助发现和安装来生态系统的技能。
npx skills add https://github.com/vercel-labs/skills --skill find-skills
```

问询示例：

- “找到xxx的技能”或“是否存在xxx的技能”

- 想要搜索工具、模板或工作流程



[`skills`](https://skills.sh/)一个用于安装和管理代理技能包的命令行界面

使用以下命令安装技能包（是给agent用的）

```
npx skills add <package>
```

