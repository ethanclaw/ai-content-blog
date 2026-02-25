# Mac Mini + OpenClaw 搭建 24/7 私人 AI 助手 - 完整部署指南

## 痛点

你是否遇到这些问题？
- 消息分散在 Telegram、微信、飞书、Discord，来回切换太麻烦
- 想让 AI 帮忙处理重复任务，但每次都要重新解释背景
- 需要一个 24/7 在线、能记住上下文的 AI 助手

本文将帮助你用 Mac Mini + OpenClaw 搭建一个完全私有化的个人 AI 助手。

---

## 方案概述

OpenClaw 是一个开源的 AI 助手框架，可以：
- 多平台消息聚合（Telegram、Discord、微信、飞书等）
- 本地 LLM 接入（支持 OpenRouter、Claude、GPT 等）
- 定时任务和主动提醒
- 自动化工作流编排

为什么选择 Mac Mini？功耗 30W、待机静音、适合 24 小时运行。

---

## 环境准备

### 硬件要求
- Mac Mini（M1/M2/M4 均可）
- 不需要外接显示器（通过 SSH 访问）

### 软件要求
- Node.js 18+
- npm

```bash
# 检查 Node.js 版本
node -v
# 检查 npm 版本
npm -v
```

---

## 安装步骤

### 步骤 1：安装 OpenClaw

```bash
# 全局安装
npm install -g openclaw
```

### 步骤 2：初始化配置

```bash
# 初始化
openclaw init
```

这会在 `~/.openclaw/` 目录创建配置文件。

### 步骤 3：配置 LLM

编辑 `~/.openclaw/settings.yaml`，添加 LLM 配置：

```yaml
llm:
  provider: openrouter
  api_key: your-api-key
  model: openai/gpt-4o-mini
```

**实际测试中发现**：OpenRouter 性价比高，支持多种模型切换。

### 步骤 4：添加消息通道

```bash
# 添加 Telegram 通道
openclaw channel add telegram

# 添加 Discord 通道
openclaw channel add discord
```

配置通道需要对应的 Bot Token：
- Telegram：通过 @BotFather 创建
- Discord：Discord Developer Portal 创建

### 步骤 5：启动服务

```bash
# 启动网关
openclaw gateway start
```

---

## 核心功能配置

### 功能 1：消息聚合

在 `settings.yaml` 中配置：

```yaml
session:
  mode: unified  # 统一会话模式
```

这样所有平台的对话会进入同一个 session，AI 会记住上下文。

### 功能 2：定时任务

创建 `~/.openclaw/crontasks/` 目录，添加定时任务：

```yaml
# 每天早上 8 点推送天气
name: morning_weather
schedule: "0 8 * * *"
action: query_weather
```

### 功能 3：连接外部服务

配置 Todoist API 实现任务管理：

```yaml
integrations:
  todoist:
    api_key: your-todoist-token
```

---

## 常见问题

### Q1：Telegram 链接超时
检查是否需要代理。实际测试中发现，国内访问 Telegram 需要配置代理。

### Q2：LLM 返回很慢
检查网络延迟，或切换到国内模型（如 MiniMax）。

### Q3：如何备份配置？
直接备份 `~/.openclaw/` 目录即可。

---

## 总结

通过本文，你可以快速搭建一个 24/7 在线的私人 AI 助手。它可以帮你：
- 聚合多平台消息
- 管理日程和任务
- 定时抓取信息
- 连接各种自动化服务

关键在于：先想清楚想让 AI 帮你解决什么问题，再进行配置。

如果觉得有帮助，欢迎评论讨论你的使用场景！
