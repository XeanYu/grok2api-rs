# Grok2API-rs

> 本项目基于 [grok2api](https://github.com/chenyme/grok2api) 重构。

> [!NOTE]
> 本项目仅供学习与研究，使用者必须在遵循 Grok 的 **使用条款** 以及 **法律法规** 的情况下使用，不得用于非法用途。

## 1. 后端重构说明

- 使用 Rust + Axum 重写服务端，保持 OpenAI 兼容接口与管理后台能力。
- 静态资源内置到二进制，支持单个二进制文件部署。
- 上游 Grok 请求统一走 `curl-impersonate`，降低被上游拦截概率。
- 配置加载合并默认值，支持后台在线修改并持久化。

系统首页截图：  
![系统首页截图](docs/images/1image.png)

## 2. 安装步骤

- 先下载并解压 [curl-impersonate](https://github.com/lwthiker/curl-impersonate/releases/)
  - 官方发布页下载对应系统版本并解压。
- 解压并在系统中配置路径
  - 将可执行文件放入系统 PATH，或在配置中指定完整路径（如 `"/root/data/curl-impersonate/curl_chrome116"`）。
- 配置文件给出完整的标准配置文件，并配有说明
  - 将 `config.defaults.toml` 复制为 `data/config.toml`，并按需修改。
  - 标准配置示例（含说明）：

```toml
[app]
# 调用 API 的 Bearer Token；为空则不校验
api_key = ""
# 后台登录密码
app_key = "grok2api"
# 对外访问地址（用于文件链接）
app_url = "http://127.0.0.1:8000"
# 图片返回格式：url / base64
image_format = "url"
# 视频返回格式：url
video_format = "url"

[grok]
# 临时对话模式
temporary = true
# 默认流式输出
stream = true
# 思维链输出
thinking = true
# 动态 Statsig 指纹
dynamic_statsig = true
# 过滤标签
filter_tags = ["xaiartifact","xai:tool_usage_card","grok:render"]
# 请求超时（秒）
timeout = 120
# Grok 基础代理地址（可留空）
base_proxy_url = ""
# 资源代理地址（可留空）
asset_proxy_url = ""
# Cloudflare 验证 Cookie（可留空）
cf_clearance = ""
# 是否启用 curl-impersonate
use_curl_impersonate = true
# curl-impersonate 伪装浏览器标识（如 chrome116）；为空则用可执行文件默认值
curl_impersonate = ""
# curl-impersonate 可执行文件路径
curl_path = "/root/data/curl-impersonate/curl_chrome116"
# 最大重试次数
max_retry = 3
# 触发重试的状态码
retry_status_codes = [401,429,403]

[token]
# 自动刷新 Token
auto_refresh = true
# 刷新间隔（小时）
refresh_interval_hours = 8
# 失败阈值
fail_threshold = 5
# 保存延迟（毫秒）
save_delay_ms = 500
# 多进程一致性刷新间隔（秒）
reload_interval_sec = 30

[cache]
# 自动清理缓存
enable_auto_clean = true
# 缓存上限（MB）
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
# 下游接口开关
enable_chat_completions = true
enable_responses = true
enable_images = true
enable_models = true
enable_files = true
```

> Grok Token 号池存储于 `data/token.json`。

### 单文件部署参考目录结构

```
/grok2api-rs
├─ grok2api-rs
└─ data
   ├─ config.toml
   ├─ token.json
   └─ curl-impersonate
      └─ curl_chrome116
```

### 二进制文件部署教程（命令行）

```bash
# 1) 准备目录
mkdir -p grok2api-rs/data/curl-impersonate

# 2) 配置文件
cp config.defaults.toml grok2api-rs/data/config.toml

# 3) Token 号池
cp /path/to/token.json grok2api-rs/data/token.json

# 4) curl-impersonate 可执行文件
cp /path/to/curl_chrome116 grok2api-rs/data/curl-impersonate/curl_chrome116
chmod +x grok2api-rs/data/curl-impersonate/curl_chrome116

# 5) 启动服务（确保 config.toml 中 curl_path 指向上面的路径）
chmod +x grok2api-rs/grok2api-rs
SERVER_HOST=0.0.0.0 SERVER_PORT=8000 ./grok2api-rs/grok2api-rs
```

系统部署执行截图：  
![系统部署执行截图](docs/images/7image.png)

### 项目编译教程（命令行）

```bash
# 常规 release 构建
cargo build --release

# 静态 musl 构建（需要 cargo-zigbuild 和 zig）
cargo zigbuild --release --target x86_64-unknown-linux-musl
```

### curl 调用示例

Chat Completions：

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-4",
    "messages": [{"role":"user","content":"你好"}]
  }'
```

Responses API：

```bash
curl http://127.0.0.1:8000/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "grok-4",
    "input": "你好"
  }'
```

Responses 调用 grok-4 文本问答截图：  
![Responses 文本问答截图](docs/images/3image.png)

Responses 图片生成：

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

Responses 调用生图截图：  
![Responses 生图截图](docs/images/4image.png)

获取可用模型：

```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```

获取可用模型列表截图：  
![可用模型列表截图](docs/images/5image.png)

sub2api 调用模型截图：  
<img src="docs/images/6image.png" alt="sub2api 调用模型截图" width="50%">

## 3. 与原项目相比缺失的内容

- 仅支持本地存储（`SERVER_STORAGE_TYPE` 其他值会降级并提示）。
- 未提供 Docker / docker-compose 部署脚本（如需可自行补充）。

## 4. 与原项目相比新增的内容

- 新增 `/v1/responses`（OpenAI Responses API 兼容）。
- 新增“下游管理”页面，支持下游接口开关。  
  ![下游管理截图](docs/images/2image.png)
- 静态资源内置，支持单文件二进制部署。
- 上游 Grok 请求统一走 `curl-impersonate`（更稳定）。
