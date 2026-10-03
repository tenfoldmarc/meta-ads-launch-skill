# Connectors

## Primary: Meta Ads connector (official MCP)

Meta's hosted MCP server: `https://mcp.facebook.com/ads`. OAuth login with Facebook, no developer app or token needed.

- **Claude Desktop / claude.ai:** Settings, Connectors, add Meta Ads (or "Add custom connector" with the URL above).
- **Claude Code terminal:** `claude mcp add --transport http meta-ads https://mcp.facebook.com/ads`, then open Claude Code and log in when prompted.

Tool names differ by client and version. At runtime, list tools containing `meta` and `ads` and map them to: list ad accounts, get insights, list / create / update campaigns, ad sets, ads, creatives, upload media, read pixel or dataset events. Read each tool's schema before calling it; don't assume parameter names.

Things that trip people up:
- **Budgets are in minor units** (cents for USD). $50/day = `5000`.
- **Insights date ranges:** use explicit `since`/`until` dates (complete days, ad account time zone), not presets like `last_7d` (presets can include today depending on the API).
- **Conversions** come back inside an `actions` list keyed by action type (`offsite_conversion.fb_pixel_purchase`, `purchase`, `lead`, `complete_registration`, `schedule`, or `offsite_conversion.custom.<id>`). Count only the campaign's optimization event, and don't double count `purchase` and `offsite_conversion.fb_pixel_purchase` (they're often the same conversions reported twice).
- **New ads show IN_REVIEW** for minutes to hours. That's normal.

## Fallback: Meta Ads CLI

Use only if the connector can read but not write (errors like "app in development mode" or missing permissions).

1. Install: `uv tool install meta-ads` (needs Python 3.12+ and `uv`). Check with `meta --help`.
2. Create a **System User** token: business.facebook.com, Business settings, Users, System users, Add (Admin role), Generate new token with `ads_management`, `ads_read`, `business_management`. Assign the ad account and Page to the system user.
3. Save the token in an environment file, never in `config.json` and never in chat. Warn the user: the token is stored as plain text on their computer, so keep that file private.
4. Test: `meta --help` lists the commands available in the installed version. Use `--json` output for anything the optimizer parses.

Set `"connector": "cli"` in `config.json` so every mode knows to use the CLI.
