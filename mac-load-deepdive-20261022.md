# Mac 负载只读深挖（2026-10-22 22:20–22:30 UTC+8）
全程只读：未改配置、未杀进程；未读 cookie/登录数据/密码/聊天邮件正文；未碰 DataPool、123pan 内容、secrets；Hermes 不动。

## 1. 整体压力（3 次采样，间隔 2 分钟）
| 采样 | swap used/total | memory_pressure 空闲% | 压缩器占用页 |
|---|---|---|---|
| 22:20 | 9153M/10240M | 29% | 446k 页(≈7.0G) |
| 22:22 | 9414M/10240M | 51% | 254k(≈4.0G) |
| 22:24 | 9350M/10240M | 38% | 373k(≈5.8G) |
| 平均 | **≈9.3G** | ≈39% | ≈5.6G |
累计 swapouts 2060 万页、swapins 1712 万页：换入换出很频繁，是“卡”的直接原因。
注：RSS 被压缩/换出后严重偏小，下面按 top 的 MEM（phys footprint，含压缩+swap）统计，单次快照，误差约 ±20%。

## 2. 按应用累计（footprint）
| 应用 | 内存 | 进程数 | 备注 |
|---|---|---|---|
| WorkBuddy（含 codebuddy CLI、WebKit 页面、MCP） | ~4.5G | 19 | 其中 codebuddy -p Remotion 任务 1.29G、--serve 会话 0.86G、内嵌 WebKit 0.79G |
| Cursor（GUI+扩展宿主） | ~2.9G | 24 | douyin-knowledge-app 扩展宿主 0.66G |
| Codex/ChatGPT 桌面 | ~2.65G | 28 | |
| 散装 node（MCP 等） | ~2.2G | 23 | **rea mcp ×7 ≈1.55G**、server.bundle ×4、cloudbase/edgeone npx |
| Grok Bot | ~1.3G | 9 | |
| 123云盘 | ~1.27G | 11 | 按约束不碰 |
| Chrome | ~1.19G | 14 | 15 个标签 |
| WindowServer | ~1.0G | 1 | CPU 40%+，窗口/显示器多导致 |
| Hermes（python 等） | ~1.0G | 8 | 保持现状 |
| Edge | ~0.84G | 12 | 0 个标签窗口但进程常驻 |
| ima.copilot | ~0.77G | 16 | |
| cursor-agent | ~0.69G | 5 | |
| opencode（Remotion 任务） | ~0.65G | 1 | 采样期 CPU 峰值 23% |
| Antigravity | ~0.62G | 7 | |
| python 其他（basic-memory mcp ×3 ≈0.43G 等） | ~0.62G | 10 | |
| DeepSeek Harness | ~0.55G | 5 | |
| UURemote | ~0.38G | 4 | CPU 常驻 ~8% |
| Spotlight(mds) | ~0.32G | 10 | 正常 |
| cline hub daemon | ~0.16G | 1 | |
| Docker | ~0 | - | 没在跑，只有 socket/vmnetd 守护 |
| kilo | 未发现进程 | | |

## 3. 三个浏览器现状
| | Chrome | Edge | 第三个：Safari |
|---|---|---|---|
| 内存 | ~1.19G（主进程+渲染器≈1.0G，GPU 137M） | ~0.84G（渲染器/扩展 5 个≈300M，GPU 79M，主进程 336M） | 未运行（系统里看到的 WebKit 进程属于 WorkBuddy，不是 Safari） |
| 标签 | 15 | 0（无窗口仍常驻） | - |
| 配置文件 | 1（Default） | 1（Default） | - |
| 登录 | 一个 gmail.com 账号 | 一个 163.com 邮箱的微软账号 | 需用户确认是否用 iCloud |
| 扩展 | 3：ChatGPT、Sider、Web Store Payments | **44 个**（见附录） | - |
| 内存节省 | 已开（high_efficiency_mode=2） | 未配置 | - |
| 后台运行 | Local State 未开启 background_mode | 同左，但开机自启（登录项 application.com.microsoft.edgemac） | - |
| 开机自启 | 是（launchd 里有 application.com.google.Chrome） | 是 | 否 |
| 更新器 | GoogleUpdater/keystone LaunchAgent | EdgeUpdater.wake | - |
推断并存原因：Chrome=Google 账号+轻量 AI；Edge=国内电商/公众号/下载/翻译工具箱（店侦探、找货神器、壹伴、IDM、SellerSprite、MetaMask 等），账号体系不同。**需用户确认**：Edge 的电商/公众号插件是否仍在日常用、MetaMask 钱包是否在 Edge 里有资产（不能随手删）。

## 4. 浏览器精简方案（全部可回滚）
1. **Chrome 做主力**（扩展少、已开内存节省、已登 Google）。Safari 不用就保持不开，无需处理。
2. **Edge 降级为“工具浏览器”，按需打开、用完 ⌘Q**：从 系统设置→通用→登录项 移除 Edge 自启；Edge 设置→系统和性能 关闭“启动增强”和“关闭后继续运行后台扩展和应用”。省 **~0.6–0.8G**（不开时）。回滚：重新加回登录项、打开开关。
3. **Edge 扩展从 44 砍到 ~10**：先“停用”不“删除”（edge://extensions 关开关，回滚就是再打开）。建议停用：重复项（AIX Downloader×2、Sider/豆包/Merlin/Manus/AIPRM/Voice Control/ChatGPT 多个 AI 侧边栏留 1 个）、Coupert、Incredible StartPage/NewTab（两个新标签页）、Ghostery 与 广告终结者 二选一、IE Tab、Listen 1、Google Docs Offline、Hunter、ModHeader、Chrome Remote Desktop（已有 UU远程）、两个 DevTools 微软内置。电商类（店侦探/店透视/找货神器/SellerSprite）与壹伴、MetaMask 需用户确认。省 **~150–400M**（Edge 开着时；常驻型扩展每个 10–60M）。
4. **Edge 开“效率模式/睡眠标签”**：edge://settings/system → 睡眠标签 5 分钟。Chrome 已开内存节省，可在 chrome://settings/performance 设为“最大”。各省 **~100–300M**（随标签数）。回滚：开关改回。
5. Chrome 15 个标签：用 OneTab/Tab Groups 收纳（开源替代：Tab Suspender 类扩展，或内置“内存节省”即可，不必再加扩展）。
6. 浏览器小计：日常可省 **~0.8–1.1G**（复核后下修，Edge 退出与扩展精简不叠加）。
参考：Chrome 内存节省 https://support.google.com/chrome/answer/12929150 ；Edge 睡眠标签/效率模式 https://learn.microsoft.com/microsoft-edge/... （Edge 官方“Sleeping tabs”文档）；macOS 登录项管理 https://support.apple.com/zh-cn/guide/mac-help/mh15189/mac

## 5. N100 迁移清单（只列收益明显、无人值守的）
| # | 项 | 现在占用 | 挪过去怎么跑 | 体验影响 | 难度 | 回滚 |
|---|---|---|---|---|---|---|
| 1 | **Remotion 渲染/批量 agent 任务**（opencode run、codebuddy -p 渲染脚本） | 采样时合计 ~1.9G，CPU 峰值 20%+，渲染时还会拉起 headless Chrome | N100 装 Node 22 + Remotion，`ssh n100` 后在项目目录跑 `npx remotion render`；工程用 git/共享目录同步，产物回传 | 无（本来就是后台批处理），渲染速度 N100 约 M4 的 1/3–1/2 | 中 | 继续在 Mac 跑即可 |
| 2 | **夜间/定时无头脚本**（bm-snapshot 03:30、l0.hygiene 03:40、123pan 冷归档 周一04:15 → 后者不碰）| 单次短时，内存小 | 可选：挪成 N100 计划任务 | 几乎无 | 中 | 重新 load plist |
第 2 项收益低（瞬时、小内存），**按要求不建议挪**，只是说明夜间负载主要来自这些定时任务而非常驻大户。

### 不是“挪”而是“去重”，收益反而最大（留在 Mac 上做）
- **rea mcp 被起了 7 份（≈1.55G）+ server.bundle 4 份 + basic-memory mcp 3 份（≈0.43G）**：每个 agent 客户端（Cursor/Codex/WorkBuddy/cursor-agent…）各自 stdio 拉一份。关掉不用的客户端会话或改成单实例 HTTP MCP，可省 **~1.2–1.6G**。改成 HTTP 后理论上可放 N100，但会增加网络依赖，先在 Mac 上去重。回滚：恢复各客户端原 MCP 配置。
- 闲置时退出的 GUI：Codex 桌面 2.65G、DeepSeek Harness 0.55G、ima 0.77G、Muse、Antigravity 0.62G——需用户确认哪些常用。

## 6. 必须留在 Mac 的大户
- WorkBuddy、Cursor、Codex、Antigravity、Grok Bot：交互式 IDE/聊天，要看页面、要登录。
- Chrome/Edge：已定浏览器不挪。
- 123云盘：约束不碰。Hermes：约束保持现状（含 hermes bridge/gateway/studio）。
- WindowServer：系统本身；可通过减少窗口/外接屏缩放降低。
- UURemote、Tailscale、clash、ssh 隧道：网络/远控，必须在本机。

## 7. 结论（实话）
- 挪 N100 真正有意义的只有“渲染/批量 agent 任务”一类，**而且是间歇性的**：跑任务时能少 ~1–2G、swap 不再顶满；不跑时收益≈0。
- 日常 swap 9.3G 的根源是“同时开太多 Electron/AI 客户端 + MCP 重复进程”，这些都不适合挪。
- 照本方案：浏览器精简 ~1G + MCP 去重 ~1.3G + 渲染任务挪走（任务期 ~1.5G）+ 闲置客户端退出（视用户 1–3G）。预计 swap 从 ~9.3G 降到 **4–6G**，memory_pressure 空闲比从 ~39% 升到 50–60%；只做 N100 迁移、不做其他，swap 只会在渲染时段降，平时基本不变。

## 8. 复核
cursor-agent（--model grok-4.7-high，只读）意见与回应：
1. 「swapouts 是累计值、无增量」——部分接受：22:20→22:22 swapouts 增加约 2.2 万页（≈350MB）、swapins 三次共增约 1.1 万页，确实在换页但不剧烈；空闲比例波动大，39% 只作参考。
2. 「漏 kernel_task、Muse/server.bundle 无数字」——接受：Muse ≈111M、server.bundle ×4 ≈124M（各 31M），MCP 去重收益下修为 ~1.1–1.4G；kernel_task 是系统，不可优化。
3. 「N100 只有 Wi‑Fi、C 盘剩 ~12G，速度偏乐观」——接受：渲染速度改为约 M4 的 1/4–1/3；必须把缓存/输出放非系统盘或渲染后清理，否则不建议挪。
4. 「MCP 改 HTTP 回滚不完整」——接受：动手前先备份各客户端 MCP 配置文件（cp 成 .bak-日期），回滚=还原备份。
5. 「收益叠算」——接受：Edge 退出与扩展精简不叠加，浏览器合计改为 ~0.8–1.1G；MCP 去重与关客户端有重叠。修正后预计：只做浏览器+MCP 去重，swap 降到 ~6–7G；再退出闲置 GUI（Codex/DeepSeek/ima/Antigravity）才可能到 4–5G；N100 仅在渲染时段额外有效。
（agy 复核未执行：cursor-agent 单次耗时近 9 分钟，时间所限。）

## 附录：Edge 扩展清单（44）
AIPRM for ChatGPT；AIX Downloader×2；Awesome Screen Recorder；ChatGPT；Checker Plus for Gmail；Chrome Remote Desktop；Coupert；Downloader for Youtube；Edge relevant text changes；Ghostery；Google Docs Offline；Hunter；IDM Integration；IE Tab；ImageAssistant；Immersive Translate；Incredible StartPage；Listen 1；Manus AI Browser Operator；Merlin AI；MetaMask；Edge DevTools Enhancements；Edge Unminification；ModHeader；NewTab新标签页；NotionCn；Proxy SwitchySharp；SellerSprite；Sider；Tampermonkey；Voice Control for ChatGPT；ima知识库；CocoCut；划词翻译；在线抠图工具；壹伴；广告终结者；店侦探&看店宝；店透视；找货神器；表答AI；豆包（另有若干内置组件，共 44 个 manifest）。
Chrome：ChatGPT、Sider、Chrome Web Store Payments。
