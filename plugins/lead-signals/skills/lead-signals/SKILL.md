---
name: lead-signals
description: Use when the user wants to watch a list of target companies, buyers or topics and hear only what is new each day; asks for "lead signals", "buying signals", "tell me every morning about new tenders", "watch these companies for changes", "alert me when a supplier's register record changes or it shows up on a sanctions list", or wants a daily or scheduled sales or procurement watch. Keeps a local watch list and a seen set, runs official EU sources through Ahel (TED tenders and awards, business-register changes, sanctions lists), and writes a short Markdown report with a source URL and run id on every line.
metadata:
  summary: A watch list in, a daily Markdown report of only the new signals out (new TED tenders, register changes, sanctions matches), each with its source URL and run id, at a few cents a day.
---

# Lead signals

A watch list in, once a day a short report out with only what is new since the last run: new
public tenders for a topic or buyer, changes in the official business register for watched
companies, and possible sanctions matches for new names. Every line carries its source URL and the
Ahel run id. A signal is reported once.

Runs on Ahel through its MCP tools (`capability_inspect`, `run_capability`,
`capability_run_status`, `balance`). If those tools are missing, Ahel is not connected: tell the user
to install the `ahel` plugin (`claude plugin install ahel@ahel`) or add the MCP address
`https://mcp.ahel.ai`, then stop. The first call opens a sign-in at ahel.ai; do that once by hand
before scheduling, because a scheduled run cannot sign in.

## Sources

| Signal | Task id | Inputs | Price (checked 2 Nov 2026) | A miss costs |
| --- | --- | --- | --- | --- |
| New tenders for a topic | `procurement.tenders.search` | `{"query": "software", "country": "EE"}`; `max_items` 10-25 | $0.004 per call that returns notices | $0 |
| New awards by a buyer | `procurement.buyer.awards` | `{"buyer": "Riigi Infosüsteemi Amet", "country": "EE"}` | $0.01 per call that returns notices | $0 |
| Register changes | `companies.register.changes` | `{"country": "EE", "registry_number": "12417834"}`; `max_items` 5 | $0.01 per call that returns rows | $0 |
| Register status (baseline) | `companies.register.status` | `{"country": "EE", "registry_number": "12417834"}` | $0.002 per call | $0 (not found) |
| Sanctions screen | `sanctions.screen.company` | `{"name": "Bolt Technology OÜ", "country": "EE", "registry_number": "12417834"}` | $0.02 per possible match | $0 (no match) |
| Company news (optional, weekly) | `companies.news` | `{"domain": "bolt.eu"}`; `max_items` 10 | $0.044 per call (PredictLeads) | **$0.044: misses are billed** |
| Funding rounds (optional, weekly) | `companies.funding` | `{"domain": "bolt.eu"}` with `endpoint_ids: ["companyenrich.companies.funding"]` | $0.01078 per company found | $0 when pinned |

Prices are what `capability_inspect` answered on 2 Nov 2026; the quote at setup is the one that
counts. Pin `companies.funding` to CompanyEnrich as shown: unpinned, a miss falls through to
PredictLeads, which reserves up to $0.05478 and bills misses.

What each one really returns:

- **Tenders and awards** come from TED (EU Official Journal notices, not all public procurement).
  Each row has `notice_id`, `title`, `publication_date`, `notice_type` (`cn-*` = contract notice,
  `can-*` = award), `cpv`, `buyer_country`, `published_total_value`, `tender_deadline`, and a
  `source_url` like `https://ted.europa.eu/en/notice/-/detail/759301-2026`. Newest first.
  `procurement.buyer.awards` rows have `match_basis: "name_candidate"`: show them as candidates.
- **Register changes** are differences between Ahel's daily snapshots of the official register,
  newest first, with `change_type` (`added`, `changed`, `removed`), `observed_from`, `observed_at`,
  `before_data`, `after_data`. The dates are snapshot dates, not the date the registry event
  happened (`date_basis` says so). The call returns the change history, not only today's: a company
  that has any recorded change bills $0.01 on every check, and the seen set (below) hides the old
  rows. A company with no change on record answers no rows, free. Countries without snapshots
  also answer no rows, free; the baseline status call tells the two apart.
- **Sanctions** rows are possible name matches of legal entities on OFAC SDN, OFAC Consolidated
  Non-SDN and the EU Financial Sanctions File; `provenance.covered_sources` gives each list's date.
- **News** rows have `headline`, `summary`, `category` (`launches`, `partners_with`, ...),
  `found_at`, `effective_date`, and `provenance.event_id`. **There is no article URL.**
- **Funding** rows have `date`, `round_type`, `amount`, `investors`; `source_urls` and the licence
  came back empty, and the newest round for Bolt was May 2024. A funding check is a weekly
  look, not a daily signal.

Not in the skill yet: **people changing jobs.** `people.signals` is temporarily unavailable on Ahel
("An Ahel source for this task is not built yet"). If the user asks for it, say so and call
`catalog_request` once for `people.signals` so the demand is recorded. Do not substitute a people
search or a LinkedIn scrape.

## Setup (once, with the user)

1. **Build the watch list** in `lead-signals.json` next to the user's work (format below). For each
   company: a name, an ISO country (EE, FI, NO, LV, IE, FR are the official registers Ahel covers)
   and the registry number. No registry number: find it once with `companies.register.search` and
   confirm the match with the user (see the eu-company-check skill). Optional: a domain, only if
   the user wants weekly news or funding. Topics: a few words plus a country (`"software"`, `EE`).
   Buyers: the buyer's registered name plus country.
2. **Quote.** Call `capability_inspect` with real arguments on each task the list uses. Tell the
   user the daily ceiling and the expected cost (see "Daily cost") and that empty answers are free,
   except news. Ask for a daily ceiling in dollars and write it to `budget_usd_per_run`. Wait for a
   yes. This yes covers the scheduled runs up to that ceiling; a bigger list is a new quote.
3. **Baseline.** For each company run `companies.register.status` (confirms the register has it;
   $0.002) and `sanctions.screen.company` (no match is free). Run each topic and buyer once. Put
   every row returned into `seen`. Report the baseline as counts plus any sanctions match; do not
   list hundreds of old notices as "new".
4. **Schedule** it (below).

## The daily run

1. Read `lead-signals.json`. If it is missing, run Setup instead.
2. **Check the ceiling.** Sum the `max_cost_usd` of today's calls (topics × $0.004 + buyers × $0.01 +
   companies × $0.01 + new names × $0.02, plus news and funding on their weekly day). If it is above
   `budget_usd_per_run`, run nothing and write that in the report. Call `balance` when the ceiling
   is above $1.
3. **Run**, a handful in parallel at most, each with
   `idempotency_key: "ls-<YYYYMMDD>-<task>-<key>"` so a retry never bills twice:
   - each topic: `procurement.tenders.search`;
   - each buyer: `procurement.buyer.awards`;
   - each company: `companies.register.changes` with `max_items: 5`;
   - each company added since the last run, and each new supplier or buyer name the user asked to
     screen: `sanctions.screen.company`;
   - on `weekly_day` only, each company with a domain: `companies.news` and `companies.funding`
     (pinned).
   A run still going after the wait (`finished: false`): read it with `capability_run_status`.
   `outcome_unknown`: list it under "Could not check" and do not retry automatically.
4. **Diff.** Build each row's key (table below). Keep a row only if its key is not in `seen`. For
   tenders, also drop notices whose `tender_deadline` has passed.
5. **Write the report** (format below) to `lead-signals-<YYYYMMDD>.md` and print it.
6. **Save** the new keys into `seen` with today's date, append the run to `runs`, set `last_run`.
   Write the file only after the report is written, so a crash re-reports rather than loses a
   signal. Drop `seen` keys older than 180 days.

## State file

`lead-signals.json`, one per watch list:

```json
{
  "budget_usd_per_run": 0.30,
  "weekly_day": "Mon",
  "topics": [{"query": "software", "country": "EE"}],
  "buyers": [{"buyer": "Riigi Infosüsteemi Amet", "country": "EE"}],
  "companies": [
    {"name": "Bolt Technology OÜ", "country": "EE", "registry_number": "12417834",
     "domain": "bolt.eu", "added": "2026-11-02", "screened": "2026-11-02"}
  ],
  "seen": {
    "ted:759301-2026": "2026-11-02",
    "reg:EE:12417834:2026-11-02T01:10:37+00:00:company:changed": "2026-11-02"
  },
  "last_run": "2026-11-02",
  "runs": [{"date": "2026-11-02", "cost_usd": "0.006", "execution_ids": ["ex_639b..."]}]
}
```

| Signal | Seen key |
| --- | --- |
| Tender or award | `ted:<notice_id>` (one key for both tasks) |
| Register change | `reg:<country>:<registry_number>:<observed_at>:<kind>:<change_type>` |
| Sanctions match | `sanc:<country>:<registry_number or name>:<entity_id>` |
| News | `news:<provenance.event_id>` |
| Funding round | `fund:<domain>:<date>:<round_type>` |

A company with `screened` set is not screened again unless the user asks for a re-screen (a
standing possible match would bill $0.02 every time).

## Report format

```markdown
# Lead signals, 2 Nov 2026

3 topics, 1 buyer, 20 companies checked. 4 new signals. Cost $0.024.

## New tenders
- 2026-11-02 · EE · Software maintenance and repair services: Erakondade Rahastamise Järelevalve Komisjoni veebilahenduste hooldus- ja arendustööd · €350,000 · deadline 2026-11-30 · https://ted.europa.eu/en/notice/-/detail/759301-2026 · run ex_639b10d72d194f0eb74a32bdc778a004

## Changed companies
- Bolt Technology OÜ (EE 12417834) · changed: status Registrisse kantud → Likvideerimisel · observed 2026-11-03 (snapshot date) · https://avaandmed.ariregister.rik.ee/en/downloading-open-data · run ex_...

## Sanctions hits
- none (OFAC SDN 28 Oct, OFAC Non-SDN 19 Oct, EU FSF 23 Oct 2026)

## News and funding (weekly)
- Bolt · launches: three ride categories in Kenya · 2026-10-30 · PredictLeads event 2fcb7589-1127-48ea-8178-656d2548bcbf (no article link) · run ex_4a58a46224f942218a14d518192dddc5

## Could not check
- none

Sources: TED (Decision 2011/833/EU), RIK e-Business Register (CC BY 4.0), OFAC (US public domain), EU FSF (Decision 2011/833/EU).
```

Rules for the report: one line per signal; every line ends with its source URL and run id; values
and statuses in the source's own words; a section with nothing new says "none"; the header has the
exact cost, the sum of `execution.cost.customer_actual_usd` over today's runs, never a list price.
A sanctions row is "possible match, a person must review", with the listed name, list and date.
The changed-companies example line above shows the shape only; it is not a real change.

## Schedule

The schedule belongs to the user's agent, not to Ahel. Run it in the folder that holds
`lead-signals.json`, after the morning register import (07:00 local is fine).

- **Claude Code, desktop app:** ask "schedule the lead-signals daily run every weekday at 07:00"
  (a local scheduled task; it uses the same MCP connections as the app).
- **Claude Code, cron:**

  ```
  0 7 * * 1-5 cd ~/signals && claude -p "Run the lead-signals daily run." --allowedTools "mcp__ahel mcp__plugin_ahel_ahel Read Write Edit" >> lead-signals.log 2>&1
  ```

  `mcp__ahel` matches Ahel added as an MCP server named `ahel`; `mcp__plugin_ahel_ahel` matches the
  `ahel` plugin. Keep the one that fits.
- **Codex:** the same cron line with `codex exec "Run the lead-signals daily run."`.
- **Cursor:** the same cron line with `cursor-agent -p "Run the lead-signals daily run."`.

Ahel also has `companies.watch` (server-side daily company checks from a monitor budget, $0.003
per company per check). It is not used here: it is untested and needs a funded monitor budget.

## Daily cost

For 20 companies and 3 topics, from the 2 Nov 2026 prices:

| Part | Calls | Most per day | Usual day |
| --- | --- | --- | --- |
| Tenders | 3 topics × $0.004 | $0.012 | $0.012 |
| Register changes | 20 × $0.01 | $0.20 | $0.01 per company with any recorded change; $0 for the rest |
| Sanctions | new names only × $0.02 | $0.02 per new name | $0 (no match is free) |
| **Official signals** | | **$0.212 + $0.02 per new name** | **about $0.01-0.05** |
| News + funding, weekly | 20 × $0.044 + 20 × $0.01078 | $1.10 per week | $0.88 + found rounds |

One-off setup: 20 status reads $0.04 plus 20 sanctions screens (free unless a possible match,
$0.02 each). So the official watch costs at most about $6.40 a month; adding weekly news and funding
for all 20 adds at most about $4.75 a month. News is the expensive part because every call bills;
watch news for the five companies that matter most.

## Never

- **No personal data beyond business contacts.** The watch list holds companies, buyers and
  topics. Do not add people, do not track individuals' job moves, do not store names or contact
  details from notices (TED rows already exclude "organisation personal fields").
- **No scraping.** Every signal comes from an Ahel task call. Do not open the source sites to fill
  gaps, do not crawl LinkedIn, and do not invent a source URL (news has none: say so).
- **Nothing the data licence forbids.** The answers carry their terms; keep them with the data:
  - TED: "Source: European Union, TED. Reused under Decision 2011/833/EU; selected and normalized by
    Ahel. Official notices prevail." Scope: "Published TED notices; not all public procurement."
  - Estonia register: "Source: RIK e-Business Register, CC BY 4.0. Selected legal-entity facts
    normalized by Ahel; no official endorsement." Keep the attribution line in every report.
  - Sanctions: OFAC lists "US federal public domain", EU FSF "Commission Decision 2011/833/EU";
    scope: "A name match is not identity proof. Person-level ownership screening and the OFAC
    50-percent ownership rule are not assessed."
  - News: "PredictLeads data is proprietary and sourced from company websites; redistribution and
    resale rights were not reviewed." Funding (CompanyEnrich): no licence returned. Use both for the
    user's own work only; do not publish or resell them.
- No verdicts. A status change is a register fact, not a solvency or risk judgement; a sanctions row
  is a possible match for a person to review.
- No spend above `budget_usd_per_run` without a new yes, and no retries that could bill twice.

## Worked example

Live runs on 2 Nov 2026, total $0.006 for the official part.

**Tenders, "software" in Estonia.** `run_capability {"id": "procurement.tenders.search",
"arguments": {"query": "software", "country": "EE"}, "max_items": 10}` → run
`ex_639b10d72d194f0eb74a32bdc778a004`, 10 notices, $0.004. First four rows, trimmed:

```json
[{"notice_id": "759301-2026", "publication_date": "2026-11-02+01:00", "notice_type": "cn-standard",
  "title": "Estonia – Software maintenance and repair services – Erakondade Rahastamise Järelevalve Komisjoni veebilahenduste hooldus- ja arendustööd",
  "published_total_value": 350000, "tender_deadline": ["2026-11-30+02:00"],
  "source_url": "https://ted.europa.eu/en/notice/-/detail/759301-2026"},
 {"notice_id": "755875-2026", "publication_date": "2026-10-30+01:00",
  "title": "Belgium, Latvia, Germany, ... Estonia, Portugal – IT services: consulting, software development, Internet and support – Development, Implementation, Maintenance and Operation of financial IT Systems",
  "published_total_value": 700000000, "tender_deadline": ["2026-11-30+01:00"],
  "source_url": "https://ted.europa.eu/en/notice/-/detail/755875-2026"},
 {"notice_id": "754368-2026", "publication_date": "2026-10-30+01:00",
  "title": "Estonia – Software-related services – ELEKTROONILISE ANDMEKOGUMISPLATVORMI (EDC) KASUTAMINE SÕELUURINGUTE LÄBIVIIMISEKS",
  "tender_deadline": ["2026-12-02+02:00"],
  "source_url": "https://ted.europa.eu/en/notice/-/detail/754368-2026"},
 {"notice_id": "744850-2026", "publication_date": "2026-10-27+01:00",
  "title": "Estonia – Software programming and consultancy services – Riigiportaali eesti.ee ja kesksete teenuste arendus- ja hooldustööd",
  "published_total_value": 4000000, "tender_deadline": ["2026-11-24+02:00"],
  "source_url": "https://ted.europa.eu/en/notice/-/detail/744850-2026"}]
```

**Register changes, Bolt Technology OÜ (EE 12417834).** `run_capability {"id":
"companies.register.changes", "arguments": {"country": "EE", "registry_number": "12417834"}}` → run
`ex_2fe29f0ce0214251ab622f22bd59176f`, `rows: []`, $0 (the $0.01 hold was returned). Provenance:
snapshot observed `2026-11-02T01:10:37Z`, source updated "Sun, 01 Nov 2026 08:37:12 GMT", licence
CC-BY-4.0. The baseline `companies.register.status` (run `ex_588d04a5a230430eb0ea584b5c9df729`,
$0.002) confirms the register has the company: "Bolt Technology OÜ", registered 07.02.2013, status
"Registrisse kantud" (entered in the register). So no rows means "no change observed", not "not
covered". The sanctions screen (run `ex_87d4d15ab0a84ff08188914f4f854e20`) found no match, $0.

**Also checked:** `procurement.buyer.awards` for "Riigi Infosüsteemi Amet" EE (run
`ex_bed8cd459c0048648c6aa45f36d2dba6`, $0.01, 5 award notices, all `name_candidate`, newest
756152-2026 of 30 Oct). `companies.news` for bolt.eu (run `ex_4a58a46224f942218a14d518192dddc5`,
$0.044, 5 events, newest "Bolt SA introduces three ride categories in Kenya", 30 Oct, no article
URL). `companies.funding` for bolt.eu pinned to CompanyEnrich (run
`ex_b5f0b815ae46407f9867e9b13938e2b7`, $0.01078, 5 rounds, newest "Debt Financing" 236,903,031 USD
on 2024-05-03, no source URLs).

The first daily report from these runs, after the baseline put every notice in `seen`, lists only
notices published after 2 Nov and any change row for Bolt with an `observed_at` not seen before.
