---
name: meta-ads-launch
description: "Launch conversion-focused Meta (Facebook + Instagram) ad campaigns through the Meta Ads connector, then manage them on autopilot against your target CPA. Onboarding audits your ad account, learns your offer, and sets up a scheduled optimizer that scales winners 20% (with your approval), rolls back, cuts losers, and asks for fresh creatives when performance slips. Use for /meta-ads-launch, 'launch my ads', 'launch a Facebook ad campaign', 'optimize my Meta campaigns', 'review my ad performance against CPA'."
---

# Meta Ads Launch

One skill, four modes. The whole system is built around ONE number: the user's **target CPA** (cost per conversion). Roughly 90% of every decision comes from "are we under it, close to it, or way over it."

| Command | Mode | What it does | File |
|---|---|---|---|
| `/meta-ads-launch setup` | Onboarding | Connects Meta, audits the ad account, asks the business questions, saves config, creates the scheduled optimizer | `onboarding.md` |
| `/meta-ads-launch` | Launch | Builds and launches a new testing campaign (conversion objective, 3 ad sets, 3 targetings) | `launch.md` |
| `/meta-ads-launch optimize` | Optimizer | Reviews every managed campaign against target CPA and proposes or applies changes. This is what the scheduled task runs. | `optimizer.md` |
| `/meta-ads-launch review` | Approvals | Shows pending proposals and applies the ones the user approves | `optimizer.md` (Approvals section) |
| `/meta-ads-launch status` | Status | One table: every managed campaign, its phase, CPA by window vs target, last action, next review date | `optimizer.md` (Report section) |

---

## Step 0. First-run check (every mode)

1. Look for `config.json` in this skill's directory.
2. **Missing, or `setupComplete` is not true:** open `onboarding.md` and run it. Do not launch or optimize anything until config exists. Exception: if this run was fired by the scheduled task and config is missing, stop and write one line to the run output: "meta-ads-launch is not set up. Run /meta-ads-launch setup."
3. **Present:** load it, then open the file for the requested mode (table above). No argument means Launch.
4. Make sure `state/` exists in this skill's directory. It holds `campaigns.json` (every campaign this skill manages and its history), `pending-actions.json` (proposals waiting for approval) and `logs/` (one markdown file per optimizer run). Create empty files if missing: `{"campaigns": {}}` and `{"pending": []}`.

---

## Connector rules (read once, apply everywhere)

All reads and writes go through the user's **Meta Ads connector** (Meta's official MCP at `https://mcp.facebook.com/ads`). Tool names vary by client and version, so discover them at runtime: list available tools whose names contain `meta` and `ads` (for example `mcp__meta-ads__*` or `mcp__claude_ai_Meta_Ads__*`). Map what you find to these jobs: list ad accounts, get insights, list/create/update campaigns, ad sets, ads and creatives, upload media, read pixel/dataset events.

Fallback: the official Meta Ads CLI (`meta` binary, installed with `uv tool install meta-ads`). See `reference/connectors.md`.

**Money gotcha:** Meta's API stores budgets in the account currency's minor unit (cents for USD). $50.00/day is `5000`. Always convert, always show the user dollars (or their currency), and double-check before every write.

---

## Hard rules

1. **Target CPA is the north star.** Every scale, rollback, cut and refresh decision cites the CPA ratio that triggered it.
2. **Nothing changes without approval.** Budget increases, budget cuts, rollbacks and pausing ads or ad sets all become proposals the user approves. The optimizer never applies them on its own.
3. **Launch PAUSED, go live on a yes.** New campaigns are created paused, shown to the user, and activated only when they say go.
4. **Pause, never delete.** No campaign, ad set or ad is ever deleted.
5. **Only touch campaigns this skill manages.** Managed = listed in `state/campaigns.json` (launched here, or adopted by the user during `setup`/`status`). Leave everything else in the account alone.
6. **Complete days only.** Use data through yesterday in the ad account's time zone. Today is partial and conversions report late.
7. **One budget change per campaign per review, minimum 3 days apart.** Meta needs time to re-stabilize after a budget edit.
8. **Rollback means the exact previous budget**, not "minus 20%" (a +20% followed by a -20% lands at 96%, not 100%).
9. **Respect the budget cap.** Never propose a daily budget above `max_daily_budget` from config.
10. **Never leave a campaign with zero active ads.** If cutting would do that, propose a campaign pause plus new creatives instead.
11. **Log everything.** Every run writes `state/logs/YYYY-MM-DD-<mode>.md`. The log is the answer to "why did it do that?"
12. **No em dashes** in any ad copy or scripts this skill writes.

---
Built by [@tenfoldmarc](https://instagram.com/tenfoldmarc). Follow for daily AI automation builds. Real systems, not theory.
