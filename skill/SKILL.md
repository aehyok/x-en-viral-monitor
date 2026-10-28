---
name: X 博主爆款帖监测
description: >-
  Use this when monitoring specific X accounts for posts whose views grow past a
  per-hour threshold (爆款帖 / viral post watch), for a one-shot scan or as the
  body of an hourly routine. Delivered hits must include a full Chinese
  translation of the post body.
---
# X 博主爆款帖监测

Scan named X accounts for posts whose **view growth rate** exceeds a threshold, then push new hits once (no duplicates).

This skill is the **scan recipe**. Account list, threshold, lookback window, timezone, and state path come from the caller (chat or routine). Do not hard-code one assistant's accounts here.

## Inputs (from caller)

| Param | Default if omitted |
| --- | --- |
| `accounts` | Required: list of X `@username`s |
| `threshold` | `5000` views per hour |
| `lookback_hours` | `24` |
| `timezone` | `Asia/Shanghai` |
| `state_path` | Caller-owned JSON under the bot store, e.g. `…/x-en-viral-monitor/state.json` |
| `language` | Hit blocks are Chinese (full translation). Surrounding status text matches the user. |

## State file shape

```json
{
  "note": "Hourly X watch state.",
  "accounts": { "username": "numeric_user_id" },
  "snapshots": {
    "post_id": { "views": 0, "checked_at": "ISO-8601 UTC" }
  },
  "pushed": ["post_id", "…"]
}
```

- `snapshots`: last seen view count + check time per post still in the lookback window.
- `pushed`: post IDs already delivered; never push again.
- Create the file on first run if missing.

## Tools

Prefer the public **`x`** MCP namespace (no user X login needed):

1. `get_users_by_usernames` — resolve usernames → user ids (cache into `state.accounts`).
2. `get_users_posts` per user id — recent posts with `post.fields` including `public_metrics,created_at,text,note_tweet` (and expansions as needed, including quoted posts). Request metrics that expose **impression / view** counts.
3. Fall back to `get_posts_by_ids` if a post needs a refresh.

Respect X rate limits (429): retry with backoff; on partial failure still save snapshots for posts you did fetch, and report which accounts failed.

## One scan pass

1. **Load state** from `state_path` (or initialize empty).
2. **Resolve accounts** if ids are missing.
3. **Fetch** each account's posts; keep originals and quotes; drop pure retweets/replies unless the caller asks otherwise.
4. **Filter** to posts with `created_at` within `lookback_hours` of now (use tweet id / timestamp; do not invent posts).
5. For each kept post, read current `views` (impressions). Body source for translation: the full `note_tweet` text when present, otherwise `text`.
6. **Rate**:
   - If the post is already in `snapshots`:  
     `rate = (views_now - views_prev) / max(hours_since_prev, 1/60)`  
     (`hours_since_prev` from `checked_at` → now, UTC).
   - If first sight:  
     `rate = views_now / max(hours_since_created, 1/60)`.
7. **Qualify** if `rate > threshold` (strictly greater) **and** `post_id` ∉ `pushed`.
8. **Update state**: rewrite `snapshots` for all posts still in the lookback window (drop older ids); append newly pushed ids to `pushed`; keep `accounts` map.
9. **Deliver** each new hit as its own block (blank line between posts). Do **not** use a ~40 character title. The post body must be a **full Chinese translation**:

   ```
   @user https://x.com/user/status/id
   浏览 N · 每小时 +R · 赞 L / 转发 T / 书签 B · YYYY-MM-DD HH:mm（北京时间）

   中文译文：
   <完整中文翻译；优先 note_tweet；产品名/代码/链接/@ 保留原文>

   引用：<仅引用帖；一两句说明被引内容；非引用省略>
   ```

   - `中文译文` is the entire post body in Chinese, not a summary. Prefer `note_tweet` over the truncated short text. Keep product names, code, URLs, and `@` handles in the original language.
   - `引用` only for quote posts: one or two sentences on what was quoted. Omit that line entirely when the post is not a quote.
   - If none: stay quiet when this is a routine tick; if the user asked for a manual scan, say briefly that nothing crossed the threshold (optionally name the closest rate).
   - Hand this exact text to the parent when running inside a routine executor.

## Archive fields

When the caller writes the hourly JSON (see the routine), each checked post keeps the original body and the translation in separate fields:

| Field | Content |
| --- | --- |
| `text` | Original body (English, or whatever language it was posted in). If `note_tweet` exists, store that full text, not the truncated tweet text. |
| `text_zh` | Full Chinese translation of `text`. Same rules as the alert: do not shorten to a title; keep product names, code, links, and `@` as written. |

Optional for quote posts only: `quoted_text` (original quoted body) and `quoted_text_zh` (one or two Chinese sentences). Do not invent either field for non-quotes. Do not invent posts, view counts, or translations of text you did not fetch.

## Rules

- Re-scan the full lookback every pass so a post that spikes in hour 2 still qualifies.
- Never re-push an id in `pushed`, even if it is still growing.
- Do not invent view counts, titles, links, or translations.
- Prefer public `x` tools over the signed-in X plugin for this read-only watch.
- Schedule (cron, hours of day) lives in a **routine**, not in this skill. A routine should say: run this skill with these accounts / threshold / state_path, then deliver any hit blocks (with full Chinese translations) to the user.

## Manual catch-up

If the user asks to补跑 after a missed hour, run **one** full pass now with the same inputs and deliver any new hits in the format above.
