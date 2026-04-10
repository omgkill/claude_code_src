# Claude Code CLI 代码结构分析指南

## 一、代码整体架构

```
/src
├── entrypoints/        # CLI 入口点
│   └── cli.tsx        # 主入口文件 (~300行)
├── main.tsx           # 主应用入口 (大型文件)
├── services/api/     # API 服务层
│   ├── client.ts      # API 客户端工厂 (~390行)
│   ├── claude.ts     # 核心查询逻辑 (~3400行)
│   └── ...
├── utils/             # 工具函数
│   ├── auth.ts       # 认证管理 (~2000行)
│   ├── cliArgs.ts    # CLI 参数解析
│   └── ...
├── commands/          # CLI 命令实现
│   ├── login/
│   ├── config/
│   └── ...
└── tools/             # 各种工具实现
```

## 二、核心模块解读

### 2.1 CLI 入口点 (cli.tsx)

**设计模式：快速路径优化**

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // 快速路径1: --version 无需加载任何模块
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // 其他路径使用动态 import 延迟加载
  const { main: cliMain } = await import('../main.js');
  await cliMain();
}
```

**关键设计**：
- 使用**动态 import** 减少启动时间
- **快速路径**处理常见标志（如 `--version`）
- 支持**特殊子命令**（daemon、bridge、bg sessions 等）

### 2.2 API 客户端工厂 (client.ts)

**多后端支持架构**：

```typescript
export async function getAnthropicClient({...}): Promise<Anthropic> {
  // 支持多种 API 提供商
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    // AWS Bedrock
    const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk');
    return new AnthropicBedrock({ awsRegion, ... });
  }

  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    // Google Vertex AI
    const { AnthropicVertex } = await import('@anthropic-ai/vertex-sdk');
    return new AnthropicVertex({ region, googleAuth, ... });
  }

  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
    // Azure Foundry
    const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk');
    return new AnthropicFoundry({ azureADTokenProvider, ... });
  }

  // 默认: Direct Anthropic API
  return new Anthropic({ apiKey, authToken, ... });
}
```

**认证方案**：
- **直接 API Key**: `ANTHROPIC_API_KEY` 环境变量
- **OAuth Token**: Claude.ai 订阅者使用
- **第三方服务认证**: AWS/ GCP/ Azure 原生认证

### 2.3 核心查询流程 (claude.ts)

**主要函数**：
1. `queryModel()` - 核心生成器，处理流式响应
2. `queryModelWithoutStreaming()` - 非流式版本
3. `queryHaiku()` - 轻量级 Haiku 模型查询
4. `queryWithModel()` - 指定模型查询

**请求构建流程**：

```typescript
async function* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options) {
  // 1. Beta 功能检查
  const betas = getMergedBetas(options.model, { isAgenticQuery });

  // 2. 工具 Schema 构建
  const toolSchemas = await Promise.all(
    filteredTools.map(tool => toolToAPISchema(tool, {...}))
  );

  // 3. 消息标准化
  let messagesForAPI = normalizeMessagesForAPI(messages, filteredTools);

  // 4. 系统提示构建
  const system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {...});

  // 5. 参数构建
  const params = {
    model: normalizeModelStringForAPI(options.model),
    messages: addCacheBreakpoints(messagesForAPI, ...),
    system,
    tools: allTools,
    betas: betasParams,
    metadata: getAPIMetadata(),
    max_tokens: maxOutputTokens,
    thinking,
    ...
  };

  // 6. 流式 API 调用
  const result = await anthropic.beta.messages.create({...}, { signal });

  // 7. 响应处理
  for await (const part of stream) {
    switch (part.type) {
      case 'message_start': ...
      case 'content_block_start': ...
      case 'content_block_delta': ...
      case 'content_block_stop': ...
      case 'message_delta': ...
    }
  }
}
```

## 三、API 调用流程梳理

```
用户输入
    ↓
cli.tsx 入口解析
    ↓
main.tsx 初始化
    ↓
query.ts 协调器
    ↓
claude.ts queryModel()
    ├── 认证检查 (auth.ts)
    ├── Beta 功能配置
    ├── 工具 Schema 构建
    ├── 消息标准化
    ├── 系统提示构建
    ├── API 参数构建
    └── 流式请求发送
    ↓
API 响应处理
    ├── 流式事件处理
    ├── 工具调用执行
    └── 结果返回
```

## 四、关键配置参数

**环境变量配置**：

```typescript
// API 配置
ANTHROPIC_API_KEY          // 直接 API 密钥
ANTHROPIC_AUTH_TOKEN       // OAuth 访问令牌
API_TIMEOUT_MS             // API 超时 (默认 600秒)

// 模型配置
CLAUDE_CODE_MAX_OUTPUT_TOKENS  // 最大输出 tokens
ANTHROPIC_VERTEX_PROJECT_ID   // GCP 项目 ID

// 功能开关
CLAUDE_CODE_USE_BEDROCK       // 使用 AWS Bedrock
CLAUDE_CODE_USE_VERTEX        // 使用 Google Vertex
CLAUDE_CODE_USE_FOUNDRY       // 使用 Azure Foundry
DISABLE_PROMPT_CACHING        // 禁用提示缓存
CLAUDE_CODE_DISABLE_THINKING  // 禁用思考模式
```

## 五、错误处理机制

**重试机制 (withRetry.ts)**：

```typescript
async function* withRetry(
  getClient,        // 获取 API 客户端
  attemptHandler,   // 处理每次尝试
  retryOptions      // 重试配置
) {
  let attempt = 0;
  while (true) {
    try {
      const result = await attemptHandler(anthropic, attempt, context);
      return result;
    } catch (error) {
      if (!isRetryable(error) || attempt >= maxRetries) {
        throw new CannotRetryError(error);
      }
      attempt++;
      await sleep(calculateBackoff(attempt));
    }
  }
}
```

**错误类型**：
- `APIConnectionTimeoutError` - 连接超时
- `APIUserAbortError` - 用户中止
- `CannotRetryError` - 不可重试错误
- `FallbackTriggeredError` - 触发回退

## 六、实际应用示例

**如何调用 Haiku 模型进行简单查询**：

```typescript
import { queryHaiku } from 'src/services/api/claude';

const result = await queryHaiku({
  systemPrompt: ['You are a helpful assistant.'],
  userPrompt: 'What is 2+2?',
  signal: AbortSignal.timeout(30000),
  options: {
    model: 'claude-3-haiku-20240307',
    isNonInteractiveSession: false,
    querySource: 'sdk',
    agents: [],
    mcpTools: [],
    // ...
  }
});
```

**如何扩展新工具**：
```typescript
// 1. 在 tools/ 目录创建新工具
// 2. 实现 toolToAPISchema 转换
// 3. 在 query.ts 中注册工具
// 4. 添加权限检查逻辑
```

## 七、学习建议

| 优先级 | 主题 | 关键文件 |
|--------|------|----------|
| ⭐⭐⭐ | API 客户端工厂 | `client.ts` |
| ⭐⭐⭐ | 核心查询逻辑 | `claude.ts` |
| ⭐⭐⭐ | 认证管理 | `auth.ts` |
| ⭐⭐ | CLI 入口解析 | `cli.tsx` |
| ⭐⭐ | 消息处理 | `utils/messages.ts` |
| ⭐ | 工具系统 | `tools/*.ts` |
| ⭐ | 重试机制 | `withRetry.ts` |

---

这个代码库展示了许多**生产级设计模式**：快速路径优化、动态加载、SWR 缓存策略、多提供商抽象、完整的错误处理和重试机制。建议按照上述优先级逐步深入学习。
