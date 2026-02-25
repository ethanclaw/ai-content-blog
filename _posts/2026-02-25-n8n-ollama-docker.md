---
layout: post
title: "n8n + Ollama 本地 AI 自动化实战：Docker 部署完全指南"
date: 2026-02-25
categories: [n8n, Ollama, Docker, AI]
---

![n8n + Ollama 架构图](https://res.cloudinary.com/dxqklbcjw/image/upload/v1772002564/wzsggwx1tixdfqud0bpn.png)

> 当开源自动化平台遇上本地大语言模型，一场关于效率、隐私与成本的革命正在你的服务器上悄然发生。无需 API 密钥，无需网络延迟，今天我们就来搭建完全自主可控的智能自动化系统。

## 开场故事

凌晨3点15分，我的手机突然震动个不停。睡眼惺忪地解锁屏幕，映入眼帘的是云服务商发来的紧急告警：「API 调用费用已超出本月预算300%」。我瞬间清醒——这个月才过去10天！

更让我脊背发凉的是，就在昨天，我们的客户数据团队报告了一个诡异现象：某些客户的对话记录中，出现了从未有过的「幻觉回答」。经过排查，我们发现第三方AI服务的API在高峰时段偶尔会返回完全无关的内容，而且——最糟糕的是——这些请求日志显示我们的数据被路由到了一个从未见过的服务器IP。

这不是我第一次为「云AI服务」买单了。每次业务量增长，伴随而来的不是喜悦，而是对账单的恐惧。数据隐私、不可控的延迟、随时可能变动的定价策略……这些问题像达摩克利斯之剑悬在头顶。

就在那个不眠之夜，我做出了决定：是时候把AI自动化搬回家了。今天，我要分享的就是如何用 n8n + Ollama + Docker，构建一个完全自托管的AI自动化平台。这不是简单的工具堆砌，而是一次彻底的技术自主权夺回行动。

## 一、为什么要自托管 AI 自动化？

在深入技术细节之前，我们先理性分析一下：为什么要放弃便捷的云服务，选择相对复杂的自托管方案？让我们用数据说话：

| 维度 | 云AI服务 | 自托管方案 | 差异分析 |
|------|----------|------------|----------|
| **延迟** | 50-500ms | 10-100ms | 本地部署消除网络往返延迟 |
| **隐私** | 数据离开本地网络 | 数据永不离开服务器 | 敏感数据完全可控 |
| **成本** | $0.002-$12/千token | 一次性硬件投入 | 月调用量超100万token后更经济 |
| **灵活性** | 受限于提供商 | 任意开源模型 | 可根据任务选择最合适模型 |

**关键洞察**：如果你每月在AI API上的花费超过$100，或者处理敏感数据，或者需要低延迟响应，那么自托管方案在6-12个月内就能收回硬件投资成本。

## 二、技术架构全景

上图展示了完整的技术架构：
- **n8n** 作为自动化大脑，负责工作流编排、触发器和数据处理
- **Ollama** 作为AI推理引擎，专门负责大语言模型的加载和推理
- **Docker** 提供环境隔离和便捷部署
- **GPU** 可选加速推理

## 三、部署环境准备

### 3.1 服务器选择

| 场景 | 推荐配置 | 可运行模型 |
|------|----------|------------|
| 个人开发/测试 | 32GB RAM + 16GB VRAM | 7B（全精度）, 13B（4-bit） |
| 小型团队 | 64GB RAM + 24GB VRAM | 13B（全精度）, 34B（4-bit） |
| 企业级 | 128GB+ RAM + 多GPU | 70B+ 模型 |

### 3.2 Docker & Docker Compose 安装

```bash
# Ubuntu 22.04 LTS
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 四、Docker 部署 Ollama

### 4.1 快速部署

```bash
docker pull ollama/ollama:latest
docker run -d \
  --name ollama \
  --restart always \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:latest

docker exec -it ollama ollama pull llama3.1:8b
```

### 4.2 docker-compose（推荐）

```yaml
version: '3.8'
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
volumes:
  ollama_data:
```

### 4.3 常用模型

| 模型 | 大小 | 显存 | 特点 |
|------|------|------|------|
| Llama 3.1 8B | 4.7GB | 8GB | 平衡性能 |
| Qwen 2.5 7B | 4.4GB | 8GB | 中文能力强 |
| Mistral 7B | 4.1GB | 8GB | 推理速度快 |

## 五、Docker 部署 n8n

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_PROTOCOL=http
      - N8N_HOST=localhost
    volumes:
      - n8n_data:/home/node/.n8n
    extra_hosts:
      - "host.docker.internal:host-gateway"
volumes:
  n8n_data:
```

**关键配置**：`extra_hosts` 让 n8n 可以访问宿主机上的 Ollama。

```bash
docker-compose up -d
# 访问 http://localhost:5678
```

## 六、第一个 AI 工作流

### 工作流设计：日志自动摘要

1. 添加 **Ollama 节点**
2. 配置 Credentials：`http://host.docker.internal:11434`
3. 添加 **Schedule Trigger**
4. 添加 **Read File** → **Code** → **Ollama** → **Send Email**

### 示例 Prompt

```
请分析以下服务器日志：
1. 错误统计
2. 主要问题摘要
3. 建议的排查步骤

日志内容：{{ $json.logs }}
```

## 七、性能优化

### GPU 加速

```bash
# 安装 NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
```

### 模型量化

```bash
# 4-bit 量化
ollama pull llama3.1:8b-q4_0
```

### 显存参考

| 显存 | 可运行模型 |
|------|-----------|
| 8GB | 7B（4-bit） |
| 16GB | 13B（4-bit） |
| 24GB+ | 70B（4-bit） |

## 八、生产环境最佳实践

### 安全配置
```bash
# 生成加密密钥
openssl rand -base64 32
```

### 监控
```bash
docker stats ollama n8n
nvidia-smi -l 1
```

## 九、常见问题

**Q1: 无法访问 GPU？**
确保安装 nvidia-container-toolkit 并重启 Docker。

**Q2: n8n 无法连接 Ollama？**
检查 extra_hosts，使用 `http://host.docker.internal:11434`。

**Q3: 如何备份？**
```bash
docker run --rm -v ollama_data:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data
```

## 十、总结

自托管 AI 不仅是一项技术选择，更是对工具自主权的回归。当你掌控了自己的 AI 基础设施，你就拥有了定义智能化未来的主动权。

**进阶方向**：
- 集成更多服务（数据库、消息队列）
- 模型微调（LoRA、QLoRA）
- 集群部署

---

*动手试试看吧！*
