# Meta Ads Launch
### A Claude Code Skill by [@tenfoldmarc](https://www.instagram.com/tenfoldmarc)

Launch Facebook and Instagram ad campaigns from Claude, then let Claude manage them against the one number that matters: your cost per customer (target CPA). It looks at your ad account to see what's already working, builds a proper conversion campaign with 3 ad sets testing 3 different audiences, and checks performance every 3 days. Under target? It asks to scale 20%. Slipping? It rolls back. Bleeding money? It cuts what isn't converting and writes you new ads to test.

You stay in control. Nothing changes without your OK.

Never run an ad before? That's fine. It asks you plain questions and handles the Ads Manager part for you.

---

## What It Does

1. **Audits your ad account.** Finds your best audiences, best ads, best countries, and where money is leaking.
2. **Learns your offer.** What you sell, your landing page, your goal (sales, leads, or booked calls) and your target cost per conversion.
3. **Launches a testing campaign.** Conversion objective, 3 ad sets (broad, interests, lookalike), your choice of countries (default: US, Canada, UK, Australia, New Zealand). Created paused, goes live when you say go.
4. **Leaves it alone for 3 days** so Meta can learn.
5. **Reviews every 3 days** using the last 3, 7, 14 and 21 days of data (whatever the campaign is old enough for).
6. **Proposes changes:** scale +20% when you're under target, hold when you're close, roll back when a scale stops working, cut losers and write new ads when you're 30%+ over.
7. **You approve** with one reply: `approve all`.

---

## The rules it follows

| Your cost per conversion vs target | What happens |
|---|---|
| At or under target (3+ conversions) | Ask to scale budget +20% |
| Up to 30% over | Hold. Let it run. |
| 30 to 50% over | Roll back the last scale, or flag it and check again in 3 days |
| More than 50% over | Pause the ads and ad sets that aren't converting, write new ads |
| Older campaigns slipping | Check the trend. Recovering? Give it 3 more days. Not recovering? Cut losers, or cut budget 30% if there's not enough left to cut. |

Full logic lives in [`optimizer.md`](optimizer.md).

---

## Requirements

- [Claude Desktop](https://claude.ai/download) or [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- A Meta ad account with a working pixel (Purchase, Lead, or booking event firing)
- The **Meta Ads connector** (Meta's official one, free, you log in with Facebook)

Don't worry about connecting these manually. The skill walks you through everything on first run.

---

## Install

No terminal needed.

### Step 1: Open Claude

Open the Claude Desktop app (or Claude Code, if you already use it).

### Step 2: Paste this message

Copy-paste this into the chat and hit Enter:

```
Install this skill for me: https://github.com/tenfoldmarc/meta-ads-launch-skill
```

Claude downloads the skill into your skills folder and tells you when it's ready. Takes a few seconds.

### Step 3: Run the skill

Type:

```
/meta-ads-launch
```

Hit Enter. On your first run, it walks you through setup: connecting Meta, auditing your account, your offer, budget, countries and target cost per conversion. It also sets up the daily optimizer for you.

<details>
<summary>Prefer the terminal? Manual install</summary>

Open Terminal (Mac: `Command + Space`, type Terminal. Windows: `Win + R`, type cmd). Paste this one line and hit Enter:

```bash
git clone https://github.com/tenfoldmarc/meta-ads-launch-skill ~/.claude/skills/meta-ads-launch
```

Then type `claude` to open Claude Code and run `/meta-ads-launch`.
</details>

---

## Usage

| Type this | What happens |
|---|---|
| `/meta-ads-launch` | Launch a new testing campaign |
| `/meta-ads-launch status` | See every campaign vs your target, at a glance |
| `/meta-ads-launch review` | Approve or skip pending changes |
| `/meta-ads-launch optimize` | Run a review right now instead of waiting for the daily check |
| `/meta-ads-launch setup` | Redo setup (new offer, new target, new countries) |

**Example:** "Launch a campaign for my $97 course, $100 a day, target $40 per sale." Claude confirms the plan, builds 3 ad sets, shows you everything paused, and switches it on when you say go. Three days later your optimizer report says something like: *"Cost per sale $33 (0.83x target), 11 sales. Scale $100 to $120/day? Reply approve 1."*

---

## Need ads to launch? Pair it with these

This skill launches and manages. These free skills make the ads. If they're installed, this skill uses them automatically when it needs new creatives.

| What you need | Skill |
|---|---|
| Image ads | [ad-image-skill](https://github.com/tenfoldmarc/ad-image-skill), [ad-image-gen-skill](https://github.com/tenfoldmarc/ad-image-gen-skill), [meta-ads-generator-skill](https://github.com/tenfoldmarc/meta-ads-generator-skill) |
| Ad copy | [ad-copy-skill](https://github.com/tenfoldmarc/ad-copy-skill) |
| Video ad scripts | [video-ad-copy-skill](https://github.com/tenfoldmarc/video-ad-copy-skill) |
| See what competitors are running | [ad-spy-skill](https://github.com/tenfoldmarc/ad-spy-skill) |

Install any of them the same way: paste `Install this skill for me:` followed by the link.

---

## Safety

- Never changes a budget or turns anything off without your approval. It analyzes and proposes. You decide.
- Never deletes anything. It only pauses.
- Only touches campaigns it launched or that you told it to manage.
- Never goes above the max daily budget you set.
- Every decision is logged with the numbers behind it.

You're still the one paying for the ads. Check in on your campaigns, especially the first couple of weeks.

---

## Updating

Paste this into Claude:

```
Update the meta-ads-launch skill from https://github.com/tenfoldmarc/meta-ads-launch-skill
```

Or in Terminal: `cd ~/.claude/skills/meta-ads-launch && git pull`

Your settings and campaign history are kept (they're never part of the download).

---

## Built By

[@tenfoldmarc](https://www.instagram.com/tenfoldmarc). Follow for daily AI automation walkthroughs. Real systems, not theory.
