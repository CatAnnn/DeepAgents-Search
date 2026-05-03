# DeepAgent Project — 多智能体协作系统

基于 DeepAgents 框架构建的多智能体协作系统。一个主智能体（Team Leader）作为任务编排中心，协调三个专业子智能体从网络搜索、数据库查询、企业知识库等异构数据源中收集信息，最终生成结构化文档（Markdown / PDF）。系统通过 FastAPI + WebSocket 提供 REST API 与实时推送能力。

---

## 1. 项目结构

```
deepagent_project/
│
├── api/                                # FastAPI Web 服务层
│   ├── server.py                       # 项目真正入口（REST 接口 + WebSocket）
│   ├── monitor.py                      # ToolMonitor 单例 + ConnectionManager
│   └── context.py                      # ContextVar 会话隔离
│
├── agent/                              # 智能体定义
│   ├── llm.py                          # LLM 初始化（通义千问，OpenAI 兼容接口）
│   ├── main_agent.py                   # 主智能体：构建 + 异步流式执行
│   ├── prompts.py                      # 从 YAML 加载系统提示词
│   └── subagents/                      # 子智能体定义
│       ├── network_search_agent.py     # 网络搜索助手（Tavily）
│       ├── database_query_agent.py     # 数据库查询助手（MySQL）
│       └── knowledge_base_agent.py     # RAGFlow 知识库助手
│
├── tools/                              # LangChain @tool 工具函数
│   ├── tavily_tool.py                  # Tavily 网络搜索
│   ├── db_tools.py                     # MySQL 查询（列出表 / 读取数据 / 执行 SQL）
│   ├── ragflow_tools.py                # RAGFlow 知识库查询
│   ├── markdown_tools.py               # 生成 Markdown 文件
│   ├── pdf_tools.py                    # Markdown → PDF（Word COM）
│   └── upload_file_read_tool.py        # 读取上传文件（.md/.docx/.pdf/.xlsx）
│
├── utils/                              # 工具模块
│   ├── path_utils.py                   # 路径解析（会话隔离 + 安全校验）
│   └── word_converter.py               # Markdown → HTML → PDF（Word COM 自动化）
│
├── prompt/
│   └── prompts.yml                     # 所有智能体的系统提示词（YAML 统一管理）
│
├── ragflow/                            # RAGFlow SDK 示例
│   ├── rag_config.py                   # 环境变量加载
│   ├── chat_assistant_demo.py          # 示例：列出助手 + 提问
│   └── knowledge_demo.py              # 示例：创建知识库 + 上传文档
│
├── output/                             # 生成文件（按会话隔离，gitignored）
├── updated/                            # 上传文件（按会话隔离，gitignored）
│
├── .env                                # 环境变量（API Key、数据库配置等）
├── .gitignore
├── pyproject.toml                      # 项目元数据与依赖
├── requirements.txt                    # Pip 依赖列表
└── main.py                             # PyCharm 占位文件（非真实入口）
```

### 各层职责

| 层 | 目录 | 职责 |
|---|---|---|
| Web 服务层 | `api/` | HTTP 接口、WebSocket 长连接、会话隔离、实时监控上报 |
| 智能体层 | `agent/` | 主智能体构建与流式执行、子智能体定义、LLM 初始化、提示词加载 |
| 工具层 | `tools/` | LangChain 工具：搜索、数据库、知识库、文档生成、文件读取 |
| 基础设施层 | `utils/` | 路径安全、格式转换等通用能力 |
| 配置层 | `prompt/` | YAML 统一管理所有智能体的系统提示词 |

---

## 2. 核心组件

### 2.1 主智能体（Main Agent）

- **构建方式**：`deepagents.create_deep_agent`，注册 3 个自有工具 + 3 个子智能体
- **自有工具**：`generate_markdown`、`convert_md_to_pdf`、`read_file_content`
- **执行方式**：`astream()` 异步流式输出，逐 chunk 解析工具调用与最终结果
- **核心规则**（由 `prompts.yml` 约束）：
  - 必须先调用子智能体获取信息，**严禁**在获取信息前生成文件
  - 生成内容不少于 1000 字，且包含 todo-list
  - 不得使用占位符内容，只有拿到完整信息后才能调用生成工具

### 2.2 子智能体

```python
# 每个子智能体是一个字典，包含 name / description / system_prompt / tools
sub_agent = {
    "name": "网络搜索助手",
    "description": "...",
    "system_prompt": "...",
    "tools": [internet_search]
}
```

| 子智能体 | 工具 | 职责 |
|---|---|---|
| 网络搜索助手 | `internet_search` | 通过 Tavily API 搜索公网信息，每个问题至少从 3 个角度检索，最多 5 次 |
| 数据库查询助手 | `list_sql_tables`、`get_table_data`、`execute_sql_query` | 查询 MySQL `pharma_db` 数据库（药品信息、库存、销售数据） |
| RAGFlow 知识库助手 | `get_assistant_list`、`create_ask_delete` | 连接 RAGFlow 平台查询企业内部知识库，每个问题至少提问 3 次 |

### 2.3 API 接口

| 接口 | 方法 | 说明 |
|---|---|---|
| `/api/task` | POST | 提交用户查询，异步启动主智能体，立即返回 `thread_id` |
| `/api/upload` | POST | 上传文件至 `updated/session_{thread_id}/`，支持多文件 |
| `/api/download` | GET | 根据绝对路径下载输出文件，含路径遍历安全检查 |
| `/api/files` | GET | 列出输出目录文件（递归），返回文件名、大小、修改时间 |
| `/ws/{thread_id}` | WebSocket | 实时双向通信：服务端推送执行进度，客户端发送心跳 |

### 2.4 实时监控（ToolMonitor）

`ToolMonitor` 是一个单例，集成在每个工具函数中，通过 WebSocket 向前端推送四类事件：

| 事件类型 | 触发时机 | 推送内容 |
|---|---|---|
| `session_created` | 会话创建 | 工作目录路径 |
| `tool_start` | 工具开始执行 | 工具名称 + 参数 |
| `assistant_call` | 调用子智能体 | 子智能体名称 + 任务描述 |
| `task_result` | 主智能体完成 | 最终结果文本 |
| `error` | 执行异常 | 错误信息 |

---

## 3. 执行流程

### 3.1 完整调用链

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端（浏览器）                             │
│  1. POST /api/task  ──►  获取 thread_id                         │
│  2. WebSocket /ws/{thread_id}  ──►  建立长连接，接收实时推送      │
│  3. （可选）POST /api/upload  ──►  上传参考资料                   │
│  4. （可选）GET /api/files + /api/download  ──►  查看/下载结果    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                   api/server.py（FastAPI）                       │
│                                                                 │
│  POST /api/task:                                                │
│    ├─ 生成或接收 thread_id                                       │
│    ├─ asyncio.create_task(run_deep_agent(query, thread_id))     │
│    └─ 立即返回 {"status": "started", "thread_id": "..."}         │
│                                                                 │
│  WebSocket /ws/{thread_id}:                                     │
│    ├─ manager.connect() 注册连接                                 │
│    └─ 循环监听前端消息（心跳 ping/pong）                          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              agent/main_agent.py — run_deep_agent()             │
│                                                                 │
│  Step 1: 准备工作                                                │
│    ├─ 创建 output/session_{session_id}/ 目录                    │
│    ├─ 检测 updated/session_{session_id}/ 是否有上传文件           │
│    │   └─ 有上传文件 → 复制到 output 目录 + 拼接提示词             │
│    ├─ ContextVar 存储 session_dir 和 thread_id                  │
│    └─ monitor.report_session_dir() 推送目录信息                   │
│                                                                 │
│  Step 2: 构建提示词                                              │
│    ├─ 拼接工作目录路径指令                                         │
│    └─ 拼接上传文件信息（如有）                                     │
│                                                                 │
│  Step 3: 流式执行主智能体                                          │
│    └─ async for chunk in main_agent.astream(...):               │
│        ├─ chunk = {"model": {messages: [...]}}                  │
│        │   ├─ last_msg.tool_calls 存在 →                        │
│        │   │   ├─ tool_call.name == "task"                      │
│        │   │   │   └─ 调用子智能体                               │
│        │   │   │       └─ monitor.report_assistant() 推送        │
│        │   │   └─ tool_call.name == 其他工具                     │
│        │   │       └─ monitor.report_tool() 推送                 │
│        │   └─ last_msg.content 存在 → 最终结果                    │
│        │       └─ monitor.report_task_result() 推送              │
│        └─ chunk = {"tools": {messages: [...]}}                  │
│            └─ 工具执行返回结果                                    │
│                                                                 │
│  Step 4: 清理                                                    │
│    └─ reset_session_context() 释放 ContextVar                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐ ┌───────────┐ ┌──────────────┐
│ 网络搜索助手  │ │数据库查询  │ │RAGFlow 知识库│
│              │ │  助手      │ │   助手       │
│ internet_    │ │ list_sql_ │ │ get_         │
│ search       │ │ tables    │ │ assistant_   │
│ (Tavily API) │ │ get_table │ │ list         │
│              │ │ _data     │ │ create_ask_  │
│ 至少3个角度   │ │ execute_  │ │ delete       │
│ 最多5次检索   │ │ sql_query │ │              │
│              │ │           │ │ 至少提问3次   │
└──────┬───────┘ └─────┬─────┘ └──────┬───────┘
       │               │              │
       └───────────────┼──────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    主智能体汇总 & 生成文档                         │
│                                                                 │
│  ├─ 调用 generate_markdown()  ──►  output/session_xxx/report.md │
│  └─ （如需 PDF）调用 convert_md_to_pdf()                         │
│      └─ Markdown → HTML（markdown_tools）                        │
│      └─ HTML → PDF（word_converter，Word COM 自动化）             │
│                                                                 │
│  最终结果通过 monitor.report_task_result() 推送到前端              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 WebSocket 消息格式

服务端通过 WebSocket 推送的消息遵循统一格式：

```json
{
  "type": "monitor_event",
  "event": "assistant_call",
  "message": "正在调用助手: 网络搜索助手",
  "data": {
    "assistant_name": "网络搜索助手",
    "args": { "description": "搜索空调行业市场趋势..." }
  },
  "timestamp": "2026-05-04T10:30:00.000000"
}
```

各类事件的 `event` 字段值：`session_created` / `tool_start` / `assistant_call` / `task_result` / `error`

### 3.3 会话隔离机制

```
请求 A（thread_id = "abc"）              请求 B（thread_id = "xyz"）
  │                                        │
  ├─ ContextVar: thread_id = "abc"         ├─ ContextVar: thread_id = "xyz"
  ├─ 工作目录: output/session_abc/          ├─ 工作目录: output/session_xyz/
  ├─ WebSocket: /ws/abc                    ├─ WebSocket: /ws/xyz
  │                                        │
  └─ ToolMonitor → manager.send_to_thread  └─ ToolMonitor → manager.send_to_thread
       (payload, "abc")                         (payload, "xyz")
```

- **ContextVar**：Python 异步上下文变量，每个协程独立持有值，天然支持异步并发隔离
- **WebSocket**：`ConnectionManager` 以 `thread_id` 为 key 管理连接，消息定向推送
- **文件系统**：每个会话的输出/上传文件存储在独立的 `session_{id}/` 目录下

---

## 4. 最终效果

### 4.1 前端实时收到的推送序列

以用户提交 "请调研空调行业 2026 年市场趋势并生成 PDF 报告" 为例，前端 WebSocket 将依次收到：

```jsonc
// 1. 工作目录创建
{"event": "session_created", "message": "工作目录已创建: .../output/session_abc", "data": {"path": "..."}}

// 2. 调用网络搜索助手
{"event": "assistant_call", "message": "正在调用助手: 网络搜索助手", "data": {"assistant_name": "网络搜索助手"}}

// 3. 调用数据库查询助手
{"event": "assistant_call", "message": "正在调用助手: 数据库查询助手", "data": {"assistant_name": "数据库查询助手"}}

// 4. 调用 RAGFlow 知识库助手
{"event": "assistant_call", "message": "正在调用助手: RAGFlow助手", "data": {"assistant_name": "RAGFlow助手"}}

// 5. 生成 Markdown 文件
{"event": "tool_start", "message": "开始执行工具: generate_markdown", "data": {"tool_name": "generate_markdown"}}

// 6. 转换为 PDF
{"event": "tool_start", "message": "开始执行工具: convert_md_to_pdf", "data": {"tool_name": "convert_md_to_pdf"}}

// 7. 最终结果
{"event": "task_result", "message": "任务执行完成", "data": {"result": "已为您生成空调行业2026年市场趋势报告..."}}
```

### 4.2 生成的文件

任务执行完毕后，`output/session_{thread_id}/` 目录下将包含：

```
output/session_abc/
├── 空调行业2026年市场趋势报告.md      # Markdown 源文件
└── 空调行业2026年市场趋势报告.pdf     # PDF 转换结果（Windows + Word 环境）
```

前端可通过以下接口获取文件：

```bash
# 查看文件列表
GET /api/files?path=/absolute/path/to/output/session_abc

# 下载文件
GET /api/download?path=/absolute/path/to/output/session_abc/空调行业2026年市场趋势报告.pdf
```

### 4.3 典型使用场景

| 场景 | 用户输入 | 系统行为 |
|---|---|---|
| 信息查询 | "空调行业目前有哪些头部品牌？" | 调用网络搜索助手 → 直接返回文字结果 |
| 数据分析 | "分析本季度药品销售趋势" | 调用数据库查询助手 → 查询销售数据 → 返回分析结果 |
| 知识检索 | "公司的售后服务流程是什么？" | 调用 RAGFlow 助手 → 检索内部知识库 → 返回知识内容 |
| 文档生成 | "调研空调市场并生成 PDF 报告" | 调用全部子智能体收集信息 → 生成 Markdown → 转换 PDF |
| 文件分析 | （上传 Excel 后）"帮我分析这份数据" | 读取上传文件 → 调用数据库助手对比 → 生成分析报告 |

---

## 5. 技术栈

| 类别 | 技术 |
|---|---|
| 编程语言 | Python 3.12+ |
| 多智能体框架 | DeepAgents 0.4.3 |
| LLM 编排 | LangChain + LangGraph |
| 大语言模型 | 阿里云 DashScope（通义千问 qwen3.5-plus 等） |
| Web 框架 | FastAPI + Uvicorn |
| 实时通信 | WebSocket（FastAPI/Starlette） |
| 网络搜索 | Tavily API |
| 知识库 / RAG | RAGFlow SDK |
| 数据库 | MySQL（mysql-connector-python） |
| 文档生成 | Markdown + pywin32（Word COM 转 PDF） |
| 文件解析 | python-docx、pypdf、pandas |
| 配置管理 | python-dotenv + YAML |
| 数据校验 | Pydantic |

---

## 6. 快速开始

### 6.1 环境准备

```bash
# 克隆项目
git clone <repository-url>
cd deepagent_project

# 创建虚拟环境
python -m venv .venv
# Windows
.venv\Scripts\activate
# Linux/Mac
source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

### 6.2 配置环境变量

在项目根目录创建 `.env` 文件：

```env
# 阿里云 DashScope LLM
DASHSCOPE_API_KEY=your_api_key
BASE_URL=https://dashscope.aliyuncs.com/compatible-mode
MODEL_NAME=qwen3.5-plus-2026-02-15

# Tavily 网络搜索
TAVILY_API_KEY=your_tavily_key

# RAGFlow 知识库
RAGFLOW_API_KEY=your_ragflow_key
RAGFLOW_BASE_URL=http://your_ragflow_server

# MySQL 数据库
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=pharma_db
```

### 6.3 启动服务

```bash
uvicorn api.server:app --host 0.0.0.0 --port 8000 --reload
```

> 真实入口为 `api/server.py`，`main.py` 仅为 PyCharm 占位文件。

### 6.4 使用方式

**提交任务：**

```bash
curl -X POST http://localhost:8000/api/task \
  -H "Content-Type: application/json" \
  -d '{"query": "请帮我调研空调行业市场趋势并生成报告", "thread_id": "session_001"}'
```

**上传文件：**

```bash
curl -X POST http://localhost:8000/api/upload \
  -F "files=@data.xlsx" \
  -F "thread_id=session_001"
```

**WebSocket 接收实时进度：**

```javascript
const ws = new WebSocket('ws://localhost:8000/ws/session_001');
ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    console.log(`[${msg.event}] ${msg.message}`);
};
```

---

## 7. 设计要点

- **会话隔离** — `ContextVar` 实现异步并发下的状态隔离，每个会话独立持有 `session_dir` 和 `thread_id`，互不干扰
- **实时监控** — `ToolMonitor` 单例内嵌于每个工具函数，通过 `ConnectionManager` 定向推送事件到对应会话的 WebSocket
- **异步解耦** — `POST /api/task` 立即返回，主智能体通过 `asyncio.create_task` 在后台流式执行，前端通过 WebSocket 接收结果
- **路径安全** — `path_utils.py` 清洗虚拟路径前缀、校验路径是否在允许范围内，防止目录遍历攻击
- **提示词驱动** — 所有智能体行为规则集中在 `prompt/prompts.yml`，修改行为无需改动代码
- **严格工作流** — 主智能体必须先收集信息再生成文件，禁止占位符生成，确保输出内容的质量和完整性
