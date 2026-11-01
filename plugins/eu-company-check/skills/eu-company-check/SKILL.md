---
name: eu-company-check
description: Use when the user wants to check or vet EU companies (a supplier, customer, partner or a list of them) by name and country or by registry code; asks "is this company real / still registered", "what is their registry number, VAT or LEI", "screen these companies for sanctions", KYB, know-your-supplier or due diligence on companies in Estonia, Finland, Norway, Latvia, Ireland or France. Quotes the price first, then returns one sourced table and the total cost.
metadata:
  summary: EU companies in, one table out with the official register record, LEI and a sanctions screen, each fact with its source and date, and the exact cost.
---

# EU company check

Companies in, one table out: the official register record (registry code, legal form, status,
registration date, VAT where the register has it), the LEI, and a sanctions screen. Every fact keeps
its source and date. The price is quoted before anything runs.

Runs on Ahel through its MCP tools (`capability_inspect`, `run_capability`,
`capability_run_status`, `balance`). If those tools are missing, Ahel is not connected: tell the user
to install the `ahel` plugin from the same marketplace (`claude plugin install ahel@ahel`) or add the
MCP address `https://mcp.ahel.ai`, then stop. The first call opens a sign-in at ahel.ai.

## Coverage

Official registers: Estonia (RIK e-Business Register), Finland (PRH/YTJ), Norway
(Brønnøysundregistrene), Latvia, Ireland, France. A country outside this list is answered free with
no record; say so rather than guessing. LEI comes from GLEIF (worldwide, only companies that hold an
LEI). The sanctions screen covers OFAC SDN, OFAC Consolidated Non-SDN and the EU Financial Sanctions
File, legal entities only.

## Tasks used

| Step | Task id | Inputs | Billing |
| --- | --- | --- | --- |
| Find the record | `companies.register.search` | `{"name": "Pipedrive", "country": "EE"}` with `endpoint_ids: ["ahel-registry.companies.register.search"]`, `max_items: 3` | Per successful call; a miss is free |
| Read the record | `companies.register.profile` | `{"country": "EE", "registry_number": "11958539"}` | Per successful call |
| Status only | `companies.register.status` | `{"country": "EE", "registry_number": "11958539"}` | Per successful call |
| LEI | `companies.lei.search` | `{"registry_number": "11958539", "country": "EE"}`; by name only when there is no registry code | Per hit; no LEI is free |
| Sanctions | `sanctions.screen.company` | `{"name": "Pipedrive OÜ", "country": "EE", "registry_number": "11958539"}` | Per hit; no match is free |

Pin `companies.register.search` to the `ahel-registry` endpoint as shown. Without the pin, a miss
falls through to a scraper that bills per result and charges misses.

## The run

1. **Read the list.** Each company needs a country (ISO code: EE, FI, NO, LV, IE, FR) and a name or
   a registry code. If the user gave a registry code, skip step 3 for that company.
2. **Quote before spending.** Call `capability_inspect` with arguments for one company on each task
   you will use (`price.max_cost_usd` is the most one run can cost). Tell the user the ceiling
   (companies × sum of the maxima) and the expected cost, and that misses and "no sanctions match"
   are free. Wait for a yes.
3. **Find the registry code** with `companies.register.search`. Search with the plain name: drop
   punctuation ("Kahoot", not "Kahoot! ASA") and try without the legal form if the first try misses.
   Check that a returned `name` really is the company. A search that returns a different company
   still bills, so never pass a doubtful match on: show the candidates and ask. No match after two
   tries: mark the company `not found by name` and ask for the registry code.
4. **Read the record** with `companies.register.profile`. Keep `name`, `registry_number`,
   `legal_form`, `status`, `registration_date`, `vat_number` (Estonia), `domain`, and for Norway
   `bankrupt`, `winding_up`, `compulsory_dissolution`, plus `source`, `source_updated_at` and
   `licence`. The profile already includes the status, so call `companies.register.status` only
   when the user wants a status re-check or a list watched over time.
   Status codes as the register writes them: Estonia `Registrisse kantud` = entered in the register;
   Finland `1` = registered; Norway `registered`.
5. **LEI** with `companies.lei.search` by `registry_number` and `country` (name search misses more).
   Report `lei`, `lei_status` (`ISSUED`, or `LAPSED` = not renewed) and the GLEIF `source_url`.
   No row means no LEI on file, which is normal for private companies.
6. **Sanctions** with `sanctions.screen.company`. No rows means no possible match on the covered
   lists; report it as "no match on OFAC SDN, OFAC Non-SDN, EU FSF (list dates …)" using the dates
   in `provenance.covered_sources`. A row is a possible name match, not proof of identity: show the
   listed entity, list and date, and tell the user a person must review it.
7. **Write the table** as `eu-company-check-<YYYYMMDD-HHMM>.csv` and show it:

   ```
   input,country,name,registry_number,legal_form,status,registered,vat,lei,lei_status,sanctions,sources,cost_usd
   ```

   `sources` names each register, GLEIF and the sanctions lists with their dates. `cost_usd` is the
   sum of `execution.cost.customer_actual_usd` for that company's calls.
8. **Report** in three lines: "10 companies → 9 records, 6 LEIs (2 lapsed), 0 sanctions matches.
   Total $0.084. File: …". Name any company not found.

## Rules

- Quote first, every time, and run only after the user agrees.
- Money comes from the answers (`customer_actual_usd`), never from a price table.
- Report what the register says, in its words, with the source and date. No solvency or risk
  verdict: the register status is a fact, not an assessment.
- The sanctions screen is a name screen of legal entities on the named lists. It does not check
  owners or directors, the OFAC 50 percent rule, or other regimes. Say this once in the report.
