# Build a give-first outbound engine for any company

Hand this file, plus `lane-scout-loopster.md`, to your coding agent with one website. That's the whole input.

## The prompt

Say this, or paste it, with your own website in it:

> Read build-for-any-company.md and lane-scout-loopster.md. My company is **yourcompany.com**. Build my outbound engine in Loopster. Spend nothing until I've looked at it.

Everything below is for the agent.

---

## Agent: the rules before anything else

1. **Spend nothing without a yes.** Read `GET /v1/wallet` first. Build tables and columns (free), use `POST /v1/search` (free), use signal previews (free, 20 a day) and `?dryRun=true` on every trigger (free). Anything that bills, ask first: a real trigger, creating a signal monitor (its first check bills straight away), or a schedule. When you ask, give the dry-run price.
2. **Autoplay off** on every table until the human has seen a row.
3. **Never send anything.** No HeyReach, Smartlead or Slack column runs on its own. The output is a table of drafts a person reads.
4. Read `lane-scout-loopster.md` all the way through, especially **API shapes** and **Traps**. That file is a worked example for one company (a recruiter placing GTM engineers). This file tells you how to swap it for any company.

## Step 1. Read the website and write the brief (2 minutes, free)

Open the homepage, the services or product page and the about page. Write five lines and show them to the human before you build:

| Line | What it is | Example (the worked build) |
|---|---|---|
| **Seller** | who they are and what they sell, in their words | Recruiter placing GTM engineers |
| **Buyer** | the job title that signs, and the company type, size and country | Founder or head of growth at a US/UK/Canada startup |
| **Moment** | the public thing that happens when that buyer needs them now | They post a GTM engineer job |
| **Proof gate** | the yes/no check that the moment is real and worth a message | Base pay $120K+, job in US/UK/Canada |
| **Give** | something useful you can make from the moment, before asking for a call | Three matched GTM engineers, three reasons each |

Take the buyer, the proof and the give from the site's own claims, case studies and offer. If the site offers "one fix you can start this week", that's the give. Don't invent services they don't sell.

If the human says the brief is wrong, fix it. Don't build until they've seen it.

## Step 2. Pick the signal

Match the **Moment** to one of these. Pick one. A second signal can come later.

| If the moment is... | Use | Cost |
|---|---|---|
| they're hiring a specific role | **Job posts.** The two-table build in `lane-scout-loopster.md`, with the role titles swapped | $0.05 a search, about $0.10 to $0.15 a qualified post |
| their team in a department is growing | Signal monitor `hiring-trends` or `hiring-velocity`, on an accounts table | about half a cent a company a week |
| a new leader just started (new VP Sales, new CRO) | Signal monitor `work-milestones`, goal New hire | about half a cent a signal |
| a champion moved to a new company | `work-milestones`, goal Job change | about half a cent a signal |
| headcount jumped | `employee-growth` | about half a cent a company check |
| they raised, launched or were in the news | `news`, `producthunt-launch` | 1 to 2 cents a signal |
| their pricing, careers or product page changed | `website-changes` | a fraction of a cent a page check |
| they posted about the problem | `linkedin-post-company` or `linkedin-post-contact` | about half a cent a signal |

`GET /v1/signal-types` lists them all with live prices. `GET /v1/signal-types/{slug}` shows each one's goals, event types and the columns it adds.

**Job posts are the default.** They're the easiest moment to read and the easiest give to make specific. Use a signal monitor when the buyer doesn't hire for a role that reveals the need.

## Step 3. Build it

### If the signal is job posts

Build `lane-scout-loopster.md` exactly, and change only these:

| Knob | Change it to |
|---|---|
| Role titles in Discover and the 7 seed queries | the role the **Moment** names (for example SDR, BDR, Head of Sales Development) |
| Drop list in Discover | look-alike titles that aren't the moment |
| Qualify prompt | the **Proof gate**, still two checks at most |
| Leaders at company seniorities | keep founder, c_suite, vp, head, director |
| Hiring contact hint | the **Buyer** title, as the person the new hire reports to |
| Give-first prompt, "from a recruiter who places GTM engineers" | the **Seller** line |
| Give-first prompt, sentence 2 | an offer of the **Give** |

Keep the Right person? check, the letter rule for openers and every gate. They exist because each one failed once.

Name the workbook after the company and the signal, like `Acme: SDR hiring`.

### If the signal is a monitor

1. **Accounts table.** `POST /v1/search` is free and returns up to 50 companies a page. `GET /v1/search/filters` lists the filter ids (country, employee count, industry and more). Search for the **Buyer** company type and load 25 to 50 rows: `company`, `domain`, `linkedin_url`. Don't add more until the human likes the first 25.
2. **Preview the signal, for free.** `POST /v1/signal-types/{slug}/preview` with `{"source": {"tableId": ..., "identifierKind": "domain", "identifierColumnId": "<domain col id>"}, "eventTypes": [...]}`. It samples up to 10 companies and shows the rows a monitor would add, plus the monthly cost. Show the human.
3. **Signals table.** Create a table called `Signals` and copy the preview's `columns` as manual columns. Load the preview's sample rows into it, so the pipeline columns have something to run on before you spend anything.
4. **Pipeline columns on Signals,** in this order. The shapes are in `lane-scout-loopster.md`:
   - **Proof gate** (mini AI, JSON `{"pass", "reason"}`): is this event the **Moment**, and is it recent?
   - **Leaders at company** (`ai-ark-find-people-at-company`, gated on pass)
   - **Buyer** (mini AI picks one LinkedIn URL from the leaders list, the **Buyer** title first. Never build a URL from a name.)
   - **Still there?** then **Right person?** (same as the worked build)
   - **Work email** (waterfall, gated on Right person?)
   - **Give-first line** (the worked build's prompt with the **Seller** and **Give** swapped in, and the event in place of the job post)
5. **Don't create the monitor.** Creating one bills its first check at once. Tell the human the monthly price from the preview, and create it only when they say yes, with `monthlyCapUsd` set. Then `POST /v1/signal-monitors/{id}/delivery-tables` with `{"table": {"kind": "existing", "tableId": "<Signals id>"}}`.

## Step 4. Show it (still free)

1. Dry-run the whole pipeline on 2 or 3 rows: `POST /tables/{t}/trigger?dryRun=true` with `rowIds`, `columnIds` and `maxCostUsd`. Report `expectedCostUsd`.
2. Mint a read-only share link for each table (`POST /tables/{t}/shares` with `expiresInSeconds: 86400`) so the human can click it.
3. Report in this shape, and nothing else:
   - the five-line brief
   - the signal you picked and why, in one sentence
   - the tables and columns you built, with the share links
   - the dry-run price for 3 rows, and what a day or a month would cost
   - the one thing you weren't sure about

Then wait for a yes before any real run.

## The traps that catch new builds

- **A brief that sounds like a brochure.** "Companies seeking growth" is no buyer. Name a title and a company size.
- **A moment nobody can see.** "They're unhappy with their outbound" isn't public. A job post for three more SDRs is.
- **A give that's a meeting in disguise.** "Free strategy call" is an ask. Make something they'd keep even if they never reply: a short teardown, a list, three people, one fix.
- **Openers that all start the same.** Keep the letter rule from the worked build.
- **Building before the human has read the brief.** Five lines up front save a rebuild.
