# Ahel

[![mcptry](https://mcptry.com/badge/ai.ahel/ahel)](https://mcptry.com/servers/ai.ahel/ahel)

Data and tools for your AI agent, from one catalog: company and people enrichment, SEO and SERP data, social media data, official EU business registers and more. One prepaid balance, pay per call, and each call's price is shown before it runs. No provider sign-ups or keys.

Website: https://ahel.ai · Catalog: https://ahel.ai/catalog · MCP address: `https://mcp.ahel.ai`

## Install

**Claude Code**

```
/plugin marketplace add ahel-technologies/ahel
/plugin install ahel@ahel
```

**Codex**

```
codex plugin marketplace add ahel-technologies/ahel
```

Then open `/plugins` in Codex and install **ahel**.

**Claude.ai, ChatGPT, Cursor and other MCP clients**

Add the MCP address `https://mcp.ahel.ai`. The first call opens a sign-in at ahel.ai; there is nothing else to configure. Setup for each client: https://ahel.ai/agents

## Job skills

Ready-made jobs that run on Ahel. Install `ahel` first, then the skill. Each one quotes the price before it runs and ends with the exact cost. Each README has a receipt from a real run.

| Skill | What it does | Real run |
|---|---|---|
| [find-work-emails](plugins/find-work-emails) | Names + company domains in, verified work emails out as a CSV | 10 people, 8 found, 5 valid: $0.074 |
| [eu-company-check](plugins/eu-company-check) | Register record, LEI and sanctions screen for EU companies, sourced | 10 companies, 9 records: $0.084 |

```
claude plugin install find-work-emails@ahel
claude plugin install eu-company-check@ahel
```

## What your agent can do

- Find a person's work email, enrich a company from its domain, search companies and people
- Check a company against official registers (Estonia, Finland, Norway, Latvia, Ireland, France), LEI records and sanctions lists, each fact with its source and date
- Google keyword volumes, SERP results, backlinks and on-page audits
- Posts, profiles, comments and trends from TikTok, Instagram, YouTube, X, Reddit and LinkedIn
- Read RSS feeds, extract named facts from a company's website, monitor pages for changes

Your agent searches the catalog, checks the price, runs the task and reads the result. Every run is in your history with its quote and final charge.

## Pricing

Free to start. Top up a prepaid balance by card or crypto; no top-up fee and no subscription. See https://ahel.ai/pricing

## Support

support@ahel.ai · Terms: https://ahel.ai/terms · Privacy: https://ahel.ai/privacy

© Ahel Technologies OÜ
