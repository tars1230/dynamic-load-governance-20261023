# 动态负载治理方案（2026-10-23）

只写方案。这次没有改配置、没有装软件、没有杀进程、没有重启任何客户端。没有打开 Hermes 目录，没有读密钥明文。

机器是 Mac Mini M4、16GB 内存。2026-10-22 深挖时 swap 曾到约 9.3G / 10G，空闲内存大约 39%。根子是 Electron / AI 客户端开得太多，再加上同名 MCP 各起一份。当时 rea 有 7 份。材料是 `~/.shared/research/mac-load-deepdive-20261022.md`、`~/.shared/backups/mcp-dedup-20261022/`、现行的 `stuck-process-watch`（已按你的决定只提醒）、以及 mac-tiered-curator 的体检和周报。

## 先看结论

1. 共享实例已经有一个正确样板：basic-memory 由 launchd 单实例听 `127.0.0.1:8778`。继续沿这个样板做，不要再套一层 MCP 网关。
2. 网关类开源项目大多是「每个客户端再起一份 stdio」，省不了内存，16GB 机器上不该装。
3. 监控挂到现有的每小时体检上，只写日志和周报。swap 的弹窗继续用已经在跑的 runtime-guard，避免同一件事响两次。
4. 「ppid=1 就清掉」会误伤这台共享 basic-memory。它的父进程就是 1，而且是故意常驻的。
5. 可自动清的孤儿只限严格正向白名单：可执行路径、cwd、启动参数特征，三选一或组合命中才可以清。未知的 ppid=1 只进周报，绝不杀。开关默认关，和已经定下的「stuck-process-watch 只提醒、不杀」一致。真要开，必须再问一次用户（重大风险）。按总监察审核修订。

## 这次只读看到的现状

采样时间大约是 2026-10-23 00:20（UTC+8）。和昨晚 22:20 不是同一时刻，数字会变，规则按坏的那次来设计。

| 项 | 昨晚深挖 | 这次 |
|---|---|---|
| swap | 约 9.3G / 10G | 4099.94M / 5120.00M |
| `memory_pressure` 空闲百分比 | 约 29%–51%，平均约 39% | 55% |
| `kern.memorystatus_vm_pressure_level` | 未记 | 1 |
| rea | 7 份，大约 1.55G footprint | 命令行匹配到 4 份；1 份的父进程在 Hermes 下，排除。另外 3 份挂在 Cursor / cursor-agent 上 |
| basic-memory | 3 份 stdio，大约 0.43G | launchd 单实例 PID 18314 在听 8778；另有 1 份 stdio，父进程是已运行约 14 小时的 cursor-agent |

空闲页数现在大约 7284。旧体检把「空闲页 < 20000」当成内存告警，这一条今晚仍会写 health 日志，但空闲百分比是 55%。页数和百分比不是一回事。新规则只用百分比和 swap 用量。

`kern.memorystatus_vm_pressure_level=1` 这次和 55% 空闲同时出现。公开资料对这个数字有两套枚举。落地前用正常、偏紧、很紧三种状态各读一次，对上 `memory_pressure` 的空闲百分比，再决定它能不能当报警线。在对上之前，它只做记录。

进程归属（只看 pid、父进程、运行时长、程序名）：

- basic-memory 共享实例：PID 18314，ppid=1，在 `launchctl list` 里，标签 `com.chengchen.mcp-shared.basic-memory`，只听本机 8778。plist 的命令是 `basic-memory mcp --transport streamable-http --host 127.0.0.1 --port 8778 --path /mcp`，KeepAlive，日志在 `~/Library/Logs/mcp-shared/`。这是要留的。
- 另一份 basic-memory：父进程是 cursor-agent（已跑约 14 小时）。`~/.cursor/mcp.json` 里 basic-memory 已经是 `http://127.0.0.1:8778/mcp`。改文件不会收回已经拉起的 stdio 子进程。
- rea 三份的父进程分别是两个 cursor-agent 和一个 Cursor 的 `mcp-process`。都有活着的父进程。Codex 的配置里 rea 已是 `enabled = false`，这次没看到 Codex 再拉一份。
- WorkBuddy 主进程下还有 `edgeone-pages-mcp` 和 `cloudbase-mcp` 的 npm 子进程（约 1 小时）。`~/.workbuddy/connectors/default/mcp.json` 里这两条标成关闭。文件和正在跑的进程不一致。处置以父进程为准。
- 这次没有看到 ppid=1、又不在 launchd 里的 MCP / broker。Codex broker 这种孤儿今晚不在。再出现时，未命中正向白名单的只进周报，绝不杀。按总监察审核修订。
- Hermes 下面有子进程。按约定不打开 `~/.hermes`，不列入可清理名单，也不在这里展开命令行。

## 1. 预防：一份注册表，外加漂移检测

### 注册表管什么

现在的 `~/.shared/mcp-servers.json` 不能直接当唯一来源。它里面的 basic-memory 仍是 stdio，还带着 rea。`~/.shared/sync-mcp-config.sh` 会把这份内容整段写进 `~/.claude.json` 的 `mcpServers`，把 Claude 现有条目换成旧的。写死：`sync-mcp-config.sh` 在 `mcp-registry.json` 就绪并校验通过之前禁止再跑。落地时要在脚本头或同目录加一行禁用说明，避免误跑把 Claude 的 basic-memory 打回 stdio。校验通过之前，不恢复这条同步。按总监察审核修订。

新注册表以后放在 `~/.shared/mcp-registry.json`（这次不创建）。它只记四件事：服务器名字、期望形态、哪个文件该怎么写、哪些客户端该跟上。期望形态只有三种：

- `shared-http`：本机只留一份，客户端写 URL。basic-memory 用这个，地址固定 `http://127.0.0.1:8778/mcp`。
- `stdio-per-client`：每个正在用的客户端自己拉一份。rea 用这个。它带分析会话，多客户端拧在同一个进程上会互相踩。
- `remote-http`：本来就是别人的网址，本机不起进程。sorftime、apify 属于这种。

注册表是「该是什么样」。各客户端自己的配置文件仍是它们真正读取的地方。检测器只读、只报告，不回写。

### 各客户端配置文件在哪

2026-10-23 只读核对过。表里的「现在」只记传输方式和服务器名字，不记参数、环境变量和网址上的查询串。

| 客户端 | 配置文件 | 这次读到的 |
|---|---|---|
| Cursor | `~/.cursor/mcp.json` | basic-memory → 本机 8778；rea → stdio |
| Claude Code | `~/.claude.json` 顶层 `mcpServers` | basic-memory → 本机 8778；sorftime、apify → 远程 HTTP；playwright-cli、playwright-mcp、douyin-mcp、minimax-mcp、minimax-coding-plan-mcp → stdio。没有 rea |
| Claude Code 用户设置 | `~/.claude/settings.json` | 没有 MCP 块 |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` | 只有 preferences，没有 MCP |
| WorkBuddy 用户 MCP | `~/.workbuddy/mcp.json` | 8 条都是关闭。其中 basic-memory 仍写成 stdio，没有改成 8778 |
| WorkBuddy 连接器 | `~/.workbuddy/connectors/default/mcp.json` | 29 条都标关闭。里面有 edgeone、cloudbase 两条 stdio。进程仍在，见上 |
| WorkBuddy 市场目录 | `~/.workbuddy/connectors-marketplace/connectors/*/mcp.json` | 上百个模板。不进注册表，不参与漂移 |
| Codex | `~/.codex/config.toml` 的 `[mcp_servers.*]` | node_repl 开着；rea、cua_repl、computer-use 关着。没有 basic-memory |
| Cline | `~/.cline/data/settings/` | 只有很小的 global-settings 和 providers。没有 `cline_mcp_settings.json`。扩展常见路径 `~/Library/Application Support/Cursor/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json` 这次也不存在 |
| Kilo | `~/.config/kilo/kilo.jsonc` | 有文件，顶层没有 MCP |
| OpenCode | `~/.config/opencode/opencode.jsonc` | 有文件，顶层没有 MCP |
| agy / Antigravity | `~/.gemini/settings.json` | 只有 basic-memory → 本机 8778。`~/.gemini/antigravity/` 下两层内没有另一份 mcp 配置 |
| 家目录 | `~/.mcp.json` | 只有 sorftime。哪个程序会读它，这次没证实，标成待确认 |
| 另一份同步副本 | `~/Library/Application Support/mcp-synchro/mcp.json` | basic-memory 和 rea 都还是 stdio，而且是开的。这是漂移源 |
| 旧的共享清单 | `~/.shared/mcp-servers.json` | 6 条。basic-memory 仍是 stdio，rea 仍在。和 Claude / Cursor 的现网不一致 |
| MiniMax 自带 | `~/.minimax/mcp/mcp.json` | matrix、playwright 为 stdio；cu、trash 指向本机 15321。算 MiniMax 自己的，不并进共享注册表 |

项目里还会有第二层，这次见到的例子：

- `xhs-mcp-src/.cursor/mcp.json`、`xhs-mcp-src/.vscode/mcp.json`：xiaohongshu，本机 18060
- `analysis/deepagents/.mcp.json`：两个 langchain 文档站
- `plugins/cowart/.mcp.json`：cowart stdio
- `homebrew/.vscode/mcp.json`：`brew mcp-server`

漂移检测只扫一份目录清单（例如常见项目根），深度限死，跳过 `node_modules`、缓存、`~/.hermes`、云盘。不要对整个家目录做 `find`。这次为了找文件，家目录深度 5 的搜索跑了大约一分钟，已停掉那个搜索本身。

Hermes 的 MCP 不进这张表，也不读它的配置。

### 漂移检测报什么

每天一次就够，挂在已有的 03:40 体检之后、用只读方式跑。比较的是「名字 + 形态 + 本机端口」，不比较密钥。

算漂移的情况：

- 注册表写了 `shared-http`，客户端文件仍是 stdio。当前已有：共享清单、mcp-synchro、WorkBuddy 的 `mcp.json`。
- 文件已经是 HTTP，机器上仍有同名 stdio 子进程。当前已有：cursor-agent 底下那份 basic-memory。
- 文件写关闭，父进程底下还有这个 MCP。当前已有：WorkBuddy 的 edgeone、cloudbase。
- 同一个本地服务器，stdio 份数比「开着的客户端数」还多。
- 有人跑了旧的 `sync-mcp-config.sh`，Claude 的 basic-memory 从 8778 变回 stdio。检测器看到这个要单独标成「回退」。

不算故障的情况：

- 两个正在用的客户端各有一份 rea。这是 `stdio-per-client` 的正常状态。
- 远程 HTTP 服务器没有本地进程。
- 项目目录里的专用 MCP（某个仓库自己的文档服务器）不在共享注册表里。报告放在「项目自带」一节，不报警。

检测结果写成一份短记录，路径建议 `~/.l0/logs/mcp-drift-YYYY-MM-DD.md`。有漂移就进周报。没有漂移就一行 `drift=0`。不要为 drift=0 弹通知。

basic-memory 的活体检查沿用已有脚本 `~/.shared/backups/mcp-dedup-20261022/check-basic-memory-http.sh`（工具数 23 为正常）。它只对 127.0.0.1 做 initialize 和 tools/list。共享实例没在听，才算失败。

回滚入口已经有了：`~/.shared/backups/mcp-dedup-20261022/ROLLBACK.md`。以后再改任何客户端 MCP 文件，先按这个目录的方式 `cp -p` 备份并记 SHA-256。

## 2. 监控：负载体检

不新开常驻进程。本机已经有三层，职责分开：

| 现有入口 | 节奏 | 已经在做的 | 负载治理怎么用它 |
|---|---|---|---|
| `com.chengchen.l0.runtime-guard` | 每 600 秒 | 记 swap、VM、数据盘；swap > 4096MiB 记 warn，≥ 8192MiB 记 critical；原因变化才弹一次通知。它还会在特定条件下停掉本地 ASR 和 UU 远控助手 | swap 继续只走这里。不要把「杀 MCP」写进这个脚本 |
| `com.chengchen.stuck-process-watch` | 每 3600 秒 | 只提醒。卡住的 cline、agent-browser、无头 Chrome 才弹通知。脚本里已经没有 kill | 在这里加一行体检：同名 MCP 个数、闲置客户端、孤儿候选。平时只写日志 |
| curator `auto-safe` | 每天 03:40，由 `l0_hygiene.sh` 调用；周一 04:00 还有 Antigravity sidecar 读周报 | 冷归档，并写 `weekly-hold-日期.md`。只有系统盘已用 ≥ 82% 才通知 | 负载周报学它：小事进周报，过线才通知。不要把进程名单塞进磁盘周报，两件事分开 |

### 五个检查项

| 检查 | 多久看一次 | 数据从哪来 | 记下来 | 写进周报 | 弹通知 |
|---|---|---|---|---|---|
| swap 用量 | 沿用每 10 分钟 | `sysctl vm.swapusage` | 始终由 runtime-guard 记 | swap 连续高于 6G 才在周报提一句 | 沿用现有线：超过 4G 只在「原因刚变成 swap」时通知一次；达到 8G 记成更严重的那一档。已经长期停在 4G 以上时，不每小时再响 |
| 内存空闲百分比 | 每小时，和 MCP 个数一起采 | `memory_pressure` 里的 `System-wide memory free percentage` | 每小时一行 | 低于 40% 且当次采样如此 | 低于 20%，并且这一轮是新出现的，才通知一次。恢复到 40% 以上算这次事件结束 |
| 同名本地 MCP 个数 | 每小时 | `ps` 的 pid、ppid、etime、pcpu，加上程序名。按服务器名字归类。父进程属于 Hermes 的单独计数、不参与后面的处置 | 个数 ≥ 2 就写进小时日志 | 个数 ≥ 3 | 个数 ≥ 4，同一次事件只通知一次。rea 到过 7，所以 4 是通知线，2 是正常的 |
| 闲置客户端 | 每小时 | 客户端进程 etime ≥ 8 小时，且这次采样 CPU < 1%，子 MCP 也没有 CPU ≥ 1% | 写入小时日志 | 写进周报，点名是哪个客户端、跑了多久 | 不通知。夜里开着 Codex / Cursor 是正常的 |
| ppid=1 | 每小时 | 先分两类。命中正向白名单（可执行路径 / cwd / 启动参数特征，三选一或组合）且满足第 3 节其余条件，才算可清候选。对不上白名单的算未知 | 两类都写小时日志 | 两类只要出现就进周报 | 未知的绝不杀、不通知。白名单候选在开关关闭时也不杀、不通知。开关打开并真的清掉之后，周报留一行，不另做弹窗。按总监察审核修订 |

空闲百分比的三档，对应 curator 看磁盘的方式：40% 及以上当正常，20%–40% 当偏高（只进周报），低于 20% 当要出声。昨晚平均约 39%，所以 40% 会让「昨晚那种状态」进周报，但不会弹窗。20% 比昨晚最差的 29% 更紧，避免百分比本身上下跳动时一直响。

swap 文件缩小得很慢。压力过去之后，used 仍可能停在几 GB。所以「swap 还高」只说明曾经紧过，不能单独当成现在就要收拾进程。现在紧不紧，看空闲百分比。

小时日志建议追加到已有的 `~/Library/Logs/stuck-process-watch.log`，行首用 `CENSUS`，和现有的 `ALERT` 分开。一行里只放数字和名字：时间、swap、空闲百分比、每个 MCP 名字的份数、孤儿候选个数。不写完整命令行，避免把 token 写进日志。

周报建议写到 `~/.l0/logs/load-hold-YYYYMMDD.md`，每天覆盖当天这一份。格式学 curator 的周报：一张表，列的是「留着没动的原因」。周一 04:00 那条 sidecar 以后可以多读这一份，仍然只有过线才通知。这次不改 sidecar。

## 3. 处置分级

三类，互不升级。

### 以后可以自动清（开关默认关）

按总监察审核修订。自动清只走严格正向白名单。对不上白名单的 ppid=1 是未知进程：只进周报，绝不杀。

必须全部满足：

1. ppid=1。
2. 这个 pid 不在 `launchctl list` 里。共享 basic-memory 会被这一条保下来。
3. 命中事先写死的正向白名单。匹配只用这三样，三选一或组合都算命中：可执行文件的绝对路径、当前工作目录 cwd、启动参数里的特征。对不上的一律当未知。
4. 已经连续活过 6 小时，并且连续两个小时采样的 CPU 都低于 1%。两次采样只确认它闲着。
5. 开关是开的。默认关。真要开，必须再问一次用户（重大风险）。以前的点头不能代替这一问。

白名单在本方案里先是空的。没有写明特征的进程，包括复核里举例的 `node_repl` 和临时 broker，现在都不能清。要加一条，得单独写明路径、cwd 或参数特征，并再经用户点头。反向排除（「不是某几个客户端就可以清」）不再使用。

未知的 ppid=1：只写小时日志和周报，绝不杀。开关开或关都不改变这一条。

清之前（仅白名单命中且开关已按重大风险再次获准打开）追加一行日志：时间、pid、程序名、命中的是哪条白名单、etime、当时的 swap 和空闲百分比。然后发 SIGTERM，等 10 秒还在，再 SIGKILL。不删它的配置，不删数据。客户端下次自己会再拉起一份，这就是「可恢复」。不要由体检脚本替它重启，免得又多一份。

今晚的共享 basic-memory 不满足第 2 条，不会被清。白名单为空，今晚也没有任何进程可以被清。

### 只提醒

- 同名 MCP 有 3 份及以上，但每一份都还有活着的客户端父进程。rea 就是这种。关掉多出来的客户端会话，份数会自己下去。
- 文件和进程不一致（HTTP 配置下还留着 stdio，或文件写了关闭进程还在）。
- 客户端闲置超过 8 小时。
- 空闲百分比在 20%–40%。
- 未知的 ppid=1。只进周报，绝不杀。按总监察审核修订。
- 已写进白名单、但杀开关仍关着的候选。只进周报，不杀。
- stuck-process-watch 已经在提醒的卡住 cline、agent-browser、无头 Chrome。保持只提醒。

### 一律不动

- 正在用的客户端：Cursor、cursor-agent、WorkBuddy、Codex / ChatGPT、Antigravity、OpenCode、Cline、Kilo、Grok，以及它们的子 MCP。判断标准是父进程还在，不看 CPU 低不低。CPU 低只说明这一分钟没算东西，会话可能还开着。
- Hermes 自身和它的全部子进程。
- launchd 管着的共享实例，包括 basic-memory 的 8778。
- 浏览器、123 云盘、远控和网络进程。runtime-guard 里已有的 ASR / UU 止损保持原样，不扩到 MCP。

## 4. 报警怎么走，才不吵

学 curator 已经在用的规矩，不新开一条通知通道。

- 每小时的正常采样只写 `CENSUS` 日志。
- 需要人知道、但不用马上动手的，写进 `load-hold-日期.md`。磁盘那份周报继续只谈磁盘。
- 弹窗只用现成的 `osascript` 通知，和 runtime-guard、curator 一样。文案只含名字和数字，不带按钮，不带「要不要杀掉」。
- 同一种严重状态只在「刚跨过线」时响一次。恢复以后再跨线，才允许再响。stuck-process-watch 曾经因为阈值太容易达到，每小时都写告警，告警就没有意义了。负载通知沿用「状态变化才响」。
- swap 的弹窗不新做。runtime-guard 已经在做，而且会在原因变化时通知。再挂一条，4G 以上的日子会响两遍。
- 会新响的只有两件：空闲百分比掉到 20% 以下；同名本地 MCP 达到 4 份。两件都是一次事件响一次。
- 闲置客户端、3 份 MCP、配置漂移、未知的 ppid=1，以及开关仍关着的白名单候选，只进周报。未知的 ppid=1 即使以后开关打开也绝不杀。按总监察审核修订。

周一 04:00 的 sidecar 现在只在系统盘 ≥ 82% 时通知。负载通知不要塞进那个磁盘句子里。等体检跑顺两周，再考虑让 sidecar 多读一份 load-hold，仍然只在上表「弹通知」那一列有新事件时出声。

## 5. 现成工具：能借鉴的借鉴

查的是 2026-10-23 的公开仓库。星数是大约值。结论：basic-memory 这种「程序自己会说 HTTP」的，用 launchd 留一份就够了。stdio 网关解决不了「7 份 rea」这种重复。

### MCP 代理 / 网关

| 项目 | 地址 | 许可 | 和这台机器的关系 |
|---|---|---|---|
| sparfenyuk/mcp-proxy | https://github.com/sparfenyuk/mcp-proxy | MIT | stdio 和 Streamable HTTP 互转，大约 2800 星。basic-memory 自己已经会听 8778，不需要再套它 |
| punkpeye/mcp-proxy | https://github.com/punkpeye/mcp-proxy | MIT | 把一条 stdio 命令挂到端口上，大约 300 星。一条 stdio 同时接多个客户端时，第二次 initialize 会踩会话 |
| supercorp-ai/supergateway | https://github.com/supercorp-ai/supergateway | MIT | 大约 2900 星。2026 年的修复是每个 SSE 会话一份 Server（issue 113，v4.0.0）。份数随客户端增加，和去重相反 |
| tbxark/mcp-proxy | https://github.com/tbxark/mcp-proxy | MIT | 大约 700 星。把好几个不同服务器合成一个 HTTP 入口，方便少写几份配置。它不负责把同名进程收成一份 |
| acehoss/mcp-gateway | https://github.com/acehoss/mcp-gateway | MIT | 大约 130 星。每个会话一条独立 stdio。这是多实例，不是共享实例 |
| stacklok/toolhive | https://github.com/stacklok/toolhive | Apache-2.0 | 大约 2000 星。用容器跑 MCP。官方说明 stdio 一次只服务一个客户端，多客户端要再起容器。16GB 上会更重 |
| kandobyte/mcpbox | https://github.com/kandobyte/mcpbox | Apache-2.0 | 很新，星数接近 0。同样是把 stdio 暴露成 HTTP。不采用 |
| shigechika/mcp-stdio | https://github.com/shigechika/mcp-stdio | MIT | 大约 10 星。双向网关，给远程服务器用。不解决本机重复进程 |

ToolHive 的 issue 428 把限制说清楚了：stdio 不是给多个客户端共用的。代理如果为了不踩会话而「每个客户端起一份」，内存和现在一样；如果硬共用一份，rea 这种有会话的服务器会串台。

值得照抄的是已经落地的那一招：服务器自己支持 Streamable HTTP，就用一个 launchd 代理听 `127.0.0.1`，客户端只写 URL。basic-memory 已是这样。下一个候选必须同时满足：自己会听 HTTP、工具没有「这份进程里的分析会话」、关掉重复进程不会弄丢正在做的事。rea 不满足第二条，继续每个正在用的客户端一份，份数到 3 进周报、到 4 才通知。

### macOS 进程和内存监控

| 项目 | 地址 | 许可 | 怎么用 |
|---|---|---|---|
| exelban/stats | https://github.com/exelban/stats | MIT | 菜单栏监控，大约 4.2 万星。能看内存压力和 swap。再装一个常驻程序会占内存，不装 |
| gabrielbrrll/binnacle | https://github.com/gabrielbrrll/binnacle | MIT | 菜单栏建议器，星数接近 0。它的做法值得学：用系统自己的内存压力、每周一份 markdown、只有严重情况才通知。不装软件 |
| vladkens/macmon | https://github.com/vladkens/macmon | MIT | Apple Silicon 监控，大约 2000 星，不用 sudo，能输出 JSON。需要看功耗时手动能跑。不做成常驻 |
| nicolargo/glances | https://github.com/nicolargo/glances | LGPL-3.0 | 功能全。4.5 起自己还带 MCP 服务器，装上等于再加一份 MCP。不装 |
| tlkh/asitop | https://github.com/tlkh/asitop | MIT | 大约 4600 星。靠 `powermetrics`，要 sudo。不采用 |
| ClementTsang/bottom | https://github.com/ClementTsang/bottom | MIT | 终端里看的，大约 1.4 万星。给人临时看，不是体检 |
| aristocratos/btop | https://github.com/aristocratos/btop | Apache-2.0 | 同上，大约 3.5 万星。不挂成定时任务 |

体检直接用系统自带的 `sysctl vm.swapusage`、`memory_pressure`、`ps`、`launchctl list`、`lsof`。这些命令这次都跑过，足够回答「几份、谁的子进程、共享端口在不在听」。

## 以后落地的顺序

这次不做。用户点头之后按步做。下面五步仍可单独停。每一步做完，报项目总监察验收，通过再做下一步。按总监察审核修订。

落地第 1 步之前的前置（不算五步之一；没做完不能进第 1 步）：

0. 把 stuck-process-watch 里废弃的 `/Volumes/DataPool` 硬编码改到现行工作盘语义（databox、archive 等现行挂载）。改前把脚本备份到 `~/.shared/backups/`。这一步只改路径语义，不打开杀开关。

1. 写 `mcp-registry.json`，只描述期望形态。先改文档，不改各客户端文件。`sync-mcp-config.sh` 在这份注册表就绪并校验通过之前禁止再跑。落地这一步时，在脚本头或同目录加一行禁用说明，避免误跑把 Claude 的 basic-memory 打回 stdio。校验通过之前，不恢复这条同步。
2. 只读漂移检测，每天一行结果。跑一周，确认不会把项目自带 MCP 报成故障。
3. 给每小时的 stuck-process-watch 加 `CENSUS` 行。仍然不杀进程。空闲页 < 20000 那条 health 日志换成空闲百分比，避免 55% 空闲还在记告警。
4. 每天写 load-hold。连续两周没有误报之后，再谈要不要让周一 sidecar 读它。
5. 自动清孤儿的开关保持关闭。真要开必须再问一次用户（重大风险）。问过并同意之后，也只清第 3 节里命中正向白名单、且其余条件全部满足的进程。未知的 ppid=1 只进周报，绝不杀。

改客户端配置时，先备份再改，回滚用 2026-10-22 那份 `ROLLBACK.md` 的办法。正在跑的 stdio 子进程要等那个客户端自己退出；体检不替用户关 Cursor、cursor-agent 或 WorkBuddy。

## 附录：DataPool（只读，2026-10-23 00:23 左右）

| 检查 | 结果 |
|---|---|
| `/Volumes/DataPool` 是否存在 | 不存在。`ls` 报 No such file or directory |
| 是否挂载 | 没有。`mount` 输出里没有 DataPool。`os.path.ismount` 为假 |
| `diskutil info /Volumes/DataPool` | 退出码 1，首行是 `Could not find disk: /Volumes/DataPool` |
| `~/DataPool` | 符号链接，`ls -ld` 显示 `Oct 12 10:49`，指向 `/Volumes/databox` |
| 链接目标 | `/Volumes/databox` 存在，是目录。`mount` 有一行：`/dev/disk8s1 on /Volumes/databox (apfs, local, nodev, nosuid, journaled, noowners)` |

`~/DataPool` 这个名字现在落到 databox，不是落到已经不在的 DataPool 卷。体检里如果还用「`/Volumes/DataPool` 在不在」判断外置盘，会一直得到「没挂上」。要看这块盘，看 `/Volumes/databox`。stuck-process-watch 里仍有一段：DataPool 不在挂载表上时，发现命令行以 `/Volumes/DataPool/` 开头就提醒。卷确实不在，这段提醒的前提还在；它现在不会因为「软链还指着旧盘」而响，因为软链已经改指向 databox。

按总监察审核修订：上面是 00:23 的只读事实，硬编码还在，这次不改脚本。落地第 1 步之前先改 stuck-process-watch，把废弃的 `/Volumes/DataPool` 换成现行工作盘语义（`/Volumes/databox`、`/Volumes/archive` 等现行挂载）。改前备份到 `~/.shared/backups/`。

## 复核意见（agy Gemini 3.8 Flash，2026-10-23 02:05 Asia/Shanghai）

判定：PASS_WITH_NOTES

作为独立复核人，对 [`dynamic-load-governance-20261023.md`](file:///Users/chengchen/.shared/research/dynamic-load-governance-20261023.md) 评审如下：

---

### (a) 总判定

**PASS_WITH_NOTES**

**核心评价**：方案保持了极高的工程克制（只读深挖、不改配置、不杀进程、不增设重型常驻、不碰 Hermes 目录与密钥），针对 16GB 内存瓶颈给出的「launchd 共享 HTTP + 其余 stdio 单独保留 + 漂移只读检测」技术路线完全契合实际。孤儿处置和历史磁盘路径两处，按总监察审核修订后的现行口径见正文第 3 节、落地顺序和文末补丁。

---

### (b) 相对用户既定规则的风险与缺口

1. **低决策清理不打扰用户**：
   - **符合度高**：小事（闲置客户端、3 份以内的 rea、配置漂移）全部收纳进周报；swap 弹窗只保留现有 `runtime-guard` 入口，避免同一事件多次响铃。
   - **潜在缺口**：方案中自动清理发生时“在周报里留一行，不另做弹窗”。若未来开关被开启，缺乏低侵入式的即时留痕（如标准系统日志通知），用户可能将清理误判为客户端偶发崩溃。
2. **仅在安全时自动**：
   - **已按总监察审核修订**：自动清只走严格正向白名单。未知的 ppid=1 只进周报，绝不杀。详见正文第 3 节和文末补丁。
3. **不碰 Hermes**：
   - **基本合规**：方案明文规定不打开 `~/.hermes`、不展开命令行、不收纳进注册表。
   - **已按总监察审核修订**：未知的 ppid=1 一律只进周报、绝不杀。Hermes 自身及其仍挂在 Hermes 下面的子进程继续一律不动。
4. **孤儿自动清杀开关默认关**：
   - **完全合规，并已加严**：结论、检查表、第 3 节和落地顺序都写明开关默认关。按总监察审核修订：真要开必须再问一次用户（重大风险）。复核原文里的「条件 6」是当时的编号，现行开关是第 3 节第 5 条。
5. **DataPool / 冷存储规则**：
   - **事实核对准确**：附录明确核实 `/Volumes/DataPool` 卷已物理移除不存在，且证实 `~/DataPool` 现指向 `/Volumes/databox`。方案全文未对任何存储卷执行格式化或写入操作。
   - **已按总监察审核修订**：`stuck-process-watch` 里废弃的 `/Volumes/DataPool` 硬编码，列为落地第 1 步之前的前置。改到现行工作盘语义（databox、archive 等现行挂载），改前备份到 `~/.shared/backups/`。这次仍不改脚本。

---

### (c) 具体改进建议（可落地、可核对）

1. **孤儿能不能清（已按总监察审核修订）**：
   - 自动清改为严格正向白名单。匹配维度是可执行路径、cwd、启动参数特征，三选一或组合命中才可清。未命中的 ppid=1 只进周报，绝不杀。复核里「路径里出现 hermes / cursor / workbuddy 就保护」是反向排除，不作为现行规则。正文第 3 节已改。白名单当前为空，`node_repl` 和临时 broker 只是「以后若要清，必须先写明特征」的例子，不是现在的许可。
2. **处置名单由“反向排除”改为“严格正向白名单”**：
   - 已采纳。未知 `ppid=1` 只进周报，绝不杀。见上一条和文末补丁。
3. **收敛清理历史 DataPool 监控语义**：
   - 已按总监察审核修订：列为落地第 1 步之前的前置。`stuck-process-watch` 里废弃的 `/Volumes/DataPool` 改为现行工作盘语义（databox、archive 等现行挂载），不单写成只认 databox。改前备份到 `~/.shared/backups/`。
4. **注册表防呆**：
   - 按总监察审核修订后的现行控制是：`sync-mcp-config.sh` 在 `mcp-registry.json` 就绪并校验通过之前禁止再跑；落地时在脚本头或同目录加一行禁用说明。本次补丁不另要求先写 `verify_mcp_registry.py`。校验通过指注册表已存在、期望形态能读、并且不会把 Claude 的 basic-memory 写回 stdio。

---

### (d) 五大支柱自洽性与冲突审查

整体设计高度自洽，五大支柱各司其职，无相互踩踏：

- **预防与监控自洽**：预防支柱定义了唯一的预期基线（`mcp-registry.json`）并拦截旧脚本覆盖；监控支柱通过每日漂移与每小时体检对比基线与现网，双方数据口径一致且全流程保持只读，不互相冲突。
- **监控与报警自洽（无警报风暴）**：监控虽然每小时采样，但报警严格受控。swap 告警完全归拢于已有的 `runtime-guard`；体检仅在真正危机（内存空闲率 < 20% 或同名 MCP $\ge$ 4）且状态翻转时才发出一次通知，其余指标入日志与周报，自洽地落实了“不吵”原则。
- **处置与现成工具调研自洽**：工具调研深入剖析了 ToolHive / supergateway 等 stdio 网关在有会话 MCP（如 rea）上的局限性，从而直接支撑了处置支柱中“basic-memory 单实例 HTTP + rea 客户端独立 stdio + 绝不加装重型网关”的技术决策，结论与调研直接闭环。
- **协调注意点**：处置支柱中指出“改客户端配置不能替用户关 Cursor”，与预防支柱中“消除漂移”存在执行时间差。方案在落地步骤中明确写明“等待客户端自行退出释放子进程”，逻辑闭环且稳妥。


- 复核模型：`gemini-3.8-flash-medium`（agy）
- 复核原文里与现行口径冲突的句子已按总监察审核修订。未改任何配置、未杀进程、未碰 Hermes、未落地实施本方案。

## 总监察审核补丁 2026-10-23

本补丁只改这份方案。没有落地、没有杀进程、没有改脚本或配置。正文里已改的口径都标了「按总监察审核修订」。

1. **正向白名单，未知只进周报。** 自动清孤儿改为严格正向白名单：可执行路径、cwd、启动参数特征，三选一或组合命中才可清。未知的 ppid=1 只进周报，绝不杀。白名单当前为空；`node_repl` 和临时 broker 不是现成许可。反向排除不再使用。
2. **落地第 1 步之前的前置。** 把 stuck-process-watch 里废弃的 `/Volumes/DataPool` 硬编码改到现行工作盘语义（databox、archive 等现行挂载）。改前备份到 `~/.shared/backups/`。这一步不算五步之一，没做完不能进第 1 步。这次不改脚本。
3. **同步脚本写死禁跑。** `sync-mcp-config.sh` 在 `mcp-registry.json` 就绪并校验通过之前禁止再跑。落地时在脚本头或同目录加一行禁用说明，避免误跑把 Claude 的 basic-memory 打回 stdio。校验通过指注册表已存在、期望形态能读、并且不会把 Claude 的 basic-memory 写回 stdio。
4. **杀开关默认关。** 保持关闭。真要开必须再问一次用户（重大风险）。以前的点头不能代替这一问。
5. **五步可单独停。** 用户点头后按步做。每步做完报项目总监察验收，通过再做下一步。第 5 步打开之后，也只清白名单命中且其余条件全部满足的进程。未知的 ppid=1 仍然只进周报，绝不杀。
