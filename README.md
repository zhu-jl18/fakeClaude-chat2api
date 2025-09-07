# TalkAI OpenAI API 适配器

这是一个将 TalkAI 的 API 格式转换为 OpenAI ChatCompletion API 格式的适配器。

⚠️ **重要提醒**：本项目使用的是公共分享的 TalkAI API 密钥，可能随时失效。如果遇到认证错误，请获取新的密钥并按照下方指南更新。

## 功能

- 兼容 OpenAI Chat API (`/v1/chat/completions`) 和模型列表 API (`/v1/models`)。
- 支持流式 (streaming) 和非流式响应。
- 通过 API 密钥进行简单的 Bearer Token 认证。
- 优先从环境变量 `PASSWORD` 读取服务认证密钥；如果环境变量不存在，则回退到 `client_api_keys.json` 文件以便本地开发。
- 支持处理多部分内容（`multipart content`），兼容新版客户端格式。
- **详细的错误处理**：区分网络错误、认证错误、API密钥失效等不同情况，提供清晰的错误信息。

## 部署到 Render（推荐）

Render 更适合长期运行的 Python/uvicorn 服务，部署简单且稳定。

1. 在 Render 注册并登录（使用 GitHub 登录）。
2. 点击 "New +" → "Web Service"，连接您的 GitHub 仓库并选择本项目。
3. 配置服务：
   - Name: 选一个名字（例如 `talkai-adapter`）。
   - Region: 选择离您近的区域。
   - Branch: 选择 `main`。
   - Runtime: 选择 `Python 3`。
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. 在 Environment Variables 中添加：
   - Key: `PASSWORD`
   - Value: `<YOUR_SERVICE_SECRET_KEY>`（用于保护您部署的服务，请使用一个安全的密钥）
5. 创建服务，Render 会自动构建并部署您的应用。

部署完成后，Render 会给出服务地址（例如 `https://your-service-name.onrender.com`）。

## 调用示例

把 `<YOUR_DEPLOYED_URL>` 替换为您在 Render 上获得的地址：

```bash
curl <YOUR_DEPLOYED_URL>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_SERVICE_SECRET_KEY>" \
  -d '{
    "model": "Claude Sonnet 4.1",
    "messages": [
      {"role": "user", "content": "你好，请介绍一下你自己"}
    ]
  }'
```

## 下游服务密钥（TalkAI）

程序会从 `client_api_keys.json` 中读取用于请求 `claude.talkai.info` 的密钥（字段示例：`["sk-talkai-..."]`）。

**密钥管理最佳实践**：
- 本地开发：可以直接编辑 `client_api_keys.json`
- 生产环境：确保使用有效的密钥，定期检查密钥状态
- 如果遇到 401/403 错误，通常表示密钥已失效，需要获取新密钥

本地测试：保留或编辑 `client_api_keys.json`；生产环境请不要提交真实密钥到仓库，Render 的 `PASSWORD` 环境变量用于保护对外请求。

## 错误处理说明

当下游 TalkAI API 出现问题时，本服务会返回详细的错误信息：

- **401 错误**："TalkAI API authentication failed - API key may be invalid or expired"
- **403 错误**："TalkAI API access forbidden - API key may lack permissions"
- **429 错误**："TalkAI API rate limit exceeded - please try again later"
- **5xx 错误**："TalkAI API server error - downstream service may be temporarily unavailable"
- **网络超时**："Connection timeout to TalkAI API - network issue or service unavailable"
- **连接失败**："Failed to connect to TalkAI API - network connectivity issue"

## 更新 TalkAI API 密钥

### 本地开发环境

1. 编辑 `client_api_keys.json` 文件：
```json
["sk-talkai-your-new-key-here"]
```

2. 重启服务即可生效。

### Render 生产环境

**注意**：生产环境的 TalkAI API 密钥存储在 `client_api_keys.json` 文件中，该文件已包含在代码仓库中。

1. 更新仓库中的 `client_api_keys.json` 文件
2. 提交并推送到 GitHub
3. Render 会自动检测到更改并重新部署

或者通过 Render 控制台：
1. 登录 Render 控制台
2. 找到你的服务
3. 点击 "Manual Deploy" → "Deploy latest commit" 来强制重新部署

## 为什么不使用 Vercel？

Vercel 主要针对静态网站和 Serverless Functions 优化，不太适合长时间运行的 Python/FastAPI 应用：

1. **执行时间限制**：Vercel 的 Serverless Functions 有执行时间限制（免费版10秒，付费版60秒），而 AI 对话可能需要更长时间。

2. **冷启动问题**：每次请求都可能触发冷启动，导致响应延迟。

3. **Python 支持限制**：Vercel 对 Python 的支持不如对 Node.js 的支持完善。

4. **依赖管理**：复杂的 Python 依赖（如 FastAPI + uvicorn）在 Vercel 上可能出现兼容性问题。

5. **流式响应**：Vercel 的 Serverless 架构对流式响应的支持有限制。

**Render 的优势**：
- 原生支持长时间运行的 Python 应用
- 更好的 FastAPI/uvicorn 兼容性
- 支持流式响应
- 更适合 API 服务的架构
