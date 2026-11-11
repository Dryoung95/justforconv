# Codex CLI 接入自定义 OpenAI 兼容中转站

在 Linux 上把 Codex CLI 配置为走第三方 OpenAI 兼容网关（而非 OpenAI 官方），并完成端到端验证。

**本文档不含任何 API Key。请替换为你自己的。**

- 测试环境：Ubuntu 24.04 LTS · x86_64 · glibc 2.39
- Codex 版本：`codex-cli 0.167.1`（独立二进制，无需 Node.js）
- 网关协议：OpenAI **Responses API**（`wire_api = "responses"`）

---

## 0. 前置条件

- 已有中转站地址（示例：`https://api.example.com/v1`）和对应的 API Key
- 中转站支持 **Responses API**（`POST /v1/responses`）。这一点很重要：Codex 的 `wire_api = "responses"` 只会打这个接口，而不是 `/v1/chat/completions`。不支持的话把 `wire_api` 改成 `"chat"`。

## 1. 先用 curl 验证 Key 和接口

**不要**先装 Codex 再排查，先确认链路本身是通的。

```bash
# 测试 Responses API（Codex 实际使用的接口）
curl -s -m 30 \
  https://api.example.com/v1/responses \
  -H "Authorization: Bearer $YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-6-astra","input":"say pong","max_output_tokens":16}'

# 也可顺带验证模型列表
curl -s -m 30 https://api.example.com/v1/models \
  -H "Authorization: Bearer $YOUR_KEY"
```

预期：两者都返回 `200`，前者 `status: "completed"`。

## 2. 安装 Codex CLI

### 方式 A：独立二进制（推荐，不依赖 Node）

```bash
# 查最新的 release tag
curl -s https://api.github.com/repos/openai/codex/releases/latest \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['tag_name'])"
# -> rust-v0.167.1

# 下载并安装（注意按机器架构选 asset：x86_64 / aarch64）
curl -sL -o /tmp/codex.tar.gz \
  https://github.com/openai/codex/releases/download/rust-v0.167.1/codex-x86_64-unknown-linux-musl.tar.gz
mkdir -p /tmp/codex-extract
tar xzf /tmp/codex.tar.gz -C /tmp/codex-extract
install -m 755 /tmp/codex-extract/codex-x86_64-unknown-linux-musl /usr/local/bin/codex
rm -rf /tmp/codex.tar.gz /tmp/codex-extract

codex --version   # codex-cli 0.167.1
```

### 方式 B：npm

```bash
npm install -g @openai/codex
```

## 3. 写配置 `~/.codex/config.toml`

```toml
model_provider = "custom"
model = "gpt-6-astra"
review_model = "gpt-6-astra"

[model_providers.custom]
name = "OpenAI"
base_url = "https://api.example.com/v1"
wire_api = "responses"
requires_openai_auth = true
```

字段说明：

| 字段 | 作用 |
|---|---|
| `model_provider = "custom"` | 指向 `[model_providers.custom]` 这张表（键名 `custom` 是固定占位，表内容随便填） |
| `name` | 展示名，无实际作用 |
| `base_url` | 网关地址，**要带 `/v1`** |
| `wire_api` | `"responses"` 打 Responses API；`"chat"` 打 Chat Completions |
| `requires_openai_auth = true` | 声明用 OpenAI 式鉴权（配合下一步的 key 存储） |

> **注意：** 在 0.167.1 上，以下配置项已被弃用并忽略，写了会刷启动警告，不要加：
> `disable_response_storage`、`network_access`、`windows_wsl_setup_acknowledged`。

顺手收紧权限（配置里只有地址，没有 key，但保持习惯）：

```bash
chmod 600 ~/.codex/config.toml
```

## 4. 存入 API Key（关键步骤）

**这是整个流程最容易卡住的一步。** 踩坑记录：

- 只 `export OPENAI_API_KEY=...` **不够** —— Codex 走文件鉴权，单跑会报 `401 Unauthorized: 缺少 API Key`。
- 老版本的 `codex login --api-key` 参数已废弃，会被拒绝。

**正确做法**（从 stdin 传入，避免 key 进 shell history）：

```bash
printenv OPENAI_API_KEY | codex login --with-api-key
# 或者直接管道明文：
# echo "sk-xxxx" | codex login --with-api-key
```

输出 `API key saved.` 即成功。Key 落盘到 `~/.codex/auth.json`（权限 600）。

```bash
# 确认已写入
python3 -c "import json; d=json.load(open('$HOME/.codex/auth.json')); print(d['auth_mode'])"
# -> apikey
```

之后**无需任何环境变量**即可使用。不再需要时：`codex logout`。

## 5. 端到端验证

```bash
# 诊断（确认配置被正确加载，且不分版本都能用）
codex doctor
# 关注 Configuration 一节应显示：
#   model  gpt-6-astra · custom
# 以及 auth 一节显示 auth is provided by environment / auth.json

# 真实调用（JSON 流，适合脚本化检查）
codex exec --skip-git-repo-check --json "Reply with exactly: pong" | tail -4
```

成功的输出特征：

```json
{"type":"item.completed","item":{"type":"agent_message","text":"pong"}}
{"type":"turn.completed","usage":{...}}
```

交互式使用直接敲 `codex` 即可，无需 `--skip-git-repo-check`（该 flag 只影响非交互模式，且要求 cwd 是 git 仓库）。

## 6. 验证过程中的排障速查

| 现象 | 原因 | 解决 |
|---|---|---|
| `401 Unauthorized: 缺少 API Key`（Codex 里） | key 没进 `~/.codex/auth.json`，只 export 了 | 第 4 步的 `codex login --with-api-key` |
| 同样的请求 curl 200、Codex 401 | 同上，先怀疑鉴权来源而不是网关 | 同上 |
| `Codex is ignoring 3 unrecognized configuration settings` | 写了已弃置的旧配置项 | 删掉 `disable_response_storage` 等 |
| `unexpected status 404` 打到 `/v1/responses` | 网关不支持 Responses API | `wire_api = "chat"` |
| `could not find bubblewrap on PATH` | 沙箱依赖缺失（仅告警，能用） | `apt install bubblewrap`，或忽略 |
| `Code Mode is unavailable ... codex-code-mode-host` | 可选组件缺失（**不影响使用**） | 忽略 |
| `codex exec` 报需要 git 仓库 | 非交互模式的默认限制 | 加 `--skip-git-repo-check` |

## 7. 安全提醒

- `~/.codex/auth.json` 是**明文**的 key，务必保持 600 权限。
- key 一旦在聊天记录 / 截图 / 日志里出现过，就去后台**轮换**一次。
- 接入第三方中转站 = 把你的全部代码和对话发给该站，自评估风险。

---

## TL;DR

```bash
# 1. curl 验证 key（/v1/responses 必须 200）
# 2. 装二进制
install -m 755 /tmp/codex-extract/codex-* /usr/local/bin/codex
# 3. 写 ~/.codex/config.toml（见第 3 节）
# 4. 存 key —— 这一步是关键
printenv OPENAI_API_KEY | codex login --with-api-key
# 5. 验证
codex exec --skip-git-repo-check --json "say pong" | tail -2
```
