# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于自研 C++ HTTP 框架的 AI 应用服务平台（第二版），使用纯 C++17 构建。支持多模型对话、RAG、轻量级 MCP 工具协议、ASR/TTS、图像识别、消息队列异步化、多会话多租户等能力。

## 构建命令

```bash
# 创建构建目录并编译
mkdir -p build && cd build
cmake ..
make -j$(nproc)

# 运行服务（默认端口 80，可通过 -p 参数指定）
./http_server -p 8080
```

## 依赖库

- **muduo** — 网络事件循环（TcpServer/EventLoop）
- **OpenSSL + CURL** — HTTPS 请求与 AI API 调用
- **MySQL Connector/C++** (mysqlcppconn8) — 数据库存储
- **OpenCV + ONNX Runtime** — 本地图像识别推理
- **SimpleAmqpClient + RabbitMQ** — 异步消息队列
- **nlohmann/json** — JSON 解析（通过 JsonUtil.h 封装）

## 环境变量

运行前需设置以下 API Key：

```bash
export DASHSCOPE_API_KEY=xxx    # 阿里百炼/通义千问
export DOUBAO_API_KEY=xxx       # 火山引擎豆包
```

## 架构

### 目录结构

```
CppAIService/
├── CMakeLists.txt              # 顶层构建配置
├── HttpServer/                 # 自研 HTTP 框架（可复用基础组件）
│   ├── include/
│   │   ├── http/               # HttpServer, HttpRequest, HttpResponse, HttpContext
│   │   ├── router/             # Router, RouterHandler（路由注册与分发）
│   │   ├── session/            # Session, SessionManager, SessionStorage
│   │   ├── middleware/         # Middleware, MiddlewareChain（含 CORS）
│   │   ├── ssl/                # SslContext, SslConnection, SslConfig
│   │   └── utils/              # JsonUtil, FileUtil, MysqlUtil
│   └── src/
└── AIApps/
    └── ChatServer/             # AI 应用业务层
        ├── include/
        │   ├── ChatServer.h    # 主服务器类（组合 HttpServer + AI 模块）
        │   ├── handlers/       # HTTP 请求处理器（登录/注册/聊天/历史/上传/语音等）
        │   └── AIUtil/         # AI 核心工具类
        ├── src/
        │   ├── main.cpp        # 入口：初始化 ChatServer + RabbitMQ 线程池
        │   ├── ChatServer.cpp  # 路由注册、会话管理、MySQL 数据加载
        │   ├── handlers/       # 各处理器实现
        │   └── AIUtil/         # AI 核心逻辑实现
        └── resource/           # 前端 HTML 页面 + config.json（MCP 工具配置）
```

### 核心设计模式

**策略模式 + 注册式工厂（多模型解耦）：**
- `AIStrategy` — 抽象策略接口（buildRequest / parseResponse / getApiUrl / getApiKey / getModel）
- `AliyunStrategy` / `DouBaoStrategy` / `AliyunRAGStrategy` / `AliyunMcpStrategy` — 具体模型实现
- `StrategyFactory` — 单例工厂，通过 `StrategyRegister<T>` 模板自动注册策略
- `AIHelper` — 持有 `shared_ptr<AIStrategy>`，封装消息历史管理与 HTTP 调用

**MCP 工具协议化（两段式推理）：**
- `AIConfig` — 加载 `config.json` 中的 prompt 模板和工具列表
- `AIToolRegistry` — 工具注册表（get_weather、get_time 等），支持动态扩展
- 流程：用户输入 → 判断是否需要工具 → 调用工具 → 将结果二次提交模型

**消息队列异步化：**
- `MQManager` — RabbitMQ 连接池（单例）
- `RabbitMQThreadPool` — 消费线程池，异步执行 MySQL 写入
- 前台同步写内存（chatInformation），异步入库

**多会话管理：**
- `chatInformation` — `unordered_map<userId, map<sessionId, AIHelper>>` 实现用户级+会话级隔离
- `AISessionIdGenerator` — 会话 ID 生成器

### 请求处理流程

```
Client → muduo::TcpServer → HttpServer(onMessage → onRequest)
  → MiddlewareChain → Router::route → Handler::handle
    → ChatServer 业务逻辑 → AIHelper::chat → AIStrategy::buildRequest
      → CURL 调用外部 AI API → parseResponse → 返回
```

### Handler 一览

| Handler | 功能 |
|---------|------|
| ChatLoginHandler / ChatRegisterHandler / ChatLogoutHandler | 用户认证 |
| ChatEntryHandler / AIMenuHandler | 页面入口 |
| ChatSendHandler | 单轮聊天 |
| ChatCreateAndSendHandler | 创建会话 + 发送消息 |
| ChatHistoryHandler | 获取历史消息 |
| ChatSessionsHandler | 获取用户会话列表 |
| AIUploadHandler / AIUploadSendHandler | 图像上传 + ONNX 识别 |
| ChatSpeechHandler | 语音识别/合成（百度 API） |

### 关键数据结构

- `AIHelper::messages` — `vector<pair<content, timestamp_ms>>`，偶数下标为用户消息，奇数为 AI 回复
- `ChatServer::ImageRecognizerMap` — 每用户一个 ONNX 图像识别器实例
- `ChatServer::onlineUsers_` — 在线用户状态表

### 前端资源

`AIApps/ChatServer/resource/` 包含完整的前端单页应用（AI.html、entry.html、menu.html、upload.html），通过 REST API 与后端交互。`config.json` 配置 MCP 的 prompt 模板和可用工具清单。
