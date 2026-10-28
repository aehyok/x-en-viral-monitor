# 英文圈 AI / 独立开发爆款帖监测

公开仓库，存放 Grok Bot skill「X 博主爆款帖监测」的英文圈配置，以及每小时扫描结果。用途是**监测英文圈 AI / 独立开发相关博主的爆款帖**。

> 数据来源为 X 公开帖子的浏览、点赞、转发、书签等公开指标，仅做趋势记录，不含任何私信或非公开内容。

流程改编自 [aehyok/x-viral-monitor](https://github.com/aehyok/x-viral-monitor)。本仓库不复制那边的历史小时归档，也不预置任何帖子或指标。

## 当前监测账号（6 个）

| 账号 | 主页 | 用户 ID | 说明 |
| --- | --- | --- | --- |
| @levelsio | https://x.com/levelsio | `1577241403` | 独立黑客 Pieter Levels |
| @marclou | https://x.com/marclou | `2311987360` | 独立开发者 Marc Lou |
| @gregisenberg | https://x.com/gregisenberg | `14642331` | 创业与社区，startup ideas |
| @steipete | https://x.com/steipete | `25401953` | OpenClaw / AI Agent，Peter Steinberger |
| @egeberkina | https://x.com/egeberkina | `1790083138959400961` | AI 视觉，ElevenLabs/Adobe 大使 |
| @karpathy | https://x.com/karpathy | `33836629` | LLM 研究者 Andrej Karpathy |

账号增减以定时任务配置为准，同时会同步写入 `data/state/state.json` 的 `accounts` 字段。完整名单也见 [ACCOUNTS.md](ACCOUNTS.md)。上表用户 ID 于 2026-10-28（北京时间）经公开查询核对，与已知 id 一致。

## 规则摘要

- **阈值**：上一小时新增浏览量 **> 5000**
- **窗口**：每天 **06:04–23:04**（北京时间 / Asia/Shanghai），含周末；**06:04 为基准轮**，只拉数据给 07:04 做对比，不推送
- **范围**：近 24 小时内的原创帖与引用帖（不含纯回复、纯转发）
- **去重**：已推送过的帖子 id 记在 `pushed`，不再重复提醒
- **告警译文**：推送给用户的每条告警必须包含帖子正文的**完整中文译文**（优先翻译 `note_tweet` 长文；产品名、代码、链接、`@` 保留原文）

## 目录说明

| 路径 | 内容 |
| --- | --- |
| `skill/SKILL.md` | 可复用的扫描流程（账号、阈值由调用方传入） |
| `routine/PROMPT.md` | 定时任务的时间表和提示词原文 |
| `ACCOUNTS.md` | 当前监测账号列表（中文） |
| `data/README.md` | 归档字段说明（`text` 英文原文，`text_zh` 中文译文） |
| `data/state/state.json` | 最新状态：账号 id、浏览量快照、已推送帖子 |
| `data/hourly/YYYY-MM-DD/HHMM.md` | 每小时一轮的中文简报 |
| `data/hourly/YYYY-MM-DD/HHMM.json` | 同一轮的结构化数据 |

## 告警格式

有达标帖时，每条按下面格式发给用户。非引用帖省略「引用」段：

```
@user https://x.com/user/status/id
浏览 N · 每小时 +R · 赞 L / 转发 T / 书签 B · YYYY-MM-DD HH:mm（北京时间）

中文译文：
<完整中文翻译；优先 note_tweet；产品名/代码/链接/@ 保留原文>

引用：<仅引用帖；一两句说明被引内容；非引用省略>
```

## 说明

- 想监测自己关注的博主：把 `skill/SKILL.md` 拿去用，账号列表、阈值和状态文件路径换成你自己的即可。
- 每小时扫描结束后，定时任务会更新 `state.json`，并新增对应的小时文件，再 `git push` 到本仓库。
- 告警消息发到 Grok Bot 对话，且必须带完整中文译文；无达标时不打扰，但仍会归档本轮数据。
- 初始 `data/state/state.json` 的 `snapshots` 与 `pushed` 为空，等第一次 06:04 基准轮写入快照。
