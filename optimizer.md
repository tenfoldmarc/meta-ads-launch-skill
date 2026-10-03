# Optimizer

The brain behind the scheduled task. Reviews every managed campaign against its target CPA, decides one of a small set of actions, and proposes it for the user's approval. Nothing that touches budget or delivery is ever applied without a yes.

Runs from the scheduled task (unattended) or `/meta-ads-launch optimize` (in chat). Same logic both ways.

---

## 0. Vocabulary

**T** = the campaign's `target_cpa` (from `state/campaigns.json`, falling back to the offer's in config).

**Window CPA** = spend / conversions for that window (conversions = the campaign's optimization event only).

**Ratio r** = window CPA / T. Lower is better.

**Bands** (defaults from `config.rules`; `close_band_pct` = 30, `way_off_pct` = 50):

| Band | Ratio | Meaning |
|---|---|---|
| `WIN` | r <= 1.00 | At or under target. Scale candidate. |
| `CLOSE` | 1.00 < r <= 1.30 | Within 30% over. Leave it alone. |
| `OFF` | 1.30 < r <= 1.50 | Over target by 30 to 50%. Something needs to change. |
| `WAY_OFF` | r > 1.50 | Over by more than 50%. Cut what's bleeding. |
| `NO_CONV` | 0 conversions | Judge by spend: under `zero_conv_cut_multiple` x T (default 2x) is `TOO_EARLY`, at or over it counts as `WAY_OFF`. |

**Windows** are complete days only, ending yesterday in the ad account's time zone: `3d`, `7d`, `14d`, `21d`, and `life` (since launch). A window is only used if the campaign is at least that many days old.

**Age** = complete days since `launched_at` (or first day with spend, if later).

**Actions** (the only things the optimizer ever does):

| Action | What happens |
|---|---|
| `HOLD` | Nothing. Log the reason. |
| `SCALE` | Daily budget x (1 + `scale_step_pct`), default +20%. Capped at `max_daily_budget`. **Always needs approval.** |
| `ROLLBACK` | Daily budget back to the exact previous value in `budget_history`. |
| `REDUCE_20` / `REDUCE_30` | Daily budget x 0.8 / x 0.7. |
| `CUT` | Pause specific losing ads and/or ad sets (rules in Section 4). |
| `REFRESH` | Ask for new creatives: write 3 new ad concepts or video scripts based on the current winners, using an installed ad-creative skill if there is one (see `launch.md` Step 4 for the companion skills). Launching them is a separate approved step. |
| `PAUSE_CAMPAIGN` | Only when nothing in the campaign is worth keeping. Always needs approval. |

At most **one budget action** (SCALE, ROLLBACK, REDUCE) per campaign per review. CUT and REFRESH can stack with a budget action.

---

## 1. Pre-flight

1. Load `config.json`, `state/campaigns.json`, `state/pending-actions.json`.
2. Confirm the Meta connector responds (list the ad account). If it fails: write the failure to the log and the run report, notify, and stop. Never guess with stale data.
3. **Sync managed campaigns:** for each campaign in state, read current status and budget from Meta.
   - Deleted or archived in Meta: mark `status: "ended"`, skip.
   - Paused by the user in Ads Manager: mark `status: "paused_by_user"`, skip, mention once.
   - Budget differs from `current_budget` (someone edited it by hand): record it in `budget_history` with reason `manual edit`, and treat that date as the last budget change.
4. **Expire stale proposals:** any pending proposal older than `proposal_expiry_hours` (48) is marked `expired`. The fresh analysis below replaces it.
5. **Tracking sanity check:** if every managed campaign shows 0 conversions in the last 2 complete days while spending normally, AND the account had conversions in the 7 days before that, suspect broken tracking. Do NOT cut or reduce anything. Report: "Conversions dropped to zero across every campaign at once. That's almost always tracking (pixel, checkout change, domain), not the ads. Check Events Manager." Skip to the report.

---

## 2. Who's due today

A campaign is reviewed today if it's `active` and either:
- age is exactly 3 or more and it has never been reviewed, or
- today >= `next_review` (last review + `review_every_days`, default 3).

Not due: skip it (it shows in the report as "next review [date]").

**Age 0 to 2 (launch phase): never act.** Only a health check, which runs daily for every active campaign:
- Ads disapproved or with delivery errors: report them with Meta's reason.
- Zero impressions after 24 hours: report (usually review stuck, payment issue, or audience too small).
- Spending more than 1.5x daily budget (unlikely but possible): report.

---

## 3. Decide (one campaign at a time)

Pull insights for the campaign, its ad sets and its ads for every available window: spend, conversions (optimization event), CPA, impressions, CTR, frequency, ad status.

First check the follow-up rule, then the phase.

### 3A. Follow-up after a scale (any phase, checked first)

If the last budget change was a `SCALE` and it happened 3+ days ago, judge the days **since that scale** (call it the post-scale window):

| Post-scale band | Action |
|---|---|
| `WIN` | `SCALE` again (+20%). The step-ladder continues as long as it keeps hitting target. |
| `CLOSE` | `HOLD` at the new budget. |
| `OFF` | `ROLLBACK` to the previous budget. |
| `WAY_OFF` or `NO_CONV` (at cut-level spend) | `ROLLBACK` + `CUT` (Section 4). |

If 3A produced a decision, that's the budget action for this review. Still run the phase rules below for CUT/REFRESH only (no second budget action).

### 3B. New campaign (age 3 to 6). Window: `life`.

| Situation | Action |
|---|---|
| `NO_CONV`, spend under 2x T (`TOO_EARLY`) | `HOLD`. Not enough data yet. |
| `NO_CONV`, spend 2x T or more | `CUT` ad sets/ads that spent 1x T or more with 0 conversions. If the whole campaign has spent 3x T with 0 conversions: propose `PAUSE_CAMPAIGN` + `REFRESH`, and ask the user to check the landing page and offer, not just the ads. |
| 1+ conversion, `WIN`, conversions >= `min_conversions_to_scale` (3) | `SCALE` +20% |
| 1+ conversion, `WIN` but fewer than 3 conversions | `HOLD`. Good sign, too little data to bet on. |
| `CLOSE` | `HOLD`. Let it run. |
| `OFF` | `HOLD` and add a strike (`strikes += 1`). If it was already scaled, 3A handled the rollback. |
| `WAY_OFF` | `CUT` the ads/ad sets with 0 conversions that are still getting real budget (Section 4). Add a strike. |

### 3C. Established campaign (age 7 to 20). Windows: `3d`, `7d`, `14d` (if age >= 14).

**L** = the longest window available: `14d` if age >= 14, otherwise `life`.

| L band | Recent performance | Action |
|---|---|---|
| `WIN` | `7d` is `WIN` and `3d` has 1+ conversion | `SCALE` +20% (if 3+ conversions in `7d`) |
| `WIN` | `3d` and `7d` are `WIN` or `CLOSE`, `3d` has 1+ conversion | `HOLD`. Keep it as is. |
| `WIN` | `7d` is `OFF` / `WAY_OFF`, or `3d` has 0 conversions with spend >= 1x T | **Strike.** First strike: `HOLD` and watch. Second consecutive strike (next review still 30%+ over): `REFRESH` + `CUT` ads/ad sets with 0 conversions in the last 7 days. |
| `CLOSE` | `7d` `WIN` or `CLOSE` | `HOLD` |
| `CLOSE` | `7d` `OFF` / `WAY_OFF` | Strike logic as above (first: watch, second: `REFRESH` + `CUT`). |
| `OFF` | any | `REFRESH` + `CUT` 7-day losers now (no waiting for a second strike). |
| `WAY_OFF` | any | `REFRESH` + `CUT` 7-day losers + `REDUCE_20`. |

Any review where `7d` is back in `WIN` or `CLOSE` resets `strikes` to 0.

### 3D. Mature campaign (age 21+). Windows: `3d`, `7d`, `14d`, `21d`.

| 21d band | 14d band | Action |
|---|---|---|
| `WIN` | `WIN` | Same as Established with L = 14d: `7d` `WIN` and 3+ conversions = `SCALE`, `7d` `CLOSE` = `HOLD`, `7d` `OFF`+ = strike logic. |
| `WIN` | `CLOSE` / `OFF` / `WAY_OFF` | **Trend check.** Is it recovering? Recovering = `7d` CPA lower than `14d` CPA, and `3d` CPA not more than 30% above `7d` CPA (or `3d` is `WIN`). Recovering: `HOLD` 3 more days, set flag `recovering`. Not recovering, or it was flagged `recovering` last review and `7d` didn't keep improving: go to **Mature cleanup** below. |
| `CLOSE` | any | Treat as "not recovering" if `7d` is `OFF`+, otherwise trend check as above. |
| `OFF` / `WAY_OFF` | any | **Mature cleanup** now, plus `REFRESH`. |

**Mature cleanup:**
1. Count what's still worth keeping over the last 7 days: ads in `WIN` or `CLOSE`, ad sets in `WIN` or `CLOSE`.
2. If there are **3 or more ads AND at least 1 ad set** in `WIN`/`CLOSE`: `CUT` everything else that qualifies under Section 4. Budget stays.
3. If **fewer than 3 ads OR zero ad sets** are in `WIN`/`CLOSE`: don't gut the campaign. `REDUCE_30` the budget and `REFRESH` creatives instead.

**Creative fatigue is a diagnosis, never a trigger.** CPA decides everything. If the campaign is in `WIN` or `CLOSE` overall and on `7d`, it is performing, so frequency and CTR are ignored completely: no flag, no `REFRESH`, no mention in the report. Only when a rule above has already fired because CPA is `OFF` or worse, check frequency (over 3.5 in `7d`) and CTR (down 30%+ vs `21d`). If either is true, add the `fatigue` flag and say so in the "Why" line ("people have seen these ads too many times"), because it tells the user that new creatives are the fix rather than new targeting.

---

## 4. CUT rules (what exactly gets paused)

Candidates, judged on the window named by the rule that triggered the cut (default `7d`, `life` for new campaigns):
- **Ads:** 0 conversions and spend >= 1x T, or CPA > 1.5x T with spend >= 2x T.
- **Ad sets:** same thresholds at ad set level. Pausing an ad set beats pausing all its ads.

Never cut:
- Anything with spend under 1x T (not enough data to judge).
- An ad set's last active ad (pause the ad set instead, if it qualifies).
- The campaign's last active ad set or last active ad (Hard rule 10: propose `PAUSE_CAMPAIGN` + `REFRESH` instead).
- Ads less than 3 days old (new creatives added during a refresh get their own 3 days).

With a campaign-level budget, pausing an ad set moves its spend to the others automatically. On an adopted campaign with ad set budgets, the paused ad set's budget is simply gone; propose moving it to the best ad set as part of the same proposal.

---

## 5. Budget mechanics

- New budget = current x factor, rounded to the nearest whole currency unit. Show both: "$100.00 to $120.00/day".
- Campaigns launched by this skill always use a **campaign-level budget** (`budget_mode: "CBO"`): change the campaign daily budget.
- Adopted campaigns that were built with ad set budgets (`budget_mode: "ABO"` in state): apply the same % to every active ad set, which is the campaign-level equivalent. For `SCALE`, raise only ad sets in `WIN`/`CLOSE` on `7d` and leave the others flat.
- Never above `max_daily_budget`. If a SCALE would cross it, propose up to the cap and say the cap is reached.
- Respect `min_days_between_budget_changes` (3). If the last change was under 3 days ago, the budget action becomes `HOLD (cooldown)`.
- Every applied change appends to `budget_history` with date, old, new, action, and the CPA numbers behind it.

---

## 6. Approvals

Every change to budget or delivery needs the user's approval. There is no automatic mode.

| Action | Approval |
|---|---|
| SCALE | Ask |
| ROLLBACK, REDUCE_20, REDUCE_30 | Ask |
| CUT (pausing ads or ad sets) | Ask |
| PAUSE_CAMPAIGN | Ask |
| Launching refreshed creatives | Ask |
| REFRESH (writing the new concepts or scripts) | No approval needed. It's only writing; nothing goes live. |

**Write each proposal** to `state/pending-actions.json`:

```json
{
  "id": "P-YYYYMMDD-1",
  "campaign_id": "...",
  "campaign_name": "...",
  "action": "SCALE",
  "from_budget": 100,
  "to_budget": 120,
  "targets": [],
  "reason": "14d CPA $31 vs $40 target (0.78x), 7d $29 (0.73x), 9 conversions in 7d",
  "created_at": "ISO timestamp",
  "status": "pending"
}
```

**In the run report**, number the proposals and end with:

> Reply **approve 1, 3**, **approve all**, or **skip 2**. You can also type `/meta-ads-launch review` anytime in the next 48 hours.

**Applying an approval** (reply in the run's session, or `/meta-ads-launch review`):
1. Re-read the live budget/status from Meta. If it no longer matches `from_budget` or the targets' current status, don't apply. Re-run the decision for that campaign and show the new proposal.
2. If the proposal is older than 48 hours: re-run the decision, don't apply the stale one.
3. Otherwise apply, confirm the new value from Meta, update `state/campaigns.json` and mark the proposal `applied`.
4. Skipped proposals are marked `skipped`; a skipped SCALE does not count as a budget change (no cooldown).

---

## 7. Update state

For every reviewed campaign: `last_reviewed` = today, `next_review` = today + 3 days, `phase` (`launch`, `new`, `established`, `mature`), `strikes`, `flags` (`recovering`, `fatigue`, `cap_reached`, `tracking_suspect`), and a one-line `last_decision`.

---

## 8. Report

Write `state/logs/YYYY-MM-DD-optimize.md` and print the same thing as the run output. Keep it scannable:

```
META ADS OPTIMIZER  |  [date]  |  [account name]

[campaign name]  (day 9, established)        target $40
  3d  $36 (0.90x) 4 conv   7d  $33 (0.83x) 11 conv   life $35 (0.88x)
  Decision: SCALE  $100 to $120/day   [needs approval: #1]
  Why: under target on every window, 11 conversions this week.

[campaign name]  (day 2, launch)
  Learning. No changes until [date]. Health: 1 ad in review, rest delivering.

[campaign name]  (day 24, mature)            target $40
  3d $58  7d $49  14d $44  21d $38
  Decision: HOLD (recovering)   7d improving vs 14d. Re-check [date].

PROPOSALS
  #1  SCALE  [campaign]  $100 to $120/day
  #2  CUT    [campaign]  pause ad set "2 Interests" ($85 spent, 0 conv in 7d)
Reply "approve 1, 2", "approve all", or "skip 2".

NOT DUE TODAY: [campaign] (next [date]), ...
```

Rules: plain language, numbers next to every claim, ratios shown as `0.83x`, no jargon without a reason. If nothing is due and nothing is wrong, the whole report is one line: "Nothing due today. Next review: [campaign] on [date]."

**Notify** when there are pending proposals, auto-applied changes, or health problems: use a push-notification tool if one exists in the session, and post to `config.notify.slack_channel` if set and a Slack connector is available. Otherwise the run report is the notification.

---

## 9. `/meta-ads-launch status`

Read-only. Same per-campaign block as the report for every managed campaign (due or not), plus pending proposals. Then offer: "Want me to adopt any other active campaigns?" (onboarding Step 6).
