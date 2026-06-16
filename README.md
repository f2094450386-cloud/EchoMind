# EchoMind

EchoMind 是一个智能客服 Agent 项目整理仓库，包含 Python 后端、Vue 前端调试台和原作者项目学习文档。

## 目录结构

```text
EchoMind/
├── backend/   # Python / FastAPI 后端，包含 Agent、RAG、记忆、监控和评测链路
├── frontend/  # Vue 3 / Vite 前端调试台，可切换 Python / Java 后端
└── docs/      # 原作者 PDF 学习文档
```

## 后端

后端核心链路：

```text
用户请求
-> FastAPI /chat
-> MemoryManager 读取 Redis 工作记忆和 ChromaDB 历史记忆
-> IntentRecognizer 三路融合识别意图
-> MCPToolManager / KnowledgeBase 检索知识库
-> AgentOrchestrator 路由到 General / Technical / Billing Agent
-> LLM 生成回复
-> 写入记忆并异步更新用户画像
```

常用启动方式见 [backend/README.md](backend/README.md)。

## 前端

前端是独立 Vue 调试控制台，支持：

- 聊天调试
- 健康检查
- 监控状态查看
- 知识库检索
- 知识库文档导入和上传
- Python / Java 后端切换

本地运行：

```bash
cd frontend
npm install
npm run dev
```

默认访问：

```text
http://localhost:5173
```

## 学习顺序

建议按下面顺序阅读 `docs/`：

1. `EchoMind 业务流程说明.pdf`：先理解一次用户请求的完整业务流。
2. `EchoMind 技术亮点详解.pdf`：建立面试中要重点讲的技术点。
3. `EchoMind 重点代码讲解.pdf`：对应源码看关键方法和调用链。
4. `EchoMind 完整使用指南.pdf`：用于部署、启动、API 调用和排障。
5. `EchoMind 简历包装指南.pdf`：最后再整理简历话术。

## 简历定位

更稳妥的表述是“复现、部署、梳理并二次整理 EchoMind 智能客服 Agent 系统”，不要写成完全从零独立原创，除非你已经能从零复现核心模块。

可以重点强调：

- FastAPI 后端接口设计
- LLM + Pattern + Embedding/相似度的三路融合意图识别
- RAG 知识库检索、查询改写、并行召回和重排
- Redis + ChromaDB 三级记忆
- 多 Agent 路由和协作
- MCP 工具调用的缓存、超时、熔断和 fallback
- Monitor 在线监控和路由降权
- `/eval/run` 端到端评测与 LLM-as-Judge

## 注意

仓库不应提交 `.env`、虚拟环境、运行日志、ChromaDB 本地数据、IDE 配置或 macOS ZIP 资源目录 `__MACOSX`。
