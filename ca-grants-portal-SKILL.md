---
name: ca-grants-portal
description: >-
  Search and filter California state grant and loan opportunities from the California
  Grants Portal (grants.ca.gov) via its nightly open-data CSV — by category, applicant
  type, status, agency, deadline, funding source, award size, matching-funds
  requirement, letter-of-intent requirement, and free-text keyword. Use whenever
  someone wants to find California state funding, check whether a grant is still open,
  compare opportunities, pull deadlines and award amounts, or get contact details for a
  grantmaker. Triggers unnamed too: "what state grants can a nonprofit apply for," "any
  California funding for youth programs," "grants closing this month," "is that CalRecycle
  grant still accepting applications," "who do I contact about this RFA," "state grants
  with no match requirement." Resolve the current CSV before answering and never quote a
  deadline, dollar amount, or status from memory — the file changes nightly.
---

# California Grants Portal — search and filter

Grant and loan opportunities offered by California state agencies, published by the
California State Library under the Grant Information Act of 2018. The portal UI at
`grants.ca.gov` renders its result list in JavaScript and returns nothing to a fetch, so
**all filtering happens against the open-data CSV**, not the website's own search.

## What works and what doesn't

Verified Aug 9, 2026:

| Approach | Result |
| --- | --- |
| Bulk CSV on data.ca.gov | **Works.** Primary source. |
| Individual grant page `grants.ca.gov/grants/{slug}/` | **Works.** Clean and complete. Use for full detail. |
| Filtered search URLs (`grants.ca.gov/?s&grant_categories_search[]=...`) | Filter registers, **no results render**. JS-only. Don't use. |
| CKAN `datastore_search_sql` / `datastore_search` API | `ROBOTS_DISALLOWED` |
| `/datastore/dump/...` | `ROBOTS_DISALLOWED` |
| `bash` / `curl` | Fails — data.ca.gov and its S3 backend are off the network allowlist |

## Step 1 — Get the CSV

Dataset page: `https://data.ca.gov/dataset/california-grants-portal`
Download: `https://data.ca.gov/dataset/e1b1c799-cdd4-4219-af6d-93b79747fffb/resource/111c8c88-21f6-453c-ae2c-b4785a0624f5/download/california-grants-portal-data.csv`

The dataset **updates nightly at 8:45pm** — re-fetch each session rather than reusing an
older result, and never answer a status or deadline question from memory. If the download
404s, the filename moved: read the current Download link off the dataset page. `web_fetch`
only accepts URLs already seen, so `web_search` the dataset name first if a URL is rejected.

## Step 2 — Know the coverage ceiling, and say so

A single fetch returns roughly the **first 40 records** before truncating.
`text_content_token_limit` does not control this — passing a small value changes nothing.

Rows arrive sorted by `LastUpdated` **descending**, so one fetch reliably covers the most
recently updated opportunities (about the last two months, in the Aug 2026 test). It is
**not** a complete sweep of every open grant.

Never present a filtered list as exhaustive. Say what the result covers, e.g. "among the
grants updated in roughly the last two months." When the user needs broader coverage:

1. `web_search` scoped to the site — `site:grants.ca.gov {keywords}` — to surface
   opportunities outside the fetched window.
2. Fetch the individual grant page for anything promising: `https://www.grants.ca.gov/grants/{slug}/`.
   These pages carry the full record including field definitions, application links, and contacts.
3. Say plainly that a complete enumeration isn't possible through this route, and point to
   `https://www.grants.ca.gov/?s` for the interactive search.

## Step 3 — Filter

For anything beyond reading one or two records, write the fetched rows to a local `.csv`
and use pandas. Fields are wide and contain commas and newlines inside quotes — parse as
CSV, don't regex the text.

```python
import pandas as pd
df = pd.read_csv("grants.csv")
open_now = df[df["Status"].str.lower() == "active"]
match = open_now[
    open_now["ApplicantType"].str.contains("Nonprofit", na=False)
    & open_now["Categories"].str.contains("Education", na=False)
    & open_now["MatchingFunds"].str.contains("Not Required", na=False)
]
```

Deadlines are strings in mixed formats — mostly `YYYY-MM-DD HH:MM:SS`, but also `Ongoing`,
a bare year, or blank. Coerce with `pd.to_datetime(..., errors="coerce")` and handle the
non-dates separately rather than dropping them silently; `Ongoing` is a real, useful value.

## Fields (36 columns)

**Identity** — `PortalID`, `GrantID` (the agency's own ID, often blank), `Title`,
`AgencyDept`, `Type` (Grant, Loan, or both), `GrantURL`, `AgencyURL`,
`AgencySubscribeURL`, `GrantEventsURL`, `ContactInfo` (name, email, phone).

**Status and timing** — `Status`, `LastUpdated`, `ChangeNotes`, `OpenDate`,
`ApplicationDeadline`, `AwardPeriod`, `ExpAwardDate`.

`Status` values: `active` (open now), `forecasted` (announced, not yet open — may have no
deadline or only a year), `closed`. **The file includes closed and forecasted rows.** Filter
on `Status` or you will hand someone a dead opportunity.

**Eligibility** — `ApplicantType` (semicolon-delimited: Business, Individual, Nonprofit,
Other Legal Entity, Public Agency, Tribal Government), `ApplicantTypeNotes` (the real
eligibility prose — read it, the type codes are coarse), `Geography`.

**Subject** — `Categories` (semicolon-delimited; a grant often carries many),
`CategorySuggestion`, `Purpose` (short), `Description` (long — the best keyword target).

The 19 categories: Agriculture · Animal Services · Consumer Protection · Disadvantaged
Communities · Disaster Prevention & Relief · Education · Employment, Labor & Training ·
Energy · Environment & Water · Food & Nutrition · Health & Human Services · Housing,
Community and Economic Development · Law, Justice, and Legal Services · Libraries and Arts ·
Parks & Recreation · Science, Technology, and Research & Development · Transportation ·
Veterans & Military.

**Money** — `FundingSource` (State, Federal, or both), `FundingSourceNotes`,
`MatchingFunds` (`Not Required` or a percentage like `25%`, `100%`), `MatchingFundsNotes`,
`EstAvailFunds` (total pool), `EstAwards` (count, often "Dependant on number of
submissions…"), `EstAmounts` (per-award range), `FundingMethod` (Reimbursement(s),
Advances & Reimbursement(s)), `FundingMethodNotes`, `AwardStats` (JSON, applications
submitted vs. grants awarded in past years — useful for gauging competitiveness).

**Process** — `LOI` (letter of intent required), `ElecSubmission` (application portal link).

Several of these — matching-funds percentage, award size, funding method, LOI, funding
source — are **not filterable on the website itself**, so the CSV answers questions the
portal UI can't.

## Answering style

- Lead with the matches: title, agency, deadline, award range, status.
- Link the portal page (`grants.ca.gov/grants/{slug}/`) and the grantmaker's own guidelines URL from `GrantURL`.
- State the coverage caveat once, briefly.
- Flag `forecasted` rows as not yet open, and give the expected open date if present.
- Quote `ApplicantTypeNotes` eligibility prose rather than relying on the coarse `ApplicantType` codes.
- Don't judge whether someone qualifies. Surface the eligibility language and let them decide.
- Cite as: California State Library, California Grants Portal Data, [date accessed], `https://data.ca.gov/dataset/california-grants-portal`.

## Scope

State-agency grants and loans offered on a competitive or first-come basis, from July 1,
2020 forward. Not federal opportunities (use grants.gov), not local or private foundation
funding, and not awarded-grant records — this is the offering side only.
