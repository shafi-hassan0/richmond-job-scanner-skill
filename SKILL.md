---
name: richmond-job-scanner
description: Scans career pages of Richmond, VA-area companies (CoStar, Dominion Energy, Honeywell, CapTech, Atlantic Casualty, Richmond National, Capital One, CarMax, Markel, Genworth, Altria, WestRock, Elephant Insurance, Snagajob, Performance Food Group) for newly posted QA and Software Engineering job openings, and reports back only what's new since the last check. Use this whenever the user asks to check Richmond job postings, scan for QA/SWE openings, see if any of "my companies" are hiring, run their job search/job scan, or references this skill by name — including on a recurring/scheduled basis. Also use it if the user wants to add, remove, or list the companies being watched.
---

# Richmond Job Scanner

Monitors a fixed list of Richmond, VA-area employers for new QA and Software Engineering job postings, and reports only what's changed since the last run — the user doesn't want to re-read the same 40 open reqs every time.

## Company list

The watch list lives in `references/companies.json`, one entry per company with a `careers_url`, `ats` (the applicant tracking system, e.g. Workday, SmartRecruiters — useful context for how to search it), and a `search_hint` (a pre-built search query). Edit this file directly to add or remove companies — don't hardcode a company list in this file. If the user asks to add a company, look up its actual careers URL yourself (via WebSearch) rather than guessing one; guessed URLs frequently 404 or point at the wrong company.

## How to search each company

Career sites for large employers (especially Workday-based ones — CoStar, Capital One, CarMax, Markel, Genworth) are JavaScript single-page apps. A plain fetch of the URL often returns an empty page shell with no job listings in the HTML, even though the site works fine in a browser. Because of this:

1. **Prefer WebSearch using the `search_hint` for each company** rather than fetching `careers_url` directly. These search hints already combine the company, relevant job titles, and Richmond-area location terms, and many ATS platforms are well-indexed by search engines (they publish structured job-posting data for Google Jobs).
2. If WebSearch results look thin, try WebFetch on `careers_url` as a supplement — it sometimes works for simpler/server-rendered career pages (e.g. CapTech's SmartRecruiters board, Atlantic Casualty's ADP page).
3. Don't spend more than one WebSearch + one optional WebFetch per company — this is a broad sweep across 15 companies, not a deep dive into any one of them.

## What counts as a match

Include roles whose title reflects software engineering or QA/testing work: Software Engineer, Software Developer, Application Developer, SDET, QA Engineer, QA Analyst, Quality Assurance Engineer/Analyst, Test Engineer, Automation Engineer, Systems Engineer (Software), DevOps Engineer, and clear variants (Senior/Staff/Principal/Lead of the above, and internships — but label internships as such). Exclude unrelated "engineer" titles (Electrical, Mechanical, Sales, Network, Civil, etc.) unless the description makes clear it's a software role.

Location matters — these are national/multi-state companies. Only count a posting as a match if it's located in the Richmond metro (Richmond, Glen Allen, Henrico, Chesterfield, Midlothian, Short Pump, Mechanicsville, Chester, Ashland, VA) or is explicitly remote with a Richmond-based team/office. Don't include, e.g., a CoStar SWE role in Washington DC or a Capital One role in McLean.

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

If `seen_postings.json` is empty/fresh (first-ever run, `companies` is `{}`), every match found is technically "new" — say so explicitly in the report (e.g. "first run — showing all current matches") rather than implying these just opened.

## Report format

Keep it scannable. Group by company; skip companies with nothing new rather than listing them as "no new postings" (except in a one-line summary count). Example:

```
Richmond QA/SWE Job Scan — [date]
3 new postings across 2 companies (12 companies checked, no new matches elsewhere)

**CoStar Group**
- Associate Software Engineer — Richmond, VA — [link]
- Senior QA Engineer — Richmond, VA — [link]

**Capital One**
- Lead Software Engineer, Card Tech — Richmond, VA (West Creek) — [link]
```

If nothing new turned up anywhere, say that plainly in one line rather than an empty report — the user should be able to tell at a glance whether this run was worth reading.

## Notes worth surfacing to the user

- Richmond National is a small insurance underwriter with no confirmed tech hiring history — it's kept on the list at the user's request, but don't be surprised if it never produces a match.
- Atlantic Casualty's actual HQ is Goldsboro, NC; Richmond is a satellite office, so posting volume there will be low.
- ATS coverage for WestRock, Elephant Insurance, Snagajob, and Performance Food Group is unconfirmed/custom — if searches for these consistently return nothing, mention it so the user can decide whether to keep them or find a better source URL.
