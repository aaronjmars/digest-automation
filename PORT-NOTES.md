# digest: aeon skill to Cursor Automation - the actual diff

Source: `aeonfun/aeon/skills/digest/SKILL.md` (leaf skill, ~180 lines).
Target: one Cursor Automation prompt (`digest.automation.md`).

## What ported unchanged (the value)
- **Whole 5-phase pipeline** (gather / filter / distill / sanity-check / send-log).
  This is the skill's actual IP and it copied verbatim.
- **xAI x_search curl** - identical call, only `./secretcurl {XAI_API_KEY}` became
  plain `curl ... $XAI_API_KEY` (see below).
- **Memory-as-git-files** - reads `memory/MEMORY.md` + `memory/logs/`, commits the
  new log + digest row back. Cursor cloud agent has repo push, so state survives
  runs exactly like on aeon. No DB, no external store. This is the clean surprise.

## What changed (the plumbing, ~6 edits)
| aeon | Cursor Automation | Why |
|---|---|---|
| `metadata:` frontmatter (`mode`, `category`, `requires:`, `var`) | deleted | No catalog/vault reads it. Trigger + secrets set in Cursor UI. |
| `./secretcurl -H "Bearer {XAI_API_KEY}"` | `curl -H "Bearer $XAI_API_KEY"` | aeon's secretcurl only exists because Claude Code's Bash analyzer denies a bare `$SECRET` on a line. Cursor's cloud agent runs real env, so plain curl works. **Simpler off-aeon.** |
| `${var}` selector (rss / topic / +rss grammar) | `TOPIC=""` constant | A scheduled Automation has no trigger-time input. Hardcode topic; one automation per topic. Minor capability loss. RSS mode dropped (was `var`-driven). |
| `./notify "<body>"` (Telegram/Discord/email/Buzz) | `curl $SLACK_WEBHOOK_URL` | No notify.sh off-aeon. Swap to a Slack webhook, or use Cursor's native Slack integration instead of a curl. |
| `requires: [XAI_API_KEY?]` -> dashboard vault -> env | Cursor background-agent env secrets | Re-declare `XAI_API_KEY` + `SLACK_WEBHOOK_URL` in the automation's environment. |
| WebSearch / WebFetch built-ins | agent web-search tool (+ curl fallback) | aeon ships these as first-class tools. On Cursor, depends on the agent's tool config; curl to HN/Reddit JSON is the guaranteed fallback. **Verify web search is enabled** for the cloud agent - the one real dependency to confirm. |

## Effort
~30 min. No logic rewritten - the port is 6 mechanical swaps around an unchanged
prompt body. The pipeline is model-agnostic prose; only the 4 plumbing verbs
(secret fetch, notify, var input, tool access) are substrate-specific.

## What you lose vs aeon
- `var`-driven RSS mode and topic rotation (static now).
- Haiku 1-5 run scoring -> `skill-health` -> `skill-repair` self-heal loop. No
  equivalent; if the automation silently degrades, nothing catches it.
- Free GitHub Actions minutes. Cursor cloud-agent runs are metered billing.

## One dependency to verify before first run
Does your Cursor cloud agent have a **web-search tool** enabled? If not, the digest
runs on X (Grok) + curl aggregators only - still valid, but note it. Everything
else in the prompt is self-contained (curl, jq, git, date all present in the agent
sandbox).
