---
layout: post
title: "n8n + Ollama 本地 AI 自动化实战：Docker 部署完全指南"
date: 2026-02-25
categories: [n8n, Ollama, Docker, AI]
---

> 当开源自动化平台遇上本地大语言模型，一场关于效率、隐私与成本的革命正在你的服务器上悄然发生。无需 API 密钥，无需网络延迟，今天我们就来搭建完全自主可控的智能自动化系统。

## 一、为什么要自托管 AI 自动化？

### 零延迟，无需 API 调用往返
想象一下：一个 AI 工作流从触发到响应需要 2-3 秒的 API 往返延迟。当处理大量数据或需要实时响应的场景时，这些毫秒级的延迟会累积成显著的时间成本。本地部署意味着 AI 推理就在你的服务器内部完成，响应速度通常比 API 调用快 5-10 倍。

### 隐私安全，数据不出本地网络
对于金融、医疗、法律等敏感行业，数据安全是红线。当你的客户数据、内部文档或商业机密需要 AI 处理时，本地部署确保所有信息都在你的防火墙内流转，彻底杜绝了数据泄露到第三方的风险。

### 零订阅费，一次硬件投入终身使用
云服务 AI API 的成本随着使用量线性增长。以中等规模企业为例，月均 API 调用费用可能高达数千美元。而本地部署只需一次性硬件投入，之后的所有调用都是"免费"的——当然，电费除外。

### 完全可控，支持任意模型
你不必受限于 OpenAI 的模型更新策略或 Anthropic 的定价调整。无论是 Llama 3、Qwen 2.5 还是 Mistral，你都可以自由选择、随时切换，甚至微调专属模型。这种技术自主权在快速发展的 AI 领域尤为宝贵。

## 二、部署环境准备

### 服务器选择
推荐使用 Ubuntu 24.04 LTS 或 Debian 13，这两个发行版对 Docker 和 NVIDIA 驱动的支持最为成熟。

### Docker 安装
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
docker --version
```

### Docker Compose 安装
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### 硬件建议
- **CPU**: 4核以上
- **内存**: 16GB+
- **显存**: 8GB+（7B 模型）

## 三、Docker 部署 Ollama

```bash
# 拉取镜像
docker pull ollama/ollama:latest

# 运行容器
docker run -d \
  --name ollama \
  --restart always \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:latest

# 拉取模型
docker exec -it ollama ollama pull llama3.1:8b
```

### 常用模型推荐
- **Llama 3.1 8B** - 平衡性能与资源
- **Qwen 2.5 7B** - 优秀中文能力
- **Mistral 7B** - 高效推理

## 四、Docker 部署 n8n

创建 `docker-compose.yml`：

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
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

启动：
```bash
docker-compose up -d
```

访问 `http://localhost:5678` 即可。

## 五、实战：第一个 AI 工作流

### 步骤 1：添加 Ollama 节点
在 n8n 界面搜索 "Ollama" 并添加节点。

### 步骤 2：配置 Credentials
- **Base URL**: `http://host.docker.internal:11434`
- **Model**: `llama3.1:8b`

### 步骤 3：构建工作流
1. **Schedule Trigger** - 设置每小时触发
2. **Read File** - 读取日志文件
3. **Ollama** - AI 分析日志
4. **Send Email** - 发送摘要

## 六、性能优化技巧

### GPU 加速
```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

### 模型量化
```bash
ollama pull llama3.1:8b-q4_0  # 4-bit 量化
```

### 显存建议
- 8GB 显存：7B 模型
- 16GB 显存：13B 模型
- 24GB+ 显存：70B 模型

## 七、总结

自托管 AI 不仅是一项技术选择，更是对工具自主权的回归。n8n + Ollama + Docker 的组合让你拥有完全可控的智能自动化系统。

---

动手试试看吧！
