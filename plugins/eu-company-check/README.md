# EU company check

A skill for Claude Code and Codex. Give it EU companies by name and country, or by registry code.
It quotes the price, reads the official register record, finds the LEI, screens sanctions lists,
and returns one table with a source and date on every fact.

Covers the official registers of Estonia, Finland, Norway, Latvia, Ireland and France; LEIs from
GLEIF; sanctions screening against OFAC SDN, OFAC Non-SDN and the EU Financial Sanctions File.

Runs on [Ahel](https://ahel.ai): pay per result from a prepaid balance, no subscription, price shown
before the run. "No LEI" and "no sanctions match" cost nothing.

## Install

**Claude Code**

```
claude plugin marketplace add ahel-technologies/ahel
claude plugin install ahel@ahel
claude plugin install eu-company-check@ahel
```

(Inside a session: `/plugin marketplace add ahel-technologies/ahel`, then `/plugin install eu-company-check@ahel`.)

**Codex**

```
codex plugin marketplace add ahel-technologies/ahel
```

Then open `/plugins` in Codex and install **ahel** and **eu-company-check**.

**Any agent, with the [`skills`](https://skills.sh) CLI** (Claude Code, Codex, Cursor and others)

```
npx skills add ahel-technologies/ahel --skill eu-company-check
```

Add `-a claude-code` (or `codex`, `cursor`) and `-y` to skip the questions. The skill lands in
`.claude/skills/` or `.agents/skills/` of the current project; `-g` installs it for the user.

The `ahel` plugin is the connection (MCP address `https://mcp.ahel.ai`); the first call opens a
sign-in at ahel.ai. New workspaces start with a small free balance.

## Use it

> Check these suppliers: Bolt Technology OÜ (EE), Supercell Oy (FI), Equinor ASA (NO).

The agent shows the most the run can cost and the expected cost, waits for your yes, then runs.

## Receipt

One real run, 1 Nov 2026: 10 Estonian, Finnish and Norwegian companies, given by name and country.

| Input | Register record | Registry code | Status | LEI | Sanctions | Cost |
| --- | --- | --- | --- | --- | --- | --- |
| Bolt Technology, EE | Bolt Technology OÜ, reg. 07.02.2013, VAT EE102090374 | 12417834 | Registrisse kantud (registered) | none | no match | $0.008 |
| Pipedrive, EE | Pipedrive OÜ, reg. 21.06.2010, VAT EE101382096 | 11958539 | registered | none | no match | $0.006 |
| Veriff, EE | Veriff OÜ, reg. 20.10.2015, VAT EE101828178 | 12932944 | registered | 98450082D0D460CD9E92 (issued) | no match | $0.010 |
| Starship Technologies, EE | Starship Technologies OÜ, reg. 11.06.2014, VAT EE101770530 | 12673365 | registered | none | no match | $0.006 |
| Supercell Oy, FI | Supercell Oy, reg. 2010-07-16 | 2336509-6 | registered | 743700MSCHABJ4XW2532 (lapsed) | no match | $0.010 |
| Wolt Enterprises Oy, FI | not found by name | | | none | no match | $0.004 |
| Nokia Oyj, FI | Nokia Oyj, reg. 1896-12-19 | 0112038-9 | registered | 549300A0JPRWG1KI7U06 (issued) | no match | $0.010 |
| Equinor ASA, NO | EQUINOR ASA, reg. 1995-03-12 | 923609016 | registered, not bankrupt | OW6OFBNCKXC4US5C7523 (issued) | no match | $0.010 |
| Kahoot! ASA, NO | KAHOOT! AS, reg. 2011-12-31 | 997770234 | registered, not bankrupt | 2549004957SZTRN8CW77 (lapsed) | no match | $0.010 |
| Telenor ASA, NO | TELENOR ASA, reg. 2000-10-06 | 982463718 | registered, not bankrupt | 549300IM1QSBY4SLPM26 (issued) | no match | $0.010 |

Sources: Estonia RIK e-Business Register (CC BY 4.0, 31 Oct 2026), Finland PRH/YTJ (CC BY 4.0,
31 Oct 2026), Brønnøysundregistrene (NLOD 2.0, 31 Oct 2026), GLEIF (CC0), OFAC SDN (28 Oct 2026),
OFAC Non-SDN (19 Oct 2026), EU FSF (23 Oct 2026).

- **10 companies → 9 register records, 6 LEIs (2 lapsed), 0 sanctions matches.**
- **Total: $0.084, under 1 cent per company.**
- What went wrong: "Wolt Enterprises Oy" did not match by name, and one looser search returned
  unrelated companies and still billed $0.004. "Kahoot! ASA" matched only as "Kahoot" (it is now an
  AS). The skill now tells the agent to check every match and ask for the registry code instead.

## Files

- `skills/eu-company-check/SKILL.md`: the steps the agent follows, with the exact Ahel task ids.

MIT licence. Questions: support@ahel.ai
