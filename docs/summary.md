# Job Alerts Automation — Summary

**What it is:** An n8n workflow called **"Job Search - LinkedIn to Discord"** that scrapes LinkedIn for relevant roles, scores each posting against a resume with AI, and posts the good matches straight to a Discord channel — no manual searching required.

## What it does, in order

1. **Runs on a schedule** — 3x a day (9am, 3pm, 9pm, Toronto time).
2. **Scrapes LinkedIn** via an Apify scraper, searching two locations in parallel: fully remote roles, and roles in a specific city (currently Toronto). Keywords target Project/Program/Operations Manager–style titles.
3. **Filters for freshness** — the morning run pulls a wider window (past week) to catch anything missed overnight; the afternoon/evening runs only look at postings from the last hour, so the same jobs aren't re-scanned all day.
4. **Checks for duplicates** — every job URL is checked against a Google Sheet log before anything else happens, so a job already seen (posted before or already scored) is skipped.
5. **Scores new jobs with AI** — each new posting is sent to Perplexity along with a resume and a scoring rubric (location/work arrangement, compensation, employment type, resume fit, seniority, bonus industry fit). Roles that are hybrid/in-office in the wrong country, junior-level, or missing work authorization are auto-excluded regardless of score.
6. **Posts good matches to Discord** — anything scoring 65+ and not excluded gets posted as a rich embed (title, company, match score, pay, key skills, why it's a fit, apply link) and the account owner is @mentioned. Jobs posted within the last hour that also score 80+ get a 🔥 "priority match" tag.
7. **Logs everything** — every job evaluated (good fit or not) is written back to the Google Sheet, which is what powers the duplicate check in step 4.

## The pieces it's built from

| Piece | Role |
|---|---|
| **n8n** | Orchestrates the whole thing |
| **Apify** (`codingfrontend~linkedin-jobs-scraper`) | Scrapes LinkedIn job listings |
| **Perplexity API** | Scores each job against the resume and rubric |
| **Google Sheets** | Dedup log — tracks every job already seen |
| **Discord webhook** | Delivers the alert |

## Where to tune it

Everything that controls *what* counts as a match — the resume text, target role titles, salary range, scoring rubric, and the 65-point cutoff — lives in one code step (`Build Perplexity Request`). The companion doc **"Build Notes"** covers the full rebuild, and **"Discord Connection & Filters"** covers exactly what to edit and where when the criteria need to change.
