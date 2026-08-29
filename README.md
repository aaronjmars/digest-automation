# digest-automation

A single aeon leaf skill (`digest`) ported to a **Cursor Automation**, to try
Cursor Origin's scheduled cloud-agent runs.

Origin natively mirrors GitHub and Automations can target a GitHub, GitLab, or
Origin repo, so this repo works either way: point an Automation straight at it, or
mirror it into Origin first (Cursor: Codebase tab, Mirror GitHub).

## What it does
Once a day a cloud agent gathers the day's signal (web search + Grok x_search on X
+ aggregators), filters to ~5-8 real items, distills a ranked 3-5 item digest, and
posts it to Slack. It reads and commits `memory/` back to this repo, so it dedups
against prior days with no database.

## Setup (5 min)
1. **Secrets** - in the Automation's background-agent environment set:
   - `XAI_API_KEY` - Grok x_search (the X signal). Optional; digest still runs on
     web + aggregators without it.
   - `SLACK_WEBHOOK_URL` - an incoming-webhook URL for the channel you want it in.
2. **Create the automation** - cursor.com/automations, or the Agents Window, or the
   `/automate` skill. Pick this repo, trigger = Schedule (daily 08:00), and paste
   the PROMPT block from [`digest.automation.md`](./digest.automation.md).
3. **Topic** - edit `TOPIC=""` at the top of the prompt. `""` = broad daily digest;
   or set `"solana"`, `"AI agents"`, etc. One automation per topic.
4. **Verify** one dependency: the cloud agent needs a **web-search tool** enabled.
   Without it the digest runs on X + curl aggregators only (still valid).

## Files
- `digest.automation.md` - config header + the full prompt to paste.
- `PORT-NOTES.md` - the exact aeon-to-Cursor diff (what ported, what changed, what
  you lose vs aeon).
- `memory/` - state the agent reads and commits back (dedup across runs).

Ported from [aeonfun/aeon](https://github.com/aeonfun/aeon) `skills/digest/SKILL.md`.

---

Built by [Aaron Elijah Mars](https://aaronjmars.com), founder of Aeon and MiroShark · [@aaronjmars](https://github.com/aaronjmars)
