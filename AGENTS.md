<!-- OPENSPEC:START -->
# OpenSpec Instructions

These instructions are for AI assistants working in this project.

Always open `@/openspec/AGENTS.md` when the request:
- Mentions planning or proposals (words like proposal, spec, change, plan)
- Introduces new capabilities, breaking changes, architecture shifts, or big performance/security work
- Sounds ambiguous and you need the authoritative spec before coding

Use `@/openspec/AGENTS.md` to learn:
- How to create and apply change proposals
- Spec format and conventions
- Project structure and guidelines

Keep this managed block so 'openspec update' can refresh the instructions.

<!-- OPENSPEC:END -->

# Project: 教学动画生成器 (Instructional Animation)

**技术栈**：Python 3.11 + FastAPI + Jinja2 + Playwright + FFmpeg；前端为原生 JS/CSS（无构建步骤）。

## 本地运行

```bash
# 启动服务
python -m uvicorn app:app --host 0.0.0.0 --port 8000
```

Docker 运行：`docker-compose up -d`（参考 `Dockerfile` / `docker-compose.yml`）。

## 目录结构

| 路径 | 说明 |
|---|---|
| `app.py` | 后端主入口：路由、SSE 流式生成、录制转码接口 |
| `AnthropicClient.py` | Anthropic/OpenAI 兼容客户端封装 |
| `templates/index.html` + `static/script.js` / `static/style.css` | 前端页面（模板渲染，加 `?v={{ time }}` 缓存失效） |
| `scripts/record_media.py` | Playwright 录制 + FFmpeg 转码工具 |
| `requirements.txt` | Python 依赖 |

## 配置

- 配置来源优先级：**环境变量**（`API_KEY` / `BASE_URL` / `MODEL`）→ `credentials.json` → 默认值。
- 本地配置写在项目根 `.env`（已被 `.gitignore` / `.dockerignore` 排除，**勿提交**）。
- 修改 `.env` / `credentials.json` 后通常需重启服务生效（`ConfigManager` 动态读取）。

## 关键接口

- **`POST /generate`**：SSE 流式生成。逐 chunk 发送 `{"token": "...", "kind": "thinking" | "content"}`。
  - `kind: thinking` → 模型思维链（对应 `delta.reasoning_content`）。
  - `kind: content` → 正式内容（HTML 代码块，用 ` ```html `...` ``` ` 包裹）。
- **`POST /record`**：接收 `html_text` 等参数，Playwright 渲染 + FFmpeg 转 MP4/GIF。
- **`POST /config`**：保存 API 配置到 `.env`。

## 前后端约定（重要）

- 前端在 `static/script.js` 中按 `kind` 区分 thinking 与 content：
  - thinking → 渲染到「思考过程」折叠块（默认 3 行预览，点击标题展开/收起；代码弹出时自动收起）。
  - content → 解析 ` ```html ` 代码块并注入播放器 iframe。
- **流式 token 可能被任意拆分**（如 `` ``` `` 与 `html` 分到不同 chunk），解析时必须用跨 chunk 状态机（`pendingCodeHeader`），不能假设一个 marker 落在同一个 token 内。
- 前端渲染大段流式文本用「缓冲 + `requestAnimationFrame` 批量刷新」，避免逐 token 写 DOM 导致卡顿。
- 修改 `templates/` / `static/` 后，用户需强刷页面（Ctrl+F5）以绕过缓存（`?v={{ time }}`）。

## 代码约定

- 路径尽量用 `pathlib.Path`；配置读取集中在 `app.py` 的 `ConfigManager`。
- 新增对模型的 thinking 转发、录制参数等改动需同时核对两条代码路径（Anthropic / OpenAI 兼容），避免只改一端。