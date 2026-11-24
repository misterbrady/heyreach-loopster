# The always-on agent, as Loopster tables

My version runs on my laptop every 15 minutes (slide 43 in the talk). This is the same agent rebuilt in Loopster, so it runs without a laptop. **Built and tested on 24 Nov 2026, apart from the link-view column.** The drafts, the rules checks and both Slack alerts ran for real, for $0.004 in total. A reply sent to the webhook landed as a row, marked the person as replied in the Outreach log and stopped their 36 hour timer, all for $0. Build it with autoplay off and test each table on one row.

What it does, same as the laptop version:

1. Watches for replies and stops chasing anyone who answered.
2. Watches your tracked links and puts whoever opened one at the top of your call list.
3. Starts a 36 hour timer on every first message, and drafts a plain-text email follow-up when it runs out with no reply.
4. Checks every draft against your writing rules.
5. **Never sends.** Drafts go to you in Slack. You press send.

The two differences: Loopster's shortest schedule is once an hour, so checks are hourly. Replies still arrive instantly through HeyReach's webhook. And Loopster can't write a Gmail draft, so the email follow-up arrives in Slack for you to paste.

---

## Before you build

- Connect **HeyReach** and **Slack** in Loopster (Settings, Connections). Both columns run on your own account and cost nothing from the wallet. `GET /v1/connections` should list both.
- Make one Slack channel for the agent, like `#outbound-agent`. The Slack column takes the name with its `#` as `channel_id`, so you don't need to look up the id.
- Your pipeline (the job-post build in `lane-scout-loopster.md`, or any other) is where people come from. This agent starts once you've messaged them.

---

## The workbook: three tables

```
Outreach log  (1 row = 1 person you messaged)
  first_touch_at, channel, tracked_link ...
  Link views     HTTP, hourly          -> Opened?  -> Slack "call now"
  Follow-up due? formula               -> Draft email -> Rules check -> Slack draft
        ^
        | send-to-table, update the row: replied = true
        |
Replies  (1 row = 1 HeyReach reply, arrives by webhook)
  Draft answer -> Rules check -> Slack alert

Call list  (a saved view on Outreach log, sorted by link views)
```

### Table 1: Outreach log

One row per person, added when you send the first message. Send rows here from your pipeline with a `send-to-table` column after you've sent, or add them by hand.

Manual columns: `name`, `linkedin_url` (url), `company`, `email`, `channel` (linkedin or email), `first_touch_at` (ISO time), `give_first` (the line you sent), `tracked_link` (url, optional), `replied` (true or false, starts false).

Auto-dedupe on `linkedin_url`, keep oldest.

Enrichment columns, in order:

1. **Link views** (`http`, GET, free). Only if your link tool has an API that returns a view count for a link. Run when `tracked_link` isn't empty. Put the link tool's key in the column's headers, never in a prompt.
2. **Opened?** (free formula). True when the view count from Link views is above 0. If the formula builder can't read the HTTP response, use a mini AI column (`ai-loopster-default-mini`, about $0.001) that returns `{"opened": true or false, "views": number}`.
3. **Call now** (`slack-send-message`). Run when `Opened? == true` and `replied != true`. Text: `Call now: @{name} at @{company} opened the link (@{views} views). @{linkedin_url}`. It runs once per row, so you get one alert per person.
4. **Follow-up due?** (free formula). True when 36 hours have passed since `first_touch_at` and `replied` is false. Check the formula builder handles dates and the current time. If it doesn't, use a mini AI column with today's date in the prompt.
5. **Draft follow-up** (`ai-loopster-default-mini`). Run when `Follow-up due? == true` and `email` isn't empty. Prompt:

```
Write a short plain-text follow-up email to @{name} at @{company}.
Two days ago I sent them this on LinkedIn: @{give_first}
They haven't replied. Say in one line what I sent, offer to send the link if they want it, and stop.
No link in the email. Under 60 words. Contractions, plain words, no dashes, no semicolons, no exclamation marks.
Never open with Just, Hope, Following up, Circling back or Bumping. Return only the email body.
```

6. **Rules check** (`ai-loopster-default-mini`, JSON). Run when Draft follow-up isn't empty. Prompt: check the draft against your writing rules and return `{"pass": true or false, "problems": "the words that broke a rule, or empty"}`. Paste your rules into the prompt. Mine: no em dashes, no semicolons, no "this, not that" contrasts, no delve, leverage, robust, seamless, unlock, game-changer.
7. **Send draft to me** (`slack-send-message`). Run when Rules check `pass == true`. Text: `Follow-up for @{name} (@{email}), no reply in 36h:` then the draft. Drafts that fail the check stay in the table for you to fix.

**What the test build found:**

- **Slack columns never run on autoplay.** Loopster marks any column that sends data outside Loopster as manual-trigger-only. So Call now, Send draft to me and Alert me need an hourly schedule (`cells: "empty"`) or a trigger that names them. An hourly schedule on them means alerts arrive within the hour.
- **Formulas work as written.** `Date.now()` and `new Date(...)` are fine in a formula. Mentions go in as `@{col_id}`. Return text like "yes" and gate on `@{col} == "yes"`.
- **Formulas recompute when a row changes, not as the clock moves.** A row that crosses 36 hours with nothing else changing stays "no". Put Follow-up due? on the hourly schedule with `cells: "all"`.
- **The webhook address only shows once.** `GET /tables/{t}/webhook` returns the last four characters. `POST /tables/{t}/webhook/rotate` returns the full address. Rotate before you give it to HeyReach.
- **Mark replied** is `send-to-table` with method `send-row`, `updateExistingRows: true`, the unique key set to Outreach log's `linkedin_url`, and `additionalData: [{"fieldName": "replied", "value": "true"}]`. That's free, and it's the line that stops the timer.

**Schedule:** one `interval` schedule on Link views, every 1 hour, `cells: "all"`, so the count refreshes. The formula columns recompute by themselves. Draft follow-up, Rules check and the two Slack columns run on autoplay once their condition becomes true, so turn **autoplay on for this table only after the one-row test**.

### Table 2: Replies

1. Create the table and get its webhook address: `GET /v1/tables/{tableId}/webhook`.
2. In HeyReach, add a webhook for **message reply received** pointing at that address.
3. Send one test event from HeyReach and look at the row it creates. Rename or add manual columns to match the payload's fields (the lead's name, LinkedIn URL, company and the message text). Don't guess the field names. Read them off the test row.

Enrichment columns:

1. **Mark replied** (`send-to-table`, free). Destination: Outreach log. Method: one row. Unique key: `linkedin_url`. `updateExistingRows: true`. Map `replied` to the text `true`. This is what stops the 36 hour timer for anyone who answered.
2. **Draft answer** (`ai-loopster-default-mini`). Give it the message text and ask for a two-line reply in your voice that answers their question and offers the next step. Same style rules as Draft follow-up.
3. **Rules check** (same as above).
4. **Alert me** (`slack-send-message`). Always runs. Text: `Reply from @{name} at @{company}:` then their message, then `Draft:` and the draft if Rules check passed.

Autoplay on for this table after the test event looks right. It costs about $0.002 a reply.

### The call list

A saved view on Outreach log (`POST /v1/tables/{t}/views`): filter `replied` is false, sort by views, most first. Open it each morning. Whoever opened a link is at the top.

---

## Test plan

1. Outreach log, one row about yourself: your own LinkedIn URL, an email you own, `first_touch_at` set to 40 hours ago, and a tracked link you've opened. Dry-run every column (`?dryRun=true`), then run it for real with `maxCostUsd: 0.05`. You should get one "Call now" message and one follow-up draft in Slack.
2. Replies: send HeyReach's test event. You should get one Slack alert, and the matching Outreach log row should flip to `replied: true`.
3. Add a second Outreach log row with `first_touch_at` set to 10 hours ago. Nothing should happen to it.

Only then: autoplay on, and the hourly schedule.

## What it costs

HeyReach, Slack, HTTP, formulas and send-to-table are free. Each draft and each check is a mini AI call at about $0.001. Twenty new people a week with a handful of replies comes to a few cents a week.

## What's still mine and not Loopster's

- **Gmail drafts.** Loopster has no Gmail column. The draft lands in Slack and you paste it.
- **Every 15 minutes.** Hourly is the floor for schedules. Replies are instant through the webhook, so this only delays link-open alerts.
- **Anything that sends.** HeyReach and Smartlead both have send columns in Loopster. This build never uses them, on purpose.
