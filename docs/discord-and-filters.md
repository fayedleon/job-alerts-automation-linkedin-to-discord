# Job Alerts Automation — Discord Connection & Filters

## How it connects to Discord

Posting to Discord is a single **HTTP POST to a Discord webhook URL** — there's no Discord app/bot install involved, just a webhook.

- **Node:** `Post to Discord` (HTTP Request) sends the payload straight to the webhook URL.
- **The message itself is built one step earlier**, in `Build Discord Embed` (a Code node), which assembles:
  - `username` — the name Discord shows as the sender ("Job Alerts")
  - `content` — the text above the embed (🔥 priority-match line, or the standard "New Job Match" line, plus the fit % and an @mention)
  - `embeds` — the rich card: author (company name + logo), title (job title, 🔥-prefixed if priority), a summary description, a colored side bar, and fields for Details / Other Notes / Key Skills / Why It Fits / Apply link / Resume Used

**To change which channel it posts to:** get a new webhook URL from that channel (Discord → channel Settings → Integrations → Webhooks → New Webhook → Copy URL) and paste it into the `url` field on the `Post to Discord` node.

**To change who gets @mentioned:** the mapping lives inside `Build Discord Embed` as `RESUME_OWNER_MAP` — a plain object of `{ "Name": "discord_user_id" }`. Update the ID (right-click a user in Discord with Developer Mode on → Copy User ID) or add more names/IDs if more than one person should be tagged.

**Priority match flag (🔥):** currently means the job was posted within the last hour *and* scored 80+. That logic is the `isPriority` line at the top of `Build Discord Embed` — change the score threshold or drop the freshness requirement there.

**Color coding:** priority = orange, score 80+ = green, score 65–79 = yellow, below 65 = red (though sub-65 jobs never reach this node, since `Is Good Fit?` already filters them out upstream).

## How to update the filters

Every filter lives in a specific node. Nothing needs to change in more than one place for a single tweak.

| What you want to change | Where to change it |
|---|---|
| **Target job titles / keywords** | `Fetch LinkedIn Jobs (Apify)` and `Fetch LinkedIn Jobs Toronto (Apify)` — the `keywords` field in each node's JSON body (currently an OR'd list of title phrases) |
| **Locations searched** | Same two nodes' `location` field (`Remote` and a specific city). To add a third location, duplicate one of these HTTP Request nodes, wire it into a third input on `Combine Job Searches` (Merge, append mode), and give it its own `maxJobs` variable in `Compute Run Window` |
| **How many jobs get pulled per run** | `Compute Run Window` — `maxJobsRemote` / `maxJobsToronto`, currently higher on the first daily run and lower afterward |
| **How far back postings can be** | `Compute Run Window` sets `datePostedFilter` (`pastWeek` on first run, `past24Hours` otherwise) which is passed to Apify; the actual go/no-go check happens in `Normalize Job Fields`'s `passesFreshnessFilter` logic (≤168 hrs on first run, ≤1 hr and same-day otherwise) |
| **Run schedule / frequency** | `Job Search Trigger` — the cron expression (currently `0 9,15,21 * * *`, 9am/3pm/9pm). Note `Compute Run Window` treats the 9am run specifically as the "wide catch-up" run — if you change the first run's hour, update that check too |
| **Resume used for scoring** | `Build Perplexity Request` — the `resumeText` constant near the top of the code |
| **Target role titles for fit-scoring** (broader than the search keywords — this is what the AI checks the resume *against*) | `Build Perplexity Request` — the `roleFamilies` constant |
| **Salary / rate target** | `Build Perplexity Request` — stated in the prompt text (currently "$60-80 USD/hour" / "$125K-$166K/year"), used in the compensation scoring section |
| **Hard exclusion rules** (hybrid/in-office by country, junior titles, work authorization, expired listings) | `Build Perplexity Request` — the "STEP 1 - HARD EXCLUSIONS" section of the prompt |
| **Scoring rubric weights** (location, comp, employment type, resume fit, seniority, industry bonus) | `Build Perplexity Request` — the "STEP 2 - SCORE" section; point values are spelled out per category |
| **Minimum score to post ("good fit" cutoff)** | `Build Perplexity Request` — currently "65 or higher," stated at the end of the prompt |
| **Discord channel** | `Post to Discord` — the webhook `url` |
| **Who gets @mentioned** | `Build Discord Embed` — `RESUME_OWNER_MAP` |
| **Priority-match (🔥) threshold** | `Build Discord Embed` — the `isPriority` line |
| **Dedup log / sheet** | The three Google Sheets nodes (`Check Seen Job`, `Save Seen Job (Fit)`, `Save Seen Job (No Fit)`) — all point to the same spreadsheet by document ID; update all three together if the sheet ever moves |

A practical note: the resume, role families, salary target, exclusions, and rubric all live in **one place** (`Build Perplexity Request`), so a full "I'm targeting different roles now" update is really one code-node edit, not a scavenger hunt across the workflow.
