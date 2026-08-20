# Cursor Automation: Daily Digest

Ported from aeonfun/aeon `skills/digest/SKILL.md`. This is the **prompt** you paste
into a Cursor Automation (cursor.com/automations, New, pick this Origin repo).

--------------------------------------------------------------------------------
## Automation config (set in the Cursor UI, not in this file)
--------------------------------------------------------------------------------
- Repository: this Origin repo (or a multi-repo env that includes it)
- Trigger:    Schedule, daily 08:00 (your tz)   [aeon cron equivalent]
- Model:      your default cloud-agent model
- Secrets (background-agent environment, not committed):
    - XAI_API_KEY        required for the X signal (Grok x_search)
    - SLACK_WEBHOOK_URL  delivery target (replaces aeon ./notify)
- Topic: hardcoded below as TOPIC. One automation per topic
  (aeon's rotating `${var}` has no direct trigger-time input on a schedule).

--------------------------------------------------------------------------------
## PROMPT (everything below this line goes in the Automation prompt box)
--------------------------------------------------------------------------------

You are running a scheduled daily digest against this repository. Deliver signal,
not volume: a 60-second skim should surface 3 things the reader didn't know this
morning, one of which changes a decision this week. Anything below that bar gets cut.

TOPIC = "" ("" = broad daily digest, no topic filter; else e.g. "solana", "AI agents")

### 0. Setup
- `TODAY=$(date -u +%Y-%m-%d)`
- Read `memory/MEMORY.md` (tracked topics) and the last 3 files in `memory/logs/`
  so you can dedup against anything already reported. If those paths don't exist,
  treat context as empty and create them at commit time.

### 1. Gather (wide net, ~15 raw candidates)
Use at least two independent source classes.

a) Web search, run 2 queries via your web-search tool:
   - `"${TOPIC}" news ${TODAY}` (broad; if TOPIC empty, query the day's notable
     stories in the tracked areas from MEMORY.md)
   - one narrower query you choose from TOPIC (e.g. solana becomes `"solana"
     launches OR funding OR exploit ${TODAY}`)

b) X signal via Grok x_search, direct curl (the Cursor agent env exposes real
   env vars, so no wrapper is needed; plain `$XAI_API_KEY` is fine):
   ```bash
   FROM=$(date -u -v-1d +%Y-%m-%d 2>/dev/null || date -u -d yesterday +%Y-%m-%d)
   TO=$(date -u +%Y-%m-%d)
   PROMPT="Search X for substantive recent posts about: ${TOPIC:-the most notable technology, AI, and crypto stories today}. Date range: $FROM to $TO. Return up to 10 high-signal posts (launches, funding, releases, exploits, hard data over hot takes). For each: @handle, full text, date, likes/retweets/replies, and https://x.com/handle/status/ID. Numbered list."
   jq -n --arg p "$PROMPT" --arg fd "$FROM" --arg td "$TO" \
     '{model:"grok-4.6", input:[{role:"user",content:$p}], tools:[{type:"x_search",from_date:$fd,to_date:$td}]}' > /tmp/xai.json
   curl -s --max-time 150 -X POST https://api.x.ai/v1/responses \
     -H "Content-Type: application/json" -H "Authorization: Bearer $XAI_API_KEY" \
     -d @/tmp/xai.json -o /tmp/xai-out.json -w 'http=%{http_code}\n'
   ```
   Parse `HTTP=200` bodies with:
   `jq -r '.output[]|select(.type=="message").content[]|select(.type=="output_text").text' /tmp/xai-out.json`
   Give x_search up to 150s; a slow curl is not a missing key. If `XAI_API_KEY`
   is unset or the call fails, fall back to a `site:x.com "<TOPIC>" after:$FROM`
   web search and record the true reason (http-code, empty, timeout, or key-unset).

c) Aggregator (only if web results are thin): fetch `https://news.ycombinator.com/`
   or `https://www.reddit.com/r/<topic>/top/.json?t=day` with curl.

### 2. Filter (kill noise, aim ~5-8 survivors)
Drop any candidate that: has no clickable source URL; is older than 36h (unless a
live story re-surfaced for a new reason); is pure speculation or hot-take (keep
items with a verifiable claim, named entity, number, release, or tx); already
appeared in the last 3 daily logs (unless a material new development); or duplicates
another survivor (keep the primary source over the recap).

### 3. Distill (3-5 strongest, lead with the most actionable)
Lead with the item a reader can act on today (subscribe/sell/fork/attend/apply).
Then descend by importance. Format exactly:

```
*${TOPIC:-Digest} - ${TODAY}*

_TL;DR: <one concrete sentence on the day's gravity, no adjectives.>_

1. *<Headline title, <=90 chars>*
   <1-2 sentence summary; lead with what happened, not who said it.>
   Why it matters: <one concrete-consequence clause, omit if you'd hand-wave>
   <link>

2. *<Title>* ...
3. *<Title>* ...

(Optional) *Also worth a glance:* <1-line> at <1-line>
```
Rules: Markdown only, no emoji, no "here's your digest" preamble, <=3000 chars.
Every item needs title + summary + real link (no `[link]` placeholders).

### 4. Sanity-check before sending
Lead item is the most actionable, not the most dramatic. Every link resolves. No
item paraphrases a hot take. No two items are the same story. Under 3000 chars. No
emoji, no corporate hedging. If fewer than 3 items clear the bar, send a short
"quiet day" digest with what survived, never pad or invent.

### 5. Send + log (persist state back to the repo)
- Deliver: `curl -s -X POST "$SLACK_WEBHOOK_URL" -H 'Content-Type: application/json' -d "$(jq -n --arg t "<digest body>" '{text:$t}')"`
- Append to `memory/logs/${TODAY}.md` under a `### digest (${TOPIC})` heading:
  source mode, sources used (+ xAI fetch reason if fallback), raw/after-filter/sent
  counts, lead-item title, dedup notes.
- Update the "Recent Digests" row in `memory/MEMORY.md`: date, topic, 3 keywords.
- Commit and push both files to this Origin repo:
  `git add memory/ && git commit -m "digest: ${TODAY}" && git push`

### Constraints
Never send placeholder links or invented items. Never repeat a story from the last
3 days of logs without a material update (and say so when you do).
