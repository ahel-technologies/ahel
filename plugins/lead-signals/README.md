# Lead signals

A skill for Claude Code, Codex and Cursor. Give it a watch list: companies (with their registry
numbers), public buyers and tender topics. Once a day it checks official EU sources, compares with
what it has already seen, and writes a short Markdown report of only the new signals: new public
tenders, changes in the business register, possible sanctions matches. Every line has its source URL
and the run id.

Sources: TED (EU public tenders and awards), the official business registers of Estonia, Finland,
Norway, Latvia, Ireland and France, and the OFAC SDN, OFAC Non-SDN and EU Financial Sanctions lists.
Optional weekly company news and funding rounds. Job changes are not in it yet.

Runs on [Ahel](https://ahel.ai): pay per result from a prepaid balance, no subscription, price shown
before the first run. An empty answer costs nothing (news is the one exception).

## Install

**Any agent, with the [`skills`](https://skills.sh) CLI** (Claude Code, Codex, Cursor and others)

```
npx skills add ahel-technologies/ahel --skill lead-signals
```

Add `-a claude-code` (or `codex`, `cursor`) and `-y` to skip the questions. The skill lands in
`.claude/skills/` or `.agents/skills/` of the current project; `-g` installs it for the user.

**Claude Code**

```
claude plugin marketplace add ahel-technologies/ahel
claude plugin install ahel@ahel
claude plugin install lead-signals@ahel
```

**Codex**

```
codex plugin marketplace add ahel-technologies/ahel
```

Then open `/plugins` in Codex and install **ahel** and **lead-signals**.

The `ahel` plugin is the connection (MCP address `https://mcp.ahel.ai`); the first call opens a
sign-in at ahel.ai. Sign in once by hand before you schedule it.

## Use it

> Watch new software tenders in Estonia and Bolt Technology OÜ (EE 12417834); report what is new
> every weekday morning.

The agent builds `lead-signals.json` (the watch list and the seen set), quotes the daily ceiling,
waits for your yes, records a baseline, and helps you schedule it: a Claude Code scheduled task, or
cron running `claude -p`, `codex exec` or `cursor-agent -p`.

## Cost

For 20 companies and 3 topics, at 2 Nov 2026 prices: at most $0.21 a day for the official signals
(tenders $0.004 per topic, register changes $0.01 per company only when it has changes on record,
sanctions $0.02 per possible match on new names only), usually a few cents. Weekly news and funding
for all 20 add at most $1.10 a week.

## Receipt

Live runs, 2 Nov 2026:

| Check | Input | Answer | Cost | Run |
| --- | --- | --- | --- | --- |
| New tenders | "software", EE | 10 TED notices, newest 759301-2026 (2 Nov, €350,000, deadline 30 Nov) | $0.004 | ex_639b10d72d194f0eb74a32bdc778a004 |
| Register changes | Bolt Technology OÜ, EE 12417834 | no change observed (snapshot 2 Nov 01:10 UTC) | $0 | ex_2fe29f0ce0214251ab622f22bd59176f |
| Register status | Bolt Technology OÜ, EE 12417834 | Registrisse kantud (registered), since 07.02.2013 | $0.002 | ex_588d04a5a230430eb0ea584b5c9df729 |
| Sanctions | Bolt Technology OÜ | no match on OFAC SDN (28 Oct), Non-SDN (19 Oct), EU FSF (23 Oct) | $0 | ex_87d4d15ab0a84ff08188914f4f854e20 |
| Buyer awards | Riigi Infosüsteemi Amet, EE | 5 award notices (name candidates), newest 30 Oct | $0.01 | ex_bed8cd459c0048648c6aa45f36d2dba6 |
| News | bolt.eu | 5 events, newest 30 Oct; no article links | $0.044 | ex_4a58a46224f942218a14d518192dddc5 |
| Funding | bolt.eu | 5 rounds, newest May 2024; no source links | $0.01078 | ex_b5f0b815ae46407f9867e9b13938e2b7 |

- **Official part: $0.016.** News and funding: $0.055, and both lack source links, so they are
  optional and weekly in the skill.
- Sources: TED (Decision 2011/833/EU), RIK e-Business Register (CC BY 4.0), OFAC (US public
  domain), EU FSF (Decision 2011/833/EU), PredictLeads, CompanyEnrich.

## Files

- `skills/lead-signals/SKILL.md`: the steps the agent follows, with the exact Ahel task ids.

MIT licence. Questions: support@ahel.ai
