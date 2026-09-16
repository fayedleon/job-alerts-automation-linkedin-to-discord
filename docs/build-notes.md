# Job Alerts Automation — Build Notes (Rebuild From Scratch)

Covers everything needed to rebuild the **"Job Search - LinkedIn to Discord"** n8n workflow from an empty canvas: accounts/credentials needed, every node in order, what each one does, and how they connect.

## 1. What you'll need before you start

| Service | Why | Get it from |
|---|---|---|
| **n8n instance** | Hosts the workflow | — |
| **Apify account** | Runs the LinkedIn job scraper actor | apify.com → API token, plus access to the `codingfrontend~linkedin-jobs-scraper` actor |
| **Perplexity API key** | Scores each job against the resume | perplexity.ai → API settings |
| **Google account** | Spreadsheet used as the dedup/seen-jobs log | Google Sheets OAuth2 credential in n8n |
| **Discord webhook URL** | Where alerts get posted | Discord server → channel Settings → Integrations → Webhooks → New Webhook → Copy URL |
| **A resume, in plain text** | Fed into every AI scoring call | — |
| **A Discord user ID** (optional) | So the alert can @mention the right person | Discord → enable Developer Mode → right-click your name → Copy User ID |

n8n credentials to set up first:
- **HTTP Query Auth** — Apify API token, used as a query parameter on the Apify HTTP calls
- **HTTP Header Auth** — Perplexity API key, sent as a bearer header
- **Google Sheets OAuth2** — connected Google account with edit access to the tracking sheet

## 2. The Google Sheet (build this first)

Create a spreadsheet with one sheet (`Sheet1`) and these columns, all plain text/string:

`jobUrl` · `jobId` · `title` · `company` · `matchScore` · `isGoodFit` · `postedToDiscord` · `seenAt` · `descriptionSummary`

`jobUrl` is the lookup/matching key — every read and write in the workflow keys off it. Grab the sheet's document ID from its URL for later steps.

## 3. Node-by-node build

### Trigger
**`Job Search Trigger`** — Schedule Trigger, cron expression `0 9,15,21 * * *`, timezone set to your local zone. Fires at 9am, 3pm, 9pm.

### Compute the run window
**`Compute Run Window`** — Code node (run once for all items). Reads the current hour off `$now`. If it's the 9am run, treat it as the "first run of the day": pull a wider freshness window (`pastWeek`) and a larger job cap (30 remote / 20 second-location). Otherwise use `past24Hours` and a smaller cap (15 / 10). Outputs `isFirstRun`, `datePostedFilter`, `maxJobsRemote`, `maxJobsToronto`, `todayDateStr`.

### Fetch jobs (two parallel branches)
**`Fetch LinkedIn Jobs (Apify)`** and **`Fetch LinkedIn Jobs Toronto (Apify)`** — both HTTP Request nodes, both connect from `Compute Run Window` directly (they run in parallel, not chained to each other). Each is a `POST` to:
```
https://api.apify.com/v2/acts/codingfrontend~linkedin-jobs-scraper/run-sync-get-dataset-items
```
Auth: the HTTP Query Auth credential. JSON body: `keywords` (the target job titles, OR'd together), `location` (one node uses `Remote`, the other uses a specific city), `maxJobs` (from the run-window output), `datePosted` (also from run-window). Set `onError: continueRegularOutput` on both so one scraper failing doesn't kill the run.

### Combine and normalize
**`Combine Job Searches`** — Merge node, mode `append`, with the two fetch nodes wired into its two inputs (index 0 and 1).

**`Normalize Job Fields`** — Code node (run once per item). Maps each scraper's inconsistent field names (`title`/`jobTitle`/`position`, etc.) into a consistent shape, parses the scraper's relative "posted X hours ago" text into an actual age in hours, and computes `passesFreshnessFilter`: true if this is the first run and the job is ≤168 hours old, or if it's a later run and the job is ≤1 hour old *and* posted today.

### Filter and loop
**`Passes Filters?`** — IF node. True when `jobUrl` is not empty AND `passesFreshnessFilter` is true. False branch goes to `All Jobs Processed` (a No-Op end node); true branch goes to the loop.

**`Loop Jobs`** — Split In Batches node (default batch size). Its "done" output also feeds `All Jobs Processed`; its "each batch" output feeds the dedup check.

### Dedup check
**`Check Seen Job`** — Google Sheets node, `read` operation, filtered on `jobUrl` equals the current job's URL, return first match only, `alwaysOutputData` on (so a no-match still produces an item to branch on).

**`Already Seen?`** — IF node. If the read found a row (jobUrl came back non-empty), the job was already logged — loop back to `Loop Jobs` and skip it. If nothing was found, it's new — continue to scoring.

### AI scoring
**`Build Perplexity Request`** — Code node. Builds the full scoring prompt: the resume text (hardcoded here), the list of target role families, and the scoring rubric (location/work arrangement, compensation vs. target hourly rate, employment type, resume/role fit, seniority, industry bonus), plus hard-exclusion rules (wrong-country hybrid/in-office, junior titles, missing work authorization, expired listings). Asks for strict JSON back.

**`Score With Perplexity`** — HTTP Request, `POST` to `https://api.perplexity.ai/chat/completions`, HTTP Header Auth credential, JSON body is the request built in the previous step (model `sonar`).

**`Parse Perplexity Score`** — Code node. Strips markdown code fences if present, parses the JSON, falls back to a safe "not a fit" object if parsing fails, and merges the result back with the original job fields (title, company, URL, etc. pulled from `Normalize Job Fields`).

**`Is Good Fit?`** — IF node on `isGoodFit === true`. False branch → `Save Seen Job (No Fit)`. True branch → `Build Discord Embed`.

### Posting to Discord
**`Build Discord Embed`** — Code node. Builds the full Discord embed JSON: author (company + logo), title (with a 🔥 prefix if it's a priority match — posted within the last hour AND scoring 80+), color-coded by score band, fields for posted date/match score/remote status/location/pay, other notes, key skills, the AI's fit reasoning, an apply link, and a resume-used link. Also builds the @mention using a name→Discord-ID lookup map and sets the message content/username.

**`Post to Discord`** — HTTP Request, `POST` straight to the Discord webhook URL, JSON body is the payload built in the previous step.

### Logging (both paths converge back into the loop)
**`Save Seen Job (Fit)`** — Google Sheets `appendOrUpdate`, matching on `jobUrl`, writes all the job/score fields plus `postedToDiscord: true` and a timestamp. Feeds back into `Loop Jobs` to continue.

**`Save Seen Job (No Fit)`** — same operation, `postedToDiscord: false`, also feeds back into `Loop Jobs`.

### End
**`All Jobs Processed`** — No-Op node. Nothing downstream; just marks the run complete.

## 4. Wiring summary

```
Job Search Trigger
  → Compute Run Window
      → Fetch LinkedIn Jobs (Apify) ─┐
      → Fetch LinkedIn Jobs Toronto ─┴→ Combine Job Searches
                                          → Normalize Job Fields
                                             → Passes Filters?
                                                 ├─ false → All Jobs Processed
                                                 └─ true  → Loop Jobs
                                                              ├─ done → All Jobs Processed
                                                              └─ each → Check Seen Job
                                                                          → Already Seen?
                                                                              ├─ true  → (back to) Loop Jobs
                                                                              └─ false → Build Perplexity Request
                                                                                           → Score With Perplexity
                                                                                              → Parse Perplexity Score
                                                                                                 → Is Good Fit?
                                                                                                     ├─ false → Save Seen Job (No Fit) → Loop Jobs
                                                                                                     └─ true  → Build Discord Embed
                                                                                                                  → Post to Discord
                                                                                                                     → Save Seen Job (Fit) → Loop Jobs
```

## 5. After building

- Test each Apify node individually first (small `maxJobs` value) to confirm the actor is returning data in the field shape the code expects.
- Run the whole thing once manually with the sheet empty to confirm dedup writes are landing correctly before turning on the schedule.
- Activate the workflow once a full manual run posts correctly to Discord.

See **"Discord Connection & Filters"** for exactly what to edit when the job criteria, resume, or Discord channel need to change later.
