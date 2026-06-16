<img src="docs/images/header.svg" alt="adk-utils-go" width="600">

# ADK Utils Go

[Google Agent Development Kit (ADK)](https://google.github.io/adk-docs/) 的 Go 语言工具与实现。

本仓库提供以下生产级实现：

- **LLM 客户端**：兼容 ADK 的 OpenAI 与 Anthropic 客户端
- **Artifact 存储**：基于文件系统的版本化 Artifact 持久化
- **Context Guard**：基于 LLM 的自动上下文窗口管理
- **Langfuse**：可观测性插件 — 通过 OTLP/HTTP 追踪每一次 LLM 调用，包含完整的提示/响应负载与 Token 用量

## 目录结构

```
├── genai/               # LLM 客户端实现
│   ├── anthropic/       # Anthropic Claude 客户端
│   └── openai/          # OpenAI 客户端（兼容 Ollama、OpenRouter 等）
├── artifact/            # Artifact 服务实现
│   └── filesystem/      # 基于文件系统的 Artifact 服务（版本化、用户隔离）
├── plugin/              # ADK 插件实现
│   ├── contextguard/    # 上下文窗口管理插件
│   └── langfuse/        # Langfuse 可观测性插件（OTLP/HTTP 追踪）
├── examples/            # 可运行示例
│   ├── anthropic-client/  # Anthropic Claude 客户端示例
│   ├── context-guard/     # ContextGuard 插件示例
│   └── openai-client/     # OpenAI/Ollama 客户端示例
├── docs/                # 文档资源
│   └── images/
├── go.mod               # Go 模块定义
├── go.sum               # 依赖校验
├── LICENSE              # Apache 2.0 许可证
└── README.md            # 本文件
```

## 安装

```bash
go get github.com/achetronic/adk-utils-go

go mod edit -replace github.com/achetronic/adk-utils-go=github.com/dtapps/adk-utils-go@master

go mod tidy
```

## LLM 客户端

### OpenAI 客户端

兼容 OpenAI API 及任何 OpenAI 兼容 API（Ollama、OpenRouter、Azure OpenAI 等）：

```go
import genaiopenai "github.com/achetronic/adk-utils-go/genai/openai"

llmModel := genaiopenai.New(genaiopenai.Config{
    APIKey:    os.Getenv("OPENAI_API_KEY"),
    BaseURL:   "http://localhost:11434/v1", // Ollama 地址
    ModelName: "gpt-4o",                      // 或 Ollama 的 "qwen3:8b"
})

agent, _ := llmagent.New(llmagent.Config{
    Name:  "assistant",
    Model: llmModel,
})
```

### Anthropic 客户端

原生 Anthropic Claude 支持：

```go
import genaianthropic "github.com/achetronic/adk-utils-go/genai/anthropic"

llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:    os.Getenv("ANTHROPIC_API_KEY"),
    ModelName: "claude-sonnet-4-5-20250929",
})

agent, _ := llmagent.New(llmagent.Config{
    Name:  "assistant",
    Model: llmModel,
})
```

#### 扩展思考（推理）

Claude 可以在最终回答前产生内部推理链。存在**两种推理 API**，Anthropic 会拒绝错误的调用方式并返回 HTTP 400，因此需要根据模型通过 `ThinkingMode` 选择：

- **经典模式（`"enabled"`）**：基于预算。推理 Token 计入输出 Token，因此 `ThinkingBudgetTokens` 必须 `>= 1024` 且严格小于 `MaxOutputTokens`。Claude 3.7、Sonnet 4 和 Opus 4 支持此模式。
- **自适应模式（`"adaptive"`）**：基于投入程度。将 `ThinkingEffort` 设为 `"low"`、`"medium"` 或 `"high"`（部分模型还支持 `"xhigh"` / `"max"`）。Opus 4.5 及更新版本需要此模式，会拒绝经典模式。

`ThinkingMode` 是可选的。留空时客户端会自动推断：设置了 `ThinkingEffort` 表示自适应；否则 `ThinkingBudgetTokens > 0` 表示经典模式。

```go
// 经典预算 API（Claude 3.7 / Sonnet 4 / Opus 4）
llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:               os.Getenv("ANTHROPIC_API_KEY"),
    ModelName:            "claude-sonnet-4-20250514",
    ThinkingMode:         genaianthropic.ThinkingModeEnabled, // 可选；由预算值推断
    MaxOutputTokens:      16000,
    ThinkingBudgetTokens: 10000, // 必须 >= 1024 且 < MaxOutputTokens
})

// 基于投入的自适应 API（Opus 4.5+）
llmModel = genaianthropic.New(genaianthropic.Config{
    APIKey:         os.Getenv("ANTHROPIC_API_KEY"),
    ModelName:      "claude-opus-4-8",
    ThinkingMode:   genaianthropic.ThinkingModeAdaptive, // 可选；由投入程度推断
    ThinkingEffort: "high",
})
```

流式传输时，推理过程以标记了 `Thought: true` 的部分内容片段形式发出，并且思考块（及其签名）会在多轮对话中保留，以确保工具调用循环正常工作。

### 自定义 HTTP 请求头

两个客户端都支持通过 `HTTPOptions` 设置自定义 HTTP 请求头，适用于测试版功能、认证代理或提供商特定标志：

```go
import "net/http"

llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:    os.Getenv("ANTHROPIC_API_KEY"),
    ModelName: "claude-sonnet-4-6-20250929",
    HTTPOptions: genaianthropic.HTTPOptions{
        Headers: http.Header{
            "anthropic-beta": []string{"context-1m-2025-08-07"},
        },
    },
})
```

### 支持的功能

两个客户端均支持：

- 流式与非流式响应
- 系统指令
- 工具/函数调用
- 图像输入（base64）
- Temperature、TopP、MaxOutputTokens、StopSequences
- 扩展思考：经典预算 API（`ThinkingBudgetTokens`）与自适应投入 API（`ThinkingEffort` + `ThinkingMode`）
- 用量元数据
- 自定义 HTTP 请求头（多值）

## Artifact 服务（文件系统）

基于本地文件系统的版本化 Artifact 存储。Agent 可以保存、加载、列出和删除文件（代码、文档、数据），并以可下载内容的形式提供给用户。

```go
import artifactfs "github.com/achetronic/adk-utils-go/artifact/filesystem"

artifactService, _ := artifactfs.NewFilesystemService(artifactfs.FilesystemServiceConfig{
    BasePath: "data/artifacts",
})

// 配合 ADK launcher 使用
launcherCfg := &launcher.Config{
    SessionService:  sessionService,
    AgentLoader:     agentLoader,
    ArtifactService: artifactService,
}
```

Artifact 存储路径为 `{BasePath}/{appName}/{userID}/{sessionID}/{fileName}/{version}.json`。以 `user:` 为前缀的文件名按用户跨会话隔离，可从任何会话访问。

## Langfuse 插件

通过 OTLP/HTTP 追踪每一次 Agent 调用与 LLM 调用到 [Langfuse](https://langfuse.com)。为 `generate_content` 跨度（span）填充完整的请求/响应负载与 Token 用量，使 Langfuse 能够展示成本、延迟以及提示/补全内容。

支持所有 ADK Agent 拓扑：单 Agent、顺序委派、SequentialAgent、LoopAgent 和 ParallelAgent。

### 配置

```go
import "github.com/achetronic/adk-utils-go/plugin/langfuse"

pluginCfg, shutdown, err := langfuse.Setup(&langfuse.Config{
    PublicKey:   os.Getenv("LANGFUSE_PUBLIC_KEY"),
    SecretKey:   os.Getenv("LANGFUSE_SECRET_KEY"),
    Host:        "https://cloud.langfuse.com", // 或自托管 URL
    Environment: "production",
    ServiceName: "my-agent",
})
if err != nil { log.Fatal(err) }
defer shutdown(context.Background())

runnr, _ := runner.New(runner.Config{
    Agent:        myAgent,
    PluginConfig: pluginCfg,
})
```

### 与 ContextGuard 结合

```go
langfuseCfg, shutdown, _ := langfuse.Setup(langfuseCfg)
guardCfg := guard.PluginConfig()

combined := runner.PluginConfig{
    Plugins: append(langfuseCfg.Plugins, guardCfg.Plugins...),
}
```

### 按请求上下文

通过上下文注入按请求属性（通常在 HTTP 中间件中）：

```go
ctx = langfuse.WithUserID(ctx, "user-123")
ctx = langfuse.WithTags(ctx, []string{"beta", "internal"})
ctx = langfuse.WithTraceName(ctx, "customer-support")
ctx = langfuse.WithTraceMetadata(ctx, map[string]string{"tenant": "acme"})
```

### 配置项

| 字段 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `PublicKey` | 是 | — | Langfuse 项目公钥（Basic Auth 用户名） |
| `SecretKey` | 是 | — | Langfuse 项目私钥（Basic Auth 密码） |
| `Host` | 否 | `https://cloud.langfuse.com` | Langfuse 服务器地址 |
| `Environment` | 否 | — | 部署环境标签 |
| `Release` | 否 | — | 应用版本标签 |
| `ServiceName` | 否 | `langfuse-adk` | OTel `service.name` 资源属性 |
| `Insecure` | 否 | `false` | 为 OTLP/HTTP 导出器禁用 TLS（用于自托管的纯 HTTP 实例） |

当凭证缺失时，可使用 `cfg.IsEnabled()` 有条件地跳过配置。

## Context Guard 插件

自动上下文窗口管理，防止对话超出 LLM 的 Token 限制。作为 ADK `BeforeModelCallback` 插件工作 — 在每次 LLM 调用前检查对话是否接近限制，并对较早的消息进行摘要以腾出空间。

### 策略

| 策略 | 触发条件 | 适用场景 |
|----------|---------|----------|
| `threshold` | Token 数量接近上下文窗口限制 | 最大化上下文利用率，已知模型限制 |
| `sliding_window` | 轮数超过配置上限 | 可预测的压缩，长时间运行的对话 |

### 配置

插件需要 `ModelRegistry` 来查询上下文窗口大小。内置 `CrushRegistry` 从 [Crush 的 provider.json](https://raw.githubusercontent.com/charmbracelet/crush/main/internal/agent/hyper/provider.json) 获取模型元数据，每 6 小时刷新一次：

```go
import "github.com/achetronic/adk-utils-go/plugin/contextguard"

// 1. 启动注册表（内置，从 Crush 获取）
registry := contextguard.NewCrushRegistry()
registry.Start(ctx)
defer registry.Stop()

// 2. 创建 guard 并添加 Agent
guard := contextguard.New(registry)
guard.Add("assistant", llmModel)

// 3. 传递给 ADK runner
runnr, _ := runner.New(runner.Config{
    Agent:        myAgent,
    PluginConfig: guard.PluginConfig(),
})
```

可通过函数式选项设置每个 Agent 的参数：

```go
guard := contextguard.New(registry)

// 阈值策略（默认）— 当 Token 接近限制时进行摘要
guard.Add("assistant", llmModel)

// 滑动窗口 — 无论 Token 数量，在 N 轮后摘要
guard.Add("researcher", llmResearcher, contextguard.WithSlidingWindow(30))

// 手动上下文窗口覆盖 — 绕过注册表
guard.Add("writer", llmWriter, contextguard.WithMaxTokens(1_000_000))

// 自定义压缩重试限制（默认：3）— 适用于两种策略
guard.Add("analyst", llmAnalyst, contextguard.WithMaxCompactionAttempts(5))
```

多 Agent 设置使用相同的 API — 多次调用 `Add` 即可：

```go
guard := contextguard.New(registry)
for _, agentDef := range agents {
    guard.Add(agentDef.ID, llmMap[agentDef.ID], optsFromDef(agentDef)...)
}
```

### 自定义模型注册表

可以实现自己的 `ModelRegistry` 替代 `CrushRegistry`：

```go
type myRegistry struct{}

func (r *myRegistry) ContextWindow(modelID string) int {
    windows := map[string]int{
        "claude-sonnet-4-5-20250929": 200000,
        "gpt-4o":                     128000,
    }
    if w, ok := windows[modelID]; ok {
        return w
    }
    return 128000
}

func (r *myRegistry) DefaultMaxTokens(modelID string) int {
    return 4096
}

guard := contextguard.New(&myRegistry{})
guard.Add("assistant", llmModel)
```

### 工作原理

1. 每次 LLM 调用前，插件根据 Agent 的配置策略进行检查
2. **阈值**：估算总 Token 数，当剩余容量低于安全缓冲区（>200k 窗口固定 20k，较小窗口为 20%）时触发摘要
3. **滑动窗口**：统计自上次压缩以来的 Content 条目数，超过限制时触发
4. 触发时，对话被拆分为"旧的"（由 Agent 自身 LLM 摘要）和"最近的"（保留原文）
5. 两种策略都会在结果摘要仍超过阈值时重试压缩，最多 3 次（`maxCompactionAttempts`）。耗尽所有尝试后按原样发送请求（尽力而为）
6. 摘要持久化在会话状态中，并在后续请求中注入，直到下次压缩
7. 工具调用链（`tool_use` + `tool_result`）永远不会在中间断开，以避免提供商错误

## 示例

`examples/` 目录下的完整可运行示例：

| 示例 | 说明 |
| --------------------------------------------- | ------------------------------------------- |
| [openai-client](examples/openai-client) | OpenAI/Ollama 客户端用法 |
| [anthropic-client](examples/anthropic-client) | Anthropic Claude 客户端用法 |
| [context-guard](examples/context-guard) | 使用 CrushRegistry、手动阈值与滑动窗口的 ContextGuard 插件 |

### 快速开始

```bash
# 启动服务
docker run -d --name postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 pgvector/pgvector:pg16
docker run -d --name redis -p 6379:6379 redis:alpine
ollama pull qwen3:8b
ollama pull nomic-embed-text

# 运行示例
go run ./examples/openai-client
```

### 环境变量

| 变量 | 默认值 | 说明 |
| -------------------- | ---------------------------------------------------------------------- | -------------------------------------- |
| `OPENAI_API_KEY` | - | OpenAI API 密钥（Ollama 不需要） |
| `OPENAI_BASE_URL` | - | OpenAI 兼容 API 端点 |
| `ANTHROPIC_API_KEY` | - | Anthropic API 密钥 |
| `MODEL_NAME` | `gpt-4o` / `claude-sonnet-4-5-20250929` | 模型名称 |
| `EMBEDDING_BASE_URL` | `http://localhost:11434/v1` | 嵌入 API 端点 |
| `EMBEDDING_MODEL` | `nomic-embed-text` | 嵌入模型 |

## 要求

- Go 1.24+
- [Google ADK](https://google.github.io/adk-docs/) v0.5.0+

## 许可证

Apache 2.0
