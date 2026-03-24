# AI Agent

## OpenClaw案例

在做ripple 以IOU形式 发行 SC+ 的 POC中，是让OpenClaw完成xrpl上的issuer的domain设置、trustline双向授权，给指定账户发行iou等上链操作

![image-20260305155514042](C:\Users\顾宇翔\Desktop\image-20260305155514042.png)





## Anthropic Cowork

北京时间2月24日晚间，Anthropic在旧金山和纽约同步举办了一场面向企业客户的发布会，核心信息是：2025年Claude Code改变了开发者的工作方式，2026年Cowork要对所有知识工作者做同样的事。

---

### Cowork vs Claude Code

|              | Claude Code | Cowork                                   |
| ------------ | ----------- | ---------------------------------------- |
| **目标用户** | 开发者      | 所有知识工作者（财务、法务、HR、设计等） |
| **场景**     | 编程        | 企业知识工作                             |
| **定位**     | AI 编程助手 | 企业 Agent 平台                          |
| **插件**     | 开发者工具  | 职能场景插件                             |

**让每个部门团队拥有“各取所需”的企业Agent**

发布会里用一个虚构的金融公司Silver & Capital串起了完整场景：财务团队用Cowork分析竞争数据、生成Excel报表；Claude在PowerPoint里转化为可汇报的Deck；法务通过Thomson Reuters的Counsel Agent审查合同；销售基于所有输出快速制定对策。从分析到决策到执行，**Claude作为底层智能层串联了整条链路。**



### Cowork 核心功能

#### 预制插件（十余个）

| 职能     | 插件功能                                     |
| -------- | -------------------------------------------- |
| **HR**   | 录用通知、入职计划、绩效评估、薪酬分析       |
| **设计** | 评审框架、UX文案、无障碍审计                 |
| **工程** | 站会摘要、事件响应、上线检查清单             |
| **金融** | 财务分析、投资银行、股票研究、私募、财富管理 |
| **法务** | 合同审查（与 Thomson Reuters 合作）          |

#### MCP 举例

- **Google Workspace**：日历、邮箱、云盘
- **DocuSign**：电子签名
- **FactSet**：金融数据
- **Harvey**：法律 AI
- **Apollo、Clay、Outreach**：销售赋能
- **MSCI**：投资分析

#### 跨应用协同

- **Excel → PowerPoint**：数据分析结果直接生成演示文稿
- **上下文不断**：跨应用操作无需从头来过

#### 私有插件市场

- **管理员搭建**：仅对内部开放的市场
- **权限管控**：哪些插件对哪些人可用
- **自动安装**：新员工入职自动安装预设插件
- **可移植**：插件可在 Cowork 和 Claude Agent SDK 上通用

```
┌─────────────────────────────────────────────────────────┐
│                  企业私有插件市场                          │
├─────────────────────────────────────────────────────────┤
│  市场部插件         │  法务部插件      │  HR插件            │
│  ───────────       │ ───────────    │  ──────────       │
│  • 社交媒体         │  • 合同审查     │  • 入职管理         │
│  • 数据分析         │  • 合规检查     │  • 绩效评估         │
│  • 内容生成         │  • 尽职调查     │  • 薪酬分析         │
└─────────────────────────────────────────────────────────┘
```

### Cowork Plugin (插件) 的核心构成

一个 Cowork Plugin 本质上是一组 Markdown 和 JSON 文件，它定义了 AI 在特定工作场景下的表现。它由四个要素组成：

- **Skills (技能)：** 插件的“大脑”。里面写满了该岗位的专业知识和标准作业程序（SOP）。例如，一个“法律插件”会包含如何审查合同条款的步骤。
- **Connectors (连接器)：** 利用 MCP 协议 将 Claude 连通到外部工具（如 Google Drive, Salesforce, Slack, GitHub）。
- **Commands (命令)：** 定义特定的“斜杠命令”（如 /invoice:generate），让用户能一键触发复杂的自动化流。
- **Sub-agents (子代理)：** 插件可以调用更小的专门 Agent 来处理特定子任务。

### 企业级特性

| 特性         | 说明                       |
| ------------ | -------------------------- |
| **数据隔离** | 用户数据绝不出现在 AI 输出 |
| **审计追踪** | OpenTelemetry 支持         |
| **权限管控** | 精细到插件级别             |
| **合规要求** | 满足企业标准               |

---



## 附录1：Coinbase AI 套件

### 1. Agentic Wallet Skills

Coinbase 推出首个专为 AI 代理设计的钱包基础设施Skills，让 AI 拥有自己的钱包、资金、身份、交易与支付能力。AI 可以被授权在“可控边界内”自动执行。

https://docs.cdp.coinbase.com/agentic-wallet/welcome

![image-20260303141024601](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260303141024601.png)

| Skill               | Description                                                |
| ------------------- | ---------------------------------------------------------- |
| authenticate-wallet | 通过邮箱一次性验证码（OTP）登录钱包                        |
| fund                | 通过 Coinbase Onramp 向钱包充值                            |
| send-usdc           | 向以太坊地址或 ENS 名称发送 USDC                           |
| trade               | 在 Base 链上进行代币交易                                   |
| search-for-service  | 在 x402 市场中搜索付费 API 服务                            |
| pay-for-service     | 通过 x402 发起付费 API 请求                                |
| monetize-service    | 构建并部署一个可通过 x402 被其他 Agent 使用的付费 API 服务 |

**把“签名权”程序化**

- 私钥隔离与执行安全：Agentic Wallet 的私钥存储在 TEE（可信执行环境）中。

- 支持 Session 级别授权
- 支持交易金额上限
- 支持行为范围限制

- 通过 Paymaster 代付 Gas
- 支持持续自动化执行
- x402 集成：机器间支付协议。代理商可通过 x402 同时消费和提供付费 API，实现代理商间商业交易。



### 3. x402 MCP、Payments MCP

当 Claude（或其他代理）调用某个工具时，MCP 服务器将执行以下操作：

1. 检测 API 是否需要付费（通过带有`PAYMENT-REQUIRED`标头的 HTTP 402 响应）
2. 通过已注册的 x402 方案，使用您的钱包自动处理付款
3. 将付费数据返还给客户（例如，Claude）。

这样，您（或您的代理）就可以以编程方式访问付费 API，而无需手动支付步骤。

**官网：**https://docs.cdp.coinbase.com/x402/mcp-server

**官网：**https://docs.cdp.coinbase.com/payments-mcp/welcome



### 2. AgentKit

使 AI Agent 能够与区块链交互的工具包，这是一个代码库和框架。如果你在写一个 AI 代理（比如基于 LangChain 或 Eliza），你会把 AgentKit 导入到你的代码中，让你的 AI 具备“读写区块链”的能力。它更偏底层，灵活性极高。

**官网：**https://docs.cdp.coinbase.com/agent-kit/welcome

**GitHub**：https://github.com/coinbase/agentkit

**核心功能**：

| 功能         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| 安全钱包管理 | 为 Agent 创建和管理多种加密钱包，可扩展设计，支持添加钱包提供商 |
| 链上操作     | 转账、Swap、部署智能合约等功能，可扩展设计，支持添加自定义操作 |
| 多链支持     | EVM兼容网络 + Solana                                         |
| 框架兼容     | LangChain、Eliza、Vercel AI SDK 等                           |



### AgentKit vs Agentic Wallet 

|              | AgentKit                                 | Agentic Wallet                                      |
| ------------ | ---------------------------------------- | --------------------------------------------------- |
| **类型**     | 嵌入代码 (SDK)                           | 通过命令行 (CLI) 或 MCP 协议调用                    |
| **集成方式** | 需要开发者手动集成导入代码               | 插拔式 (Plug-and-play)，几分钟内即可启用            |
| **功能范围** | 全链路能力、安全逻辑通常需要开发者自己写 | 钱包操作：send/trade/x402、内置了**可编程安全护栏** |
| **网络**     | 多网络 (EVM + Solana)                    | 目前高度优化于 **Base** 网络                        |


---

## 附录2：Stripe AI Payments 开发套件

> 来源：https://docs.stripe.com/building-with-llms

### 1. Stripe Agent Toolkit SDK

**描述**：为 Agent 添加 Stripe 支付功能的 SDK

**官方文档**：https://docs.stripe.com/agents

**支持框架**：

- OpenAI Agents SDK
- Vercel AI SDK
- LangChain
- CrewAI

**核心功能**：

- 创建 Stripe 对象（Payment Links、Products 等）
- 支付收款
- 客服工作流集成

---

### 2. Stripe MCP Server

**描述**：Stripe Model Context Protocol 服务器

**功能**：

- AI Agent 可以直接与 Stripe API 交互
- 搜索 Stripe 知识库（文档、支持文章）
- 在 Stripe 账户中完成任务

**连接方式**：

```bash
# Claude Code 安装
claude /plugin install stripe@claude-plugins-official

# Claude (桌面版)
# 访问 https://claude.com/connectors/stripe

# Cursor
# 安装 Cursor Stripe 插件
```



### 支付场景案例

#### 案例1：自动创建 Payment Link

```
用户："帮我创建一个 $50 的支付链接"

Agent → Stripe Agent Toolkit → 创建 Product + Price + Payment Link
                                    ↓
返回："https://buy.stripe.com/..."
```

#### 案例2：客服退款处理

```
用户："订单 #12345 需要退款"

Agent → Stripe API → 查询订单 → 发起退款
                          ↓
返回："退款已处理，$100 已退回原支付方式"
```

#### 案例3：Agent 收费

```
Agent 提供付费 API → 用户调用
                          ↓
Stripe → 自动扣费 → Agent 获得收入
```



### 相关资源

- **文档**：https://docs.stripe.com/building-with-llms
- **Agent SDK**：https://docs.stripe.com/agents
- **GitHub**：https://github.com/stripe/ai



# 附录4：MoonPay CLI for AI agents

MoonPay Agents 是一个**非托管软件层**，它为 AI 代理提供**54 种加密货币专用工具，**涵盖**17 项关键技能**。

该软件于 2026 年 2 月 24 日发布，允许 AI 代理通过 MoonPay 的命令行界面 (CLI) 、本地 MCP 服务器 ( mp mcp ) 或 REST API 多种方式代表您**管理钱包、执行交易并自主进行交易**。

**出入金通道**：

- **存款：**支持多链存款链接，并可自动转换为稳定币。
- **法币存取通道：**使用法币（美元）兑换加密货币
- **虚拟账户**: 基于 KYC 的法币入账通道
- **创建存款链接：**生成多链存款链接，用于接收加密货币支付并自动转换为稳定币。

**综合交易**

- **交易：**掉期交易、桥接交易、转账交易、定投交易、限价单、止损单
- **定期购买**：您可以设置自动定期采购，以便您的代理商始终拥有所需的资金。
- **x402 兼容性**：无需人工干预即可实现机器对机器支付
- **市场情报**——代币分析、价格数据和价格提醒

**特性**

- **非托管钱包**：采用操作系统密钥链加密的本地硬盘钱包；密钥永远不会离开机器。

- **多种访问方式**：命令行界面 (CLI)、本地 MCP 服务器、REST API 或 Web 聊天

- **支持多链**：可在 Solana、以太坊、Base、Polygon、Arbitrum、Optimism、BNB、Avalanche、TRON 和比特币上运行。

  

MoonPay Agents包含**17种技能**和**54种工具**。

该列表将持续更新，您在使用MoonPay Agents构建功能时也可以添加自己的技能

https://support.moonpay.com/en/articles/586487-moonpay-agents-fund-your-ai



OpenClaw + MoonPay CLI 设置指南：https://support.moonpay.com/en/articles/592627-openclaw-moonpay-cli-setup-guide

官网：https://www.moonpay.com/agents

https://x.com/moonpay/status/2026371974037983504

https://support.moonpay.com/en/articles/586583-moonpay-cli-for-ai-agents



# 附录5：Securitize MCP 

RWA平台 Securitize 推出的 Securitize MCP 服务器，这是一个集成层，旨在让其 RWA 数据集更容易被大型企业系统和人工智能工具访问。

```
https://mcp.securitize.io/mcp 
```

MCP 服务器作为 Securitize 的 RWA 数据集与各种应用程序之间的安全网关，将数据请求转换为标准化调用，使金融平台、机构投资者和 AI 助手能够通过简单的函数调用获取资产供应量、分发情况和代币元数据等实时信息。

**询问示例**

- Securitize平台上有哪些可用资产？
- 当前有多少资产上架？
- 显示ACRED的资产详情。
- 请提供ACRED的代币信息（地址、链ID）。
- 哪些代币支持跨链桥接？

![img](https://cdn.builder.io/api/v1/image/assets%2Fd39b51a544e84e2fbb2445f58c6c6f2c%2F77ee8cade43940a5b3c592b32382f109?width=900)



# 附录6：Phantom MCP

加密钱包 Phantom 宣布推出 MCP 服务器，AI 代理可以查看钱包地址、签署交易、转移代币以及在 Solana、以太坊、比特币和 Sui 等加密货币之间签署消息。

#### 特征

- **单点登录认证**：与 Phantom 的嵌入式钱包单点登录流程（Google/Apple 登录）无缝集成
- **会话持久性**：使用本地存储的时间戳密钥进行自动会话管理。
- **多链支持**：可与 Solana、以太坊、比特币和 Sui 网络配合使用
- **五种MCP工具**
  - `get_wallet_addresses`— 获取已认证嵌入式钱包的区块链地址
  - `sign_transaction`— 在支持的链上签署交易
  - `transfer_tokens`— 在 Solana 上转移 SOL 或 SPL 代币
  - `buy_token`— 从 Phantom 报价 API 获取 Solana 互换报价
  - `sign_message`— 使用自动链特定路由对 UTF-8 消息进行签名



## 附录7：Solana AI 开发套件

#### 1. Solana Agent Kit (SendAI)

可以将任何 AI  Agent连接到 30 多个 Solana 协议并执行 50 多个 Solana 操作：

- 交易代币
- 资产转账
- 余额查询
- 部署 SPL Token
- 创建 NFT 系列（Collection）
- 上架市场，挂单出售你的 NFT
- NFT元数据管理（Metadata Management）
- 通过 Jupiter Exchange 进行兑换（Swap）
- 发送空投
- 通过 Pyth 价格预言机获取资产价格
- 通过 PumpPortal 在 Pump 上发射项目
- 创建 Raydium 资金池（CPMM、CLMM、AMMv4）
- 创建 OpenBook 市场、下限价单
- 通过 Wormhole 进行跨链桥接代币
- ……

**官网：**https://kit.sendai.fun/

**Github:**https://github.com/sendaifun/solana-agent-kit

**官网：**https://solana.com/developers/ai