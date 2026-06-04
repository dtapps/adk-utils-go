<img src="docs/images/header.svg" alt="adk-utils-go" width="600">

# ADK Utils Go

针对 Google 智能体开发工具包 [Google's Agent Development Kit (ADK)](https://google.github.io/adk-docs/) 的 Go 语言扩展工具包。

本项目专为轻量化、免 CGO 的常驻守护进程设计，剥离了沉重的外部数据库依赖，提供以下生产环境就绪的核心组件：

- **大模型客户端（LLM Clients）**：完美兼容 Google ADK 规范的 OpenAI 与 Anthropic (Claude) 原生驱动。
- **产物存储（Artifact Storage）**：基于本地文件系统的版本化、用户域隔离的产物持久化管理。
- **上下文守护插件（Context Guard）**：基于大模型自动摘要的上下文窗口管理插件，防止长对话超出 Token 限制。
- **Langfuse 观测插件**：通过 OTLP/HTTP 协议将每一轮 LLM 调用的 Prompt、Response 及 Token 消耗完整上报至 [Langfuse](https://langfuse.com) 的观测插件。

## 目录结构


```

├── genai/            # 大模型客户端实现
│   ├── openai/       # OpenAI 客户端（完美适配 Ollama, OpenRouter, DeepSeek 等）
│   └── anthropic/    # Anthropic Claude 原生客户端
├── artifact/         # 产物服务实现
│   └── filesystem/   # 基于本地文件系统的版本化产物服务
├── plugin/           # ADK 插件实现
│   ├── contextguard/ # 上下文窗口管理与自动压缩插件
│   └── langfuse/     # Langfuse 链路追踪观测插件
└── examples/         # 运行示例

```

## 安装

```bash
go get github.com/achetronic/adk-utils-go

go mod edit -replace github.com/achetronic/adk-utils-go=github.com/dtapps/adk-utils-go@master

go mod tidy
```

---

## 大模型客户端（LLM Clients）

### OpenAI 客户端

支持 OpenAI 官方 API 以及任何兼容 OpenAI 格式的端点（如 Ollama、OpenRouter、DeepSeek 等）：

```go
import genaiopenai "github.com/achetronic/adk-utils-go/genai/openai"

llmModel := genaiopenai.New(genaiopenai.Config{
    APIKey:    os.Getenv("OPENAI_API_KEY"),
    BaseURL:   "https://api.deepseek.com/v1", // 可自由更换为 DeepSeek 或本地 Ollama 端点
    ModelName: "deepseek-chat",               
})

agent, _ := llmagent.New(llmagent.Config{
    Name:  "assistant",
    Model: llmModel,
})

```

### Anthropic 客户端

原生支持 Anthropic Claude 模型：

```go
import genaianthropic "github.com/achetronic/adk-utils-go/genai/anthropic"

llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:    os.Getenv("ANTHROPIC_API_KEY"),
    ModelName: "claude-3-5-sonnet-latest",
})

agent, _ := llmagent.New(llmagent.Config{
    Name:  "assistant",
    Model: llmModel,
})

```

#### 深度思考（Extended Thinking）支持

Claude 支持在输出最终答复前生成内部推理链。通过设置 `ThinkingBudgetTokens` 可以为推理链分配合理的 Token 预算：

```go
llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:               os.Getenv("ANTHROPIC_API_KEY"),
    ModelName:            "claude-3-5-sonnet-latest",
    MaxOutputTokens:      16000,
    ThinkingBudgetTokens: 10000, // 必须 >= 1024 且小于 MaxOutputTokens
})

```

### 自定义 HTTP 请求头（Custom HTTP Headers）

两个客户端均支持通过 `HTTPOptions` 注入自定义 HTTP Header，便于对接特定的认证网关、代理或 Beta 特性：

```go
import "net/http"

llmModel := genaianthropic.New(genaianthropic.Config{
    APIKey:    os.Getenv("ANTHROPIC_API_KEY"),
    ModelName: "claude-3-5-sonnet-latest",
    HTTPOptions: genaianthropic.HTTPOptions{
        Headers: http.Header{
            "X-Custom-Proxy-Auth": []string{"your-token"},
        },
    },
})

```

### 支持的核心特性

OpenAI 与 Anthropic 驱动全面支持：

* 流式响应（Streaming）与非流式响应
* 系统指令（System Instructions / System Prompt）注入
* 工具调用 / 函数调用（Tool/Function Calling）
* 多模态图片输入（Base64 格式）
* 完整的模型控制参数（Temperature, TopP, MaxOutputTokens, StopSequences）
* 深度思考元数据与 Usage 元数据解析

---

## 本地文件系统产物服务（Artifact Service）

提供基于本地文件系统的版本化产物存储。智能体可以安全地保存、读取、列出和删除文件（如生成的代码、资产、数据），这些文件作为可分发内容交付给最终用户。

```go
import artifactfs "github.com/achetronic/adk-utils-go/artifact/filesystem"

artifactService, _ := artifactfs.NewFilesystemService(artifactfs.FilesystemServiceConfig{
    BasePath: "data/artifacts", // 本地持久化资产的根路径
})

```

产物会安全地隔离存储在 `{BasePath}/{appName}/{userID}/{sessionID}/{fileName}/{version}.json` 路径下。前缀为 `user:` 的文件名具有全局用户域作用域，允许跨不同的会话进行读取。

---

## Langfuse 观测插件

通过 OTLP/HTTP 标准协议，将每一次 Agent 编排流转和底层 LLM 调用链路完整追踪并同步至 [Langfuse](https://langfuse.com)。可以捕获完整的输入输出 Payload 报文、耗时以及 Token 实际消耗量。

支持单智能体、顺序委托、LoopAgent 及 ParallelAgent 等全部 ADK 拓扑架构。

### 快速接入

```go
import langfuse "github.com/achetronic/adk-utils-go/plugin/langfuse"

pluginCfg, shutdown, err := langfuse.Setup(&langfuse.Config{
    PublicKey:   os.Getenv("LANGFUSE_PUBLIC_KEY"),
    SecretKey:   os.Getenv("LANGFUSE_SECRET_KEY"),
    Host:        "https://cloud.langfuse.com", // 支持自建集群端点
    Environment: "production",
    ServiceName: "hubcore-daemon",
})
if err != nil { log.Fatal(err) }
defer shutdown(context.Background())

runnr, _ := runner.New(runner.Config{
    Agent:        myAgent,
    PluginConfig: pluginCfg,
})

```

---

## 上下文守护插件（Context Guard）

自动化的上下文窗口管理插件，防止长对话历史超出 LLM 的最大 Token 限制。作为 ADK 的 `BeforeModelCallback` 拦截器工作——在每次投递给大模型推理之前，它会评估当前上下文，并调用大模型自动对久远的历史消息进行摘要压缩，腾出上下文空间。

### 核心压缩策略

| 策略名称 | 触发机制 | 适用场景 |
| --- | --- | --- |
| `threshold` | Token 总计数逼近上下文窗口极限（默认触发缓冲） | 最大化利用上下文、适用于已知严格限制的模型 |
| `sliding_window` | 会话轮数（Turn Count）超出了配置的最大轮数 | 可预测的确定性压缩、超长对话生命周期 |

### 快速接入

插件需要一个 `ModelRegistry` 来检索各模型的上下文窗口配置。内置了功能强大的 `CrushRegistry`，它会自动从远程定期同步各大主流模型的最新元数据，并进行内存缓存：

```go
import contextguard "github.com/achetronic/adk-utils-go/plugin/contextguard"

// 1. 启动模型元数据注册中心
registry := contextguard.NewCrushRegistry()
registry.Start(ctx)
defer registry.Stop()

// 2. 创建上下文守护并添加你的冷装配智能体
guard := contextguard.New(registry)
guard.Add("assistant", llmModel) // 默认采用 threshold 动态按 Token 比例压缩

// 也可以配置固定轮数的滑动窗口策略
// guard.Add("researcher", llmModel, contextguard.WithSlidingWindow(30))

// 3. 喂给 ADK 执行器
runnr, _ := runner.New(runner.Config{
    Agent:        myAgent,
    PluginConfig: guard.PluginConfig(),
})

```

---

## 依赖环境要求

* Go 1.24+
* [Google ADK](https://google.github.io/adk-docs/) v0.5.0+

## 开源协议

本项目采用 **Apache 2.0** 开源许可证，基于原开源项目 `achetronic/adk-utils-go` 进行裁剪与定制。
