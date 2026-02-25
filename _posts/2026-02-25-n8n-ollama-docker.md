# n8n + Ollama 本地 AI 自动化实战：Docker 部署完全指南

## 开场故事

凌晨3点15分，我的手机突然震动个不停。睡眼惺忪地解锁屏幕，映入眼帘的是云服务商发来的紧急告警：「API 调用费用已超出本月预算300%」。我瞬间清醒——这个月才过去10天！

更让我脊背发凉的是，就在昨天，我们的客户数据团队报告了一个诡异现象：某些客户的对话记录中，出现了从未有过的「幻觉回答」。经过排查，我们发现第三方AI服务的API在高峰时段偶尔会返回完全无关的内容，而且——最糟糕的是——这些请求日志显示我们的数据被路由到了一个从未见过的服务器IP。

这不是我第一次为「云AI服务」买单了。每次业务量增长，伴随而来的不是喜悦，而是对账单的恐惧。数据隐私、不可控的延迟、随时可能变动的定价策略……这些问题像达摩克利斯之剑悬在头顶。

就在那个不眠之夜，我做出了决定：是时候把AI自动化搬回家了。今天，我要分享的就是如何用 n8n + Ollama + Docker，构建一个完全自托管的AI自动化平台。这不是简单的工具堆砌，而是一次彻底的技术自主权夺回行动。

## 一、为什么要自托管 AI 自动化？

在深入技术细节之前，我们先理性分析一下：为什么要放弃便捷的云服务，选择相对复杂的自托管方案？让我们用数据说话：

| 维度 | 云AI服务（OpenAI/Anthropic等） | 自托管方案（n8n+Ollama） | 差异分析 |
|------|--------------------------------|--------------------------|----------|
| **延迟** | 50-500ms（依赖网络质量） | 10-100ms（本地网络） | 本地部署消除了网络往返延迟，对于需要频繁调用的自动化工作流，累积优势显著 |
| **隐私** | 数据需离开本地网络 | 数据永不离开你的服务器 | 敏感数据（客户信息、内部文档）完全可控，符合GDPR等严格合规要求 |
| **成本** | $0.002-$0.12/千token<br>+ API调用费 | 一次性硬件投资<br>+ 电费（约$20-$50/月） | 月调用量超过100万token后，自托管方案通常更经济，且成本可预测 |
| **灵活性** | 受限于提供商模型列表<br>和更新节奏 | 任意开源模型<br>随时可切换/微调 | 可根据具体任务选择最合适的模型（代码生成、文档分析、多语言等） |
| **可用性** | 99.9% SLA<br>但可能突发限流 | 取决于自身运维水平<br>完全自主控制 | 自托管避免「被供应商锁定」，业务连续性掌握在自己手中 |
| **自定义** | 基本不支持模型微调 | 支持LoRA、QLoRA等<br>轻量级微调 | 可针对行业术语、公司文档进行领域适配，提升任务准确率 |

**关键洞察**：如果你每月在AI API上的花费超过$100，或者处理敏感数据，或者需要低延迟响应，那么自托管方案在6-12个月内就能收回硬件投资成本。

## 二、技术架构全景

让我们先理解这个技术栈是如何协同工作的：

```
┌─────────────────────────────────────────────────────────────┐
│                   你的本地服务器/工作站                     │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    Docker Network    ┌─────────────┐      │
│  │             │  ┌─────────────────┐ │             │      │
│  │   n8n容器   │◀─┤   HTTP API调用  ├─▶│ Ollama容器  │      │
│  │             │  └─────────────────┘ │             │      │
│  │  • 工作流引擎 │                      │  • LLM推理引擎 │      │
│  │  • Web UI    │  ┌─────────────────┐ │  • 模型管理    │      │
│  │  • API端点   │◀─┤   数据交换      ├─▶│  • GPU加速    │      │
│  └──────┬──────┘  └─────────────────┘ └──────┬──────┘      │
│         │                                     │             │
│  ┌──────▼──────┐                    ┌───────▼───────┐      │
│  │  Postgres   │                    │  模型存储卷    │      │
│  │  (持久化)   │                    │  (~10-50GB)   │      │
│  └─────────────┘                    └───────────────┘      │
└─────────────────────────────────────────────────────────────┘
                                 ▲
                                 │
                          ┌──────┴──────┐
                          │  NVIDIA GPU  │
                          │  (可选加速)  │
                          └─────────────┘
```

**架构核心思想**：
1. **n8n** 作为自动化大脑，负责工作流编排、触发器和数据处理
2. **Ollama** 作为AI推理引擎，专门负责大语言模型的加载和推理
3. **Docker** 提供环境隔离和便捷部署，确保一致性
4. **共享网络** 让两个服务可以安全地内部通信，无需暴露到公网

## 三、部署环境准备

### 3.1 服务器选择

自托管的第一道坎就是硬件选择。这里没有「一刀切」的方案，只有「最适合你场景」的选择：

**场景一：个人开发/测试**
- 推荐配置：32GB RAM + 16GB VRAM（RTX 4060 Ti）
- 成本：约 $800-$1200
- 可运行模型：7B参数模型（全精度），13B模型（4-bit量化）

**场景二：小型团队生产环境**
- 推荐配置：64GB RAM + 24GB VRAM（RTX 4090）
- 成本：约 $2000-$3000
- 可运行模型：13B模型（全精度），34B模型（4-bit量化）

**场景三：企业级部署**
- 推荐配置：128GB+ RAM + 多张A100/H100
- 成本：$10000+
- 可运行模型：70B+参数模型，支持多用户并发

**避坑指南**：
- 避免使用消费级显卡的共享内存模式（如RTX 3060 12G），实际推理性能可能只有专用VRAM的60%
- 如果预算有限，优先保证RAM容量（至少32GB），模型可以量化运行
- 考虑电源功率，高配GPU可能需要850W+电源

### 3.2 Docker & Docker Compose 安装

为什么选择Docker？因为AI模型的依赖环境复杂，不同模型可能需要不同版本的CUDA、Python库。Docker提供了完美的环境隔离。

```bash
# Ubuntu 22.04 LTS 完整安装脚本
#!/bin/bash

# 1. 卸载旧版本（如果有）
sudo apt-get remove docker docker-engine docker.io containerd runc

# 2. 安装依赖
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 3. 添加Docker官方GPG密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 4. 设置存储库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. 安装Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 6. 验证安装
sudo docker run hello-world

# 7. 配置非root用户使用（重要！）
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker

# 8. 测试非root权限
docker run hello-world

# 9. 安装独立版docker-compose（兼容性更好）
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 10. 验证docker-compose
docker-compose --version
```

**关键解释**：
- 第7步至关重要：避免每次都要用`sudo`，这会简化后续操作
- 我们同时安装了Docker Compose插件和独立版本，确保兼容性
- 生产环境建议使用特定版本而非latest，避免意外更新破坏稳定性

### 3.3 硬件配置参考表

不同模型对硬件的要求差异巨大。以下是实测数据：

| 模型名称 | 参数量 | 最小显存（FP16） | 推荐显存（FP16） | 4-bit量化显存 | 适合任务 |
|----------|--------|------------------|------------------|---------------|----------|
| Llama 3.2 | 1B | 2GB | 4GB | 1GB | 简单分类、路由 |
| Llama 3.1 | 8B | 16GB | 20GB | 5GB | 通用对话、文案生成 |
| Qwen2.5 | 7B | 14GB | 18GB | 4.5GB | 代码生成、中文任务 |
| Llama 3.1 | 70B | 140GB | 160GB | 40GB | 复杂推理、分析 |
| Mixtral 8x7B | 47B | 94GB | 100GB | 28GB | 多专家混合任务 |
| Gemma 2 | 9B | 18GB | 22GB | 6GB | 快速推理、边缘部署 |

**显存计算公式**：
```
FP16显存 ≈ 参数数量 × 2字节 × 1.2（KV缓存开销）
4-bit量化显存 ≈ 参数数量 × 0.5字节 × 1.2
```

例如：8B模型FP16需要 8×10⁹ × 2 × 1.2 ≈ 19.2GB

### 3.4 NVIDIA GPU 加速配置

如果你有NVIDIA显卡，这一步能让推理速度提升5-10倍：

```bash
# 1. 检查GPU是否被Docker识别
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# 2. 如果上面命令失败，安装NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker

# 3. 再次验证
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# 4. 验证CUDA版本（重要！）
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvcc --version
```

**为什么需要特定CUDA版本？**
- Ollama底层使用llama.cpp，需要与CUDA版本匹配
- CUDA 12.2是目前最稳定的选择，兼容性最好
- 如果遇到"CUDA out of memory"错误，通常是版本不匹配导致

## 四、Docker 部署 Ollama

### 4.1 为什么选择 Ollama？

在众多本地LLM部署方案中，Ollama脱颖而出有几个关键原因：

1. **简单性**：`ollama run llama3.1` 就能运行一个70B参数的模型
2. **模型生态**：支持GGUF格式，这是目前最流行的量化格式
3. **API兼容**：提供OpenAI兼容的API接口，n8n可以直接使用
4. **资源管理**：自动处理模型缓存、多模型切换
5. **活跃社区**：每周都有新模型和优化发布

对比其他方案：
- **text-generation-webui**：功能强大但配置复杂
- **vLLM**：适合高并发但资源占用大
- **LocalAI**：更灵活但需要更多手动配置

### 4.2 快速部署（基础命令）

先来一个最简单的部署，验证基础功能：

```bash
# 1. 拉取Ollama镜像
docker pull ollama/ollama:latest

# 2. 创建数据目录（模型会下载到这里）
mkdir -p ~/ollama_data

# 3. 运行容器（无GPU版本）
docker run -d \
  --name ollama \
  -p 11434:11434 \
  -v ~/ollama_data:/root/.ollama \
  ollama/ollama:latest

# 4. 查看日志
docker logs -f ollama

# 5. 测试API
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Hello, how are you?",
  "stream": false
}'
```

### 4.3 docker-compose 完整配置（含 GPU 支持）

单命令部署虽然简单，但生产环境需要更完整的配置：

```yaml
# docker-compose.ollama.yml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama_server
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      # 1. 模型存储卷 - 为什么需要单独挂载？
      #    - 模型文件很大（7B模型约4GB，70B模型约40GB）
      #    - 方便备份和迁移
      #    - 避免容器删除后重新下载
      - ./ollama_data:/root/.ollama
      
      # 2. 可选：自定义模型目录
      # - ./custom_models:/models
    environment:
      # 3. 环境变量配置
      - OLLAMA_HOST=0.0.0.0  # 监听所有接口
      - OLLAMA_NUM_PARALLEL=2  # 并行请求数，根据CPU核心数调整
      - OLLAMA_MAX_LOADED_MODELS=3  # 最大同时加载模型数
      
      # 4. GPU相关环境变量
      - NVIDIA_VISIBLE_DEVICES=all  # 使用所有GPU
      - CUDA_VISIBLE_DEVICES=0  # 如果多GPU，可以指定使用哪一张
    deploy:
      resources:
        reservations:
          devices:
            # 5. GPU资源预留
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - ai_network  # 自定义网络，与n8n通信
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

# 6. 自定义网络 - 为什么需要？
#    - 隔离性：避免端口冲突
#    - 安全性：内部通信不暴露到宿主机
#    - 便捷性：容器间通过服务名访问
networks:
  ai_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

启动命令：
```bash
# 启动服务
docker-compose -f docker-compose.ollama.yml up -d

# 查看GPU是否生效
docker exec ollama_server nvidia-smi

# 查看服务状态
docker-compose -f docker-compose.ollama.yml ps
```

### 4.4 拉取模型实测

Ollama支持的命令行操作：

```bash
# 进入容器执行命令
docker exec -it ollama_server ollama list

# 或者从宿主机直接调用API
# 1. 拉取模型（以llama3.1:8b为例）
curl http://localhost:11434/api/pull -d '{
  "name": "llama3.1:8b"
}'

# 2. 查看下载进度（另开终端）
docker logs -f ollama_server

# 3. 推荐的首批模型
MODELS=(
  "llama3.2:1b"        # 轻量级，快速响应
  "llama3.1:8b"        # 平衡型，通用任务
  "qwen2.5:7b"         # 中文优化，代码能力强
  "mistral:7b"         # 英文任务优秀
  "gemma2:9b"          # Google出品，推理能力强
)

for model in "${MODELS[@]}"; do
  echo "正在拉取模型: $model"
  curl -X POST http://localhost:11434/api/pull -d "{\"name\":\"$model\"}"
  echo ""
done

# 4. 查看已下载模型
curl http://localhost:11434/api/tags | python -m json.tool
```

**模型选择策略**：
- **开发测试**：从1B或3B参数模型开始，快速验证流程
- **生产使用**：根据任务类型选择专用模型（代码、文案、分析）
- **内存优化**：使用`:q4_0`后缀的量化版本，如`llama3.1:8b-q4_0`

### 4.5 API 测试示例

验证Ollama API是否正常工作：

```bash
#!/bin/bash
# test_ollama_api.sh

API_URL="http://localhost:11434"
MODEL="llama3.1:8b"

echo "测试1: 简单对话"
curl -s -X POST "$API_URL/api/generate" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"$MODEL\",
    \"prompt\": \"请用中文自我介绍\",
    \"stream\": false,
    \"options\": {
      \"temperature\": 0.7,
      \"top_p\": 0.9,
      \"num_predict\": 100
    }
  }" | python -c "
import sys, json
data = json.load(sys.stdin)
print('响应:', data['response'])
print('生成耗时:', data['total_duration']/1e9, '秒')
print('每秒token数:', data['eval_count']/(data['eval_duration']/1e9) if data['eval_duration'] > 0 else 'N/A')
"

echo -e "\n测试2: 流式响应（适合长文本）"
curl -s -X POST "$API_URL/api/generate" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"$MODEL\",
    \"prompt\": \"写一篇关于Docker的简短介绍\",
    \"stream\": true,
    \"options\": {
      \"temperature\": 0.3
    }
  }" | while read -r line; do
    if [[ -n "$line" ]]; then
      echo "$line" | python -c "
import sys, json
try:
    data = json.loads(sys.stdin.read())
    if 'response' in data:
        print(data['response'], end='', flush=True)
except:
    pass
"
    fi
done
echo ""

echo -e "\n测试3: OpenAI兼容接口"
curl -s -X POST "$API_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"$MODEL\",
    \"messages\": [
      {\"role\": \"system\", \"content\": \"你是一个有帮助的助手\"},
      {\"role\": \"user\", \"content\": \"什么是机器学习？\"}
    ],
    \"temperature\": 0.7
  }" | python -m json.tool
```

## 五、Docker 部署 n8n

### 5.1 n8n 简介

n8n（发音为"n-eight-n"）是一个开源的工作流自动化工具，它通过可视化的方式连接各种应用和服务。为什么选择n8n而不是Zapier或Make？

1. **完全开源**：可以自托管，无使用限制
2. **节点丰富**：800+内置节点，包括Ollama专用节点
3. **代码友好**：支持JavaScript/Python代码节点，灵活扩展
4. **企业级特性**：用户管理、审计日志、Webhook等
5. **活跃社区**：持续更新，问题响应快

### 5.2 完整 docker-compose.yml

这是生产环境推荐的配置：

```yaml
# docker-compose.n8n.yml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n_server
    restart: unless-stopped
    ports:
      - "5678:5678"  # Web UI端口
    environment:
      # 1. 基础配置
      - N8N_PROTOCOL=http
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_EDITOR_BASE_URL=http://localhost:5678/
      
      # 2. 加密密钥（重要！）
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:-your-32-char-encryption-key-here-change-me}
      
      # 3. 数据库配置（使用PostgreSQL）
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD:-change_me_please}
      
      # 4. 执行模式
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168  # 保留7天执行记录
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000  # 最多保留1万条
      
      # 5. 安全配置
      - N8N_SECURE_COOKIE=false  # 本地部署可关闭
      - WEBHOOK_URL=http://localhost:5678/
      - N8N_USER_MANAGEMENT_DISABLED=false  # 启用用户管理
      
      # 6. 邮件配置（用于密码重置）
      - N8N_SMTP_HOST=smtp.gmail.com
      - N8N_SMTP_PORT=587
      - N8N_SMTP_USER=your-email@gmail.com
      - N8N_SMTP_PASS=${SMTP_PASSWORD}
      - N8N_SMTP_SENDER=your-email@gmail.com
      
      # 7. 性能优化
      - EXECUTIONS_PROCESS=main
      - EXECUTIONS_TIMEOUT=3600  # 1小时超时
      - N8N_MAX_PAYLOAD_SIZE=16MB
      
      # 8. Ollama连接配置
      - OLLAMA_HOST=ollama:11434  # 通过服务名访问
    volumes:
      # 9. 数据持久化
      - ./n8n_data:/home/node/.n8n
      
      # 10. 本地文件访问（可选）
      # - /path/to/local/files:/files
    networks:
      - ai_network  # 与Ollama同一网络
    depends_on:
      - postgres
    extra_hosts:
      # 11. 解决容器内部访问宿主机服务
      - "host.docker.internal:host-gateway"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:15-alpine
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${DB_PASSWORD:-change_me_please}
      - POSTGRES_DB=n8n
    volumes:
      # 12. 数据库数据持久化
      - ./postgres_data:/var/lib/postgresql/data
      
      # 13. 初始化脚本（可选）
      # - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - ai_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 30s
      timeout: 10s
      retries: 3

# 14. 网络配置
networks:
  ai_network:
    external: true  # 使用之前创建的共享网络
    name: ollama_ai_network  # 确保网络名称一致
```

### 5.3 关键配置解析

**1. 加密密钥（N8N_ENCRYPTION_KEY）**
```bash
# 生成安全的32字符密钥
openssl rand -base64 24
# 或使用pwgen
pwgen -s 32 1
```
为什么重要？这个密钥用于加密n8n中的敏感数据（API密钥、凭证等）。一旦丢失，所有加密数据将无法解密。

**2. extra_hosts配置**
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```
为什么需要？当n8n需要访问宿主机上的其他服务（如本地数据库、Redis）时，这个配置让容器内可以通过`host.docker.internal`访问宿主机。

**3. 共享网络配置**
```yaml
networks:
  ai_network:
    external: true
    name: ollama_ai_network
```
关键点：n8n和Ollama必须在同一Docker网络中，这样n8n容器内可以通过`http://ollama:11434`访问Ollama服务，而不是`http://localhost:11434`。

**4. 数据卷映射**
```yaml
volumes:
  - ./n8n_data:/home/node/.n8n
  - ./postgres_data:/