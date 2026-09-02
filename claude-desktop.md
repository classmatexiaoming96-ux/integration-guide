# Runbook:把 Claude 接入 OpenViking + Obsidian 记忆栈

**本文档供 agent 执行。** 按顺序做,每步先跑「验证」,已满足就跳过该步。
标记 🙋 的步骤 agent 做不了,必须停下来交给人。

最后验证:2026-09-02(Claude Code 2.1.250 / 桌面端 1.37937.3 / Obsidian 1.12.7 / OpenViking 0.4.17.2)

---

## 目标状态

完成后应达到:

- Claude Code 能自动召回 OpenViking 里的长期记忆,无需人工提示
- 会话结束后自动把结论沉淀进记忆
- Claude Code 能读写 Obsidian vault

---

## 开工前必读的三条

违反任何一条都会浪费大量时间。

**1. Chat 和 Code 是两个东西,配置不通用。**

| | Claude Chat | Claude Code |
| --- | --- | --- |
| 配置文件 | `~/Library/Application Support/Claude/claude_desktop_config.json` | `~/.claude/settings.json` |
| 手工编辑 | **会被应用覆盖,无效** | 有效 |
| 生命周期钩子 | 无 | **有** |

自动记忆能力只有 Code 有。**本 runbook 全程操作 Code 侧。**

**2. `plugin_hooks` 已废弃,不要照插件文档抄。**

OpenViking 插件的 README 和 doctor 仍在提示 `[features] plugin_hooks`,那是针对
旧版 Codex 的检查。Codex 已移除该键,当前正式键是 `hooks`。

| 位置 | 当前正确写法 |
| --- | --- |
| Codex(桌面端与新版 CLI) | `~/.codex/config.toml` 的 `[features] hooks = true` |
| Claude Code | `~/.claude/settings.json` 的 `hooks` 字段 |

**不要为了消除 doctor 误报去加 `plugin_hooks = true`。**

**3. 判断是否生效,看运行时产物,不看配置文件。**

配置写对了不等于生效——进程可能没重启。始终用「验证」块里的命令确认。

---

## 步骤 0:环境检查

```bash
export PATH="$HOME/.local/bin:$PATH"
echo "claude:     $(claude --version 2>&1 | head -1)"
echo "openviking: $(lsof -nP -iTCP:1933 -sTCP:LISTEN >/dev/null 2>&1 && echo '✅ :1933' || echo '❌ 未运行')"
echo "obsidian:   $(lsof -nP -iTCP:27124 -sTCP:LISTEN >/dev/null 2>&1 && echo '✅ :27124' || echo '❌ 未运行')"
echo "vault:      $(ls -d <你的 Obsidian Vault 路径> 2>/dev/null || echo '❌ 不存在')"
```

**全部 ✅ 才继续。** 否则:

| 缺什么 | 怎么办 |
| --- | --- |
| `claude` 命令 | `npm install -g @anthropic-ai/claude-code` |
| OpenViking 未运行 | `openviking-server`(后台跑) |
| Obsidian REST API 未运行 | 见步骤 3 |

---

## 步骤 1:接 OpenViking 到 Claude Code

**验证(已完成则跳过):**

```bash
python3 -c "import json,os;p=json.load(open(os.path.expanduser('~/.claude/settings.json')));print(p.get('enabledPlugins',{}))" | grep -q openviking && echo "已完成,跳过" || echo "需要执行"
```

**执行:**

```bash
claude plugin marketplace add volcengine/OpenViking
claude plugin install openviking-memory@openviking
```

**确认:**

```bash
claude plugin list 2>&1 | grep -A2 openviking
```

应看到 `openviking-memory` 且状态为 enabled。插件自动读
`~/.openviking/ovcli.conf`,不需要另外配连接信息。

> **装上并 enabled ≠ 钩子在跑。** Codex 侧需要在 `/hooks` 里显式信任插件自带的
> 非托管钩子,否则装了也不执行;Claude Code 的 `settings.json` 里没有对应的
> 信任字段,机制不同。**不要靠推断,用步骤 7 的运行时产物判定**——
> `codex-plugin-state/` 里出现真实 session id 的 json 才算钩子真的在跑。

---

## 步骤 2:打开诊断开关

首次接入必做——**没有这个,失败是完全静默的**(异步 worker 的
`stdio` 是 `["pipe","ignore","ignore"]`,错误无处可见)。

```bash
python3 - <<'PY'
import json, pathlib
p = pathlib.Path.home()/".openviking"/"ovcli.conf"
d = json.loads(p.read_text())
d.setdefault("codex", {}).update({
    "debug": True,              # 写 ~/.openviking/logs/codex-hooks.log
    "writePathAsync": False,    # 同步写入,失败会报出来
    "commitTokenThreshold": 1000,  # 默认 20000,短会话测不出来
})
p.write_text(json.dumps(d, indent=2, ensure_ascii=False) + "\n")
p.chmod(0o600)
print("已写入:", json.dumps(d["codex"], ensure_ascii=False))
PY
```

> GUI 启动的应用**不继承终端环境变量**,所以诊断开关必须走配置文件,
> `export` 无效。验证通过后可把 `commitTokenThreshold` 调回 20000。

---

## 步骤 3:🙋 装 Obsidian 插件(交给人)

**验证(已完成则跳过):**

```bash
lsof -nP -iTCP:27124 -sTCP:LISTEN >/dev/null 2>&1 && echo "已完成,跳过" || echo "需要人工操作"
```

**这一步 agent 做不了**,需要人在 Obsidian 界面里操作。请这样告知:

> 需要你在 Obsidian 里做三件事:
> 1. 装 **Local REST API with MCP** 插件。若 Obsidian 版本低于 1.13.1,
>    要装 **5.0.3** 而不是最新的 5.1.0(5.1.0 要求 ≥1.13.1,装了不会加载)
> 2. 切换到目标 vault,切换时同意「开启社区插件」
> 3. 完成后告诉我
>
> 插件会自动生成 API key,不需要你复制给我,我从配置文件读。

人做完后重跑上面的验证。

---

## 步骤 4:提取证书

**不要关闭 TLS 校验。** 把插件的自签证书提出来单独信任:

```bash
VAULT=<你的 Obsidian Vault 路径>
mkdir -p ~/.config/obsidian-rest
python3 - <<PY
import json, pathlib
src = pathlib.Path("$VAULT/.obsidian/plugins/obsidian-local-rest-api/data.json")
out = pathlib.Path.home()/".config/obsidian-rest/obsidian-local-rest-api.pem"
cert = json.loads(src.read_text())["crypto"]["cert"]
out.write_text(cert if cert.endswith("\n") else cert + "\n")
print("证书已写入", out)
PY
```

**确认(必须返回 200):**

```bash
VAULT=<你的 Obsidian Vault 路径>
KEY=$(python3 -c "import json;print(json.load(open('$VAULT/.obsidian/plugins/obsidian-local-rest-api/data.json'))['apiKey'])")
curl -s --cacert ~/.config/obsidian-rest/obsidian-local-rest-api.pem \
  -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $KEY" https://127.0.0.1:27124/vault/
```

返回 401 说明 key 读错了;返回证书错误说明 pem 提取失败,重跑本步。

> **不要把 API key 打印到对话里。** 用上面这种从文件读进变量的方式。

---

## 步骤 5:接 Obsidian 到 Claude Code

**验证(已完成则跳过):**

```bash
claude mcp list 2>&1 | grep -q obsidian-vault && echo "已完成,跳过" || echo "需要执行"
```

**执行:**

```bash
VAULT=<你的 Obsidian Vault 路径>
KEY=$(python3 -c "import json;print(json.load(open('$VAULT/.obsidian/plugins/obsidian-local-rest-api/data.json'))['apiKey'])")
claude mcp add obsidian-vault \
  --env AUTH_HEADER="Bearer $KEY" \
  --env NODE_EXTRA_CA_CERTS="$HOME/.config/obsidian-rest/obsidian-local-rest-api.pem" \
  -- npx -y mcp-remote https://127.0.0.1:27124/mcp --header 'Authorization:${AUTH_HEADER}'
```

`--header` 必须用**无空格**形式(`Authorization:${AUTH_HEADER}`)。带空格的写法
在某些客户端调用 `npx` 时不转义,值会被弄坏。

**确认:**

```bash
claude mcp list 2>&1 | grep obsidian-vault
```

---

## 步骤 6:🙋 重启(交给人)

配置改完必须让进程重新加载。**关窗口不算,必须完全退出。**

请这样告知:

> 请完全退出 Claude 桌面端(`Cmd+Q`,不是关窗口),然后重新打开,再告诉我。

人操作后**必须确认进程真的是新的**——旧进程可能变成孤儿继续跑:

```bash
ps -o pid,ppid,lstart,command -ax | awk '$2==1' | grep -E "Claude.app" | head -5
```

`lstart` 若不是今天,说明是孤儿,`kill` 掉再让人重开。

---

## 步骤 7:验收

**先让人正常用一会儿**(聊几轮,超过 `commitTokenThreshold`),然后**正常结束会话**
——`SessionEnd` 才触发沉淀。之后跑:

```bash
export PATH="$HOME/.local/bin:$PATH"
echo "--- 状态文件 ---"; ls ~/.openviking/codex-plugin-state/ 2>/dev/null
echo "--- 会话 ---";     ov session list 2>&1 | head -5
echo "--- 记忆 ---";     ov ls viking://user/default/memories 2>&1 | grep viking:// | head -5
echo "--- 日志 ---";     tail -20 ~/.openviking/logs/codex-hooks.log 2>/dev/null
```

**判读:**

| 现象 | 结论 | 下一步 |
| --- | --- | --- |
| 状态目录有真实 session id 的 json，`memories/` 非空 | ✅ 全通 | 把 `commitTokenThreshold` 调回 20000 |
| 状态目录只有 `unknown.json` 或为空 | ❌ 钩子没触发 | 回步骤 6 查进程;再查步骤 1 插件是否 enabled |
| 有状态文件但 `memories/` 空 | ⚠️ 追加成功、提交失败 | 看日志尾部的堆栈 |
| 完全没有日志文件 | ⚠️ 诊断开关没生效 | 回步骤 2,确认写的是 `ovcli.conf` |

**最终验收(端到端):** 写一条模型不可能猜到的事实进记忆,新开会话,
**不作任何提示**地提问,看它能否答对。这是唯一可信的验收方式。

---

## 排查顺序

卡住时按这个顺序查,**顺序反了会绕很多圈**。

1. **进程是不是新的** —— 常驻进程不热加载配置;应用退出后子进程可能变成
   `PPID=1` 的孤儿继续跑,看起来像重启了其实没有
2. **字段名是否是当前版本的** —— 插件文档可能落后于客户端。见开头第 2 条
3. **最后才怀疑功能本身**

本机曾两次把配置错误误判成能力缺失:据"没数据"断言 Codex 桌面端不支持钩子
(实际是照插件文档写了已废弃的 `plugin_hooks`,且钩子未被信任);
据"配置文件被覆盖"断言 Claude 桌面端不支持钩子(实际那是 **Chat** 的配置,
**Code** 的钩子一直是好的)。

**在穷尽前两项之前,不要下"不支持"的结论。**

---

## 可选:Claude Chat

Chat 侧只有 MCP,没有钩子,拿不到自动召回和自动沉淀。

**不要编辑 `claude_desktop_config.json`**——桌面端会在启停时重写它,手工加的
`mcpServers` 会被清掉(实测:写入后重启,该键完全消失)。只能在应用设置界面添加:

| 字段 | 值 |
| --- | --- |
| 名称 | `obsidian-vault` |
| 命令 | `npx` |
| 参数 | `-y mcp-remote https://127.0.0.1:27124/mcp --header Authorization:${AUTH_HEADER}` |
| 环境变量 | `AUTH_HEADER` = `Bearer <API key>`;`NODE_EXTRA_CA_CERTS` = pem 绝对路径 |

---

## 相关

- vault 写入规则:同目录 `AGENTS.md`
- 校验与晋升脚本:`scripts/`
