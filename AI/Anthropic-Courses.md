# Anthropic 官方课程汇总

> 来源：https://anthropic.skilljar.com/
> 整理日期：2026-03-02

---

## 课程列表（共13门）

| # | 课程名称 | 简介 |
|---|----------|------|
| 1 | **Claude 101** | 了解 Claude 日常使用方法和核心功能 |
| 2 | **Claude Code in Action** | 将 Claude Code 集成到开发工作流 |
| 3 | **AI Fluency: Framework & Foundations** | 与 AI 系统有效协作的fluency技能 |
| 4 | **Building with the Claude API** | 使用 Claude API 构建应用（全面） |
| 5 | **Introduction to Model Context Protocol** | 用 Python 构建 MCP 服务器和客户端 |
| 6 | **Model Context Protocol: Advanced Topics** | MCP 高级主题（采样、通知、传输） |
| 7 | **Introduction to agent skills** | 在 Claude Code 中构建和共享 Skills |
| 8 | **Claude with Amazon Bedrock** | 通过 AWS Bedrock 使用 Claude |
| 9 | **Claude with Google Cloud's Vertex AI** | 通过 Google Vertex AI 使用 Claude |
| 10 | **AI Fluency for educators** | 教育工作者 AI fluency |
| 11 | **AI Fluency for students** | 学生 AI fluency |
| 12 | **Teaching AI Fluency** | 教授 AI fluency |
| 13 | **AI Fluency for nonprofits** | 非营利组织 AI fluency |

---

## 核心课程详细大纲

### 1. Claude 101
**目标人群**：Claude 新手

**内容**：
- Claude 基础使用
- 核心功能介绍
- 进阶学习资源

---

### 2. Building with the Claude API（最全面）
**目标人群**：后端/全栈开发者、数据工程师、架构师

**课程大纲**：
```
├── 入门
│   ├── Anthropic 概述
│   ├── Claude 模型概览
│   └── API 访问与认证
├── 多轮对话
│   ├── 消息格式化
│   ├── 上下文处理
│   └── System Prompts
├── 响应控制
│   ├── Temperature
│   ├── Streaming
│   └── Structured Output
├── Prompt 评估
│   ├── 测试数据集生成
│   ├── 模型评分
│   └── 代码评分
├── Prompt 工程技巧
│   ├── 清晰直接的指令
│   ├── 具体明确的说明
│   ├── XML 标签结构
│   └── 示例学习
├── Tool 使用
│   ├── 工具函数定义
│   ├── JSON Schema
│   ├── 多工具调用
│   ├── 批量操作
│   └── Web Search
├── RAG（检索增强生成）
│   ├── 文本分块策略
│   ├── Embeddings 向量
│   ├── BM25 搜索
│   └── 上下文检索
├── Claude 特性
│   ├── Extended Thinking
│   ├── 图像支持
│   ├── PDF 处理
│   ├── 引用生成
│   └── Prompt 缓存
├── MCP
│   ├── MCP 服务器/客户端
│   ├── 定义工具/资源/提示
│   └── Server Inspector
├── Anthropic Apps
│   ├── Claude Code
│   └── Computer Use
└── Agent 与工作流
    ├── 并行化
    ├── 链式
    └── 路由
```

---

### 3. Introduction to Model Context Protocol
**前置要求**：Python、JSON、HTTP

**核心内容**：
- MCP 架构与原理
- 三大原语：**Tools**、**Resources**、**Prompts**
- Python SDK 构建 MCP 服务器
- 使用装饰器定义工具
- Server Inspector 调试
- 资源暴露（静态/模板）
- 提示模板创建

---

### 4. Model Context Protocol: Advanced Topics
**前置要求**：Python 异步编程、JSON、SSE

**核心内容**：
- **Sampling**：服务器请求语言模型调用
- **Notifications**：实时日志与进度报告
- **Roots**：基于目录的文件访问权限
- **JSON 消息架构**：请求/响应 vs 通知
- **传输机制**：
  - Stdio 传输
  - StreamableHTTP 传输（SSE）
  - 有状态 vs 无状态
- **生产部署**：水平扩展、负载均衡

---

### 5. Introduction to agent skills
**目标**：创建可复用的 Skill

**核心内容**：
- Skills vs CLAUDE.md vs Hooks vs Subagents
- SKILL.md 元数据编写
- 触发描述的编写技巧
- 目录结构与渐进式披露
- 高级配置：
  - allowed-tools 限制工具
  - 执行脚本（不消耗上下文）
- 团队共享：
  - Git 仓库
  - Plugins
  - Enterprise managed settings
- 自定义 Subagents
- 故障排查指南

---

### 6. Claude Code in Action
**前置要求**：CLI、Git 基础

**核心内容**：
- AI 编码助手架构
- 工具使用系统
- 上下文管理技术
- 视觉输入工作流
- 自定义命令与自动化
- MCP 服务器集成
- GitHub 工作流集成
- Thinking 与 Planning 模式

---

### 7. Claude with Amazon Bedrock
**前置要求**：Python、AWS Bedrock 基础

**核心内容**：
- Bedrock API 调用
- 多轮对话
- Prompt 评估
- Tool 定义（JSON Schema）
- RAG 管道
- 高级特性配置
- Claude Code 自动化
- MCP 实现
- Agent 构建

---

## 课程特点

| 课程 | 形式 | 时长 | 证书 |
|------|------|------|------|
| 大部分课程 | 视频 + 实操 | 不等 | ✅ 完成可获证书 |

## 访问方式

1. 访问 https://anthropic.skilljar.com/
2. 注册账号（无需 Anthropic 账号）
3. 选择课程开始学习

---

*注意：完整课程内容需要登录后访问，本文整理自课程公开介绍页面*
