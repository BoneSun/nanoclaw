# Telegram 模型强制指定修复

**日期**: 2026-03-22
**版本**: v1.0.0

## 问题

用户发送 `@Andy 测试` 后，Bot 回复显示使用 **Claude 3.5 Sonnet** 模型，但期望使用 **Claude Haiku** (`claude-haiku-4-5-20251001`)。

## 根本原因

Anthropic SDK 在使用 `ANTHROPIC_API_KEY` 模式时：
- 不读取 `CLAUDE_CODE_MODEL` 环境变量
- 不读取 `settings.json` 中的模型配置
- 默认发送 `claude-3-5-sonnet-20241022` 到 API

SDK 的 `query()` 函数的 `model` 选项也不被支持。

## 解决方案

在 **credential proxy** 层拦截 API 请求，强制修改请求体中的 `model` 参数。

### 修改文件：`src/credential-proxy.ts`

#### 1. 添加 `.env` 文件读取函数

```typescript
function readFullEnvFile(): Record<string, string> {
  const envPath = path.join(process.cwd(), '.env');
  const env: Record<string, string> = {};
  try {
    const content = fs.readFileSync(envPath, 'utf-8');
    for (const line of content.split('\n')) {
      const match = line.match(/^([A-Z_]+)=(.*)$/);
      if (match) {
        env[match[1]] = match[2];
      }
    }
  } catch (err) {
    logger.warn({ err }, 'Failed to read .env file');
  }
  return env;
}
```

#### 2. 在代理启动时读取模型设置

```typescript
const fullEnv = readFullEnvFile();
const modelOverride = fullEnv.ANTHROPIC_SMALL_FAST_MODEL || fullEnv.CLAUDE_CODE_MODEL;
if (modelOverride) {
  logger.info({ model: modelOverride }, 'Using model from ANTHROPIC_SMALL_FAST_MODEL/CLAUDE_CODE_MODEL');
}
```

#### 3. 在请求处理中强制修改 model 参数

```typescript
if (modelOverride && (urlPath.includes('/v1/messages') || urlPath.includes('/v1/chat/completions'))) {
  try {
    const parsedBody = JSON.parse(body.toString());
    logger.info({ originalModel: parsedBody.model, newModel: modelOverride }, 'Applying model override');
    parsedBody.model = modelOverride;
    finalBody = Buffer.from(JSON.stringify(parsedBody));
    headers['content-length'] = finalBody.length;
    logger.info({ model: parsedBody.model }, 'Forced model in proxy via ANTHROPIC_SMALL_FAST_MODEL');
  } catch (err) {
    logger.warn({ err }, 'Failed to parse request body for model override');
  }
}
```

## 验证日志

```
[19:35:06.300] Using model from ANTHROPIC_SMALL_FAST_MODEL/CLAUDE_CODE_MODEL: claude-haiku-4-5-20251001
[19:37:09.257] Credential proxy request
[19:37:09.259] Checking model override
[19:37:09.261] Applying model override
[19:37:09.263] Forced model in proxy via ANTHROPIC_SMALL_FAST_MODEL
```

## 配置文件

`.env` 文件需要包含：

```bash
ANTHROPIC_API_KEY=sk-...
ANTHROPIC_BASE_URL=http://183.36.37.124:4000
ANTHROPIC_SMALL_FAST_MODEL=claude-haiku-4-5-20251001
```

或

```bash
CLAUDE_CODE_MODEL=claude-haiku-4-5-20251001
```

## 经验教训

1. **SDK 限制**: Anthropic SDK 对模型配置支持有限，环境变量不被读取
2. **代理层修改**: 在 proxy 层修改请求是最可靠的方式
3. **支持自建模型**: 此方案对自建模型 API 同样有效
4. **日志追踪**: 添加详细日志便于调试模型覆盖逻辑

## 相关文件

- `src/credential-proxy.ts` - 核心修改
- `.env` - 模型配置
- `container/agent-runner/src/index.ts` - SDK 调用（已移除无效的 `model` 选项）
