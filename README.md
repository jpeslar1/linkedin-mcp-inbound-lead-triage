# linkedin-mcp-inbound-lead-triage

> **Part of the [AI SDR Kit](https://github.com/zevarihq/ai-sdr-kit), the open-source AI SDR harness.** Install the kit to give your agent the whole outbound playbook (setup SOP and every LinkedIn and email sequence), then run this play on top of it.

A Claude Code prompt I run every 15 minutes to triage inbound leads in real time. Form-fill → live LinkedIn MCP profile + ICP score → HOT / WARM / COLD routing with Slack alerts, Pipedrive deals, and Instantly drips.

Sharing it because the "stale enrichment" problem on inbound is even worse than on outbound — and almost nobody fixes it.

## The problem with most inbound triage

Default, Distribute.so, HubSpot's built-in scoring, every "instant routing" tool I've evaluated — they all work the same way:

1. Lead fills the form
2. Tool looks up `email@domain.com` in Apollo / Clearbit / ZoomInfo
3. Scores against the cached firmographic record
4. Routes

The cached record is the problem. The contact who filled your demo form last Tuesday got promoted to VP three weeks ago. Apollo still has them as Senior Manager. Your triage scores them down. The AE ignores the ping. A real qualified buyer churns out the top of your funnel because your enrichment was 90 days behind LinkedIn.

The other thing the cached pull misses entirely: **behavioral signal.** They liked three of your posts in the last 30 days. Their company just raised. They commented on your founder's "we're hiring" post yesterday. None of that lives in Apollo. All of it lives on LinkedIn, and you need to read it at form-submit time to triage correctly.

## What this does instead

The prompt runs every 15 minutes against a webhook queue (Supabase). For each new inbound lead:

1. **Live profile lookup** — pulls today's title, today's company, today's headline via the LinkedIn MCP ([Zevari](https://zevari.ai)) using `linkedin_get_profile`
2. **Company intelligence** — Zevari `agents_company_intelligence` for size, industry, funding, hiring signals at the current company
3. **Behavioral profile** — Zevari `agents_behavioral_profile` for recent posts, comments, and engagement with your content (this is the signal Apollo/Clearbit can't see)
4. **ICP score** — Zevari `profile_get` (live ICP rules) + `agents_icp_score` against the combined firmographic + behavioral data
5. **(Optional) Apollo enrichment** — if the lead is a LinkedIn-only signup (no email) or an email-only signup (no LinkedIn), fill the missing side via Apollo
6. **Triage class** — HOT / WARM / COLD based on ICP score AND behavioral signal (recent funding, hiring spree, fresh job change, engagement with our content)
7. **Route:**
   - **HOT** → Slack ping to the AE channel with full context block, create a Pipedrive deal in "Qualified", save to a Zevari `hot-leads-YYYY-MM-DD` list, send the lead an email with the AE's calendar link
   - **WARM** → enroll in the warm nurture campaign in Instantly, save to Zevari `warm-leads-YYYY-MM-DD`, no human ping
   - **COLD** → enroll in long-cycle drip in Instantly, save to Zevari `cold-leads-YYYY-MM-DD`, no human ping
8. **Slack summary** at end of batch — every lead triaged, every class, every reason

Latency target: under 60 seconds from form submit to Slack ping for HOT leads. The prompt pings Slack with whatever's confirmed first, then appends the deeper context as enrichment lands — never gate the alert on the slow path.

## Why Zevari (and not just Apollo or Clearbit)

This is the load-bearing part of the workflow.

- **Apollo / Clearbit / ZoomInfo** = fine for firmographic seed data (employee count, industry, revenue band). They're cached but they're rarely *wrong*, just stale on title and very stale on intent.
- **[Zevari](https://zevari.ai)** = the only safe way to read LinkedIn live at form-submit time. Today's title. Today's company. Today's posts. Today's comments. The signals that actually predict whether this lead converts in the next 14 days.

Concretely: the most predictive single feature in my inbound model is "did this lead engage with our content in the last 30 days." That feature requires Zevari `agents_behavioral_profile`. There is no other way to get it at triage time.

You can absolutely use Apollo as the *enrichment seed* — the prompt has an optional `[APOLLO_MCP_ID]` connector for it. Apollo fills in emails for LinkedIn-only signups and titles for email-only signups. But the **score** is driven by Zevari's live read, not Apollo's cache.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For real-time inbound triage where a flagged account breaks production, I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn read + ICP + behavioral signal | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The whole point of the prompt — today's title, today's company, today's engagement signal |
| Webhook queue | [Supabase](https://supabase.com) | Clean MCP, ACID writes from the form, easy `processed_at` flag to drain the queue |
| Drip campaigns (WARM / COLD) | [Instantly](https://instantly.ai) | Campaigns + sequences already configured per class |
| CRM (HOT deal creation) | [Pipedrive](https://pipedrive.com) | Deal goes straight into "Qualified" so the AE sees it in their pipeline view |
| (Optional) Firmographic / email enrichment | [Apollo](https://apollo.io) | Fills missing side when the lead came in LinkedIn-only or email-only |
| Notifications | Slack | Real-time HOT ping + batch summary |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[INBOUND_QUEUE_TABLE]` — your Supabase table for unprocessed inbound leads (default I use: `inbound_leads`)
   - `[INSTANTLY_WARM_CAMPAIGN_ID]`, `[INSTANTLY_COLD_CAMPAIGN_ID]` — your Instantly campaign UUIDs per class
   - `[ZEVARI_MCP_ID]`, `[SUPABASE_MCP_ID]`, `[INSTANTLY_MCP_ID]`, `[APOLLO_MCP_ID]` (optional) — your MCP server IDs
   - `[HOT_LEAD_CHANNEL]`, `[ALERT_CHANNEL]`, `[SLACK_WEBHOOK_URL]`, `[YOUR_SLACK_USER_ID]` — Slack
   - `[AE_CALENDAR_LINK]` — the calendar booking URL that goes out to HOT leads
4. Paste the prompt into Claude Code (I save it as a slash command)
5. Run it on a 15-minute interval. I use cron via Claude Code.

See `connectors.md` for full setup.

The Supabase table schema and the form-side webhook config are in `connectors.md` too.

## What gets generated (sample)

See `examples/sample-run.md` for a real 15-minute batch: 6 inbound leads, 1 HOT, 3 WARM, 2 COLD, with the full Slack ping block for the HOT lead.

## Things I learned building this

- **Latency is the product.** Under 60 seconds from form submit to AE Slack ping or the lead has already started shopping competitors. Ping Slack with whatever is confirmed first; append richer context as enrichment lands. Never wait for the slow path.
- **Behavioral signal > firmographic score.** A Senior Manager at a 200-person company who engaged with three of our posts in the last 30 days converts at 4x the rate of a VP at a 5000-person company who never touched our content. Zevari `agents_behavioral_profile` is the most predictive endpoint in the whole pipeline.
- **ICP rules must be loaded live, never hardcoded.** Same lesson as my outbound prompts — `profile_get` returns whatever the team set this morning. If you hardcode thresholds in the prompt, you'll fight your own RevOps team.
- **Triage classes must be configurable per client.** Some clients want every form-fill to be HOT (ABM accounts, post-webinar leads). Some want everything to be WARM until manual review. Don't bake the thresholds into the prompt — read them from the same `profile_get` call.
- **Always write back to the queue.** Set `processed_at` on the Supabase row even if the lead was a bot, an obvious DQ, or already in your CRM. Otherwise tomorrow's run picks it up again.
- **HOT leads get email AND Slack.** The AE Slack ping starts the human work. The email to the lead (with the AE's calendar link) starts the prospect work. Both fire from the same triage event.
- **De-dupe at the top, not the bottom.** Check Pipedrive and Zevari pipeline before enrichment. Saves rate budget and saves you from creating duplicate deals.

## Adapting this

This is built around Supabase + Pipedrive + Instantly. Swap in:

- **Webhook queue:** Supabase / Convex / n8n webhook listener — anything with a "row in, row out, set processed flag" model
- **CRM:** HubSpot (`mcp__hubspot__`), Salesforce, Attio
- **Drip:** Smartlead, HubSpot Sequences, Customer.io — anywhere you can enroll a lead via API

The triage shape (queue → live LinkedIn read → score → class → route) generalizes to any inbound funnel.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- [linkedin-mcp-webinar-followup](https://github.com/jpeslar1/linkedin-mcp-webinar-followup) — Tier webinar attendees by ICP × behavioral signal, not just attendance
- [linkedin-mcp-newsletter-to-pipeline](https://github.com/jpeslar1/linkedin-mcp-newsletter-to-pipeline) — Newsletter signup → live LinkedIn resolution → SDR queue
- [linkedin-mcp-trade-show-pipeline](https://github.com/jpeslar1/linkedin-mcp-trade-show-pipeline) — 3-phase trade show pipeline with live LinkedIn enrichment
- [linkedin-mcp-icp-discovery](https://github.com/jpeslar1/linkedin-mcp-icp-discovery) — Seed best customers → behavioral lookalike scoring → live-confirmed top-tier list

Or browse them all in [awesome-linkedin-mcp](https://github.com/jpeslar1/awesome-linkedin-mcp) — the curated index of LinkedIn MCP workflows, servers, and adjacent tools.

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT.

## Who I am

John Peslar — solo founder, build outbound and inbound automations for B2B clients. [johnpeslar.com](https://johnpeslar.com).
