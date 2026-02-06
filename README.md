# Grok2API-rs

> 本项目基于 [grok2api](https://github.com/chenyme/grok2api) 重构。

> [!NOTE]
> 本项目仅供学习与研究，使用者必须在遵循 Grok 使用条款及当地法律法规的前提下使用，不得用于非法用途。

## 项目定位

- 使用 Rust + Axum 重写后端，提供 OpenAI 兼容接口与管理后台。
- 静态资源内置到二进制，支持单文件部署。
- 所有 Grok 上游请求统一走内置 `wreq` 链路，无需外部 `curl-impersonate`。
- 支持 Docker 快捷部署与 GHCR 自动发布。

系统首页截图：  
![系统首页截图](docs/images/1image.png)

## 本次迭代重点

- NSFW 开启链路增强：支持专用指纹、失败重试与回退策略，失败详情可在后台直接查看。
- 新增管理后台「对话」页面：支持 Chat / Responses / Images / Images NSFW 四接口联调。
- 对话页面支持流式展示、Markdown 渲染、图片（URL/Base64）展示和自适应网格布局。
- 新增「下游管理」页面：按接口维度启停对外 API。

下游管理截图：  
![下游管理截图](docs/images/2image.png)

## 下游接口列表（OpenAI 兼容）

| 接口 | 路径 | 开关项 |
| --- | --- | --- |
| Chat Completions | `/v1/chat/completions` | `downstream.enable_chat_completions` |
| Responses API | `/v1/responses` | `downstream.enable_responses` |
| Images Generations | `/v1/images/generations` | `downstream.enable_images` |
| Images NSFW | `/v1/images/generations/nsfw` | `downstream.enable_images_nsfw` |
| Models | `/v1/models` | `downstream.enable_models` |
| Files | `/v1/files` | `downstream.enable_files` |

> 后台路径：`/admin`（Token 管理 / 配置管理 / 缓存管理 / 下游管理 / 对话）。

## 安装与部署

### 1) 单文件部署（命令行）

```bash
# 1. 准备目录
mkdir -p grok2api-rs/data

# 2. 配置文件
cp config.defaults.toml grok2api-rs/data/config.toml

# 3. Token 池
cp /path/to/token.json grok2api-rs/data/token.json

# 4. 启动
chmod +x grok2api-rs/grok2api-rs
SERVER_HOST=0.0.0.0 SERVER_PORT=8000 ./grok2api-rs/grok2api-rs
```

单文件部署目录参考：

```text
/grok2api-rs
├─ grok2api-rs
└─ data
   ├─ config.toml
   └─ token.json
```

系统部署执行截图：  
![系统部署执行截图](docs/images/7image.png)

### 2) Docker 快捷部署（推荐）

```bash
# 1. 拉取项目
git clone https://github.com/XeanYu/grok2api-rs.git
cd grok2api-rs

# 2. 准备数据目录
mkdir -p data
cp config.defaults.toml data/config.toml

# 3. 准备 token 池（至少 1 个可用 ssoBasic）
cat > data/token.json <<'JSON'
{
  "ssoBasic": []
}
JSON

# 4. 启动（默认镜像：ghcr.io/xeanyu/grok2api-rs:latest）
docker compose pull
docker compose up -d

# 5. 查看日志
docker compose logs -f
```

本地构建镜像后再启动：

```bash
docker build -t grok2api-rs:local .
IMAGE=grok2api-rs:local docker compose up -d
```

### 3) GitHub Actions 自动构建镜像

仓库已包含 `.github/workflows/docker-publish.yml`：

- 推送到 `main`：发布 `ghcr.io/<owner>/grok2api-rs:latest`
- 推送标签（如 `v1.0.0`）：发布同名 tag 镜像
- 构建架构：`linux/amd64` + `linux/arm64`

## 编译

```bash
# 本地 release
cargo build --release

# Linux x86_64 musl 静态构建（需安装 cargo-zigbuild 和 zig）
cargo zigbuild --release --target x86_64-unknown-linux-musl
```

## 标准配置示例（完整）

将 `config.defaults.toml` 复制到 `data/config.toml` 后按需调整：

```toml
[grok]
temporary = true
stream = true
thinking = true
dynamic_statsig = true
filter_tags = ["xaiartifact","xai:tool_usage_card","grok:render"]
timeout = 120
base_proxy_url = ""
asset_proxy_url = ""
cf_clearance = ""
wreq_emulation = "chrome_136"
wreq_emulation_usage = ""
wreq_emulation_nsfw = ""
max_retry = 3
retry_status_codes = [401,429,403]
imagine_default_image_count = 4
imagine_sso_daily_limit = 10
imagine_blocked_retry = 3
imagine_max_retries = 5

[app]
app_url = "http://127.0.0.1:8000"
app_key = "grok2api"
api_key = ""
image_format = "url"
video_format = "url"

[token]
auto_refresh = true
refresh_interval_hours = 8
fail_threshold = 5
save_delay_ms = 500
reload_interval_sec = 30

[cache]
enable_auto_clean = true
limit_mb = 1024

[performance]
assets_max_concurrent = 25
media_max_concurrent = 50
usage_max_concurrent = 25
assets_delete_batch_size = 10
assets_batch_size = 10
assets_max_tokens = 1000
usage_batch_size = 50
usage_max_tokens = 1000
nsfw_max_concurrent = 10
nsfw_batch_size = 50
nsfw_max_tokens = 1000

[downstream]
enable_chat_completions = true
enable_responses = true
enable_images = true
enable_images_nsfw = true
enable_models = true
enable_files = true
```

配置说明（关键项）：

- `app.api_key`：下游调用鉴权 Bearer Token，留空表示不校验。
- `app.app_key`：后台登录密码。
- `app.image_format`：默认图片返回格式（`url` / `base64`）；若请求里传了 `response_format`，以请求参数为准。
- `grok.wreq_emulation*`：上游浏览器指纹模板，支持全局/Usage/NSFW 分场景覆盖。
- `grok.base_proxy_url` / `grok.asset_proxy_url`：可选上游代理地址。

## curl 调用示例

### Chat Completions

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-4",
    "messages": [{"role":"user","content":"你好"}]
  }'
```

### Responses API（文本）

```bash
curl http://127.0.0.1:8000/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-4",
    "input": "你好"
  }'
```

Responses 文本问答截图：  
![Responses 文本问答截图](docs/images/3image.png)

### Responses API（生图）

```bash
curl http://127.0.0.1:8000/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-imagine-1.0",
    "input": [
      {"type":"input_text","text":"画一只在太空漂浮的猫"}
    ]
  }'
```

Responses 生图截图：  
![Responses 生图截图](docs/images/4image.png)

### NSFW 专用生图

```bash
curl http://127.0.0.1:8000/v1/images/generations/nsfw \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-imagine-1.0",
    "prompt": "绘制一张夜店风格的人像海报",
    "n": 1,
    "response_format": "url"
  }'
```

### 获取模型列表

```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```

可用模型列表截图：  
![可用模型列表截图](docs/images/5image.png)

sub2api 调用模型截图：  
<img src="docs/images/6image.png" alt="sub2api 调用模型截图" width="50%">

## 与原项目差异

### 新增

- `/v1/responses`（OpenAI Responses API 兼容）。
- `/v1/images/generations/nsfw`（NSFW 专用图片生成接口）。
- 管理后台新增「下游管理」「对话」页面。
- 对话页面支持 SSE、Markdown 渲染、图片渲染与图文混排。
- 上游统一 `wreq` 实现，不再依赖外部 `curl-impersonate`。
- 增加 Docker 部署与 GHCR 自动发布工作流。

### 暂缺

- 当前仅支持本地存储（`SERVER_STORAGE_TYPE` 其他值会降级并提示）。
- 暂未提供多节点/分布式部署能力（当前以单实例为主）。
