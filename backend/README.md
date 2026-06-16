# EchoMind — 智能客服 AI 系统 v2.0

基于大语言模型的企业级智能客服系统，支持**多 Agent 编排**、**三路融合意图识别**、**三级记忆架构**、**MCP 工具调用优化链路**，内置**实时监控闭环**和**端到端评测框架**。

```
    ʕ•ᴥ•ʔ  ʕ•ᴥ•ʔ  ʕ•ᴥ•ʔ
   ╔══════════════════════╗
   ║   EchoMind  v2.0     ║
   ║   智能客服 AI 系统    ║
   ╚══════════════════════╝
    ʕ•ᴥ•ʔ  ʕ•ᴥ•ʔ  ʕ•ᴥ•ʔ
```

---

## 架构总览

```
用户请求 → Nginx → FastAPI → 意图识别（三路融合）
                           → 记忆读取（三级架构）
                           → Agent 路由（三层决策）
                           → 工具调用（改写→召回→重排）
                           → 记忆写入 + 用户画像更新
                ↑
           Monitor 监控闭环（采集 → 异常检测 → 路由反馈）
```

### 核心亮点

| 模块 | 核心技术 | 解决什么问题 |
|------|---------|-------------|
| **意图识别** | LLM + Embedding + Pattern 三路加权投票 | 复杂语义理解 + 零延迟兜底 |
| **Agent 编排** | 意图路由 → 性能路由 → 降级路由 | 多 Agent 精准分发 + 容灾 |
| **工具调用** | 查询改写 → 并行召回 → LLM 重排 | 检索不全、召回不好的优化 |
| **记忆管理** | Redis 工作记忆 + ChromaDB 情景/画像 | 多轮对话上下文 + 跨会话记忆 |
| **性能监控** | Z-score 异常检测 + Prometheus | Agent 在线表现实时追踪 |
| **质量评测** | LLM-as-Judge + 回归检测 | 规模化、可重复的端到端评测 |

---

## 技术栈

| 组件 | 选型 |
|------|------|
| Web 框架 | FastAPI 0.115 + Uvicorn |
| LLM 后端 | Anthropic Claude API（兼容 DeepSeek 等第三方 API） |
| 工作记忆 | Redis 7（TTL 24h，自动压缩） |
| 向量数据库 | ChromaDB 0.5（内置 ONNX Embedding，三路复用） |
| 反向代理 | Nginx（限流、Gzip、安全头） |
| 监控 | Prometheus + 自定义 Z-score 异常检测 |
| 容器化 | Docker 多阶段构建 + Docker Compose 5 服务编排 |

---

## 快速开始

### 前置条件

- Docker & Docker Compose
- Anthropic API Key（或兼容的第三方 API Key）

### 部署（推荐）

```bash
# 1. 配置环境变量
cp .env.example.env .env
# 编辑 .env：设置 ANTHROPIC_API_KEY

# 2. 一键安装启动
./docker-deploy.sh install
./docker-deploy.sh start

# 3. 验证
curl http://localhost:8000/health
```

### 本地开发

```bash
# 需要本地 Redis 和 ChromaDB 服务
cp .env.example .env
pip install -r requirements.txt

# 启动 API
python api/main.py

# 或交互式命令行
python api/main.py --cli
```

---

## API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/health` | 健康检查 + Agent 运行时统计 |
| `POST` | `/chat` | 主对话接口（意图识别 → Agent 路由 → 回复） |
| `GET` | `/monitor` | 监控摘要（Agent/工具统计、告警、优化建议） |
| `GET` | `/metrics` | Prometheus 指标端点 |
| `POST` | `/search` | 检索优化链路演示（查询改写 → 召回 → 重排） |
| `POST` | `/knowledge/add` | 批量导入文档到知识库 |
| `POST` | `/knowledge/upload` | 文件上传导入（支持 txt/md/json） |
| `GET` | `/knowledge/stats` | 知识库统计 |
| `POST` | `/eval/run` | 运行端到端评测 |

### 对话请求示例

```json
POST /chat
{
  "message": "应用登录一直报错 401",
  "user_id": "user_001",
  "conv_id": "conv_abc"
}
```

```json
// 响应
{
  "conv_id": "conv_abc",
  "response": "收到您关于应用登录报错 401 的问题...",
  "intent": "technical",
  "agent_type": "technical",
  "escalated": false,
  "latency_ms": 1234.5,
  "knowledge_used": true
}
```

---

## 模块详解

### 1. 意图识别 — `core/intent_recognizer.py`

三路融合策略：
- **LLM 语义**（70%）：Few-shot prompt，理解复杂语义和对话上下文
- **Embedding 相似度**（20%）：模板向量匹配，字符 n-gram 哈希兜底
- **关键词模式**（10%）：零延迟同步识别

三路加权投票，置信度低于阈值（默认 0.5）降级为 `other`。LLM 和 Embedding 并行调用，不串行等待。

### 2. Agent 编排 — `agents/agent_orchestrator.py`

三层路由决策：
1. **意图路由**：`IntentCategory` → `AgentType` 静态映射表
2. **性能路由**：同类多实例按 `routing_score()` 选最优（成功率 + 延迟加权）
3. **降级路由**：专属 Agent 不可用时自动降级到 `GeneralAgent`

复杂问题（如"登录报错且被重复扣款"）自动触发并行协作，派发给多个 Agent 后合并结果。

### 3. MCP 工具框架 — `mcp/tool_manager.py`

检索优化全链路：
```
用户查询 → 查询改写（多角度子查询）→ 并行召回 → 去重 → LLM 重排 → Top-K
```

内置**三态熔断器**（CLOSED → OPEN → HALF_OPEN）、**TTL 缓存**、**参数校验**、**降级策略**。

### 4. 知识库 — `mcp/knowledge_base.py`

基于 ChromaDB 的 RAG 实现：
- 文档自动切片（每 500 字，按句号/换行切分）
- ChromaDB 内置 Embedding 自动向量化
- 与记忆模块使用不同的 collection，互不干扰
- 内置默认客服知识（退款政策、订单查询、账户安全等）

### 5. 三级记忆 — `memory/conversation_memory.py`

- **工作记忆**（Redis）：当前会话最近消息，TTL 24h，超 15 条自动压缩
- **情景记忆**（ChromaDB `episodic`）：跨会话历史，语义检索相关片段
- **用户画像**（ChromaDB `user_profile`）：LLM 提炼偏好和实体

压缩流程：旧消息 → LLM 摘要（存 Redis）→ 原文存入情景记忆 → 工作记忆保留最近 5 条。

### 6. 监控反馈 — `monitor/performance_monitor.py`

闭环反馈：
```
Monitor 采集 → 发现 Agent 成功率下降
  → routing_score 自动降低
  → Orchestrator._best_agent() 路由时自动绕开
```

- Z-score 异常检测（滑动窗口 60 点，灵敏度 2.5σ）
- 阈值告警（成功率 < 90%、延迟 > 3s 等）
- 可选 Prometheus 指标暴露 + Webhook 告警

### 7. 评测框架 — `evaluation/evaluator.py`

- **意图识别评测**：Accuracy / Macro-F1 / 每类 Precision/Recall
- **LLM-as-Judge**：四维评分（相关性、准确性、完整性、有用性），及格线 0.75
- **回归检测**：与历史基线对比，退化 > 5% 自动标记
- 内置 8 个意图测试用例 + 5 个对话测试用例（含多轮）

---

## 部署架构

```
                    ┌──────────────┐
                    │    Nginx      │  (80)
                    └──────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐  ┌──────▼──────┐  ┌─────▼─────┐
     │ EchoMind   │  │  Prometheus  │  │  Redis     │
     │ FastAPI    │  │  (9090)      │  │  (6379)    │
     └─────┬─────┘  └─────────────┘  └───────────┘
           │
     ┌─────▼─────┐
     │ ChromaDB   │
     │  (8001)    │
     └───────────┘
```

---

## 配置说明

关键环境变量（完整列表见 `.env.example.env`）：

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ANTHROPIC_API_KEY` | LLM API 密钥 | 必填 |
| `ANTHROPIC_BASE_URL` | 第三方 API 地址（DeepSeek 等） | 空（使用官方） |
| `ANTHROPIC_MODEL` | 模型名称 | `claude-3-5-sonnet-20241022` |
| `REDIS_URL` | Redis 连接 | `redis://localhost:6379/0` |
| `CHROMA_HOST` | ChromaDB 地址 | `localhost` |
| `CHROMA_PORT` | ChromaDB 端口 | `8001` |
| `PROMETHEUS_PORT` | Prometheus 指标端口 | `9091` |

---

## License

MIT
