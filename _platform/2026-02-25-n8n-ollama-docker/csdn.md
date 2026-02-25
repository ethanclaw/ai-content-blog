# n8n + Ollama 本地 AI 自动化平台 - Docker 部署完全指南

## 痛点

你是否面临这些问题？
- 每月 AI API 调用费用高昂，超出预算
- 敏感数据不敢上传到第三方服务
- 网络延迟影响 AI 响应体验
- 想用 AI 自动化工作流，但不想被云服务绑定

本文将帮助你用 n8n + Ollama + Docker 搭建完全本地化的 AI 自动化平台。

---

## 方案概述

| 维度 | 云服务 | 本地部署 |
|------|--------|----------|
| 延迟 | 50-500ms | 10-100ms |
| 隐私 | 数据离开本地 | 数据不出服务器 |
| 成本 | 月付 $100+ | 一次性硬件 |
| 可控性 | 受限 | 完全掌控 |

**适用场景**：月 API 花费超 $100、数据敏感、需要低延迟、想完全自控。

---

## 硬件配置推荐

| 场景 | 配置 | 可运行模型 |
|------|------|------------|
| 个人开发 | 32GB RAM + 16GB VRAM | 7B, 13B(4-bit) |
| 小团队 | 64GB RAM + 24GB VRAM | 13B, 34B(4-bit) |
| 生产环境 | 128GB+ RAM + 多GPU | 70B+ |

---

## Docker 安装 Ollama

### 步骤 1：安装 Docker

```bash
# Ubuntu
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

### 步骤 2：拉取镜像

```bash
docker pull ollama/ollama:latest
```

### 步骤 3：启动容器

```bash
docker run -d \
  --name ollama \
  --restart always \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:latest
```

### 步骤 4：下载模型

```bash
# 进入容器
docker exec -it ollama bash

# 下载 Llama 3.1
ollama pull llama3.1:8b

# 或下载中文能力强的 Qwen
ollama pull qwen2.5:7b
```

**实际测试中发现**：4-bit 量化模型显存需求减半，推荐使用 `ollama pull llama3.1:8b-q4_0`。

---

## Docker 安装 n8n

### 步骤 1：创建 docker-compose.yml

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

### 步骤 2：启动服务

```bash
docker-compose up -d
```

### 步骤 3：访问 n8n

打开浏览器访问 http://localhost:5678

---

## 配置 n8n 连接 Ollama

### 步骤 1：创建 Credentials

1. n8n 界面 → Settings → Credentials
2. 新增 → Ollama
3. URL 填写：`http://host.docker.internal:11434`

### 步骤 2：创建测试工作流

1. 添加 **Ollama 节点**
2. 选择 Credentials
3. 输入 Prompt 测试

**常见错误**：如果连接失败，检查 `extra_hosts` 配置是否正确。

---

## 常用模型参考

| 模型 | 大小 | 显存 | 适用场景 |
|------|------|------|----------|
| Llama 3.1 8B | 4.7GB | 8GB | 通用 |
| Qwen 2.5 7B | 4.4GB | 8GB | 中文 |
| Mistral 7B | 4.1GB | 8GB | 推理 |

---

## GPU 加速配置

### 步骤 1：安装 NVIDIA Container Toolkit

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### 步骤 2：修改 docker-compose

```yaml
services:
  ollama:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

---

## 备份与恢复

### 备份

```bash
# 备份 Ollama 数据
docker run --rm -v ollama:/data -v $(pwd):/backup alpine \
  tar czf /backup/ollama.tar.gz /data

# 备份 n8n 数据
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n.tar.gz /data
```

### 恢复

```bash
docker run --rm -v ollama:/data -v $(pwd):/backup alpine \
  tar xzf /backup/ollama.tar.gz -C /
```

---

## 总结

通过本文，你可以快速搭建一个本地 AI 自动化平台：
- 完全私有，数据不出本地
- 成本可控，一次性投入
- 延迟更低，体验更好
- 灵活可控，无供应商绑定

如果觉得有帮助，欢迎评论讨论你的部署经验！
