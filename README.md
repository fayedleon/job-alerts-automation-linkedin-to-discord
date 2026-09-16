# Job Alerts Automation — LinkedIn/Job Board → AI Scoring → Discord

An n8n workflow that scrapes job postings, scores each one against a resume using AI, and posts the good matches straight to a Discord channel.

## What it does

1. Runs on a schedule (default: 3x/day)
2. Scrapes job postings from two locations in parallel via [Apify](https://apify.com)'s `codingfrontend~linkedin-jobs-scraper` actor
3. Filters out anything already seen or stale
4. Scores each new posting against a resume + rubric using the [Perplexity API](https://www.perplexity.ai)
5. Posts good matches (score ≥ 65) to Discord as a rich embed, with a 🔥 priority flag for very fresh, high-scoring roles
6. Logs every job it evaluates to a Google Sheet, which powers the dedup check

Full write-up: [`docs/summary.md`](docs/summary.md)

## Contents

```
job-alerts-automation/
├── README.md
├── workflows/
│   └── job-alerts-discord-template.json   ← import this into n8n
└── docs/
    ├── summary.md              ← what the automation does
    ├── build-notes.md          ← full rebuild-from-scratch walkthrough
    └── discord-and-filters.md  ← how Discord posting works + where every filter lives
```

## Quick start

1. **Import the workflow.** In n8n: **Add workflow → Import from File**, and select `workflows/job-alerts-discord-template.json`. (Or drag the file onto an empty canvas.)
2. **Read the sticky note** on the canvas titled **"🛠️ START HERE — Template Setup Checklist"** — it lists everything below in the order you'll hit it.
3. **Connect four credentials:**
   - An Apify API key (Query Auth) — for the two "Fetch Jobs" nodes
   - A Perplexity API key (Header Auth) — for "Score With AI (Perplexity)"
   - A Google Sheets OAuth2 connection — for the three Google Sheets nodes
   - A Discord webhook URL — pasted directly into "Post to Discord"
4. **Create a Google Sheet** with these columns: `jobUrl`, `jobId`, `title`, `company`, `matchScore`, `isGoodFit`, `postedToDiscord`, `seenAt`, `descriptionSummary`. Paste its ID into the three Google Sheets nodes (replacing `YOUR_GOOGLE_SHEET_ID`).
5. **Fill in your own criteria** in the `Build Scoring Request` node: resume text, target role titles, and pay range (each block is marked `// TEMPLATE:`).
6. **Fill in your Discord user ID** in `Build Discord Embed` (`MENTION_MAP`) if you want to be @mentioned.
7. **Adjust the schedule and search keywords/locations** to match your own job search — see [`docs/discord-and-filters.md`](docs/discord-and-filters.md) for exactly which node controls what.
8. **Test with a manual run** before turning the workflow on. It's imported inactive on purpose.

## How it's built

See [`docs/build-notes.md`](docs/build-notes.md) for the full node-by-node walkthrough and wiring diagram if you want to rebuild it from scratch instead of importing the JSON.

## Tuning filters & Discord behavior

See [`docs/discord-and-filters.md`](docs/discord-and-filters.md) — covers exactly which node to edit for: search keywords, locations, freshness window, run schedule, resume/role criteria, salary target, scoring rubric, minimum score to post, Discord channel, @mentions, and the priority-match threshold.

## Requirements

- An [n8n](https://n8n.io) instance (cloud or self-hosted)
- [Apify](https://apify.com) account with access to the LinkedIn jobs scraper actor
- [Perplexity API](https://www.perplexity.ai) key
- A Google account (for the Sheets dedup log)
- A Discord server you can add a webhook to

## License / use

This is a personal automation shared as a starting point — adapt it freely.
