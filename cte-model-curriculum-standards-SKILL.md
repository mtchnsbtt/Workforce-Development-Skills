---
name: cte-model-curriculum-standards
description: >-
  Fetch and parse California's CTE Model Curriculum Standards (CDE, adopted January 2013)
  — the 12 Standards for Career Ready Practice, the 11 Knowledge and Performance Anchor
  Standards, and the pathway standards for all 15 industry sectors, straight from the
  source PDFs on cde.ca.gov. Use whenever a question involves CTE standards, anchor or
  pathway standard codes, career pathways within a sector, sample occupations, or writing
  and aligning curriculum, course outlines, or lesson plans to them. Triggers unnamed too:
  "what are the anchor standards," "what does B5.3 mean," "which pathways are in the
  Building and Construction Trades sector," "standards for a welding course," "align this
  syllabus to state CTE standards," "Career Ready Practice list." Always fetch the sector
  PDF and quote the real standard text — never reconstruct a standard code or its wording
  from memory.
---

# CTE Model Curriculum Standards — fetch and parse

California Department of Education, adopted by the State Board January 16, 2013 under
Education Code §51226. Landing page:
`https://www.cde.ca.gov/ci/ct/sf/ctemcstandards.asp`

**Check for a newer edition first.** CDE has an active revision underway at
`https://www.cde.ca.gov/ci/ct/sf/ctemcsupdate.asp`. Fetch that page when currency matters
and say which edition an answer came from. The 2013 standards remain the adopted ones
until replaced.

## What works

Verified Aug 9, 2026:

- Landing page fetches cleanly and lists all 19 PDFs with sizes.
- Sector PDFs extract well with `web_fetch` and `web_fetch_pdf_extract_text: true`.
- `bash` / `curl` cannot reach cde.ca.gov — it is off the network allowlist. `web_fetch` only.
- `web_fetch` only accepts URLs already seen in the conversation. The URLs below are in this
  file, not in the conversation, so if one is rejected, fetch the landing page first (it links
  every PDF) or `web_search` for the document, then fetch the result.

## Documents

General:

| Document | URL |
| --- | --- |
| CTE Standards Introduction (4MB) — executive summary, full front matter | `https://www.cde.ca.gov/ci/ct/sf/documents/ctestdfrontpages.pdf` |
| Standards for Career Ready Practice — flyer | `https://www.cde.ca.gov/ci/ct/sf/documents/ctescrpflyer.pdf` |
| Standards for Career Ready Practice — poster | `https://www.cde.ca.gov/ci/ct/sf/documents/ctescrpposter.pdf` |
| CTE Career Pathways — poster | `https://www.cde.ca.gov/ci/ct/sf/documents/ctecpwposter.pdf` |

The 15 industry sectors, all at `https://www.cde.ca.gov/ci/ct/sf/documents/{file}`:

| Sector | File | Size |
| --- | --- | --- |
| Agriculture and Natural Resources | `agnatural.pdf` | 1MB |
| Arts, Media, and Entertainment | `artsmedia.pdf` | 1MB |
| Building and Construction Trades | `buildingconstruct.pdf` | 5MB |
| Business and Finance | `bizfinance.pdf` | 1MB |
| Education, Child Development, and Family Services | `edchildfamily.pdf` | 4MB |
| Energy, Environment, and Utilities | `energyutilities.pdf` | 3MB |
| Engineering and Architecture | `enginearchit.pdf` | 3MB |
| Fashion and Interior Design | `fashioninterior.pdf` | 3MB |
| Health Science and Medical Technology | `healthmedical.pdf` | 1MB |
| Hospitality, Tourism, and Recreation | `hosptourrec.pdf` | 1MB |
| Information and Communication Technologies | `infocomtech.pdf` | 4MB |
| Manufacturing and Product Development | `manproddev.pdf` | 5MB |
| Marketing Sales and Service | `mktsalesservices.pdf` | 1MB |
| Public Services | `pubservices.pdf` | 1MB |
| Transportation | `transportation.pdf` | 1MB |

Every sector PDF is self-contained: it repeats the overview and all 12 Standards for Career
Ready Practice before its own sector content. **Fetch one sector, not the introduction plus
a sector** — the front matter comes free.

## Structure inside each sector PDF

1. **Table of contents** — lists that sector's pathways by letter. Read this first to learn how many pathways exist and what they are; the count and names differ by sector.
2. **Overview** — how the standards are organized and intended to be used.
3. **California Standards for Career Ready Practice** — 12 numbered practices, identical in every sector PDF.
4. **Sector Description** — one paragraph naming the pathways.
5. **Knowledge and Performance Anchor Standards** — 11 standards, common across all 15 sectors but *customized per sector* with sector-specific additions. Numbered `1.0`–`11.0`, each with performance indicators (`2.1`, `2.2`, …). Fixed titles: 1.0 Academics · 2.0 Communications · 3.0 Career Planning and Management · 4.0 Technology · 5.0 Problem Solving and Critical Thinking · 6.0 Health and Safety · 7.0 Responsibility and Flexibility · 8.0 Ethics and Legal Responsibilities · 9.0 Leadership and Teamwork · 10.0 Technical Knowledge and Skills · 11.0 Demonstration and Application.
6. **Pathway Standards** — one section per pathway, lettered `A`, `B`, `C`… Each opens with a pathway description and a short list of sample occupations, then standards coded `A1.0`, `A1.1`, `A2.0`. Pathways carry 8–12 pathway-specific standards by design.
7. **Academic Alignment Matrix** — see the warning below.
8. **Contributors** and **References** — at the very end, often past the extraction cutoff.

**Code notation:** a bare number (`5.0`, `5.2`) is an anchor standard; a letter prefix
(`B5.0`, `B5.3`) is a pathway standard, where the letter identifies the pathway *within that
sector only*. `B5.3` means something different in every sector, so always name the sector
alongside a code. Anchor standards 2–10 carry an inline Common Core alignment note such as
"(Direct alignment with LS 9-10, 11-12.6)" — that notation is reliable in extracted text.

## Two extraction limits

**The Academic Alignment Matrix does not survive text extraction.** It is a wide
multi-column table with rotated headers; extracted output collapses pathway columns
together and degenerates into scrambled characters. Do not read alignment data out of it,
and do not present a reconstructed matrix — the errors are silent and look plausible.
When someone needs academic alignment:
- Use the inline "(Direct alignment with …)" notes on anchor standards 2–10, which extract correctly.
- Otherwise point them to the matrix by page in the sector PDF and let them read it directly.

**A single fetch truncates before the end of a sector PDF.** In the Aug 2026 test on the
1MB health sector file, extraction ran to roughly page 43 of 60 — all narrative content
came through, but the later matrix pages, Contributors, and References were cut. The larger
5MB files will cut proportionally earlier. Never claim a sector has no pathway X because you
didn't see it; check the table of contents at the top of the PDF, which is always in range.

## Answering style

- Quote standards verbatim. The wording is the deliverable, and paraphrase loses the measurable verb.
- Always pair a code with its sector: "Health Science and Medical Technology, B5.3," not "B5.3."
- Give the full code path when citing an indicator, e.g. anchor 6.0 → 6.4, or pathway A3.0 → A3.4.
- When someone asks which sector or pathway fits a course, name the candidates and quote the pathway descriptions and sample occupations so they can judge — the descriptions are the actual selection criteria.
- Note that pathway standards are not designed to be taught as single courses; they get collected and sequenced, and standards from more than one sector can be combined for a course.
- Cite as: California Department of Education, *California Career Technical Education Model Curriculum Standards*, adopted January 2013, with the sector name and PDF URL.

## Scope

The 2013 CTE Model Curriculum Standards only. Not the CTE Framework, not Perkins
requirements, not course approval or A–G, and not any other state's standards.
