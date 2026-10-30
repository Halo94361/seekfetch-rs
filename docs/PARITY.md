# PARITY — seekfetch-rs → seekfetch-go 验收清单

逐条对照 `../seekfetch-rs` 的外部行为。Go 版达到 parity 的定义：**同一命令、同一输入，退出码与 stdout/stderr 一致**；凭据/缓存/配置文件与 Rust 版**字节兼容**（用户无需重新配置、缓存不失效）。

> 注：Go 重写已取消，本清单保留为 seekfetch-rs 外部行为的参考文档。

## 0. 原则

- 二进制名保持 `seekfetch`；仓库名 `seekfetch-go`。
- 行为兼容优先，重写的同一 PR 里不顺手改设计。
- 唯一允许的第三方依赖：`golang.org/x/crypto`（Argon2/XChaCha20）与 `github.com/pelletier/go-toml/v2`（TOML 解析），均为纯 Go、无 cgo。其余全部标准库。

## 1. CLI 命令面

| 子命令 | flag（默认值） |
|---|---|
| `search [QUERY]` | `-c/--count`(10, 1–50)，`--need-content`，`--max-chars`(300)，`--json`，`--min-rerank`，`--min-authority`，`--domain`(可重复)，`--no-cache`，`--refresh`(与 no-cache 互斥)，`-o/--output` |
| `fetch [URLS...]` | `--input <PATH>`，`--max-chars`(20000)，`--json`，`--no-cache`，`--refresh`，`-o/--output` |
| `balance` | `--json`，`-o/--output` |
| `cache` | `stats` / `clear` / `path` / `list [KEYWORD]` / `show KEYWORD [--max-chars 2000]` |
| `usage` | `--json`，`--since YYYY-MM-DD`，`--until YYYY-MM-DD`，`--last 30d\|4w\|6m\|1y`（since/until 与 last 互斥），`--group-by day\|week\|month` |
| `config` | `set-key [--stdin\|--from-env NAME]`，`import --dotenv PATH [--name DEEPSEEK_API_KEY]`，`show`，`path`，`clear` |
| `completions <bash\|zsh\|fish\|...>` | 输出补全脚本 |
| `mcp` | stdio MCP server |
| `doctor` | 诊断 |

stdin 行为：`search` 省略 QUERY 且 stdin 为管道时从 stdin 读；`fetch` 无 URL 参数且 stdin 为管道时逐行读 URL；`--input` 文件逐行读，空行与 `#` 注释忽略。`set-key` 在非终端 stdin 时从 stdin 读 key。

过滤：`--min-rerank`/`--min-authority`/`--domain`（匹配子域）在拿到结果后应用，CLI 与 MCP 一致。

## 2. 退出码

`0` 成功；`1` 其他（配置/凭据/IO）；`2` 用法错误；`3` Network；`4` Timeout；`5` Auth；`6` Payment；`7` RateLimited；`8` Api；`9` fetch 批量部分失败。

## 3. 配置文件

- 路径：`$XDG_CONFIG_HOME/seekfetch/config.toml`（Linux 默认 `~/.config/seekfetch/`）。
- 字段与默认值：`base_url`=`https://api.deepseek.com`，`timeout_s`=35，`default_count`=10，`default_max_chars`=300，`min_interval_ms`=200，`[cache] enabled`=true/`dir`=平台缓存目录 override 可空/`ttl_s`=86400，`[usage] enabled`=true。
- 文件不存在 = 全部默认值；解析失败报错退出 1。

## 4. 凭据文件（字节兼容，关键）

单一 provider：

- DeepSeek：文件 `credentials`，env 覆盖 `DEEPSEEK_API_KEY`

（Rust 版的 OpenCode provider 随重写移除，`credentials-opencode` 文件不再读写，遗留在磁盘上无害。）

文件格式（单行 base64）：`b64(版本字节 0x01 ‖ 24 字节随机 nonce ‖ XChaCha20-Poly1305 密文)`。

机器绑定 key：`Argon2id(material = machine_id + "\n" + user_name, salt = "seekfetch credential v1", m=19456 KiB, t=2, p=1, outlen=32)`。
- `machine_id`：Unix 依次读 `/etc/machine-id`、`/var/lib/dbus/machine-id`，都失败回退 `"unknown-machine"`；Windows 用 `%COMPUTERNAME%`。
- `user_name`：`$USER`，否则 `$USERNAME`，否则 `"unknown-user"`。
- Go 实现：`argon2.IDKey(material, salt, 2, 19456, 1, 32)` + `chacha20poly1305.NewX(key)`。
- env 覆盖忽略空白值；文件权限 0600、目录 0700。
- `mask_key`：长度 ≤8 显示 `****`，否则 `****`+末 4 字符。
- 验收：不重新 set-key，直接用 Go 版解密现有凭据文件成功。

## 5. DeepSeek API client

- 唯一出口函数（等价 `request_json`）：所有请求走它，进程级 `min_interval` 节流（缓存命中绕过），`Authorization: Bearer`，UA `seekfetch/<version>`。
- 端点：`POST /v1/tools/search` `{query, count, need_content}` → `{results: [SearchResult]}`；`POST /v1/tools/fetch` `{url}` → `{title, content}`；`GET /user/balance` → `{is_available, balance_infos: [{currency, total_balance, granted_balance, topped_up_balance}]}`（数字或字符串都接受）。
- `SearchResult` 字段：title, url, summary?, content?, snippet?, rerank_score?, authority?, site_name?, date?（空字符串视作无该字段；序列化时省略）。
- 重试：最多 4 次尝试；退避 `500ms × 2^(attempt-1)`；`Retry-After` 响应头优先于退避；可重试状态码 408、429、5xx **除 504**；传输层错误仅在 connect/request 错误（非超时）时重试。
- 错误映射（→ 退出码）：401/403→Auth(5)；402→Payment(6)；404→Api(8)；429（4 次后仍失败）→RateLimited(7)；504→Api(8)；其他→Api(8)。响应体解析 `{error:{message, code, type}}`；非 JSON 取原文截 200 字符；错误文本中 scrub API key。
- 超时：整体 `timeout_s`（35s），超时错误文案带 `timeout_s`。

## 6. 缓存

- 根目录：`$XDG_CACHE_HOME/seekfetch`（Linux `~/.cache/seekfetch`），可被 `[cache] dir` 覆盖。
- 布局：`<dir>/search/<key>.json`、`<dir>/fetch/<key>.json`。
- 条目 JSON：`{created_at: unix秒, request: <原始请求JSON>, payload: <响应JSON>}`。
- key：`fnv1a-64("search\0{normalized}\0{count}\0{need_content}")`；fetch 为 `fnv1a-64("fetch\0{url.trim()}")`；`normalized` = 查询转小写、空白折叠为单空格。
- TTL 86400 秒惰性过期（读到过期条目顺手删除）。缺失/损坏/过期一律视为 miss；**缓存 IO 失败绝不使请求失败**。
- 模式：Normal（读穿透+miss 写入）/ NoCache（完全不读写）/ Refresh（跳过读、覆盖写）。

## 7. usage 日志

- 路径：`$XDG_DATA_HOME/seekfetch/usage.jsonl`（Linux `~/.local/share/seekfetch/`）。
- 每行一个 JSON：`{ts: unix秒, kind: "search"|"fetch"|"balance", ok: bool, cached: bool}`；追加写，失败静默；`[usage] enabled=false` 不写。
- 成本估算：每 1000 次「成功且未命中缓存」的 search/fetch = 5.0 CNY；失败与缓存命中不计；balance 不计费。
- `usage` 命令：since/until/last 过滤、group-by 日/周/月汇总、人类可读 + `--json`。

## 8. 输出格式

- search 人类可读：空结果输出 `no results\n`；否则逐条 `N. {title}\n   {url}\n   {truncate(summary,max_chars)}\n   {truncate(content,max_chars)}\n\n`。`--json` 为结果数组 pretty JSON。
- fetch 人类可读：多条之间以 `\n---\n\n` 分隔；每条 `{title}\n{url}\n\n{truncate(content,max_chars)}\n`。失败的 URL 输出错误（见 source `output.rs`）。
- balance 人类可读：`available: yes|no\n` + 各币种 `{currency}: total {total} (granted {granted}, topped-up {topped})\n`（空列表输出 `no balance information`）。
- `-o/--output` 写文件代替 stdout。
- MCP 工具输出必须截断（不把原始 API JSON 灌进模型上下文）。

## 9. MCP server（stdio JSON-RPC 2.0）

- `initialize` 返回 `serverInfo.name = "seekfetch"`。
- 三个工具（schema 与 Rust 版完全一致，见 `mcp.rs tool_definitions`）：
  - `deepseek_search`：required `query`；count(1–50, 默认 config.default_count)、need_content(false)、max_chars(默认 config.default_max_chars)、min_rerank、min_authority、domain、no_cache、refresh。
  - `deepseek_fetch`：`url` 或 `urls`（二选一；urls 最多 **20** 条）；max_chars 默认 **20000**；no_cache、refresh；批量内单条失败不中断，逐条上报。
  - `deepseek_balance`：无参数。

## 10. markdown 清洗（clean.rs 移植）

- 散文中的 `\` + ASCII 标点 取消转义；围栏代码块（` ``` `/`~~~`，≥3）保持原文，块内若含疑似转义产物则在块首注入一行 `[seekfetch note] this code block may contain extractor escape artifacts (e.g. \\( or \\{ standing for the literal character); interpret them by meaning`。
- 未闭合围栏、行尾换行保持等边界行为以 Rust 版单测为准（原样移植其测试用例）。

## 11. doctor 输出

依次打印：config 路径、credentials 路径、base_url、cache dir（或错误）、api key 状态（configured/错误摘要）、connectivity（调 balance 实测，ok/错误摘要）。

## 12. 对拍验证方案

1. `go build` 后并排放 `seekfetch-rs` 目标二进制与 Go 版（改名 `seekfetch-go`）。
2. 同一组 golden case（search 带各 flag 组合、fetch 单条/批量含失败 URL、balance、usage 各过滤、cache 各子命令、config show/path、doctor）跑两版，`diff` stdout/stderr 与退出码。
3. 凭据：Go 版直接解密 Rust 版写出的文件，两边 `balance` 对拍。
4. 缓存：Rust 版写入的条目，Go 版 `cache list/show` 能读且命中。
5. MCP：stdio 脚本化握手（initialize → tools/list → tools/call 各工具）比对响应 JSON。
6. 全部通过后替换 `~/.local/bin/seekfetch`，Rust 版归档 `~/repo/archive/seekfetch-rs`，观察期一周。

## 13. 显式不做的改动（parity 期间）

- 不改 CLI 名称/flag/默认值/输出格式。
- 不改凭据、缓存、usage 的文件格式与路径。
- 不新增功能（`serve` 子命令留到 parity 达成后的第二阶段）。
- 移除 OpenCode Go 功能（`opencode`/`oc` 子命令、`opencode_usage` MCP 工具、`credentials-opencode` 凭据）——Go 版直接不实现；Rust 版在替换前保持原样。
