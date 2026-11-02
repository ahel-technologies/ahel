# Find work emails

A skill for Claude Code and Codex. Give it a list of people (name + company domain, or a LinkedIn
URL). It quotes the price, finds each work email, checks the mailbox, and writes a CSV with a status
per row and the total cost.

Runs on [Ahel](https://ahel.ai): pay per result from a prepaid balance, no subscription, price shown
before the run. Misses are usually free.

## Install

**Claude Code**

```
claude plugin marketplace add ahel-technologies/ahel
claude plugin install ahel@ahel
claude plugin install find-work-emails@ahel
```

(Inside a session: `/plugin marketplace add ahel-technologies/ahel`, then `/plugin install find-work-emails@ahel`.)

**Codex**

```
codex plugin marketplace add ahel-technologies/ahel
```

Then open `/plugins` in Codex and install **ahel** and **find-work-emails**.

**Any agent, with the [`skills`](https://skills.sh) CLI** (Claude Code, Codex, Cursor and others)

```
npx skills add ahel-technologies/ahel --skill find-work-emails
```

Add `-a claude-code` (or `codex`, `cursor`) and `-y` to skip the questions. The skill lands in
`.claude/skills/` or `.agents/skills/` of the current project; `-g` installs it for the user.

The `ahel` plugin is the connection (MCP address `https://mcp.ahel.ai`); the first call opens a
sign-in at ahel.ai. New workspaces start with a small free balance.

## Use it

> Find and verify work emails for the people in leads.csv.

The agent shows the most the run can cost and the expected cost, waits for your yes, then runs.

## Receipt

One real run, 1 Nov 2026: 10 founders and CEOs of European tech companies, names taken from public
sources. Emails are redacted here.

| # | Company | Email | Found by | Mailbox check |
| --- | --- | --- | --- | --- |
| 1 | bolt.eu | m\*\*\*@bolt.eu | QuickEnrich | valid |
| 2 | wise.com | k\*\*\*@wise.com | Hunter | catch-all |
| 3 | klarna.com | s\*\*\*@klarna.com | QuickEnrich | valid |
| 4 | spotify.com | not found | | |
| 5 | supercell.com | i\*\*\*@supercell.com | QuickEnrich | catch-all |
| 6 | wolt.com | not found | | |
| 7 | veriff.com | k\*\*\*@veriff.com | QuickEnrich | valid |
| 8 | pipedrive.com | t\*\*\*@pipedrive.com | Hunter | invalid |
| 9 | revolut.com | n\*\*\*@revolut.com | Hunter | valid |
| 10 | n26.com | v\*\*\*@n26.com | Hunter | valid |

- **10 people → 8 emails found, 5 valid, 2 catch-all, 1 invalid, 2 not found.**
- **Total: $0.074** (8 finds $0.062 + 8 checks $0.012).
- **$0.009 per email found, $0.015 per valid email, all-in for this run** (finds plus checks, divided by the valid addresses).
- One find is about $0.005 per found email (QuickEnrich, measured on 14 calls), priced before every run; the exact quote is shown before each run.
- The 2 misses cost nothing. Their last provider timed out, so Ahel still holds $0.037 each until
  that provider settles; it bills only when it finds an email.

## Files

- `skills/find-work-emails/SKILL.md`: the steps the agent follows, with the exact Ahel task ids.

MIT licence. Questions: support@ahel.ai
