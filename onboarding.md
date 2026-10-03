# Onboarding

Runs on first use or with `/meta-ads-launch setup`. Takes about 10 minutes. Ends with a saved `config.json`, a scheduled optimizer, and an offer to launch the first campaign.

Open with:

> Let's get your ads set up. First I'll look at your ad account to see what's already working, then I'll ask a few questions about your offer. About 10 minutes, and you only do it once.

Ask questions **one or two at a time**, never as a wall. Every question gets a suggested answer based on the account audit, so the user can just say "yes".

---

## 1. Connect Meta

Discover the Meta Ads connector tools (see SKILL.md, Connector rules). Smoke test: list the user's ad accounts.

**Works:** continue.

**No tools found:** stop and give the fix, then wait:

> Your Meta Ads connector isn't connected yet. Pick whichever matches how you use Claude:
>
> **Claude Desktop or claude.ai:** Settings, then Connectors, then add Meta Ads (if it isn't listed, choose "Add custom connector" and paste `https://mcp.facebook.com/ads`). Log in with Facebook and give it access to your ad account.
>
> **Claude Code in the terminal:** run `claude mcp add --transport http meta-ads https://mcp.facebook.com/ads`, open Claude Code again, and log in with Facebook when it asks.
>
> Then run `/meta-ads-launch setup` again.

**Connected but writes fail later** (permissions or "app in development mode"): switch to the CLI path in `reference/connectors.md`.

---

## 2. Pick the ad account

List accounts the user can access with name, ID, currency, time zone and status. One account: confirm it. Several: ask which. Save `ad_account_id` (format `act_123...`), `currency`, `timezone`.

Also fetch and confirm:
- **Facebook Page** and **Instagram account** the ads run from (list Pages tied to the account; if one, confirm it).
- **Pixel / dataset** on the account and which conversion events fired in the last 7 days (Purchase, Lead, CompleteRegistration, Schedule, custom conversions). If nothing fired in 7 days, warn clearly: conversion campaigns can't optimize without a working pixel. Offer to continue setup anyway but block launches until it's fixed.

---

## 3. Account audit (what's actually working)

Pull the last 90 days (fall back to lifetime if less), at campaign, ad set and ad level. Fields: spend, impressions, CPM, CTR, conversions by event, CPA, ROAS if available, frequency. Read ad set targeting for the top spenders.

Work out:
- **Main conversion event** the account has been optimizing for, and its blended CPA.
- **Top 3 ad sets by CPA** (minimum 3 conversions each so one lucky sale doesn't win): countries, age, gender, interests, lookalikes, custom audiences, Advantage+ audience on/off, placements.
- **Top 5 ads by CPA**: format (video, image, carousel), hook / first line, offer angle.
- **What's burning money**: ad sets or ads with spend over 2x their CPA and zero conversions.
- **Country split**: CPA by country if data exists.
- **Audiences available**: customer lists, website visitors, lookalikes (needed for exclusions and the lookalike ad set).

Show a short summary, plain English, no jargon dump:

> **Here's what your account tells me (last 90 days)**
> - You spent $X and got Y purchases. Blended cost per purchase: $Z.
> - Best targeting: [ad set] at $A per purchase ([what the targeting is]).
> - Best ads: [2-3 ads with their hook].
> - Money leaks: [what's spending without converting].
> - Countries: [where it converts cheapest].

**No history** (new account or no conversions): say so, skip the stats, and use proven defaults in Step 4.

---

## 4. Business questions

Ask in this order. Pre-fill from the audit wherever possible.

1. **What are you selling?** Product or offer name, one-line description, price. If more than one, collect each (saved as `offers`).
2. **What's the goal?** Map it to the objective and event:

   | Goal | Objective | Optimization event |
   |---|---|---|
   | Sell a product or course | Sales (`OUTCOME_SALES`) | Purchase |
   | Get leads / opt-ins | Leads (`OUTCOME_LEADS`) | Lead or CompleteRegistration |
   | Book calls | Leads (`OUTCOME_LEADS`) | Schedule or a custom conversion on the booking thank-you page |

   Always a conversion objective. Never traffic or engagement for these goals, even if the user asks for cheaper clicks. Explain once: "Traffic campaigns find clickers, not buyers."
3. **Landing page URL** for each offer. Fetch it once to confirm it loads.
4. **Target CPA** (in account currency). Help them pick a real number:
   - Show history: "Your account has been getting purchases at $Z."
   - Show break-even: price x margin. "If you keep $70 of a $97 sale, anything under $70 per sale is profitable on the first purchase."
   - Suggest: the lower of history and a profitable number. Save as `target_cpa` per offer.
5. **Daily testing budget.** Suggest one. Sanity check: the budget should be **at least 1x target CPA per day**, ideally 2x to 3x. Below 1x, a campaign can go days without a single conversion and the 3-day reviews become guesswork. Say that plainly if they go lower.
6. **Countries.** Default: the Big 5, `US, CA, GB, AU, NZ`. Offer add-ons from `reference/countries.md` (Tier 1 English-friendly first: Ireland, then Netherlands, Nordics, Germany, Switzerland, and so on). If the audit shows a country with clearly better CPA, mention it.
7. **Special ad category.** Ask only if the offer touches credit, financing, jobs, housing, or politics/social issues. These restrict targeting and must be declared.
8. **Exclusions.** "Want me to exclude people who already bought?" If a customer list or purchaser audience exists, suggest it.
9. **Max daily budget cap.** "What's the most you'd ever want a single campaign to spend per day?" The optimizer will never propose more. Default: 5x the starting test budget.
10. **Where should updates go?** Default: the scheduled task's run report (plus a push notification if available). Optional: a Slack channel if a Slack connector is present.

---

## 5. Save config

Write `config.json` in this skill's directory (it is gitignored, never commit it). No tokens or passwords in it. Shape (see `config.example.json`):

```json
{
  "setupComplete": true,
  "setupDate": "YYYY-MM-DD",
  "connector": "mcp",
  "ad_account_id": "act_...",
  "currency": "USD",
  "timezone": "America/New_York",
  "page_id": "...",
  "instagram_account_id": "...",
  "pixel_id": "...",
  "offers": [
    {
      "name": "...",
      "description": "...",
      "price": 97,
      "goal": "sales",
      "objective": "OUTCOME_SALES",
      "conversion_event": "PURCHASE",
      "landing_page": "https://...",
      "target_cpa": 40
    }
  ],
  "default_daily_budget": 100,
  "max_daily_budget": 500,
  "countries": ["US", "CA", "GB", "AU", "NZ"],
  "special_ad_category": "NONE",
  "exclude_audience_ids": [],
  "notify": { "slack_channel": null },
  "rules": {
    "close_band_pct": 30,
    "way_off_pct": 50,
    "scale_step_pct": 20,
    "deep_cut_pct": 30,
    "min_conversions_to_scale": 3,
    "zero_conv_cut_multiple": 2,
    "min_days_between_budget_changes": 3,
    "review_every_days": 3,
    "proposal_expiry_hours": 48
  },
  "audit_summary": "short paragraph from Step 3: best targeting, best ads, best countries",
  "scheduled_task": { "type": "desktop|cloud|manual", "id": "..." }
}
```

Read back a 5-line summary and ask "Anything to change?"

---

## 6. Adopt existing campaigns (optional)

Ask: "Want the optimizer to also manage any campaigns that are already running?" If yes, list active campaigns with 7-day spend and CPA, let them pick, and add each to `state/campaigns.json` with its real launch date, current budget, `budget_mode` (`CBO` if the budget sits on the campaign, `ABO` if it sits on the ad sets), `target_cpa` (ask per campaign if offers differ), and `adopted: true`. Never add campaigns they didn't pick.

---

## 7. Create the scheduled optimizer

The optimizer runs **daily**, but each campaign is only reviewed when it's due (every 3 days, and exactly on day 3 after launch). A daily check is what makes "review on day 3" accurate; a plain every-3-days schedule would hit some campaigns on day 4 or 5 and skips unevenly at month end.

Use the first option available:

**A. Desktop scheduled tasks** (a tool named like `create_scheduled_task` exists):
- `taskId`: `meta-ads-optimizer`
- `cronExpression`: `0 9 * * *` (9am local; ask if they prefer another time)
- `title`: `Meta Ads Optimizer`
- `description`: `Daily CPA check on campaigns managed by meta-ads-launch`
- `prompt`: the prompt in `scheduled-task.md`, with `{SKILL_DIR}` replaced by this skill's absolute path.
- Tell the user: "This runs while the Claude app is open. If it's closed at 9am, it runs next time you open it."

**B. Cloud routine** (the `/schedule` skill is available and the Meta connector is a claude.ai connector): create a daily routine with the same prompt. Note that the routine needs access to the Meta Ads connector, and that state files must live somewhere the routine can read (a cloud routine can't see local files, so only use this if the skill folder is in a repo the routine checks out).

**C. Manual fallback:** print the prompt from `scheduled-task.md` and these steps: open Claude Desktop, go to Scheduled tasks, click New task, paste the prompt, set it to daily at 9am.

Save the result under `scheduled_task` in config.

---

## 8. Finish

> You're set. Here's how it works from now on:
> - `/meta-ads-launch` launches a new testing campaign.
> - Every day at 9am the optimizer checks your campaigns. Anything due for review gets analyzed against your $[target] CPA.
> - Nothing changes without your OK: no budget increase, no budget cut, no ad turned off. You'll see proposals in the run report, or type `/meta-ads-launch review`.
> - `/meta-ads-launch status` shows everything at a glance.
>
> Want to launch your first campaign now?

If yes, open `launch.md`.
