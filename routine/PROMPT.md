# 定时任务：英文圈 AI / 独立开发爆款帖监测

- **时间表**：`CRON_TZ=Asia/Shanghai 4,34 6-23 * * *`（北京时间每天 06:04–23:34，每半小时一次，:04 与 :34，含周末；小时范围仍是 06 点到 23 点。06:04 为基准轮，只记数据不推送给用户）
- **回看窗口**：`lookback_hours = 12`
- **调用的 skill**：[`skill/SKILL.md`](../skill/SKILL.md)
- **结果去向**：有达标帖时推到 Grok Bot 对话，且每条带**完整中文译文**；没有达标时不提醒，但仍会归档
- **归档仓库**：`aehyok/x-en-viral-monitor`
- **运行时状态**：占位路径 `/cursor/stores/<STORE_ID>/x-en-viral-monitor/state.json`；本 bot 的预期路径为 `/cursor/stores/bot/x-en-viral-monitor/state.json`

## 提示词（原文）

```text
每半小时按 skill「X 博主爆款帖监测」扫一轮，并把本轮数据归档到仓库 aehyok/x-en-viral-monitor。

时间表：CRON_TZ=Asia/Shanghai 4,34 6-23 * * *（北京时间 :04 与 :34，从 06 点到 23 点，含周末）。归档目录仍是 data/hourly/YYYY-MM-DD/HHMM.md 与同名 .json，HHMM 为 0604、0634、0704……直到 2334。

特殊规则：北京时间 06:04 这一轮只是基准轮，照常 git pull、拉数据、更新 snapshots 并归档（commit 后 push 到 aehyok/x-en-viral-monitor），但不判定达标、不写入 pushed、不给用户发任何消息。它的作用是给 06:34 那轮提供约半小时前的对比基准。06:34 及之后的轮次按下面规则正常判定和推送。

监测这 6 个英文圈 X 账号近 12 小时的原创帖与引用帖（不含回复、纯转发）：@levelsio、@marclou、@gregisenberg、@steipete、@egeberkina、@karpathy。用公开 X 数据工具拉每个账号过去 12 小时内发的帖子及其浏览量（impression_count）。lookback_hours = 12。正文优先取 note_tweet 长文，没有长文再用 text。

状态文件（运行时）：/cursor/stores/<STORE_ID>/x-en-viral-monitor/state.json（本 bot 的预期路径为 /cursor/stores/bot/x-en-viral-monitor/state.json）。里面有账号 id、每条帖子上次记录的浏览量和时间（snapshots），以及已推送过的帖子 id（pushed）。阈值：浏览增速超过每小时 5000（views per hour）。时区：Asia/Shanghai。账号 id 若仍为空，本轮用公开查询补上再写入 accounts，不要凭记忆编造。

判定规则（与上一轮快照的间隔经常约为 0.5 小时；用实际间隔小时数，不要把间隔改成 1 小时。结果仍是每小时浏览量，阈值 > 5000）：
- 帖子在 snapshots 里有上次记录：rate =（本次浏览量 − 上次浏览量）÷ max(两次间隔小时数, 1/60)，结果 > 5000 就算达标。
- 第一次见到的帖子：rate = 浏览量 ÷ max(发帖至今小时数, 1/60)，结果 > 5000 就算达标。
- 已在 pushed 里的帖子不再重复推送。
每次跑完都把仍在 12 小时窗口内的帖子的最新浏览量和时间写回 snapshots（窗口外的 snapshot 删掉），把新推送的 id 加进 pushed。pushed 只增不删，用来去重。写回时保留 state 里已有的 accounts（别覆盖掉新加的账号）。06:04 基准轮只更新 snapshots 与归档，不改 pushed，也不判定达标。

归档到 GitHub（本机目录 /workspace/x-en-viral-monitor，远端仓库 aehyok/x-en-viral-monitor）：
1. 先 git pull --rebase，再把更新后的 state.json 复制到 data/state/state.json。
2. 按北京时间写本轮文件：data/hourly/YYYY-MM-DD/HHMM.md 与同名 .json（HHMM 为 0604、0634 这类四位时间）。JSON 至少包含 run_at、accounts、每条检查过的帖子（id、author、text、text_zh、views、rate_per_hour、qualified、already_pushed）、本轮新达标列表（06:04 基准轮写明 baseline: true，新达标为空）。text 存英文（或原发语言）正文，有 note_tweet 时存长文全文；text_zh 存该正文的完整中文译文，产品名、代码、链接、@ 保留原文，不要收成约 40 字标题。引用帖可加 quoted_text 与 quoted_text_zh（一两句中文说明被引内容）；非引用帖不要写引用字段。Markdown 用中文写简报：扫了几个号、几条帖、新达标几条、最接近阈值的几条；新达标帖附上完整中文译文。
3. 若账号列表或本提示词有变更，同步更新 ACCOUNTS.md、README.md 里的账号表和 routine/PROMPT.md。
4. git add、commit（说明写本轮北京时间与达标条数）、push 到 origin。若目录缺失则先 gh 克隆 aehyok/x-en-viral-monitor。提交身份用 aehyok / aehyok@163.com（与姊妹仓库 aehyok/x-viral-monitor 归档提交所用邮箱一致）。push 失败时在结果里说明，不要假装已归档。

有达标帖子时（06:34 及之后的轮次），用中文给用户发消息。每条帖子单独一块，块与块之间空一行，格式必须是：

@user https://x.com/user/status/id
浏览 N · 每小时 +R · 赞 L / 转发 T / 书签 B · YYYY-MM-DD HH:mm（北京时间）

中文译文：
<完整中文翻译；优先 note_tweet；产品名/代码/链接/@ 保留原文>

引用：<仅引用帖；一两句说明被引内容；非引用省略>

中文译文必须是帖子正文的完整翻译，不是简短标题。非引用帖整段省略「引用」行。「每小时 +R」里的 R 是浏览增速（次/小时），扫描本身是每半小时一轮。数字、链接、时间只使用本轮工具实际返回的值，不要编造帖子或指标。可在全部帖子之后顺带一句「本轮数据已推到 aehyok/x-en-viral-monitor」。

结果发给这个对话里的用户本人。没有达标帖子时、以及 06:04 基准轮，什么都不发，保持安静（仍要完成归档提交）。
```
