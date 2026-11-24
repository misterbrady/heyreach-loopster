# Rebuild Lane Scout as a Loopster in-sheet agent

Paste this file into your coding agent (Claude, Cursor, Codex) and tell it: **build this in my Loopster workspace, autoplay off, spend nothing until I inspect.**

This recreates a daily job-post scout that used to live in a Grok Bot writing to a Google Sheet. The Loopster version finds live GTM Engineer postings, fans each posting into its own row, qualifies it, finds the hiring manager, pulls a work email, and writes a two-sentence give-first opener from the actual JD.

Swap the role, the pick rule, and the search queries for your niche. Leave the two-table shape and the traps alone. Those are what burned the first build.

---

## What you are copying

**Before (Grok Bot "Lane Scout"):** a weekday 07:00 job that Googled ATS boards, dumped every matching posting into a Google Sheet, then a human or a later agent qualified them.

**After (this Loopster workbook):** two tables in one workbook. Table 1 searches. Table 2 is the live pipeline. A `send-to-table` column explodes each search's `jobs` JSON into one pipeline row per posting, with dedupe on `posting_url`.

Verified on 24 Nov 2026 against a live Ashby query:

- Discover found 5 postings for **$0.05**.
- Dedupe on `posting_url` skipped repeats on a second run (`rowsSent: 1`, `skippedAsDuplicate: 4`).
- Full enrich on 2 qualified rows (Baseten, Forgepoint) landed a hiring LinkedIn, a verified work email on the row whose domain was right, and a JD-specific opener.
- End-to-end test spend was about **$0.45**. Leave autoplay **off** until the 1-row test looks right.

---

## Connect Loopster

| | |
|---|---|
| REST | `https://app.loopster.ai/api/v1` |
| OpenAPI | `https://app.loopster.ai/openapi.json` |
| MCP | `https://app.loopster.ai/api/mcp` |
| Docs | `https://docs.loopster.ai` |
| Auth | `Authorization: Bearer <LOOPSTER_API_KEY>` |

The key needs **editor** plus **canSpend: true**. Confirm with `GET /v1/key` and `GET /v1/wallet` before you create anything.

App URLs are team-scoped (`/app/{team-slug}/tables/{id}`). Bare `/tables/{id}` 404s. If you need a link the human can click from chat, mint a read-only share: `POST /v1/tables/{tableId}/shares` with `expiresInSeconds`. Do not paste logged-in app URLs and expect them to open.

**API habits that matter:**

- Create **enrichment** columns one at a time (manual columns can go in one call). `POST /v1/tables/{id}/columns` with a `templateId` from `GET /v1/tables/{id}/columns/templates`.
- `PATCH` on a column **merges** `aiConfig` / `taskConfig`. Arrays replace wholesale.
- `POST /v1/tables/{id}/trigger` with **no `rowIds` runs every row**. Always send `rowIds`. Always send `maxCostUsd`. Price first with `?dryRun=true`.
- Set **autoplay: false** on both tables until the test pass is done.

---

## Architecture (do not flatten this into one table)

```
Table 1  Search Queries          Table 2  Live Job Pipeline
-------------------------        --------------------------------
1 row = 1 saved Google query     1 row = 1 job posting
Discover jobs (Loopgent $0.05)   Scrape listing  ->  Resolve job URL
  -> JSON { "jobs": [ ... ] }    Scrape JD
Send to Live Pipeline (FREE)     Qualify  ($120K+ base, US/UK/Canada)
  send-row-for-each-item         if pass:
  on json path `jobs`              Resolve domain -> Leaders -> Hiring contact
                                   Still there? -> paid profile fallback
                                   Right person? (current company check)
                                   if match: Work email, Give-first line
```

Why two tables: Loopgent (`loopster-default` + `serpSearch` + `retrieveUrl`) is a **fixed $0.05/row**. If Discover also has to open every ATS listing and extract individual job URLs, it blows the budget and returns `{"jobs": []}`. Search returns company boards. Table 2 resolves the real posting with a $0.001 mini model plus a $0.0002 scrape. That split is the whole design.

---

## Create the workbook

1. `POST /workbooks` name: `Lane Scout (In-Sheet Agent)`.
2. Create **Table 2 first** (the destination has to exist before send-to-table can point at it).
3. Create Table 1 in the same workbook.
4. Both tables: `autoplay: false`. Table 2: `keepExistingResults: true`. Table 2 auto-dedupe on `posting_url`, keep oldest.

---

## Table 2: Live Job Pipeline

Create these **manual** columns, headers exactly as written so auto-map from Table 1 works:

| Header | Type / dataShape |
|---|---|
| `company` | manual text (this is the table `input` column; rename the input header to `company`) |
| `domain` | manual text |
| `job_title` | manual text |
| `location` | manual text |
| `remote` | manual text |
| `salary` | manual text |
| `posting_url` | manual url |
| `ats` | manual text |
| `hiring_team_title` | manual text |
| `first_posted` | manual text |
| `source_query` | manual text |

Then add the enrichment columns **in this order**. Template ids are from Loopster's Add column menu.

### 1. Scrape posting
- Template: `web-scraper`
- URL: `@{posting_url}` (the column id of `posting_url`)
- format: markdown, timeout 15s, no images
- Cost: ~$0.0002/row
- This often scrapes an ATS **company board**, not the JD. That is expected.

### 2. Resolve job URL
- Template: `ai-loopster-default-mini` (~$0.001)
- Prompt:

```
Extract the individual job posting URL for this role.

Company: @{company}
Job title: @{job_title}
Listing or posting URL: @{posting_url}
Scraped page: @{scrape_posting}

Rules:
1. If the given URL already is an individual job page (Ashby URL with a UUID after the company, Greenhouse /jobs/{id}, Lever URL with a job slug after the company, Workable URL with a job slug), return it unchanged.
2. Otherwise find, in the scraped page, the markdown or HTML link whose visible title matches this job title. Return that full https URL.
3. Do not return a company listing URL (jobs.ashbyhq.com/{company} with no UUID, boards.greenhouse.io/{company} or job-boards.greenhouse.io/{company} with no /jobs/{id}, jobs.lever.co/{company} with no slug).
4. Return only the URL, or Not found.
```

### 3. Scrape JD
- Template: `web-scraper`
- URL: the Resolve job URL column
- Same scraper settings as step 1
- Give-first and hiring-contact read **this**, never the listing scrape

### 4. Qualify
- Template: `ai-loopster-default-mini` (~$0.001)
- **Keep this at two checks.** Adding "duties must mention outbound" failed real GTM Engineer posts when the scrape was still a listing page.
- Prompt (swap the pay floor and geos for your niche):

```
You qualify a GTM engineer job post for a recruiting agency. Both parts must hold for a pass.
1. Pay: the top of the base pay band is $120K USD or more. Use base pay and leave out bonus and equity, unless the post only gives OTE. Convert other currencies to USD first. No pay listed is a fail.
2. Location: the job is in the United States, the United Kingdom or Canada. Remote counts when it is open to one of those.

Company: @{company}
Job title: @{job_title}
Location: @{location}
Salary: @{salary}
Scraped job post: @{Scrape JD}

If the salary field is blank, read the pay band from the job post.

Return only JSON: {"pass": true or false, "pay_top_usd": number or null, "reason": "one short sentence naming the part that passed or failed"}
```

`optionalMentions`: location, salary. Title filtering already happened in Discover. Qualify is pay + geo. A 404 on Scrape posting already stops the row.

### 4b. Resolve domain
Discover guesses `domain` from search snippets and gets it wrong (Forgepoint the AI company came through as `forgepointcap.com`, the VC, so the email waterfall returned `no_match` on a real CEO). This mini column reads the real domain from the JD, and both Leaders at company and Work email use it.
- Template: `ai-loopster-default-mini` (~$0.001)
- Run only when `Qualify.pass == true`
- `optionalMentions`: domain
- Prompt:

```
Company: @{company}
Domain guessed from search: @{domain}
Scraped job post: @{Scrape JD}

Return the hiring company's own website domain, such as acme.com, read from the job post. Use the guessed domain only when the post gives nothing better. Never return an ATS domain (ashbyhq.com, greenhouse.io, lever.co, workable.com) or an investor's domain. Return only the domain, or Not found.
```

### 5. Leaders at company
- Template: `ai-ark-find-people-at-company`
- Domain: `@{Resolve domain}`
- Seniorities: founder, c_suite, vp, head, director
- `max_results`: **8** (15 dumps enough JSON that the next Loopgent hits its $0.05 ceiling and returns "model's reply wasn't a usable answer")
- Run only when `Qualify.pass == true`
- Cost: charged on result, roughly $0.01

### 6. Hiring contact
- Template: `ai-web` (Loopgent, $0.05)
- Run only when `Qualify.pass == true`
- `optionalMentions`: hiring_team_title, Leaders at company, Resolve domain
- **It can invent people.** In the 24 Nov test it skipped 8 real Baseten leaders and returned a LinkedIn URL for a stranger at another company. The "never build a URL from a name" line and the Right person? check (7b) are both there because of that.
- **Do not paste the listing scrape into this prompt.** URL + leaders list is enough. A 29KB board scrape plus 15 people is what blew the first Baseten run.
- Prompt:

```
Find the person most likely to be the hiring manager for this job, and return their LinkedIn profile URL.

Company: @{company}
Company domain: @{resolve_domain}
Job title: @{job_title}
Hiring team title on the post: @{hiring_team_title}
Job post URL: @{resolve_job_url}

Senior people at the company, from a people database:
@{leaders_at_company}

A GTM engineer usually reports to a founder or CEO at a small company (~50 people or fewer), or to the head of RevOps, Growth, Sales or Marketing Ops at a larger one. Use the hiring team title when it is given.

Pick from the senior people list first, and copy their linkedin_url exactly as the list gives it. Search the web only when nobody on the list fits, and then only return a profile you actually opened that shows this person works at @{company} today. Never build a LinkedIn URL from a name.

Your entire reply must be one LinkedIn profile URL (https://www.linkedin.com/in/...) and nothing else. Return Not found only when there is no plausible person at all.
```

### 7. Still there?
- Template: `loopster-data-enrich-person` (FREE)
- LinkedIn URL: Hiring contact column
- Run only when `Qualify.pass == true`

### 7b. Right person?
- Template: `ai-loopster-default-mini` (~$0.001), `dataShape: json`
- Run when `Qualify.pass == true` AND (`Still there?.found == true` OR `Profile.success == true`)
- `optionalMentions`: Still there?, Profile (paid fallback), Resolve domain
- Build it after Profile (paid fallback), since it reads both profiles.
- Prompt:

```
Check that this profile is someone who works at the hiring company today.

Hiring company: @{company}
Company domain: @{resolve_domain}
Free profile check: @{still_there}
Paid profile: @{profile_paid_fallback}

Use whichever profile has data. Match when the profile's current company name or website is this company (allow small spelling differences, such as Inc or a .ai domain). A different company, a past job only, or no current company is not a match.

Return only JSON: {"match": true or false, "current_company": "the company the profile shows today", "reason": "one short sentence"}
```

Work email and Give-first line both gate on `Right person?.match == true`. Without it, a wrong contact still gets an email address built from their name at the hiring company's domain, and that address looks real.

### 8. Profile (paid fallback)
- Template: `enrich-professional-profile` (~$0.003)
- Profile: Hiring contact column
- Run when `Qualify.pass == true` AND `Still there?.found != true`

### 9. Work email
- Template: `waterfall-email`
- Validation: Balanced, `onAllRejected: use_best_rejected`, Findymail verify
- Providers in order: Loopster work email, Findymail (name), Icypeas
- Map `linkedin_url` from Hiring contact, `domain` from Resolve domain, name fields from Still there?
- On the paid fallback path, Still there? has no name, so the waterfall gets only `linkedin_url` and `domain`. Findymail by name needs a name and will usually miss on that path. Check the first row that takes the paid path, and if emails miss there, add a free `merge-columns` name column that takes Still there?'s name first and the paid profile's second.
- **Run when Qualify passed AND (Still there found the person OR paid profile `success == true`) AND `Right person?.match == true`.** If you only gate on the free check, a paid-fallback hit never gets an email. Live test: free enrich returned `found: false` on a co-founder who then resolved on the paid profile, and the waterfall only ran after this condition was widened.

### 10. Give-first line
- Template: `ai-web` ($0.05)
- Run only when `Qualify.pass == true` AND `Right person?.match == true`
- **Why the letter rule:** asked only to "vary the opening", the model opened two of three lines with "Saw you're". The model can't see the other rows, so it can't vary across a batch by itself. Keying the shape to the company name makes the variety come from the data.
- Read Scrape JD, not the listing. Otherwise the opener quotes a random other job on the same board.
- Prompt (swap the recruiter one-liner for your offer):

```
Read the job post for the @{job_title} role at @{company} (URL: @{resolve_job_url}).
Scraped JD: @{scrape_jd}

Write the opening of a cold email from a recruiter who places GTM engineers, in two short sentences.

Sentence 1: one concrete detail only this post has. A named tool, system, number, team or outcome, in the post's own words. Never a generic phrase like 'build GTM systems' or 'own outbound'.

Shape sentence 1 by the first letter of the company name, so a batch of emails doesn't all open the same way:
A to E: make the detail itself the subject of the sentence, then react to it in a few words.
F to J: ask a short, specific question about that detail.
K to O: say what the person in this seat will be stuck with or judged on, using the detail.
P to T: open with the number, tool or team name, then one plain clause about it.
U to Z, or a digit: say what usually goes wrong with that kind of work, tied to the detail.

Never open with Saw, Noticed, I see, Looks like, Came across, Spotted, Love, Just, Hope, Hey or "You're looking for". Don't start with "I" or "Your".

Sentence 2: a short question offering three profiles of people who have done that specific thing. Vary how you ask. Never use the words 'exactly', 'perfect fit', 'game changer' or 'I came across'.

Under 35 words in total. Write like a person typing fast on a phone: contractions, plain words. No dashes of any kind, no semicolons, no exclamation marks, no flattery. Return only the two sentences.
```

---

## Table 1: Search Queries

Manual columns:

| Header | What it holds |
|---|---|
| `Source name` | rename the table input. One saved search per row. |
| `Search query` | the exact Google query |
| `ATS or source` | Ashby, Greenhouse, Lever, Workable, HN, Newsletter |
| `Notes` | extra guidance for Discover |

### Discover jobs
- Template: `ai-web` (Loopgent, **$0.05/row every run**)
- Prompt:

```
You are Lane Scout, an automated job-post discovery agent finding companies currently hiring GTM Engineers.

Run this exact search:
Source: @{source_name}
Query: @{search_query}
ATS / source type: @{ats_or_source}
Extra guidance: @{notes}

Rules:
1. Find live, currently open job postings matching this query (prioritize roles posted or updated recently).
2. Keep a posting ONLY if the title is GTM Engineer, GTM Engineering, Go-to-Market Engineer, or Growth Engineer with outbound or enrichment in the duties. Drop RevOps analyst, SDR, sales engineer, and solutions engineer roles.
3. Prefer the individual job page URL when search results include one. A company ATS listing URL is acceptable as a starting point; Table 2 will resolve it to the specific job. Leave fields as empty strings "" when unknown. Never guess.
4. Open each candidate posting URL when needed to extract the company domain, location, remote status, salary band, hiring team title, and first posted date.
5. Return ONLY valid JSON with this exact schema (up to 15 postings per run):
{"jobs": [{"company": "", "domain": "", "job_title": "", "location": "", "remote": "", "salary": "", "posting_url": "", "ats": "", "hiring_team_title": "", "first_posted": "", "source_query": "@{source_name}"}]}
```

Do **not** tell Discover "never return a listing URL." It will spend the $0.05 trying, fail, and return `{"jobs": []}`.

### Send to Live Pipeline
- Template: `send-to-table` (FREE)
- Destination: Table 2
- Method: `send-row-for-each-item`
- List source: Discover jobs, JSON path `jobs`
- `autoMapExistingColumns: true`
- `skipDuplicates: true`
- Unique key: Table 2 `posting_url`
- `updateExistingRows: false`

JSON keys in each job object must match Table 2 manual headers (`company`, `domain`, `job_title`, ...).

---

## Seed rows (Table 1)

Load these 7 rows. Swap `"GTM Engineer"` / `"GTM Engineering"` / `"Go-to-Market Engineer"` for your title.

| Source name | Search query | ATS or source | Notes |
|---|---|---|---|
| Ashby - GTM Engineer | `site:jobs.ashbyhq.com ("GTM Engineer" OR "GTM Engineering")` | Ashby | US/UK/Canada roles preferred; extract exact base pay band if shown on Ashby page. |
| Greenhouse - GTM Engineer | `(site:boards.greenhouse.io OR site:job-boards.greenhouse.io) ("GTM Engineer" OR "GTM Engineering")` | Greenhouse | Check Greenhouse posting body for compensation range and hiring team title. |
| Lever - GTM Engineer | `site:jobs.lever.co ("GTM Engineer" OR "GTM Engineering")` | Lever | Capture exact Lever URL and location/remote details. |
| Workable - GTM Engineer | `site:apply.workable.com ("GTM Engineer" OR "GTM Engineering")` | Workable | Drop pure SDR, RevOps analyst, or solutions engineer roles. |
| ATS - Go-to-Market Engineer | `(site:jobs.ashbyhq.com OR site:boards.greenhouse.io OR site:jobs.lever.co) "Go-to-Market Engineer"` | Ashby/Greenhouse/Lever | Catches hyphenated Go-to-Market Engineer titles missed by GTM Engineer queries. |
| Hacker News - Who is Hiring | `site:news.ycombinator.com "Ask HN: Who is hiring?" ("GTM Engineer" OR "Go-to-Market Engineer")` | Hacker News | Read the latest monthly Who is hiring thread for GTM Engineer or Growth Engineer (outbound/enrichment) posts. |
| The Shortlist - GTM Engineer Search | `"gtmengineersearch.com" OR "The Shortlist" "GTM Engineer" jobs latest` | Newsletter | Read the latest public edition of The Shortlist by GTM Engineer Search and pull live posting URLs. |

---

## Test plan (do this before any schedule)

Autoplay stays **false** on both tables.

1. **Table 1, one row** (Ashby or your cheapest ATS query). Trigger Discover jobs + Send to Live Pipeline. Cost ~**$0.05**.
2. Open Table 2. You should see N new rows, one per job, mapped into the 11 manuals. `skippedAsDuplicate` should be 0 on the first run.
3. **Table 2, 2 or 3 rows.** Pick one likely pass (pay listed, US/UK/CA), one likely fail (no pay, or a 404). Trigger the enrichment columns. Dry-run first. Cap `maxCostUsd` at **0.80**.
4. Check:
   - Resolve job URL is a **specific** posting, not `jobs.ashbyhq.com/{company}`.
   - Scrape JD title matches the role.
   - Qualify JSON has `pass` true/false with a one-line reason.
   - Failures skip Leaders / Hiring contact / email / give-first.
   - Passes produce a LinkedIn URL, then an email when domain is right, then a two-sentence opener that quotes the JD.

Cost to expect on a passing row after discovery: roughly **$0.10 to $0.15** (two Loopgents at $0.05, plus mini + scrape + Ark + waterfall). Failures should die at Qualify for about **$0.002**.

---

## Then, and only then, turn it on

- Table 2 **Auto-run ON** so new rows from send-to-table enrich themselves.
- Table 1 schedule: Discover jobs + Send to Live Pipeline, **07:00 UTC**, weekdays. Loopster weekly cadence is one `dayOfWeek` (0=Sunday) per schedule, so either five weekly schedules (Mon-Fri) or one daily and live with weekend spend. Grok Bot was weekdays.
- Optional last columns on Table 2: `slack-send-message`, or `heyreach-add-leads-to-campaign`. Gate them on Qualify.pass. Do not auto-send email from this table. The give-first line is copy for a human or a later sequence.

A schedule spends from the wallet with nobody watching. A key with spending turned off cannot create one.

---

## Traps (bake these in, do not rediscover them)

1. **ATS search returns the company board, not the job.** `https://jobs.ashbyhq.com/baseten` is 108 open roles. Scrape that, and Give-first will quote a random posting. Resolve job URL + second scrape is mandatory.
2. **Loopgent is $0.05 with no retry.** If Hiring contact's prompt includes the board scrape plus a 15-person JSON dump, you get charged and an empty cell. Cap leaders at 8. Output "one LinkedIn URL, nothing else."
3. **Qualify is two lines.** Pay floor + geos. Title hygiene lives in Discover. Extra duty-matching on a listing scrape false-fails good jobs.
4. **Work email must accept either profile path.** Free enrich `found: false` plus paid enrich `success: true` is a common real case.
5. **Domain from search is a guess.** Wrong domain = waterfall `no_match` on a real person. Resolve domain from the JD if email miss rate is high.
6. **Never trigger a table without `rowIds`.** Empty body = every row, full Loopgent price.
7. **Forcing Discover to refuse listing URLs returns zero jobs.** Let Table 2 resolve them.
8. **Dedupe key is `posting_url`.** `jobs.ashbyhq.com/growtherapy` and `jobs.ashbyhq.com/grow-therapy` are two rows. That is fine; a 404 dies at scrape.

---

## Swap for your niche

Change these. Leave the rest. `build-for-any-company.md` lists every GTM-specific line to rewrite.

| Knob | This build | Your build |
|---|---|---|
| Role titles in Discover and seed queries | GTM Engineer / GTM Engineering / Go-to-Market Engineer / Growth Engineer | your titles |
| Drop list in Discover | RevOps analyst, SDR, sales engineer, solutions engineer | your noise titles |
| Qualify pay + geo | $120K+ base, US/UK/Canada | your floor and markets |
| Give-first sentence 2 | three profiles of people who have done that specific thing | whatever you actually send (asset, teardown, intro) |

Hiring-contact reporting-line hint is GTM-specific (founder / head of RevOps / Growth). Rewrite that paragraph for your role.

---

## Cost cheat sheet

| Step | Template | Typical |
|---|---|---|
| Discover jobs (per saved query, per run) | `ai-web` | $0.05 |
| Send to Live Pipeline | `send-to-table` | free |
| Scrape posting / Scrape JD | `web-scraper` | ~$0.0002 each |
| Resolve job URL / Qualify | `ai-loopster-default-mini` | $0.001 each |
| Leaders at company | `ai-ark-find-people-at-company` | ~$0.01, pass-only |
| Hiring contact / Give-first | `ai-web` | $0.05 each, pass-only |
| Still there? | `loopster-data-enrich-person` | free |
| Profile fallback | `enrich-professional-profile` | ~$0.003, only if free miss |
| Work email | `waterfall-email` Balanced | a few cents, pass-only |

Daily ballpark if you run all 7 seed queries and a handful of new posts qualify: **about $0.35 to $0.80**, dominated by Loopgent.

---

## Agent checklist

- [ ] Wallet checked. Autoplay false on both tables.
- [ ] Workbook + Table 2 manuals + 12 enrichments (Resolve domain is 4b, Right person? is 7b), then Table 1.
- [ ] Send-to-table points at Table 2, `jobs` path, skipDuplicates on `posting_url`.
- [ ] 7 seed rows loaded.
- [ ] 1-row Discover test ~$0.05, rows landed in Table 2, zero dupes.
- [ ] 2-3 Table 2 rows: Resolve URL is specific, Qualify gates the paid columns, give-first quotes the JD.
- [ ] Human inspects. Then Table 2 auto-run, then Table 1 weekday 07:00 UTC schedule.
- [ ] No email auto-send from this workbook.

---

## API shapes (read this before the first call)

Everything above says *what* to build. This section says *how the API wants it*, so a fresh session builds it first time without opening another table to copy from.

**Auth and base.** `https://app.loopster.ai/api/v1`. Every path in this file is relative to that, so `/workbooks` and `/v1/workbooks` both mean `https://app.loopster.ai/api/v1/workbooks`. Send header `Authorization: Bearer <key>`. Read the key from the `LOOPSTER_API_KEY` environment variable and never print it. App links are `https://app.loopster.ai/app/<team>/tables/<tableId>`.

**Build order.**
1. `POST /workbooks` with `{"name": ...}`, then `POST /tables` with `{"title": ..., "autoplay": false}` for each table, then move each into the workbook with `POST /workbooks/{workbookId}/tables` `{"tableId": ...}`.
2. Every new table starts with one manual column whose id is `input`. Rename it with `PATCH /tables/{t}/columns/input` `{"header": "company"}` and add the other manuals in one call: `POST /tables/{t}/columns` `{"columns": [{"templateId": "manual", "header": "domain"}, ...]}`. posting_url also takes `"dataShape": "url"`.
3. `GET /tables/{t}/columns` and keep a map of header to column id. Every later step needs ids.
4. Add enrichments one at a time, in order, re-reading the map as you go, because each one references the ids of the ones before it.
5. `PATCH /tables/{t}` with `{"autoplay": false, "keepExistingResults": true}` before any rows go in. Dedupe on Table 2 is `{"autoDedupe": {"enabled": true, "columnId": "<posting_url id>", "keep": "oldest"}}`.
6. Rows: `POST /tables/{t}/rows` with the body as a bare JSON array `[{...}, {...}]` keyed by header. `{"rows": [...]}` is refused.

**Column references.** Inside a prompt or a task field, a column is `@{col_abc123}`, the id. A header name in braces (`@{posting_url}`) is refused. The `@{posting_url}` style used earlier in this doc is shorthand for "the id of that column".

**A field inside a JSON column** (for example `pass` in Qualify's output) is a ref token:

```
@{ref:v1:<base64url of {"columnId":"col_abc123","source":"value","path":["pass"]} with no = padding>}
```

Build the JSON with no spaces (`separators=(',',':')` in Python), base64url encode it, strip the `=`.

**Gating ("only run when Qualify passed").** Add `runCondition` to the column body:

```json
"runCondition": {"condition": "<ref to Qualify.pass> == true", "description": "Only for posts that pass Qualify"}
```

Combine with `&&`, `||` and brackets. Profile fallback is `<pass> == true && <ref to Still there?.found> != true`. Work email is `<pass> == true && (<found> == true || <ref to Profile.success> == true)`.

**Blank inputs block a row by default.** Any column a prompt mentions that can legitimately be empty (salary, location, notes, hiring_team_title, domain, Leaders at company) goes in `"optionalMentions": ["col_...", ...]`. Without it, the cell sits waiting forever.

**AI column (mini or ai-web).**

```json
{"templateId": "ai-web", "header": "Hiring contact",
 "runCondition": {...},
 "optionalMentions": ["col_domain", "col_hiring_team_title", "col_leaders"],
 "aiConfig": {"model": "loopster-default", "tools": ["serpSearch", "retrieveUrl"], "prompt": "... @{col_company} ..."}}
```

Mini columns use `"templateId": "ai-loopster-default-mini"` and `"model": "loopster-default-mini"` with no tools. Add `"dataShape": "json"` when the prompt returns JSON (Qualify, Discover jobs).

**Task columns** use `taskConfig: {"taskType": X, X: {...}}`:

| Column | templateId | taskConfig inner body |
|---|---|---|
| Scrape posting / Scrape JD | `web-scraper` | `jinaScraper`: `{"url": "@{col}", "format": "markdown", "timeoutSeconds": 15, "scrapeImages": false, "scrapeLinks": true}` |
| Leaders at company | `ai-ark-find-people-at-company` | `aiArkFindPeopleAtCompany`: `{"domain": "@{col}", "seniorities": ["founder","c_suite","vp","head","director"], "departments": [], "max_results": 8}` |
| Still there? | `loopster-data-enrich-person` | `loopsterDataEnrichPerson`: `{"linkedin_url": "@{col}"}` |
| Profile (paid fallback) | `enrich-professional-profile` | `enrichProfessionalProfile`: `{"profile": "@{col}", "execution_mode": "managed"}` |

**Work email waterfall.** `templateId: "waterfall-email"`, `taskType: "waterfallEmail"`:

```json
{"providers": [
   {"id": "loopsterDataFindWorkEmail-default", "taskType": "loopsterDataFindWorkEmail", "enabled": true, "order": 0},
   {"id": "findymailFindByName-default", "taskType": "findymailFindByName", "enabled": true, "order": 1, "executionMode": "managed"},
   {"id": "icypeasFindEmail-default", "taskType": "icypeasFindEmail", "enabled": true, "order": 2, "executionMode": "managed"}],
 "validation": {"enabled": true, "strategy": "balanced", "onAllRejected": "use_best_rejected", "validatorTaskType": "findymailVerifyEmail", "maxCandidatesPerStep": 3},
 "inputMapping": {"linkedin_url": "@{Hiring contact id}", "domain": "@{Resolve domain id}",
   "first_name": "<ref Still there?.first_name>", "last_name": "<ref Still there?.last_name>", "full_name": "<ref Still there?.full_name>"}}
```

**Send to Live Pipeline.** `templateId: "send-to-table"`, `taskType: "sendToTable"`. It needs the destination workbook id as well as the table id:

```json
{"destinationProjectId": "<Table 2 id>", "destinationProjectName": "<Table 2 title>", "destinationWorkbookId": "<workbook id>",
 "method": "send-row-for-each-item", "listSourceColumnId": "<Discover jobs id>", "listSource": "value", "listJsonPath": "jobs",
 "columnMapping": [], "additionalData": [], "autoMapExistingColumns": true, "autoExtractNewColumns": false,
 "skipDuplicates": true, "uniqueKeyColumnId": "<Table 2 posting_url id>", "updateExistingRows": false}
```

**Check the response.** A column create returns `warnings` and `appliedFixes`. Read both. Loopster sometimes switches on extras for you (it turned on Findymail verification and a managed account in the first build).

**Price before you run.** `POST /tables/{t}/trigger?dryRun=true` with `{"rowIds": [...], "columnIds": [...], "maxCostUsd": 0.10}` returns `expectedCostUsd` and spends nothing. Drop `dryRun` to run for real, and always keep `rowIds` and `maxCostUsd`. `GET /wallet` shows the balance.
