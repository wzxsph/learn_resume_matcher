# Resume Matcher 中文学习教程

> **仓库声明 / Repository Notice**
>
> - 本仓库是 [srbhr/Resume-Matcher](https://github.com/srbhr/Resume-Matcher) 的**个人中文学习笔记**，
>   **非官方文档**，目前仅作作者本人学习使用。
> - 上游项目基于 **Apache License 2.0** 发布；本仓库教程散文以 [MIT License](LICENSE) 发布。
> - 教程中引用的代码片段版权归上游项目作者所有，遵循 Apache 2.0 协议。
> - 如需转载或二次发布，请保留本声明与上游项目署名。

> 本目录是 Resume Matcher 项目的系统化中文学习教程，从环境搭建到代码逐模块讲解，帮助你吃透这个 AI 简历定制工具。

## 项目一句话

Resume Matcher 是一个 **AI 驱动的简历定制平台**：上传主简历 → 粘贴职位描述 (JD) → AI 给出改进建议 → 调整后导出 PDF。本地可跑 Ollama 免费使用，云端可连 OpenAI / Anthropic / Gemini 等多家 LLM。

## 阅读路径建议

| 阶段 | 教程 | 适合谁 |
|------|------|--------|
| 上手 | [01-项目总览](01-项目总览.md) → [02-环境搭建](02-环境搭建.md) | 所有人 |
| 后端学习 | [03-后端架构](03-后端架构.md) → [04-后端核心模块](04-后端核心模块.md) → [05-AI集成与提示词](05-AI集成与提示词.md) | 后端 / AI 工程师 |
| 前端学习 | [06-前端架构](06-前端架构.md) → [07-前端数据层与Hooks](07-前端数据层与Hooks.md) → [08-前端核心组件](08-前端核心组件.md) | 前端工程师 |
| 进阶特性 | [09-前端特性与流程](09-前端特性与流程.md) → [10-部署与扩展](10-部署与扩展.md) | 准备部署 / 二次开发 |

## 教程索引

### 一、基础篇
- [01-项目总览](01-项目总览.md) — 项目背景、核心能力、技术栈一览
- [02-环境搭建](02-环境搭建.md) — 本地运行、AI 提供商配置、Docker 启动

### 二、后端篇
- [03-后端架构](03-后端架构.md) — FastAPI 入口、路由组织、模块划分
- [04-后端核心模块](04-后端核心模块.md) — Routers / Services / Schemas 详解
- [05-AI集成与提示词](05-AI集成与提示词.md) — LiteLLM 封装、提示词模板、改进工作流

### 三、前端篇
- [06-前端架构](06-前端架构.md) — Next.js 16 + React 19 项目结构
- [07-前端数据层与Hooks](07-前端数据层与Hooks.md) — API 客户端、Context、Hooks
- [08-前端核心组件](08-前端核心组件.md) — Builder / Tailor / Enrichment 等核心组件

### 四、特性与部署
- [09-前端特性与流程](09-前端特性与流程.md) — i18n、模板、富集、JD 匹配、设计系统
- [10-部署与扩展](10-部署与扩展.md) — Docker 部署、扩展点、二次开发指南

## 教程约定

- 所有代码引用使用 `文件路径:行号` 格式，例如 `apps/backend/app/main.py:42`
- 中文为主，技术术语保留英文（如 `router`、`schema`、`context`）
- 代码块标注语言（`python`、`typescript`、`bash` 等）
- 教程内容基于项目代码事实，不包含未实现的「画饼」功能

## 原始项目资源

- 项目主页: [README.md](https://github.com/srbhr/Resume-Matcher/blob/main/README.md)（多语言版本见 [README.zh-CN.md](https://github.com/srbhr/Resume-Matcher/blob/main/README.zh-CN.md)）
- 安装指南: [SETUP.md](https://github.com/srbhr/Resume-Matcher/blob/main/SETUP.md) / [SETUP.zh-CN.md](https://github.com/srbhr/Resume-Matcher/blob/main/SETUP.zh-CN.md)
- Agent 文档: [docs/agent/README.md](https://github.com/srbhr/Resume-Matcher/blob/main/docs/agent/README.md)
- 设计系统: [docs/portable/swiss-design-system/README.md](https://github.com/srbhr/Resume-Matcher/blob/main/docs/portable/swiss-design-system/README.md)

## 贡献

本教程系列只新增内容到 `learn_resume_matcher/`，**不修改**项目任何源代码。如果发现错误或想补充内容，请直接在对应 `.md` 中编辑。

---

> 开始阅读 → [01-项目总览](01-项目总览.md)
