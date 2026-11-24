# Zero to one outbound: HeyReach + Loopster

The build files from my talk at the HeyReach, Loopster, Smartlead and ZapMail event in Lisbon on 24 Nov 2026.

I'm Steven Brady. I run [lfgtme.com](https://lfgtme.com), a recruiting agency for GTM engineers, and I co-founded [Kith](https://www.kith.contact). Everything here is what I actually run, at small numbers, two weeks in.

## What's here

| File | What it is |
|---|---|
| [`lane-scout-loopster.md`](lane-scout-loopster.md) | The full spec for a two-table Loopster workbook. It finds fresh job posts, qualifies them, finds the hiring manager, checks they still work there, gets a work email and writes a give-first opening line from the actual job post. |
| [`table-config/`](table-config/) | The column settings (prompts, gating, providers) exported from a working build of that spec. Use them as a reference when a column won't behave. |

## Build it in 5 minutes

1. Get a Loopster API key with spending turned on, and set it as `LOOPSTER_API_KEY`.
2. Open Claude Code (or Cursor, or Codex) in an empty folder.
3. Hand it `lane-scout-loopster.md` and say: **build this in my Loopster workspace, autoplay off, spend nothing until I inspect.**
4. When it's done, run the test plan in the file. The first test is one search for $0.05.

A fresh Claude session built both tables from this file in about 3 minutes on the day of the talk. Running all 7 searches plus the follow-up on whatever qualifies costs roughly $0.35 to $0.80 a day.

## Make it yours

The spec is built for GTM engineer job posts, but only four things are specific to that. The "Swap for your niche" table in the spec lists them: the role titles, the titles to drop, your pay and location rule, and what you offer in the opening line. Change those and keep the rest.

## The traps

Each of these cost me money or a wrong contact before I wrote it down. They're all in the spec with the fix:

- Search gives you the company's job board, not the job. Resolve the real posting before you read it.
- The AI will invent a hiring manager if you let it search alone. Give it a list of real senior people first, then check the person it picks works at the company today.
- A blank salary field fails a job that lists pay in the post. Let the qualifier read the post.
- Ask an AI to "vary the opening" and half your emails still start with "Saw you're". Tie the opening style to something in the data, like the company name.

## Coming next

A build guide for the whole loop as one Loopster flow: find the job post, pick three matching candidates, write the message and push it into HeyReach.

## Contact

[lfgtme.com](https://lfgtme.com) · [kith.contact](https://www.kith.contact)
