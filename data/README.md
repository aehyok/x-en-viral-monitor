# data

每半小时扫描的归档与运行状态。目录名仍是 `hourly/`；文件名为 `HHMM`（`0604`、`0634`，直到 `2334`）。起步时只有空的 `hourly/` 和一份空快照的 `state/state.json`，没有历史归档。不要从中文账号仓库 [aehyok/x-viral-monitor](https://github.com/aehyok/x-viral-monitor) 复制旧归档，也不要写入没有经工具读到的帖子或指标。归档仓库是 `aehyok/x-en-viral-monitor`。回看窗口为 12 小时。

## state/state.json

运行状态：

| 字段 | 内容 |
| --- | --- |
| `note` | 这份状态文件是做什么的 |
| `accounts` | 用户名 → 数字用户 id |
| `snapshots` | 帖子 id → `{ views, checked_at }`（UTC）。初始为 `{}` |
| `pushed` | 已经推送给用户的帖子 id，用于去重。初始为 `[]` |

## hourly/YYYY-MM-DD/HHMM.json

一轮扫描的结构化结果。每条检查过的帖子除浏览量与速率外，正文分成两个字段：

| 字段 | 内容 |
| --- | --- |
| `text` | 帖子原文（英文，或原发语言）。有 `note_tweet` 长文时存长文全文，不用截断后的短文本。 |
| `text_zh` | 该正文的**完整中文译文**。产品名、代码、链接、`@` 用户名保留原文。不是大约 40 字的标题。 |

引用帖可以另加 `quoted_text`（被引原文）和 `quoted_text_zh`（一两句中文说明）。非引用帖不要写这两个字段。

浏览、点赞、转发、书签只记录工具实际返回的数字。

## hourly/YYYY-MM-DD/HHMM.md

同一轮的中文简报：扫了几个号、几条帖、新达标几条、最接近阈值的几条。新达标帖附上与告警相同的完整中文译文。
