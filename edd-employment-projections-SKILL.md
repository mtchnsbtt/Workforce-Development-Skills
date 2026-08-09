---
name: edd-employment-projections
description: >-
  Fetch and analyze California EDD Long-Term (10-year) employment projections from
  data.ca.gov — industry (NAICS) and occupational (SOC) growth or decline, job
  openings, exits, transfers, median wages, entry-level education, and typical job
  training, for the state, metro areas, and labor market regions. Use whenever a
  question turns on projected or future California employment: fastest-growing
  industries, occupations with the most openings, whether a job is growing or shrinking
  in a county or metro, wages and education for an occupation, or demand evidence for a
  training program or grant narrative. Triggers
  unnamed too: "is welding a growing field in California," "how many RN openings in
  the Inland Empire," "demand data for my Perkins application," "top jobs in
  Sacramento," "which industries are shrinking here." Always defaults to the newest
  published vintage — resolve it live from the dataset pages, never ask the user which
  year, and never quote a projection from memory.
---

# CA EDD Long-Term Employment Projections

Two CSVs on data.ca.gov, each ~360 KB. Industry file for NAICS questions; occupational
file for SOC questions (it also carries openings, wages, and education). Never ask which
vintage the user wants — resolve the current one.

## Step 1 — Resolve the latest CSV (mandatory, every session)

**Never answer from a remembered filename, a cached vintage, or a year in this file.**
EDD overwrites these resources and renames the file each cycle. Resolve first, every time.
Never ask the user which vintage — the answer is always the newest one available.

Both portals serve the same resource UUIDs but **their Download links drift out of sync**.
Verified Aug 9, 2026: data.ca.gov's industry link still pointed at `lt-ind-2023-2033.csv`,
which **404s**, while lab.data.ca.gov carried the live `lt-ind-emp-2024-2034.csv`.
So check both and take the newer.

| File | Portal A (lab) | Portal B (main) |
| --- | --- | --- |
| Industry | `https://www.lab.data.ca.gov/dataset/long-term-industry-employment-projections/5642307f-30c2-4ddb-b811-507b338e0b4d` | `https://data.ca.gov/dataset/long-term-industry-employment-projections` |
| Occupational | `https://www.lab.data.ca.gov/dataset/long-term-occupational-employment-projections` | `https://data.ca.gov/dataset/long-term-occupational-employment-projections` |

### Resolution procedure

1. Fetch a resource page and read its Download link plus the "Last updated" date. If the page URL is rejected because it wasn't seen in conversation, `web_search` the dataset name, then fetch the result.
2. Fetch the other portal's page for the same dataset and read its Download link too.
3. **Pick the filename with the later end year** (`lt-*-YYYY-YYYY.csv`). A newer "Last updated" date on a page with an older filename does not win — the filename's period is what decides.
4. Fetch that CSV. On a 404, the filename moved again: re-read both pages and take the newest link that actually returns data. Never fall back to an older vintage that happens to fetch without saying so.
5. Confirm the vintage from the data itself — read the `Period` column on the rows you use. If it is older than the filename implies, that is the mixed-vintage case below, not a resolution failure.

URLs follow this shape (dataset UUID / resource UUID are stable; only the filename changes):

- Industry: `https://data.ca.gov/dataset/b1ac39b1-33cc-4577-b584-6259406ce835/resource/5642307f-30c2-4ddb-b811-507b338e0b4d/download/lt-ind-emp-YYYY-YYYY.csv`
- Occupational: `https://data.ca.gov/dataset/715d1324-ac02-4b11-b922-86bafa6eb80f/resource/274e273c-d18c-4d84-b8df-49b4d13c14ce/download/lt-occ-emp-YYYY-YYYY.csv`

Newest seen Aug 9, 2026: `lt-ind-emp-2024-2034.csv`, `lt-occ-emp-2024-2034.csv`. This is a
**floor, not an answer** — if resolution turns up anything later, use it. If resolution
somehow yields something earlier, resolution failed; retry rather than answer from it.

If the user explicitly names an older vintage, use it and say the current one is newer.

## Step 2 — Fetch

- `web_fetch` only. `bash`/`curl` fail (data.ca.gov and its S3 backend are off the network allowlist).
- The datastore dump endpoint (`/datastore/dump/...`) returns `ROBOTS_DISALLOWED`, and the Data API page is blocked. Don't try them.
- `text_content_token_limit` does **not** meaningfully truncate these CSVs — expect a large dump (tens of thousands of tokens) whatever you pass. Fetch **once per session** and reuse.
- Rows are alphabetical by Area Name, then by NAICS/SOC hierarchy. Statewide rows sit under Area Name "California" (Area Type State), between Bakersfield and Chico.

## Step 3 — Parse and answer

Simple lookup (one area + one industry or occupation): read the value straight from the
fetched text.

Ranking, sorting, or aggregation: filter the fetched text to the rows you need, write
**only that subset** to a local `.csv`, and use pandas. Writing the whole file back out is
wasteful.

```python
import pandas as pd
df = pd.read_csv("subset.csv")
df["Percentage Change"] = pd.to_numeric(df["Percentage Change"], errors="coerce")
top = (df[df["NAICS Level"].astype(str) == "3"]
         .sort_values("Percentage Change", ascending=False).head(10))
```
Occupational equivalent: filter `SOC Level == "4"`, coerce `Total Job Openings` /
`Median Annual Wage`. Always coerce with `errors="coerce"` so `No data` and `N/A` become
NaN instead of breaking the sort.

## Columns

Shared: `Area Type, Area Name, Period, <level>, <code>, <title>, Base Year Employment
Estimate, Projected Year Employment Estimate, Numeric Change, Percentage Change`.

**Industry** — `NAICS Level, NAICS Code, Industry Title`.
- Levels 1–4, bigger = finer. Level 4 detail exists for statewide rows; metros and consortia usually stop at Level 3.
- Level 1 = `Goods-Producing` (101) / `Service-Providing` (102). Level 2 includes sector rollups (`1011` Natural Resources and Mining, `1013` Manufacturing) alongside real NAICS sectors. Government rows carry Level 2 with NAICS Code `No data`.
- Summary rows with Level `No data`: **Total Employment**, **Self Employment**.
- Never add a rollup to its own children.

**Occupational** — `SOC Level, Standard Occupational Classification (SOC), Occupational
Title`, plus `Exits, Transfers, Total Job Openings, Median Hourly Wage, Median Annual
Wage, Entry Level Education, Work Experience, Job Training`.
- SOC Level 1–4; 4 = detailed occupation. `00-0000` = Total, All Occupations.
- `Total Job Openings` = growth + exits + transfers over the period. This is the headline "how many openings" number.
- Wages show `0.00`/`0` on summary levels 1–3, and on some Level 4 rows too (suppressed). Education/experience/training show `N/A` on summary lines.
- `Job Training` values include `Apprenticeship` — useful for isolating apprenticeable occupations (electricians, carpenters, sheet metal workers, elevator installers, etc.).

Pick the measure that matches intent and say which you used: **Percentage Change** for
"fastest growing," **Numeric Change** for "most new jobs," **Total Job Openings** for
"most hiring." They rank very differently.

## Caveats to state when they affect the answer

- **Mixed vintages in one file.** Confirmed Aug 9, 2026: `lt-ind-emp-2024-2034.csv` carries `Period` 2024-2034 on California statewide rows but **2023-2033** on metro and consortium rows. The filename is not the row's period. Always report the `Period` value from the row you used, and don't compare a 2024-2034 statewide figure against a 2023-2033 metro figure as if they were the same vintage.
- **Suppression.** Confidential cells are withheld, so detail doesn't always sum to summary lines. Industry shows `No data`; some areas show long runs of `0`.
- **Geography floor.** Nothing below labor market regions/metros. Area Types: State, Metropolitan Area, Consortium (some metro names carry the county in parentheses).
- Cite the dataset name and the resolved period every time.
