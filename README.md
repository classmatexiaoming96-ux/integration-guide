# Codex Desktop 接入 OpenViking 与 Obsidian

这是一份面向 macOS 的可复现接入指南，目标是在 Codex Desktop 中同时获得：

- OpenViking MCP 工具：显式搜索、读取和写入长期记忆；
- OpenViking 生命周期 Hooks：自动召回、增量捕获、提交和异步记忆提取；
- Obsidian MCP 工具：直接检索和维护人类可读的 Vault 笔记；
- 清晰的数据分工：Obsidian 是长期知识真源，OpenViking 是面向 Agent 的记忆与检索层。

本文记录的是已经实际跑通的滚动最新版方案，不需要修改业务项目，也不需要把密钥写进
任何仓库。文中的版本是历史验收快照，不代表安装命令会永久得到同一版本；如果用于团队
生产环境，应把 marketplace 固定到审查过的 tag 或分支。

> 验证基线：2026-09-02，Codex Desktop 内置 CLI `0.151.0-alpha.7.2`，
> `openviking-memory@openviking` `0.8.1`。后续版本的菜单名称可能变化，
> 但 MCP、Hook 信任和验收原则不变。

## 1. 先理解两条链路

```text
                         ┌─ MCP 工具 ──────── 检索 / 显式记忆
Codex Desktop ───────────┤
                         └─ 生命周期 Hooks ─ 自动召回 / 捕获 / commit
                                           │
                                           ▼
                                    OpenViking Server
                                    Agent 记忆与语义索引

Codex Desktop ── Obsidian MCP ─────► Obsidian Vault
                                    人类可读的知识真源
```

MCP 和 Hooks 是两个独立能力：

- MCP 正常，只能证明模型能调用 OpenViking 工具；
- 自动召回和自动沉淀依赖 Hooks，插件安装成功不等于 Hooks 已被信任；
- Obsidian MCP 直接操作 Vault，不负责 OpenViking 的自动记忆提取；
- 本文不启用任何“根据远端清单自动删除本地笔记”的同步桥。

推荐的数据治理原则：

- 需要长期审阅、人工维护、版本化的知识进入 Obsidian；
- 会话偏好、人物实体、事件和工作记忆交给 OpenViking；
- OpenViking 内容应被视为可重建索引，不应成为唯一事实来源；
- Obsidian 写入先落到 `_inbox/`，人工确认后再晋升到正式目录。

## 2. 前置条件

- macOS 已安装 ChatGPT/Codex Desktop；
- Python 3.10 或更高版本；
- Node.js 18 或更高版本；
- `curl`；诊断示例还使用 `rg` 与 `jq`，没有它们时可用 `grep` 和人工查看 JSON 代替；
- 选择 `uv tool install` 路径时还需要先安装 `uv`；
- OpenViking Server 已安装，或有一个可访问的远端 OpenViking Server；
- 如需 Obsidian 链路，Obsidian 已安装并打开目标 Vault；
- 有权配置 `~/.codex/config.toml`、`~/.openviking/` 和自己的 Obsidian 插件。

先确认桌面端真正使用的 Codex 版本：

```bash
CODEX_DESKTOP_BIN="/Applications/ChatGPT.app/Contents/Resources/codex"

"$CODEX_DESKTOP_BIN" --version
codex --version
```

这两个版本可能不同。桌面端接入、插件检查和 Hook 信任应以
`/Applications/ChatGPT.app/Contents/Resources/codex` 为准，不能只看 PATH 中的
`codex`。本机曾同时存在桌面内置 `0.151.0-alpha.7.2` 和 PATH 中 `0.144.6`，
后者会造成非常具有误导性的诊断结果。

再查看当前 Hook 能力：

```bash
"$CODEX_DESKTOP_BIN" features list | rg 'hooks|plugin_hooks|plugins'
```

当前 Codex 的正式开关是 `hooks`。`plugin_hooks` 已移除，不要为了让旧版 doctor
变绿而重新添加它。

## 3. 启动 OpenViking Server

已有可用远端服务器可以跳过本机安装与启动步骤，但应由服务端管理员提供对应环境的
health/ready、Embedding、VLM、存储和鉴权检查证据。

### 3.1 安装

二选一：

```bash
uv tool install openviking --upgrade
```

或：

```bash
pip install openviking --upgrade --force-reinstall
```

### 3.2 首次配置与诊断

```bash
openviking-server init
openviking-server doctor
```

初始化向导会要求配置 Embedding 与 VLM。不要猜测模型名、API 地址或密钥；
它们必须与实际使用的模型服务一致。

### 3.3 启动与健康检查

```bash
openviking-server
```

默认本地地址是 `http://127.0.0.1:1933`。另开终端检查：

```bash
curl -fsS http://127.0.0.1:1933/health
```

`/health` 只证明服务可达。Embedding、VLM、存储和鉴权仍应以
`openviking-server doctor` 的完整结果为准。

## 4. 配置 OpenViking 客户端身份

OpenViking 插件的 MCP 与 Hooks 共用 `~/.openviking/ovcli.conf`，因此两边应连接
到同一 URL 和同一套有效身份。`account`、`user` 是否必填取决于服务器 auth mode；
本地 dev 默认身份或某些 API-key 模式可以省略，不要为了照抄示例而填写虚构值。

本地示例：

```json
{
  "url": "http://127.0.0.1:1933",
  "account": "<your-account>",
  "user": "<your-user>"
}
```

远端鉴权示例：

```json
{
  "url": "https://openviking.example.com",
  "api_key": "<your-api-key>",
  "account": "<your-account>",
  "user": "<your-user>"
}
```

收紧权限：

```bash
chmod 600 ~/.openviking/ovcli.conf

if [ -f ~/.openviking/ov.conf ]; then
  chmod 600 ~/.openviking/ov.conf
fi
```

注意：

- 不要把 `ovcli.conf`、`ov.conf` 或 API key 提交到 Git；
- 不要在排障截图或终端输出里打印完整密钥；
- `OPENVIKING_*` 环境变量优先级可能高于配置文件；
- 可只检查是否存在覆盖变量，不打印值：

```bash
env | cut -d= -f1 | rg '^OPENVIKING_'
```

## 5. 安装 OpenViking Memory 插件

下面两种安装方式只能选一种。混用可能让同一组 Hooks 被加载两次。

### 5.1 方式 A：官方滚动脚本（仅便捷试用）

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/volcengine/OpenViking/main/examples/memory-plugin-shared/install.sh) \
  --harness codex \
  --codex-bin /Applications/ChatGPT.app/Contents/Resources/codex
```

安装脚本会配置客户端、注册 `openviking` marketplace、安装并启用
`openviking-memory@openviking`，然后检查 stdio MCP。显式传入 `--codex-bin` 是为了
避免安装脚本误调用 PATH 中另一个旧版 `codex`。

这条命令会下载并执行 `main` 上的滚动脚本，只适合个人便捷试用。它不满足团队生产的
可复现和供应链审查要求。生产环境应先审查脚本，把 URL 中的 `main` 换成审查过的
commit SHA 并校验内容，或改用下一节的固定 ref 方式。

截至插件 0.8.1，安装器仍可能向 `[features]` 写入已移除的
`plugin_hooks = true`。安装后打开 `~/.codex/config.toml`，只删除这一条旧键，再按第 6 节
确认 `hooks = true`；不要让旧键留在配置中。

### 5.2 方式 B：桌面内置 CLI（推荐；生产时固定 ref）

如果希望明确避免 PATH 中旧版 Codex，可以手动执行：

```bash
CODEX_DESKTOP_BIN="/Applications/ChatGPT.app/Contents/Resources/codex"

"$CODEX_DESKTOP_BIN" plugin marketplace add https://github.com/volcengine/OpenViking.git --ref main
"$CODEX_DESKTOP_BIN" plugin add openviking-memory@openviking
```

上例的 `main` 是滚动更新。如果要让团队安装结果可重复，把它换成已经确认存在并完成审查的
tag 或 commit SHA，同时把该 ref 记录到内部变更单；不要凭空填写一个未验证的版本号。

检查安装结果：

```bash
"$CODEX_DESKTOP_BIN" plugin list
```

应能看到 `openviking-memory@openviking` 已安装并启用。

## 6. 启用正确的 Hooks 开关

在现有 `~/.codex/config.toml` 中合并以下配置。不要创建重复的 `[features]`
或重复的插件表。

```toml
[features]
hooks = true

[plugins."openviking-memory@openviking"]
enabled = true
```

Hooks 在当前 Codex 中默认开启，显式写出 `hooks = true` 的好处是便于诊断。

不要写：

```toml
# 已移除的旧键，不要使用
plugin_hooks = true
```

## 7. 审查并信任 5 个 Hooks

这是最关键、也最容易遗漏的一步。插件安装或启用后，Codex **不会自动信任**
插件自带的非托管 Hooks。

从准备日常使用的工作目录启动桌面内置 CLI：

```bash
CODEX_DESKTOP_BIN="/Applications/ChatGPT.app/Contents/Resources/codex"

"$CODEX_DESKTOP_BIN" -C "/absolute/path/to/your/workspace"
```

在 Codex TUI 中输入：

```text
/hooks
```

找到来源 `openviking-memory@openviking`，逐项审查并信任：

1. `SessionStart`
2. `UserPromptSubmit`
3. `Stop`
4. `SessionEnd`
5. `PreCompact`

这些 Hook 的职责是：

| Hook | 作用 |
| --- | --- |
| `SessionStart` | 始终注入画像；仅 `startup`/`clear` 重试未成功提交或超时闲置的会话，`resume` 不 sweep |
| `UserPromptSubmit` | 根据本轮问题自动召回相关记忆并注入上下文 |
| `Stop` | 每轮结束后增量捕获新的对话与工具记录 |
| `PreCompact` | Codex 压缩上下文前提交完整会话 |
| `SessionEnd` | 会话正常结束时补齐尾部记录并提交 |

信任记录绑定 Hook 定义的 hash。升级插件或修改 `hooks.json` 后，定义会变化，
必须重新进入 `/hooks` 审查。不要把 `--dangerously-bypass-hook-trust` 当作长期方案。

## 8. 完全重启 Codex Desktop

完成安装、改配置或信任 Hooks 后：

1. 完全退出 ChatGPT/Codex Desktop；
2. 确认不是只关闭窗口；
3. 重新打开应用；
4. 最好新建一个短任务做首次验收。

桌面端会复用长期运行的 MCP proxy 和 app-server，只关闭窗口不一定加载新配置。

## 9. 验收 OpenViking MCP

在桌面端输入：

```text
/mcp
```

应能看到 OpenViking 服务器。也可以直接要求 Codex：

```text
请调用 OpenViking 的 health 工具，只返回健康检查结果。
```

成功结果应类似：

```text
OpenViking is healthy (service initialized, storage: VikingFS)
```

这一项通过说明 MCP 正常，但还不能单独证明自动 Hooks 正常。

## 10. 验收自动召回与自动沉淀

强烈建议用**全新的短任务**验收，不要先在一个包含大量工具输出的旧任务里开启。

### 10.1 验收自动召回

用 OpenViking 中已知存在的独特关键词提问，例如某个明确偏好、项目代号或人物关系。
命中时，`UserPromptSubmit` 会向模型注入类似内容：

```xml
<openviking-context source="auto-recall" format="digest">
...
</openviking-context>
```

没有看到召回内容并不必然是故障：本轮问题可能没有足够相关的记忆。判断 Hook 是否运行，
应同时检查下一节的本地状态。

### 10.2 验收增量捕获

每完成一轮对话后查看：

```bash
ls -lt ~/.openviking/codex-plugin-state
```

找到当前任务对应的 JSON 文件，然后检查：

```bash
jq '{codexSessionId,ovSessionId,capturedTurnCount,createdAt,lastUpdatedAt}' \
  ~/.openviking/codex-plugin-state/<codex-session-id>.json
```

判据：

- `capturedTurnCount` 随新的 `Stop` 事件增长，说明自动捕获正在运行；
- 它统计的是规范化 transcript 单元，不等于肉眼看到的自然对话轮数；
- commit 完成后 `ovSessionId` 变成 `null` 是正常状态，不代表失败；
- 不应仅凭一个旧的 debug 日志判断 Hook 是否工作。

### 10.3 验收服务端会话

仅对本机可信模式，可用以下只读请求。远端密钥不要直接写进命令历史。

```bash
curl -sS \
  -H 'X-OpenViking-Account: <account>' \
  -H 'X-OpenViking-User: <user>' \
  'http://127.0.0.1:1933/api/v1/sessions/cx-<codex-session-id>' \
  | jq '{status,session_id:.result.session_id,total_message_count:.result.total_message_count,commit_count:.result.commit_count,pending_tokens:.result.pending_tokens,memories_extracted:.result.memories_extracted,llm_token_usage:.result.llm_token_usage}'
```

主要判据：

- `total_message_count` 持续增长；
- `commit_count > 0` 说明至少有一次提交；
- `message_count = 0` 可能只是 live tail 已归档，应看累计的 `total_message_count`；
- `pending_tokens = 0` 常见于刚完成 commit 的状态。

### 10.4 理解异步提取

OpenViking 的 commit 分两阶段：

1. Phase 1 同步归档并增加 `commit_count`；
2. Phase 2 在后台调用模型提取长期记忆。

因此刚看到 `commit_count > 0` 时，`memories_extracted = 0` 是合法的中间态。
通常应等待几分钟再查，而不是立刻判定失败。若最终仍为 0，也可能是内容没有可提炼的
长期信息；如果有明确可提炼内容却长期为 0，再检查 VLM、Embedding 和服务器任务日志。

默认触发 commit 的路径包括：

- `Stop` 捕获后，`pending_tokens >= 20000`；
- `PreCompact`；
- `SessionEnd`；
- 下次 `SessionStart` 在来源为 `startup` 或 `clear` 时，对异常退出或闲置会话补交；
  `resume` 只恢复上下文，不执行 sweep/commit。

桌面 app-server 的 `SessionEnd` 可能延后，所以“切换任务后没有立刻提取”并不等于失败。

## 11. 接入 Obsidian MCP（可选但推荐）

OpenViking 负责 Agent 记忆，Obsidian 负责人类可读的正式知识。两者可以同时接入 Codex，
但应保持职责分离。

### 11.1 安装 Obsidian 插件

在 Obsidian 中安装并启用社区插件 **Local REST API with MCP**，然后进入：

```text
Settings → Local REST API with MCP
```

默认安全端点为：

```text
https://127.0.0.1:27124/mcp/
```

它需要 `Authorization: Bearer <api-key>`，并使用插件生成的本地 CA 证书。

### 11.2 私密保存 Authorization Header 与 CA

不要把 API key 直接写进 `~/.codex/config.toml`。先创建私密目录：

```bash
install -d -m 700 ~/.config/obsidian-rest
```

用编辑器创建：

```text
~/.config/obsidian-rest/authorization-header
```

文件只有一行：

```text
Authorization: Bearer <your-obsidian-api-key>
```

然后限制权限：

```bash
chmod 600 ~/.config/obsidian-rest/authorization-header
```

先在 Obsidian 插件设置中确认 HTTPS 端口确实是 `27124`，再检查监听进程属于 Obsidian：

```bash
lsof -nP -iTCP:27124 -sTCP:LISTEN
```

确认无误后，从本机 HTTPS 端点下载 CA：

```bash
curl -k \
  https://127.0.0.1:27124/obsidian-local-rest-api.crt \
  -o ~/.config/obsidian-rest/obsidian-local-rest-api.pem

chmod 600 ~/.config/obsidian-rest/obsidian-local-rest-api.pem

openssl x509 \
  -in ~/.config/obsidian-rest/obsidian-local-rest-api.pem \
  -noout -subject -issuer -fingerprint -sha256
```

这里的 `-k` 只用于第一次从 `127.0.0.1` 获取尚未信任的本地 CA。后续连接通过
`NODE_EXTRA_CA_CERTS` 明确信任该证书，不应全局关闭 TLS 校验。应保存首次核对后的 SHA-256
指纹；证书意外变化时先回到 Obsidian 插件设置确认，不要直接覆盖。

### 11.3 在 Codex 中配置 Obsidian MCP

Codex Desktop 原生支持 Streamable HTTP MCP。下面使用 `mcp-remote` 的原因，是让
Authorization 存在独立的 600 权限文件中，同时让 Node 只为该 MCP 进程加载 Obsidian CA，
避免把密钥明文放进 Codex 配置。

先找到稳定的 `npx` 绝对路径：

```bash
command -v npx
```

把以下配置合并进 `~/.codex/config.toml`，并将所有 `<mac-user>` 与 npx 路径替换成
本机实际值：

```toml
[mcp_servers.obsidian]
startup_timeout_sec = 30
tool_timeout_sec = 60
default_tools_approval_mode = "writes"
enabled_tools = ["vault_list", "vault_read", "vault_get_document_map", "search_query", "search_simple", "tag_list", "active_file_get_path", "vault_write", "vault_append", "vault_patch"]
disabled_tools = ["vault_delete", "vault_move", "vault_copy", "command_execute", "command_list", "open_file"]
command = "/absolute/path/to/npx"
args = ["-y", "mcp-remote@0.8.3", "https://127.0.0.1:27124/mcp/", "--header-file", "/Users/<mac-user>/.config/obsidian-rest/authorization-header", "--silent"]

[mcp_servers.obsidian.env]
NODE_EXTRA_CA_CERTS = "/Users/<mac-user>/.config/obsidian-rest/obsidian-local-rest-api.pem"

[mcp_servers.obsidian.tools.vault_write]
approval_mode = "prompt"

[mcp_servers.obsidian.tools.vault_append]
approval_mode = "prompt"

[mcp_servers.obsidian.tools.vault_patch]
approval_mode = "prompt"
```

这个权限模型的含义：

- 读、搜索和标签工具可用；
- `default_tools_approval_mode = "writes"` 按 MCP 工具注解识别写操作；
- 三个写工具额外配置了 `approval_mode = "prompt"`，避免依赖注解是否准确；
- 删除、移动、复制、任意命令执行和打开文件被禁用；
- 工具 allowlist 是硬边界，`_inbox/` 写入约束仍需要 Agent 指令或额外代理层落实。

如果只需要检索，可以进一步从 `enabled_tools` 移除 `vault_write`、`vault_append`、
`vault_patch`，获得真正的只读配置。

最后收紧 Codex 配置权限并完全重启桌面端：

```bash
chmod 600 ~/.codex/config.toml
```

### 11.4 验收 Obsidian MCP

保持 Obsidian 打开，在 Codex Desktop 输入 `/mcp`，确认 `obsidian` 已连接。然后依次测试：

```text
请使用 Obsidian MCP 列出 Vault 根目录，只读，不修改任何文件。
```

```text
请使用 Obsidian MCP 搜索一个确定存在的独特关键词，只返回命中的笔记路径。
```

首次写入测试应明确指定 `_inbox/`，并检查 Codex 是否出现写操作批准：

```text
请在 Obsidian 的 _inbox/ 下新建一篇验收笔记；先告诉我目标路径，获得批准后再写入。
```

建议给 Codex 增加长期指令：

```markdown
## Obsidian 写入规则

- 默认只读。
- 需要沉淀时只写入 `_inbox/`。
- 不删除、移动或覆盖现有笔记。
- 从 `_inbox/` 晋升到正式目录必须由人确认。
```

注意：提示词规则不是强制访问控制。真正的硬限制仍来自禁用工具、审批模式和服务端权限。

## 12. Doctor 与常见误判

### 12.1 运行两层 Doctor

服务器诊断：

```bash
openviking-server doctor
```

先定位插件 doctor：

```bash
find ~/.codex/plugins/cache/openviking/openviking-memory \
  -path '*/scripts/ov-memory-doctor.mjs' \
  -print
```

再把找到的最新绝对路径交给 Node：

```bash
node /absolute/path/to/ov-memory-doctor.mjs --no-color
```

### 12.2 OpenViking 0.8.1 的 `plugin_hooks` 误报

0.8.1 的 installer/doctor 仍可能提示：

```text
[features] plugin_hooks is not set
```

这是针对旧 Codex 的检查。当前 Codex 的正式键是 `hooks`，`plugin_hooks` 已移除。
当以下四项都成立时，应忽略这一条旧检查：

- 桌面内置 binary 的 `features list` 显示 `hooks` 可用；
- `~/.codex/config.toml` 中是 `hooks = true`；
- `/hooks` 中 5 个 Hook 已信任；
- `capturedTurnCount` 或真实召回注入证明 Hook 已运行。

不要为了消除误报而添加废弃的 `plugin_hooks = true`。

### 12.3 排障矩阵

| 现象 | 更可能的原因 | 优先检查 |
| --- | --- | --- |
| OpenViking MCP 不出现 | 插件未启用、配置未重载、proxy 启动失败 | 桌面 binary 的 `plugin list`、`/mcp`、完整重启 |
| MCP 通，但状态文件不增长 | Hooks 未信任或运行时没加载新配置 | `[features].hooks`、`/hooks` 的 5 项信任、完整重启 |
| `commit_count > 0`，记忆仍为 0 | Phase 2 仍在异步提取，或内容不可提炼 | 等几分钟后重查；再跑 server doctor |
| `ovSessionId = null` | 很可能刚完成 commit | 看 `capturedTurnCount`、`total_message_count`、`commit_count` |
| 自动召回为空 | 没有足够相关的语义命中 | 用已知独特关键词重试；不要只凭空结果判断 Hook |
| doctor 显示旧 Codex 版本 | PATH 中 `codex` 不是桌面内置 binary | 用 `/Applications/ChatGPT.app/Contents/Resources/codex` 复核 |
| Obsidian 连接被拒绝 | Obsidian 未运行、端口不对或 token 不对 | Obsidian 插件设置、27124、header 文件 |
| Obsidian TLS 失败 | Node 未加载插件 CA | `NODE_EXTRA_CA_CERTS` 路径和 PEM 文件权限 |
| 日志出现 `unknown conversation` | 桌面渲染/路由日志噪声 | 若主任务状态增长、MCP/召回成功，可暂不视为链路故障 |

### 12.4 Debug 日志的限制

可以用 `OPENVIKING_DEBUG=1` 开启完整 Hook 日志，但它必须存在于**启动 Codex 的进程
环境**；在桌面已经启动后才在普通 shell 中设置变量，不会追溯生效。0.8.1 也兼容把
`debug: true` 合并到 `~/.openviking/ov.conf` 的 `codex` 对象中。无论使用哪种方式，
修改后都要完整重启桌面端。因此：

- 日志为空或陈旧不能证明 Hooks 没运行；
- 优先使用状态文件、MCP health、服务端会话和 memory diff 作为证据；
- 需要 debug 时，设置环境后必须完整重启桌面端。

## 13. 成本、隐私与首次回灌

OpenViking 自动捕获的不只是自然语言对话，也可能包括工具调用和工具输出。启用前需要确认：

- 当前 workspace 是否包含敏感代码、日志、凭证或客户数据；
- account、user 和 peer scope 是否指向正确的租户；
- 服务器存储和模型提供商是否符合数据合规要求；
- 是否允许自动召回额外调用压缩模型。

插件 0.8.1 的关键默认值：

| 配置 | 默认值 | 含义 |
| --- | ---: | --- |
| `OPENVIKING_COMMIT_TOKEN_THRESHOLD` | `20000` | 达到阈值后自动 commit |
| `OPENVIKING_COMMIT_KEEP_RECENT_COUNT` | `10` | 阈值 commit 后保留的近期记录数 |
| `OPENVIKING_CAPTURE_TOOL_MAX_CHARS` | `1000000` | 单个异常巨大工具载荷的保护上限 |

第一次在已有超长历史的任务中启用 `Stop`，会从本地 cursor 0 开始补录旧 transcript。
本机首次真实验收曾补录约 450 个规范化消息单元，并在一次提取中消耗约 65.3 万模型
tokens；这不是每个短任务的常规成本，但足以说明为什么应从新任务开始验收。

降低风险的做法：

- 首次启用后新建一个短任务，不要直接回到工具输出很多的旧任务；
- 根据业务情况降低 `OPENVIKING_CAPTURE_TOOL_MAX_CHARS`；
- 不需要自动召回时可评估 `OPENVIKING_AUTO_RECALL=0`；
- 不需要自动捕获时可评估 `OPENVIKING_AUTO_CAPTURE=0`；
- 若关闭召回压缩，可评估 `OPENVIKING_RECALL_COMPRESS=0`，同时接受召回长度和质量变化；
- 调整前先记录基线，不要仅为了省 token 盲目调大 commit 阈值。

后续正常情况下按 `capturedTurnCount` 增量发送，不会每轮重传完整历史。

## 14. 更新插件

```bash
CODEX_DESKTOP_BIN="/Applications/ChatGPT.app/Contents/Resources/codex"

"$CODEX_DESKTOP_BIN" plugin marketplace upgrade openviking
```

更新后依次执行：

1. 查看 `plugin list` 确认实际加载版本；
2. 重新进入 `/hooks` 审查所有变更项；
3. 完全退出并重启桌面端；
4. 在新短任务中重跑 MCP、捕获和异步提取验收。

## 15. 本方案不做什么

本文只接入 Codex、OpenViking 和 Obsidian，不部署自动双向同步桥，也不会根据不完整的
远端清单自动删除任何笔记。

如果未来要增加 OpenViking/云端与 Obsidian 的自动同步，至少先满足以下上线门槛：

- 远端清单有完整性证明，或删除前逐页精确确认；
- SQLite 做物理文件身份隔离，而不只是字符串路径判断；
- Bridge 有单实例 lease/lock；
- 写接口有明确幂等键或条件版本契约；
- 所有非 2xx HTTP response body 都被消费或释放；
- 启动时校验 workspace、project、Vault 与同步目录角色完全一致。

在这些条件满足前，保持 Obsidian 删除/移动/复制工具禁用，并采用 `_inbox/ + 人工晋升`
更稳妥。

## 16. 最终验收清单

- [ ] 使用的是桌面内置 Codex binary，而不是误用 PATH 旧版
- [ ] 本地部署的 `openviking-server doctor` 通过；远端部署已有管理员提供的等价证据
- [ ] 目标 OpenViking 环境的 health/ready 与真实鉴权检查通过
- [ ] `ovcli.conf` 的 URL 与该 auth mode 所需的有效身份字段指向正确环境
- [ ] 所有实际存在且含敏感配置的 `ovcli.conf`、`ov.conf`、`config.toml` 权限为 600
- [ ] `openviking-memory@openviking` 已安装并启用
- [ ] `[features] hooks = true`
- [ ] 没有依赖已移除的 `plugin_hooks`
- [ ] 5 个插件 Hooks 已通过 `/hooks` 审查和信任
- [ ] 完全重启过 Codex Desktop
- [ ] `/mcp` 中 OpenViking 可见，health 调用成功
- [ ] 新任务完成一轮后 `capturedTurnCount` 增长
- [ ] 服务端 `total_message_count` 增长
- [ ] commit 后等待异步 Phase 2，再判断 memory extraction
- [ ] Obsidian MCP 使用私密 header 文件与独立 CA
- [ ] Obsidian 删除、移动、复制和任意命令执行已禁用
- [ ] Obsidian 写入需要批准，默认只进入 `_inbox/`

## 17. 参考资料

- [OpenAI：Model Context Protocol](https://learn.chatgpt.com/docs/extend/mcp)
- [OpenAI：Codex Hooks](https://learn.chatgpt.com/docs/hooks)
- [OpenAI：Plugins 中的 MCP 与生命周期 Hooks](https://developers.openai.com/plugins/build/plugins#bundled-mcp-servers-and-lifecycle-hooks)
- [OpenViking：项目主页](https://github.com/volcengine/OpenViking)
- [OpenViking：Server Mode Quick Start](https://github.com/volcengine/OpenViking/blob/main/docs/en/getting-started/03-quickstart-server.md)
- [OpenViking：Codex Memory Plugin](https://github.com/volcengine/OpenViking/tree/main/examples/codex-memory-plugin)
- [Obsidian Local REST API with MCP](https://github.com/coddingtonbear/obsidian-local-rest-api)

---

一句话判断是否真的接通：**MCP health 成功、5 个 Hooks 已信任、`capturedTurnCount`
持续增长、服务端 commit 完成并在异步阶段产出可解释的 memory diff。**
