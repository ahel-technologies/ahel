---
name: find-work-emails
description: Use when the user has a list of people (name plus company domain, or a LinkedIn URL) and wants their work email addresses found and checked; asks to "find emails", "enrich this lead list with emails", "get contact emails for these people" or "verify these emails"; or pastes a CSV of names and companies for outreach. Quotes the price first, then returns a CSV with a status per row and the total cost.
metadata:
  summary: A list of people in, a CSV of verified work emails out, with the price quoted before the run and the exact cost after.
---

# Find work emails

A list of people in, a CSV out: one row per person with the email, where it came from, whether the
mailbox checks out, and what that row cost. The price is quoted before anything runs.

Runs on Ahel through its MCP tools (`capability_inspect`, `run_capability`,
`capability_run_status`, `balance`). If those tools are missing, Ahel is not connected: tell the user
to install the `ahel` plugin from the same marketplace (`claude plugin install ahel@ahel`) or add the
MCP address `https://mcp.ahel.ai`, then stop. The first call opens a sign-in at ahel.ai.

## Tasks used

| Step | Task id | Inputs | Billing |
| --- | --- | --- | --- |
| Find | `people.email.find` | `{"full_name": "...", "domain": "acme.com"}`, or `{"first_name", "last_name", "domain"}`, or `{"linkedin_url": "https://www.linkedin.com/in/..."}` | Ahel tries providers cheapest first; most providers bill only when they find an email |
| Verify | `people.email.verify` | `{"email": "..."}` | One check per address; answers `status` (`valid`, `risky`, `invalid`), `catch_all`, `disposable`, `role_account` |

A LinkedIn URL unlocks the cheapest finder, so pass it when the user has one. `domain` is the bare
company domain (`bolt.eu`, not `https://www.bolt.eu/`).

## The run

1. **Read the list.** Accept a pasted table, a CSV or JSON file. Each row needs a name and a
   company domain, or a LinkedIn URL. Turn company names into domains only when the user gave them
   or the domain is unambiguous; otherwise ask. Drop exact duplicates and say how many.
2. **Quote before spending.** Call `capability_inspect` with `id: "people.email.find"` and the
   first row's `arguments`, and once with `id: "people.email.verify"` and a sample email. Each answer
   has `price.max_cost_usd` (the most one run can cost) and the per-provider rates. Tell the user, in
   two lines:
   - the ceiling: rows × (find max + verify max), e.g. "at most $0.42 for 10 people";
   - the expected cost: found rows × the first provider's rate + verify rate, and that misses are
     usually free.
   Call `balance` if the ceiling is more than a few dollars. Wait for a yes before running.
3. **Find**, one `run_capability` per person, `id: "people.email.find"`, with
   `idempotency_key: "fwe-<date>-<row>"` so a retry never bills twice. Do not set `verify` here;
   step 4 checks the address. Run a handful in parallel at most.
   - `state: "succeeded"` with a row: keep `email`, `confidence` (when given) and
     `execution.routing.served_by`.
   - `state: "succeeded"` with no rows: not found, $0.
   - Still running after the wait (`finished: false`): read it with `capability_run_status`.
   - `outcome_unknown` (a provider timed out): mark the row `unresolved`. The hold stays reserved
     until it settles. Do not retry automatically; offer one retry with
     `exclude_endpoint_ids: ["<the endpoint that timed out>"]` and say what that retry could cost.
4. **Verify** every found address with `run_capability`, `id: "people.email.verify"`. Map the answer:
   `valid` → `valid`; `risky` with `catch_all: true` → `catch-all` (the server accepts any
   address, so it can't be confirmed); `invalid` → `invalid`; anything else → `unknown`.
5. **Write the CSV** next to the user's work as `work-emails-<YYYYMMDD-HHMM>.csv`:

   ```
   name,domain,email,found_by,verify_status,catch_all,cost_usd,find_execution_id,verify_execution_id
   ```

   `cost_usd` is the sum of `execution.cost.customer_actual_usd` for that row's find and verify.
   Never a list price: always the settled amount from the answer.
6. **Report** in three lines: "10 people → 8 found, 5 valid, 2 catch-all, 1 invalid, 2 not found.
   Total $0.074 (≈ $0.015 per valid email). File: work-emails-….csv". Add unresolved holds if any.

## Rules

- Quote first, every time, and run only after the user agrees. A new list is a new quote.
- Money comes from the answers (`customer_actual_usd`), never from a price table.
- Never guess an email pattern yourself and present it as found.
- Never send the same person twice: every hit bills, repeats included.
- These are work emails for business outreach. Remind the user once that their outreach must
  follow the law where the recipient lives (in the EU: GDPR legitimate interest and an opt-out).
