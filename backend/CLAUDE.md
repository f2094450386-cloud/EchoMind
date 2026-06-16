# EchoMind 开发与讲解规则

本文件用于规范 Claude Code 在 EchoMind 项目中的工作方式。目标是快速理解 EchoMind 的完整架构链路、源码实现，并在修改代码时遵循项目既定模式。默认只读源码和文档，修改代码前必须确认意图。

## 项目定位

`D:\Agent\EchoMind` 是一个基于大语言模型的企业级智能客服 AI 系统（v2.0），使用 Anthropic Claude API（兼容 DeepSeek 等第三方协议）作为 LLM 后端。核心能力：

- **三路融合意图识别**：LLM 语义（70%）+ Embedding 相似度（20%）+ 关键词模式（10%），加权投票，低于阈值降级为 OTHER。
- **多 Agent 编排**：三层路由（意图路由 → 性能路由 → 降级路由），复杂问题并行派发多 Agent 合并结果。
- **MCP 工具调用框架**：查询改写 → 并行召回 → LLM 重排 → Top-K，内置熔断、缓存、降级。
- **三级记忆架构**：Redis 工作记忆（TTL 24h，自动压缩）→ ChromaDB 情景记忆（跨会话语义检索）→ ChromaDB 用户画像（LLM 提炼偏好）。
- **Monitor 闭环反馈**：Z-score 异常检测 → 路由惩罚系数写回 Orchestrator → Agent 路由自动调权。
- **端到端评测**：意图 Accuracy/F1 + LLM-as-Judge 四维评分 + 回归检测。

不要把普通 FastAPI 项目的单体思维套到本项目。EchoMind 的架构核心是 **构造注入 + 异步协作 + 闭环反馈**，每个模块有明确的职责边界和可替换的依赖接口。

## 固定资料读取顺序

每次开始工作前，先读：

```
D:\Agent\EchoMind\EchoMind\CLAUDE.md
D:\Agent\EchoMind\EchoMind\README.md
```

再按本次目标读取：

```
1. .env.example（环境变量配置）
2. docker-compose.yml / Dockerfile（部署架构）
3. requirements.txt（依赖清单）
4. 对应模块源码，按以下顺序：
   a. api/main.py（入口、lifespan 初始化、路由注册）
   b. 目标模块的 __init__ 或类定义
   c. 目标模块的公开方法
   d. 目标模块依赖的其他模块
5. config/（Nginx、Prometheus 配置）
6. data/eval/baseline.json（评测基线，了解预期表现）
```

如果文档和源码冲突，以当前源码为准。

## 目录职责

```
api/main.py
FastAPI 入口。lifespan 中初始化所有全局组件（Orchestrator、MemoryManager、
MCPToolManager、KnowledgeBase、PerformanceMonitor、EndToEndEvaluator），
通过构造注入传递依赖。路由包括 /chat、/health、/monitor、/metrics、/search、
/knowledge/*、/eval/run。CLI 模式（--cli）提供交互式命令行。

agents/agent_orchestrator.py
多 Agent 编排器。定义了 Request、AgentResponse、OrchestratorResult 数据结构，
BaseAgent 基类（封装 LLM 调用和统计），三个专属 Agent（General/Technical/Billing），
和 AgentOrchestrator 编排器（三层路由 + 并行协作 + 降级）。Agent 路由评分
通过 AgentStats.routing_score() 计算，综合成功率和延迟。

core/intent_recognizer.py
意图识别器。定义了 10 种 IntentCategory 和 4 级 UrgencyLevel。三路融合识别：
_llm_recognize（Few-shot prompt）、_embedding_recognize（模板向量匹配，
字符 n-gram 哈希兜底）、_pattern_recognize（关键词同步匹配）。加权投票合并结果，
LLM 和 Embedding 并行调用。支持 LRU 缓存和在线学习（learn()）。

mcp/tool_manager.py
MCP 工具调用框架。Tool 数据类持有 handler、schema、cache_ttl、fallback 等配置。
MCPToolManager 提供完整的工具调用链路：缓存检查 → 熔断检查 → 参数校验 →
执行（含超时）→ 缓存写入 → 可选重排。search_with_rewrite() 实现检索优化全链路
（查询改写 → 并行召回 → 去重 → LLM 重排）。CircuitBreaker 为三态熔断器。

mcp/knowledge_base.py
基于 ChromaDB 的 RAG 知识库。文档自动切片（500 字/片，按句号切分），
ChromaDB 内置 Embedding 自动向量化。作为 knowledge_search 工具的 handler
注册到 MCPToolManager。内置默认客服知识文档。

memory/conversation_memory.py
三级记忆管理器。Message 和 MemoryContext 数据结构定义上下文格式。
MemoryManager：add_message 写入 Redis 工作记忆（超 15 条触发压缩），
get_context 融合三级记忆构建完整上下文，update_profile 用 LLM 提炼用户画像。
压缩流程：旧消息 → LLM 摘要（存 Redis）→ 原文存入 ChromaDB 情景记忆。

monitor/performance_monitor.py
性能监控器。AnomalyDetector 基于滑动窗口 Z-score 做异常检测。
PerformanceMonitor 每 N 秒采集 Orchestrator + ToolManager 统计，
检测异常、触发阈值告警、生成路由惩罚系数写回 Orchestrator、生成优化建议。
可选 Prometheus 指标暴露和 Webhook 告警。

evaluation/evaluator.py
端到端评测框架。IntentEvaluator 计算 Accuracy/Macro-F1/每类 P/R。
LLMJudge 用 LLM 从四维度（相关性、准确性、完整性、有用性）评判回复质量。
EndToEndEvaluator 编排完整评测流程（意图评测 → 对话评测 → 回归检测）。
内置默认测试用例，支持与历史基线对比。

config/     Nginx、Prometheus 配置文件。
data/       ChromaDB 持久化数据、评测基线、示例知识库文档。
```

## 每个模块的讲解链路

讲任何一个模块，必须按这条链路展开：

```
模块解决什么核心问题（为什么需要它）
→ 入口在哪里（哪个类/哪个方法被谁调用）
→ 输入是什么（数据结构、参数来源）
→ 内部处理流程（关键方法调用链、数据流转）
→ 输出是什么（返回什么、给谁用）
→ 依赖哪些外部服务（Redis / ChromaDB / Anthropic API）
→ key / collection / collection name 如何设计
→ 异常路径是什么（失败怎么办、降级是什么）
→ 与哪些模块联动（Orchestrator ↔ Monitor 反馈闭环等）
→ 为什么这样设计（取舍、trade-off）
```

讲源码时，必须说明：

- 类负责什么。
- 方法被谁调用（追溯到 API 路由或内部调用链）。
- 参数从哪里来（环境变量 / 构造注入 / 请求体 / 其他模块）。
- 返回值给谁用（API 响应 / 下一环节 / LLM prompt）。
- 哪些地方访问 Redis（key 设计、TTL）。
- 哪些地方访问 ChromaDB（collection、Embedding 由谁生成）。
- 哪些地方调用 Anthropic API（model、max_tokens、temperature）。
- 为什么这样设计。
- 关键代码片段必须在回答中加详细中文注释；除非明确要求修改源码，注释不写进源码文件。

## 基础概念补齐

默认已理解 Python FastAPI 基础。第一次出现以下概念时，要先解释概念，再结合源码讲：

- Anthropic Messages API、system prompt、max_tokens、temperature。
- Embedding 向量、余弦相似度、字符 n-gram 哈希向量（本地兜底）。
- Few-shot prompting、LLM-as-Judge。
- Redis List（LPUSH/LRANGE）、TTL、key 命名空间设计。
- ChromaDB collection、Embedding 自动生成、语义检索 vs 关键词检索。
- 查询改写、并行召回、结果重排（Rerank）。
- 熔断器三态（CLOSED / OPEN / HALF_OPEN）、TTL 缓存。
- Z-score 异常检测、滑动窗口、Prometheus Counter/Gauge/Histogram。
- 构造注入、lifespan 生命周期、asyncio 并行。
- RAG（检索增强生成）、向量数据库、文档切片策略。
- LLM 温度参数对输出的影响（0.0 用于评测和实体提取，0.3 用于查询改写，0.7 可用于对话）。
- Unicode 代理字符问题（`_clean_text()` 的必要性）。

## 中间件/技术点讲解模板

每讲 Redis、ChromaDB、Anthropic API、Nginx、Docker、Prometheus 等技术点，都补齐：

```
业务问题是什么
为什么只用 Python 内存或同步代码不够
为什么这个技术适合
用了什么数据结构或机制
key / collection / API endpoint 如何设计
数据保存多久（TTL）
失败风险是什么
并发风险是什么
Python 代码在哪里调用
真实生产还需要什么兜底
为什么这样设计
```

## 架构约束与修改规则

### 禁止事项

除非明确要求：

- 不引入新的第三方依赖（numpy、pandas 等重依赖在 EchoMind 里刻意避免）。
- 不把构造注入改成全局变量或单例模式。
- 不删除 `_clean_text()` 调用（Unicode 代理字符会导致 LLM API 调用崩溃）。
- 不把 ChromaDB 内置 Embedding 替换为外部 Embedding API（会影响离线部署场景）。
- 不在模块间引入循环依赖。
- 不跳过异常处理和降级逻辑。
- 不把异步改为同步（整个链路都是异步设计的）。

### 修改代码前的检查清单

1. 确认要改的模块是否在 lifespan 中被初始化（全局组件变更需要检查初始化顺序）。
2. 确认是否影响多个 Agent 的 system_prompt（改 Agent 行为需要检查所有子类）。
3. 确认是否引入了新的环境变量（需要同步更新 `.env.example` 和 `.env.example.env`）。
4. 确认 ChromaDB collection 名是否冲突（知识库用 `knowledge_base`，记忆用 `episodic` 和 `user_profile`）。
5. 确认新增的 LLM 调用是否传了正确的 `max_tokens` 和 `temperature`（不同用途参数不同）。
6. 确认异常路径有降级处理（不把 raw exception 直接抛给用户）。

### 新增 Agent 规则

1. 继承 `BaseAgent`，设置 `agent_type` 和 `system_prompt`。
2. 在 `AgentOrchestrator._pool` 中注册。
3. 在 `_INTENT_ROUTING` 映射表中添加对应的意图 → Agent 类型映射。
4. 更新 `_collaboration_targets()` 中的领域关键词（如需并行协作）。
5. Evaluator 的默认测试用例中包含新 Agent 覆盖的意图。

### 新增 MCP 工具规则

1. 定义 `Tool` 数据类实例（name、description、handler、schema、cache_ttl、supports_rerank、fallback）。
2. 在 `lifespan` 中（`api/main.py`）调用 `_tool_manager.register()`。
3. handler 签名必须是 `async def handler(params: Dict[str, Any], context: Any) -> Any`。
4. 如果是检索类工具，设置 `supports_rerank=True` 以获得查询改写和重排能力。
5. 提供 fallback 函数用于工具不可用时的降级结果。

## 代码规范

- Python 3.12+，类型注解使用 `typing` 模块（`Dict[str, Any]`、`Optional[X]`、`List[X]`）。
- 模块间依赖通过构造注入，接口类型通过 Duck Typing（不定义抽象基类）。
- 所有面向 LLM 的字符串必须经过 `_clean_text()` 处理。
- 日志使用 `logging.getLogger(__name__)`，warning 以上级别用于外部依赖失败。
- 统计值用纯 Python 实现（`statistics` 模块），不引入 numpy。
- 中文字段名和注释用中文，代码标识符用英文。
- 模块注释用 `"""..."""` 写在文件顶部，一句话说明模块职责。

## 工作方式限制

除非明确要求：

- 不改业务源码。
- 不自动重构（简单的东西用简单代码，不需要炫技抽象）。
- 不自动提交 Git。
- 不默认启动项目（需要 Redis + ChromaDB 服务）。
- 不跳过模块间的联动关系。
- 不跳过异常路径和降级策略。
- 不用抽象总结代替源码讲解。
- 在回答代码相关问题时，必须引用具体文件路径和行号。
