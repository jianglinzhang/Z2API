# Z2API

一个为 Z.AI API 提供 OpenAI 兼容接口的代理服务器，支持 cookie 池管理、智能内容过滤和灵活的响应模式控制。

> **💡 核心特性：** 支持流式和非流式两种响应模式，非流式模式下可选择性隐藏 AI 思考过程，提供更简洁的 API 响应。

## ⚠️ 已知问题

### 🌊 流式传输失败

**问题描述：**
目前流式传输功能存在稳定性问题，可能导致以下情况：

- 流式响应中断或失败
- 连接超时
- 响应内容不完整

**影响范围：**

- 所有使用 `stream=true` 的请求
- 实时交互场景

**解决方案：**

1. **使用非流式模式**（推荐）：

   ```python
   response = client.chat.completions.create(
       model="GLM-4.5",
       messages=[{"role": "user", "content": "你的问题"}],
       stream=False  # 明确设置为非流式
   )
   ```

2. **环境变量配置**：

   ```env
   DEFAULT_STREAM=false  # 确保默认使用非流式模式
   ```

3. **客户端配置**：
   - 在 OpenAI SDK 中不设置 `stream` 参数或设置为 `false`
   - 使用 cURL 时移除 `stream: true` 参数

**临时修复：**
如果必须使用流式传输，建议：

- 减少请求内容长度
- 增加超时时间设置
- 实现重试机制

**修复进度：**

- 🔴 问题已确认
- 🟡 正在调查原因
- 🟢 预计下个版本修复

---

### 🔄 其他已知问题

#### Request Error 错误

- **现象**：日志中出现 "Request error" 错误
- **原因**：网络连接问题或 Cookie 失效
- **解决方案**：使用 `debug_connection.py` 脚本诊断连接问题

#### Cookie 健康检查

- **现象**：Cookie 可能被标记为失效但实际可用
- **原因**：Z.AI API 的临时性认证问题
- **解决方案**：系统会自动尝试恢复失效的 Cookie

---

**提示：** 如遇到其他问题，请查看项目 Issues 或使用 `debug_connection.py` 进行诊断。

## ✨ 特性

- 🔌 **OpenAI SDK 完全兼容** - 无缝替换 OpenAI API
- 🍪 **智能 Cookie 池管理** - 多 token 轮换，自动故障转移
- 🧠 **智能内容过滤** - 非流式响应可选择隐藏 AI 思考过程
- 🌊 **灵活响应模式** - 支持流式和非流式响应，可配置默认模式
- 🛡️ **安全认证** - 固定 API Key 验证
- 📊 **健康检查** - 自动监控和恢复
- 📝 **详细日志** - 完善的调试和监控信息

## 🚀 快速开始

### 环境要求

- Python 3.8+
- pip

### 安装步骤

1. **克隆项目**

```bash
git clone https://github.com/LargeCupPanda/Z2API.git
cd Z2API
```

2. **安装依赖**

```bash
pip install -r requirements.txt
```

3. **配置环境变量**

```bash
cp .env.example .env
# 编辑 .env 文件，配置你的参数
```

4. **启动服务器**

```bash
python main.py
```

服务器将在 `http://localhost:8000` 启动

## ⚙️ 配置说明

在 `.env` 文件中配置以下参数：

```env
# 服务器设置
HOST=0.0.0.0
PORT=8000

# API Key (用于外部认证)
API_KEY=sk-z2api-key-2024

# 内容过滤设置 (仅适用于非流式响应)
# 是否显示AI思考过程 (true/false)
SHOW_THINK_TAGS=false

# 响应模式设置
# 默认是否使用流式响应 (true/false)
DEFAULT_STREAM=false

# Z.AI Token配置
# 从 https://chat.z.ai 获取的JWT token (不包含"Bearer "前缀),多个用`,`分隔,比如：token1,token2
Z_AI_COOKIES=eyJ9...

# 速率限制
MAX_REQUESTS_PER_MINUTE=60

# 日志级别 (DEBUG, INFO, WARNING, ERROR)
LOG_LEVEL=INFO
```

### 🔑 获取 Z.AI Token

1. 访问 [https://chat.z.ai](https://chat.z.ai) 并登录
2. 打开浏览器开发者工具 (F12)
3. 切换到 **Network** 标签
4. 发送一条消息给 AI
5. 找到对 `chat/completions` 的请求
6. 复制请求头中 `Authorization: Bearer xxx` 的 token 部分
7. 将 token 值（不包括"Bearer "前缀）配置到 `Z_AI_COOKIES`

## 📖 使用方法

### OpenAI SDK (推荐)

```python
import openai

# 配置客户端
client = openai.OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="sk-z2api-key-2024"  # 使用配置的API Key
)

# 发送请求
response = client.chat.completions.create(
    model="GLM-4.5",  # 固定模型名称
    messages=[
        {"role": "user", "content": "你好，请介绍一下自己"}
    ],
    max_tokens=1000,
    temperature=0.7
)

print(response.choices[0].message.content)
```

### cURL

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-z2api-key-2024" \
  -d '{
    "model": "GLM-4.5",
    "messages": [
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "max_tokens": 500
  }'
```

### 不同响应模式示例

#### 非流式响应（默认，支持思考内容过滤）

```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="sk-z2api-key-2024"
)

# 非流式响应，会根据SHOW_THINK_TAGS设置过滤内容
response = client.chat.completions.create(
    model="GLM-4.5",
    messages=[{"role": "user", "content": "解释一下量子计算"}],
    stream=False  # 或者不设置此参数（使用DEFAULT_STREAM默认值）
)

print(response.choices[0].message.content)
```

#### 流式响应（包含完整内容）

```python
# 流式响应，始终包含完整内容（忽略SHOW_THINK_TAGS设置）
stream = client.chat.completions.create(
    model="GLM-4.5",
    messages=[{"role": "user", "content": "写一首关于春天的诗"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content is not None:
        print(chunk.choices[0].delta.content, end="")
```

## 🎛️ 高级配置

### 响应模式控制

系统支持两种响应模式，通过以下参数控制：

```env
# 默认响应模式 (推荐设置为false，即非流式)
DEFAULT_STREAM=false

# 思考内容过滤 (仅对非流式响应生效)
SHOW_THINK_TAGS=false
```

**响应模式说明：**

| 模式       | 参数设置              | 思考内容过滤              | 适用场景           |
| ---------- | --------------------- | ------------------------- | ------------------ |
| **非流式** | `stream=false` 或默认 | ✅ 支持 `SHOW_THINK_TAGS` | 简洁回答，API 集成 |
| **流式**   | `stream=true`         | ❌ 忽略 `SHOW_THINK_TAGS` | 实时交互，聊天界面 |

**效果对比：**

- **非流式 + `SHOW_THINK_TAGS=false`**: 只返回答案（~80 字符），简洁明了
- **非流式 + `SHOW_THINK_TAGS=true`**: 完整内容（~1300 字符），包含思考过程
- **流式响应**: 始终包含完整内容，实时输出

**推荐配置：**

```env
# 推荐配置：默认非流式，隐藏思考过程
DEFAULT_STREAM=false
SHOW_THINK_TAGS=false
```

这样配置可以：

- 提供简洁的 API 响应（适合大多数应用场景）
- 需要完整内容时可通过 `stream=true` 获取
- 需要思考过程时可通过 `SHOW_THINK_TAGS=true` 开启

### Cookie 池管理

支持配置多个 token 以提高并发性和可靠性：

```env
# 单个token
Z_AI_COOKIES=token1

# 多个token (逗号分隔)
Z_AI_COOKIES=token1,token2,token3
```

系统会自动：

- 轮换使用不同的 token
- 检测失效的 token 并自动切换
- 定期进行健康检查和恢复

## 🔍 API 端点

| 端点                   | 方法 | 描述                       |
| ---------------------- | ---- | -------------------------- |
| `/v1/chat/completions` | POST | 聊天完成接口 (OpenAI 兼容) |
| `/health`              | GET  | 健康检查                   |
| `/`                    | GET  | 服务状态                   |

## 🧪 测试

### 基本测试

```bash
# 运行示例测试
python example_usage.py

# 测试健康检查
curl http://localhost:8000/health
```

### API 测试

```bash
# 测试非流式响应
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-z2api-key-2024" \
  -d '{
    "model": "GLM-4.5",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": false
  }'

# 测试流式响应
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-z2api-key-2024" \
  -d '{
    "model": "GLM-4.5",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

## 📊 监控和日志

### 日志级别

```env
LOG_LEVEL=DEBUG  # 详细调试信息
LOG_LEVEL=INFO   # 一般信息 (推荐)
LOG_LEVEL=WARNING # 警告信息
LOG_LEVEL=ERROR  # 仅错误信息
```

### 健康检查

访问 `http://localhost:8000/health` 查看服务状态：

```json
{
  "status": "healthy",
  "timestamp": "2025-08-04T17:30:00Z",
  "version": "1.0.0"
}
```

## 🔧 故障排除

### 常见问题

1. **401 Unauthorized**

   - 检查 API Key 是否正确配置
   - 确认使用的是 `sk-z2api-key-2024`

2. **Token 失效**

   - 重新从 Z.AI 网站获取新的 token
   - 更新 `.env` 文件中的 `Z_AI_COOKIES`

3. **连接超时**

   - 检查网络连接
   - 确认 Z.AI 服务可访问

4. **内容为空或不符合预期**

   - 检查 `SHOW_THINK_TAGS` 和 `DEFAULT_STREAM` 设置
   - 确认响应模式（流式 vs 非流式）
   - 查看服务器日志获取详细信息

5. **思考内容过滤不生效**

   - 确认使用的是非流式响应（`stream=false`）
   - 流式响应会忽略 `SHOW_THINK_TAGS` 设置

6. **服务启动失败**
   - 检查端口是否被占用：`netstat -tlnp | grep :8000`
   - 查看详细错误：直接运行 `python main.py`
   - 检查依赖是否安装：`pip list | grep fastapi`

### 调试模式

```bash
# 启用详细日志
export LOG_LEVEL=DEBUG
python main.py

# 或者直接在.env文件中设置
echo "LOG_LEVEL=DEBUG" >> .env
```

## 📋 配置参数

| 参数              | 描述            | 默认值              | 必需 |
| ----------------- | --------------- | ------------------- | ---- |
| `HOST`            | 服务器监听地址  | `0.0.0.0`           | 否   |
| `PORT`            | 服务器端口      | `8000`              | 否   |
| `API_KEY`         | 外部认证密钥    | `sk-z2api-key-2024` | 否   |
| `SHOW_THINK_TAGS` | 显示思考内容    | `false`             | 否   |
| `DEFAULT_STREAM`  | 默认流式模式    | `false`             | 否   |
| `Z_AI_COOKIES`    | Z.AI JWT tokens | -                   | 是   |
| `LOG_LEVEL`       | 日志级别        | `INFO`              | 否   |

## 🛠️ 服务管理

### 基本操作

```bash
# 启动服务（前台运行）
python main.py

# 后台运行
nohup python main.py > z2api.log 2>&1 &

# 查看日志
tail -f z2api.log

# 停止服务
# 找到进程ID并终止
ps aux | grep "python main.py"
kill <PID>
```

## 🤝 贡献

**特别说明：** 作者为非编程人士，此项目全程由 AI 开发，AI 代码 100%，人类代码 0%。由于这种开发模式，更新维护起来非常费劲，所以特别欢迎大家提交 Issue 和 Pull Request 来帮助改进项目！

## 📄 许可证

MIT License
