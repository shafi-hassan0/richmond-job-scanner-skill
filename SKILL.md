---
name: richmond-job-scanner
description: Scans career pages of Richmond, VA-area companies (CoStar, Dominion Energy, Honeywell, CapTech, Atlantic Casualty, Richmond National, Capital One, CarMax, Markel, Genworth, Altria, WestRock, Elephant Insurance, Performance Food Group) for newly posted security and network job openings (Security Engineer, Security Analyst, SOC Analyst, Cybersecurity Analyst, Network Engineer, Network Analyst), and reports back only what's new since the last check. Use this whenever the user asks to check Richmond job postings, scan for security/cybersecurity/SOC/network openings, see if any of "my companies" are hiring, run their job search/job scan, or references this skill by name — including on a recurring/scheduled basis. Also use it if the user wants to add, remove, or list the companies being watched.
---

# Richmond Job Scanner

Monitors a fixed list of Richmond, VA-area employers for new security and network job postings, and reports only what's changed since the last run — the user doesn't want to re-read the same 40 open reqs every time.

## Company list

The watch list lives in `references/companies.json`, one entry per company with `ats` (confirmed platform), `official_domains` (the allowlist for that company — see "Only trust the company's own site" below), a `method` (`workday_api`, `smartrecruiters_api`, `bamboohr_api`, `oracle_cloud_api`, `sap_rmk_api`, or `direct_url`), and either an `api` block (exact endpoint, HTTP method, headers, body template, and how to extract title/location/URL from the JSON response) or a `search_url`/`careers_url` plus `extraction` notes for HTML-based boards. Every entry was verified with a live HTTP request on 2026-08-29 — re-verify with the same method (see each entry's `notes`) rather than assuming a URL or facet ID stays valid forever, ATS vendors and tenant names do change (WestRock and CarMax both turned out to differ from earlier assumptions).

Edit this file directly to add or remove companies — don't hardcode a company list in this file. If the user asks to add a company, find and verify its actual API/search URL yourself the same way this file was built (see "How to search each company" below) rather than guessing one; guessed URLs frequently 404 or point at the wrong company, and a WebSearch-only lookup tends to surface aggregator links (Indeed, ZipRecruiter, Glassdoor, etc.) instead of the company's own site.

## Only trust the company's own site

**Never surface a link from a third-party job aggregator** (Indeed, ZipRecruiter, Glassdoor, LinkedIn, TheLadders, BuiltIn, TheMuse, Jooble, SimplyHired, and similar) — these links rot fast and frequently point at expired or mismatched postings. Every URL in the report must resolve to a domain listed in that company's `official_domains` array in `companies.json`. If a search or fetch turns up a promising posting on a domain not in that list, either find the same posting on the company's own ATS (most listings are cross-posted) or drop it — don't include it.

## How to search each company

Use each company's `method`/`api`/`search_url` from `companies.json` directly — this is faster and far more reliable than a generic web search, and it's what was actually tested when this file was built. **But first, check whether direct network access even works in this environment** — some sandboxes (notably scheduled cloud routines) run behind an egress proxy that blocks essentially all outbound HTTPS except GitHub/package-registry domains, while a local/interactive session usually has normal internet access. Don't discover this company-by-company (that burns 15 failed requests before you notice the pattern): make ONE test request first (e.g. `curl` the CoStar Workday endpoint, or WebFetch any `direct_url` entry) and check its result.

- **If the test request succeeds** (real JSON/HTML comes back, not a proxy/egress error): you have direct access. Proceed company by company:
  1. **`workday_api` / `smartrecruiters_api` / `bamboohr_api` / `oracle_cloud_api` / `sap_rmk_api`** — call the documented `api.endpoint` with Bash (`curl`) or WebFetch, using the `body_template`/`query_params` given (the `"software engineer"` search text stored in `companies.json` is a placeholder left over from the original SWE/QA scan — replace it and run one query per term: `"security"`, `"cyber"`, `"SOC"`, `"network"`; or clear the search text to pull the full Richmond board and filter client-side, paging through `offset` if `total` exceeds `limit`). These return real JSON with no JS rendering needed — parse per the entry's `extraction` notes to get title, location, and a working job URL.
  2. **`direct_url`** — WebFetch the `search_url`/`careers_url`. These are confirmed server-rendered (real listings appear in the raw HTML) unless the entry's `notes` say otherwise (a few sites — elephant.com, jobs.performancefoodservice.com — sit behind bot-protection that blocks a plain `curl` even though the same URL renders fine in a real browser; if WebFetch comes back blocked/empty for one of these, that's expected, not a sign the URL is wrong).
  3. Several ATS search boxes (Dominion, Capital One, Markel, Altria, CoStar) rank by relevance rather than doing a strict keyword AND — results can include unrelated titles (e.g. "Network Marketing", "Social Security Benefits Specialist", "Security Officer") even when searching "security" or "network". Always apply the title regex from "What counts as a match" below client-side; don't trust the API/page's own ranking as a filter.
- **If the test request fails with a proxy/egress/policy-denial error** (not a normal 404 or timeout — look for language like "egress proxy," "connect_rejected," "policy," or `EGRESS_BLOCKED`): direct access is blocked for this entire run, not just this one company. Don't keep retrying other companies' direct methods one by one — switch immediately to WebSearch for all 15 (e.g. `"<company name> security analyst OR security engineer OR SOC OR network engineer Richmond VA"`), and run every result through the "Only trust the company's own site" filter above before including it. Note in the report that this run used the WebSearch fallback due to a network restriction, so the user understands why results might be sparser than a direct-access run.

## What counts as a match

Include roles whose title reflects cybersecurity or network work: **Security Engineer, Security Analyst, SOC Analyst, Cybersecurity Analyst, Network Engineer, Network Analyst**, and clear variants — Information Security Analyst/Engineer, Cyber Security Engineer, Security Operations (SOC) Analyst/Engineer, Incident Response Analyst, Threat Detection/Intelligence Analyst, Network Administrator, Network Security Engineer, and Junior/Associate/Senior/Staff/Principal/Lead or Tier 1/2/3 versions of the above. Internships count, but label them as such.

Exclude:
- Software engineering, software development, and QA/test roles (Software Engineer, SDET, QA Engineer, Test Engineer, etc.), even if "security" appears in the title (e.g. "Software Engineer, Security Platform") — unless the posting is clearly a security operations/engineering role rather than product development.
- Non-IT uses of the words: physical security/guard/officer roles, "Social Security", "network" in the sales/marketing/provider-network sense, electrical/utility "network" or grid engineering (watch for this at Dominion Energy).

Location matters — these are national/multi-state companies. Only count a posting as a match if it's located in the Richmond metro (Richmond, Glen Allen, Henrico, Chesterfield, Midlothian, Short Pump, Mechanicsville, Chester, Ashland, VA) or is remote and open to candidates in Virginia (remote-US counts; remote limited to other states does not). Don't include, e.g., an on-site Capital One role in McLean.

## Details to pull for each new posting

For each posting that's new this run (not ones already in state), open the job detail page/API record and capture:

- **Title, company, location or remote status** (Richmond on-site / hybrid / remote).
- **Seniority level** — from the title or the posting (Entry/Associate, Mid, Senior, Lead/Principal, Manager, Intern). If the posting doesn't say, infer from required years of experience and note it as inferred.
- **Date posted** — from the API (e.g. Workday `postedOn`) or the page. If unavailable, write "date not listed".
- **Link** — on an `official_domains` domain, per the rule above.
- **Flags** — note if the description mentions any of: SIEM tools (Wazuh, Splunk, Microsoft Sentinel, QRadar, Elastic, etc.), firewalls (Palo Alto, Fortinet, Cisco ASA/Firepower, etc.), or network monitoring (SolarWinds, PRTG, Zeek, Wireshark, NetFlow, etc.). Name the specific tools mentioned.
- **Certifications** — list any required or preferred certs (Security+, CySA+, CCNA, CCNP, CISSP, GSEC, etc.), marking which are required vs. preferred.

Postings should be recent: if a new posting's date posted is more than 7 days old, still include it (it's new to this tracker) but note its date so the user can see it's not fresh.

## Tracking what's new

State is stored in `state/seen_postings.json`:

```json
{
  "last_run": "2026-08-27T00:00:00Z",
  "companies": {
    "CoStar Group": [
      {"title": "Associate Software Engineer", "location": "Richmond, VA", "url": "...", "first_seen": "2026-08-27"}
    ]
  }
}
```

Before searching, read this file. After finding current matching postings for a company, compare against the stored list for that company (match on title + location, since URLs can change between crawls). Anything not already present is new — report it. Anything already present, skip in the report (it was already surfaced on a prior run).

After compiling the report, **update the file**: for each company, replace its list with the current set of matching postings found this run (so postings that have since been filled naturally drop off), and set `last_run` to the current timestamp. Write the file back — this is what makes the next run only show genuinely new postings.

If `seen_postings.json` is empty/fresh (first-ever run, `companies` is `{}`), or none of its stored entries match the current role criteria (e.g. the first run after the role list changed from SWE/QA to security/network), every match found is technically "new" — say so explicitly in the report (e.g. "first run — showing all current matches") rather than implying these just opened.

## Report format

Keep it scannable. Group by company; skip companies with nothing new rather than listing them as "no new postings" (except in a one-line summary count). Example:

```
Richmond Security/Network Job Scan — [date]
3 new postings across 2 companies (12 companies checked, no new matches elsewhere)

**Capital One**
- Senior Cybersecurity Analyst — Richmond, VA (hybrid) — Senior — posted 2026-09-21 — [link]
  🚩 SIEM: Splunk; Firewalls: Palo Alto · Certs: Security+ (required), CySA+ (preferred)

**Dominion Energy**
- Network Engineer II — Glen Allen, VA — Mid — posted 2026-09-20 — [link]
  🚩 Network monitoring: SolarWinds · Certs: CCNA (preferred)
- SOC Analyst (Tier 1) — Remote (US) — Entry — posted 2026-09-22 — [link]
  Certs: none listed
```

Only show the 🚩 line when at least one SIEM/firewall/network-monitoring mention was found; always show certs (or "none listed").

If nothing new turned up anywhere, say that plainly in one line rather than an empty report — the user should be able to tell at a glance whether this run was worth reading.

## Notes worth surfacing to the user

- Richmond National actually has an active tech hiring history (confirmed live SWE/QA/ML openings as of 2026-08-29) — the earlier assumption that it never posts tech roles was wrong; don't be surprised either way run to run.
- Performance Food Group's Richmond HQ IT roles run infra/security-flavored (Network Services Manager, Info Security Analyst seen 2026-08-29) — filter its board by the 'Information Systems' career area and 'West Creek-Richmond Corp (0999)' location rather than a keyword search (see its `notes`).
- Many `notes` and `verified_on` counts in `companies.json` ("zero SWE/QA openings") describe the old SWE/QA scan — they confirm the endpoint works, not that there are zero security/network roles.
- Atlantic Casualty's actual HQ is Goldsboro, NC; Richmond is a satellite office, so posting volume there will be low, and the ADP job list page doesn't show location — check the (JS-rendered) job detail page before counting a hit as Richmond-based.
- WestRock has a plant literally called "North Richmond" — in Sydney, Australia. A location search on the word "Richmond" alone will pull those in; always confirm ", VA" / "Virginia" before counting a WestRock match.
- CoStar, Capital One, and CarMax's ATS tenant/site names differ from what you might guess (CoStar: `CoStarCareers` not `Costar_Campus`; Capital One: tenant `wd12` not `wd1`; CarMax: Workday, not Oracle Cloud as previously assumed) — use the values already recorded in `companies.json` rather than re-guessing.
- Full company-by-company verification detail (API bodies, facet IDs, confirmed domains, what was tried and ruled out) lives in `references/companies.json` — read it before troubleshooting a company that stops returning results.
