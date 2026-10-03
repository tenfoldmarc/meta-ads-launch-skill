# Scheduled Task

The optimizer runs once a day. Each campaign is only reviewed when it's due (day 3 after launch, then every 3 days), so a daily run is mostly a quick "nothing due" check. Onboarding creates this task for you. This file is the source of truth for its prompt.

**Schedule:** `0 9 * * *` (every day, 9am local)
**Task ID:** `meta-ads-optimizer`
**Title:** `Meta Ads Optimizer`

## Prompt

Replace `{SKILL_DIR}` with the absolute path of this skill folder before saving the task.

```
Run the Meta Ads optimizer from the meta-ads-launch skill.

1. Read {SKILL_DIR}/SKILL.md, then follow {SKILL_DIR}/optimizer.md exactly, step by step.
2. Config is {SKILL_DIR}/config.json. State is in {SKILL_DIR}/state/. Read both before doing anything.
3. Use the Meta Ads connector for all reads and writes. If it isn't available or errors, stop, log it, and report "Meta connector unavailable" as the first line.
4. Only touch campaigns listed in state/campaigns.json. Never touch anything else in the ad account.
5. Never change a budget and never pause or activate anything without my approval. Analyze, then propose. The only thing you may do without asking is write new ad concepts or scripts.
6. Use complete days only (through yesterday, ad account time zone).
7. Write the run log to {SKILL_DIR}/state/logs/ and end with the report from optimizer.md Section 8, including numbered proposals I can approve by replying in this session.
8. If nothing is due and nothing is wrong, reply with one line and stop.
```

## Approving from a run

Open the run in your scheduled tasks list and reply `approve 1, 2` (or `approve all`, or `skip 1`). Or type `/meta-ads-launch review` in any chat within 48 hours. Older proposals expire and get re-analyzed on the next run.

## Turning it off

Scheduled tasks list, find `Meta Ads Optimizer`, toggle it off. Your campaigns keep running as they are; nothing gets paused when the optimizer stops.
