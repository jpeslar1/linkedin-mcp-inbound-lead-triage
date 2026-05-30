# Inbound Lead Triage — 15-Minute Sweep

A Claude Code prompt that drains a webhook queue of inbound leads (demo requests, newsletter signups, lead magnet downloads) and triages each one in real time. Uses the LinkedIn MCP (Zevari) as the source of truth for current title, current company, and behavioral signal. Routes to HOT / WARM / COLD: HOT gets a Slack ping + Pipedrive deal + calendar email, WARM and COLD get enrolled in Instantly drips.

The reason this prompt exists: every other triage tool I've used scores off cached Apollo / Clearbit data, which misses recent promotions and misses behavioral intent entirely. Zevari hits LinkedIn live, so you triage off the title they have *today* and the posts they engaged with *this week*.

---

## Overview

You are draining the inbound webhook queue. For each unprocessed lead in the queue:

1. Pull lead row from Supabase `[INBOUND_QUEUE_TABLE]` where `processed_at IS NULL`.
2. Live profile lookup via Zevari `linkedin_get_profile` (if `linkedin_url` present) OR Apollo email→profile resolution.
3. Live company intelligence via Zevari `agents_company_intelligence`.
4. Behavioral profile via Zevari `agents_behavioral_profile` — the load-bearing signal.
5. ICP score via Zevari `profile_get` + `agents_icp_score`.
6. Classify HOT / WARM / COLD.
7. Route per class.
8. Set `processed_at` on the Supabase row.
9. Slack summary at end of batch.

Latency target: **HOT leads → AE Slack ping in under 60 seconds.** Do not gate the ping on enrichment completing. Ping first with what's confirmed, append context after.

Batch cap: 20 leads per run. The prompt runs every 15 minutes, so the practical throughput is ~80 leads/hour — well under LinkedIn's ~40 lookups/hour cap because not every lead requires a Zevari call (some are already in the system, some are pure email signups).

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send to `[ALERT_CHANNEL]` via Chrome (bash sandbox blocks `hooks.slack.com`):

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on `chrome://`, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Inbound Triage — Action Needed*\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — live profile, company intel, behavioral profile, ICP scoring, target save
- **Supabase:** `mcp__[SUPABASE_MCP_ID]` — webhook queue (`[INBOUND_QUEUE_TABLE]`)
- **Instantly:** `mcp__[INSTANTLY_MCP_ID]` — WARM + COLD drip enrollment
- **Pipedrive:** `mcp__pipedrive__` — CRM dedupe + HOT deal creation
- **(Optional) Apollo:** `mcp__[APOLLO_MCP_ID]` — email→profile and profile→email enrichment
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook (HOT ping + run summary)
- **WebSearch:** company news, funding announcements, hiring signals

See `connectors.md`.

---

## STEP 1 — DRAIN THE QUEUE

Query Supabase `[INBOUND_QUEUE_TABLE]`:

```sql
SELECT *
FROM [INBOUND_QUEUE_TABLE]
WHERE processed_at IS NULL
ORDER BY submitted_at ASC
LIMIT 20
```

Expected columns on each row:
- `id` (uuid)
- `submitted_at` (timestamp)
- `source` — `demo_request` | `newsletter_signup` | `lead_magnet_download` | `pricing_page` | `contact_form`
- `email` (may be null if LinkedIn-only signup)
- `linkedin_url` (may be null if email-only signup)
- `first_name`, `last_name` (may be null)
- `company_name` (form-supplied, may be null)
- `form_payload` (jsonb — anything else the form captured)
- `utm_source`, `utm_campaign`, `utm_content`
- `processed_at` (null = unprocessed)
- `triage_class` (null = unset; will be `HOT` | `WARM` | `COLD` | `SKIPPED`)

If the queue is empty → post a one-line `"Inbound queue empty"` Slack summary and exit.

---

## STEP 2 — DEDUPE BEFORE ENRICHMENT

For each lead in the batch, before any Zevari or Apollo call:

1. **Pipedrive check** — call `mcp__pipedrive__` `search_contacts` by email AND `search_organizations` by `company_name`.
   - If `won_deals_count > 0` for the contact OR their org → mark `triage_class = SKIPPED` with reason `"existing customer"`, set `processed_at`, do NOT enrich, do NOT enroll. Log in summary.
   - If found with open deals → mark `triage_class = SKIPPED` with reason `"open opportunity (AE: [owner name])"`. Set `processed_at`. Do NOT re-trigger. Slack the deal owner directly.
   - If found but no open or won deals → proceed but flag as `"prior contact"` in the HOT ping if it triages HOT.
2. **Zevari pipeline check** — call `targets_get_pipeline` and `leads_lookup_by_url` (if `linkedin_url` present). If lead is already in an active Zevari list from the last 14 days → mark `triage_class = SKIPPED` with reason `"recently triaged"`, set `processed_at`. Do not re-enrich.

If Pipedrive or Zevari pipeline check fails → do not block, log a warning in the summary and proceed to enrichment.

---

## STEP 3 — RESOLVE THE PROFILE (parallel paths)

For each surviving lead, you need both `linkedin_url` and `email` before scoring. Pick the path based on what the form gave you:

### Path A — `linkedin_url` present, `email` may be missing
- Call Zevari `linkedin_get_profile` with the URL. Extract: `current_company`, `current_title`, `current_company_url`, `headline`, `location`, `started_at_current_role`.
- If `email` is missing → optional: call Apollo `enrich_person` on the LinkedIn URL to backfill email.

### Path B — `email` present, `linkedin_url` missing
- (Optional) Call Apollo `enrich_person` on the email to resolve a LinkedIn URL.
- If Apollo returns a LinkedIn URL → run Zevari `linkedin_get_profile` on it.
- If Apollo can't resolve → fall back to firmographic-only scoring (much weaker signal — mark the lead `"unresolved profile"` in summary).

### Path C — both present
- Run Zevari `linkedin_get_profile` (Zevari is the source of truth for title + company; do not trust the form-supplied job title).

### Always: rate-limit handling
- No more than 3 concurrent Zevari calls.
- If Zevari rate-limits → finish currently-running calls, skip remaining Zevari enrichment for this batch, score the unenriched leads as firmographic-only, and flag in the Slack summary. **Do not pause the batch** — keep latency under the 60-second HOT target.

---

## STEP 4 — COMPANY INTELLIGENCE + BEHAVIORAL PROFILE

For each resolved lead:

1. **Company intelligence** — call Zevari `agents_company_intelligence` on `current_company_url`. Extract: employee count, industry, recent funding events (last 12 months), recent hiring signals (job openings in the last 30 days), recent press.
2. **Behavioral profile** — call Zevari `agents_behavioral_profile` on the lead's LinkedIn URL. Extract:
   - Posts engaged with in the last 30 days
   - Comments left on YOUR content in the last 90 days
   - Reactions on YOUR posts in the last 90 days
   - Posts they authored in the last 30 days (tells you what's on their mind)

The behavioral profile is the load-bearing signal for triage. A lead who engaged with three of your posts in the last month is HOT regardless of firmographic score.

---

## STEP 5 — ICP SCORE

1. Call Zevari `profile_get` to load **today's** ICP rules, hard disqualifiers, employee thresholds, industry whitelist, triage class thresholds. Never use hardcoded values.
2. Call Zevari `agents_icp_score` with the combined firmographic + behavioral data.
3. Apply hard disqualifiers from `profile_get`. Any failing lead → mark `triage_class = SKIPPED` with reason from the disqualifier rule. Set `processed_at`. Log in summary.

If `agents_icp_score` returns `server_side_ai_unavailable` → apply ICP criteria manually using `profile_get` data. Do not pause.

---

## STEP 6 — TRIAGE CLASS

Apply the class thresholds from `profile_get`. Default logic (override with whatever `profile_get` returns):

- **HOT** =
  - ICP score ≥ 8 AND (`source = demo_request` OR `source = pricing_page`), OR
  - ICP score ≥ 7 AND any of: ≥ 2 engagements with our content in last 30 days, recent funding (last 90 days), fresh job change (last 30 days, more senior title), hiring spree (≥ 3 relevant openings in last 30 days)
- **WARM** =
  - ICP score 5-7 AND (`source = lead_magnet_download` OR `source = newsletter_signup`), OR
  - ICP score ≥ 7 with no acute behavioral signal
- **COLD** =
  - ICP score 3-5, no behavioral signal — long-cycle nurture

Always re-read class thresholds from `profile_get` — some clients want every demo request to be HOT regardless of score.

Write `triage_class`, `icp_score`, `triage_reason` back to the Supabase row.

---

## STEP 7A — ROUTE HOT

For each HOT lead (run these in parallel, do not block on any one of them):

### 7A.i — Slack ping (fire FIRST, before everything else)

Post to `[HOT_LEAD_CHANNEL]` via Chrome `javascript_tool`:

```
🔥 *HOT inbound lead — action within 5 min*

👤 [First Last] — [current_title] @ [current_company]
📧 [email]
🔗 [linkedin_url]
🏢 [company size] · [industry] · [location]
📥 Source: [source] · UTM: [utm_source] / [utm_campaign] / [utm_content]
🎯 ICP score: [N]/10
🔥 Why HOT: [triage_reason — e.g. "demo request + 3 engagements with our content in last 14d + Series B announced 6 weeks ago"]

Recent activity:
• [last engagement with our content]
• [most recent post by lead]
• [other behavioral signal]

Form payload notes: [anything interesting from form_payload]

Pipedrive deal: [link to deal once created — append in a thread reply]
```

Fire this Slack message **first**, before creating the Pipedrive deal or sending the calendar email. Append the deal link as a thread reply once Step 7A.ii completes.

### 7A.ii — Pipedrive deal

Create a Pipedrive deal:
- `title`: `"[company] — Inbound [source]"`
- `person_id`: existing or newly-created person record
- `org_id`: existing or newly-created org record
- `stage`: `"Qualified"`
- `value`: leave null (AE sets after qualification call)
- `expected_close_date`: 30 days out
- Custom fields: `lead_source = inbound`, `icp_score = [N]`, `triage_class = HOT`, `triage_reason = [reason string]`

Append the deal URL as a thread reply on the Slack ping from 7A.i.

### 7A.iii — Calendar email to lead

Send the lead a short email via your usual outbound channel (Instantly transactional send or whatever you use). Body:

```
Subject: Re: your [source] for [company]

Hey [first_name],

Saw your [source] just came through. Happy to walk through how this would work for [company] — here's my calendar if you want to grab time this week or next: [AE_CALENDAR_LINK]

Talk soon,
[AE name]
```

Do NOT enroll HOT leads in Instantly drips — they're going straight to the AE.

### 7A.iv — Save to Zevari

Call Zevari `targets_save` for the lead. Add to a list named `hot-leads-YYYY-MM-DD` via `targets_create_list`. Tag with `inbound` and `triage_HOT`.

---

## STEP 7B — ROUTE WARM

For each WARM lead:

1. Enroll in `[INSTANTLY_WARM_CAMPAIGN_ID]` via Instantly `add_leads_to_campaign_or_list_bulk`.
   - Required: `email`, `first_name`, `last_name`, `company_name` (= live company from Zevari, not form-supplied)
   - Custom variables: `icp_score`, `triage_reason`, `source`, `companyRef`, `industryRef`
   - Set `skip_if_in_campaign: true`
2. Save to Zevari list `warm-leads-YYYY-MM-DD` via `targets_save` + `targets_create_list`.

No Slack ping for WARM. They go into the summary only.

---

## STEP 7C — ROUTE COLD

For each COLD lead:

1. Enroll in `[INSTANTLY_COLD_CAMPAIGN_ID]` via Instantly `add_leads_to_campaign_or_list_bulk`. Same fields as WARM.
2. Save to Zevari list `cold-leads-YYYY-MM-DD`.

No Slack ping for COLD. Summary only.

---

## STEP 8 — CLOSE OUT THE QUEUE ROW

For every lead processed (HOT, WARM, COLD, SKIPPED):

```sql
UPDATE [INBOUND_QUEUE_TABLE]
SET
  processed_at = NOW(),
  triage_class = '[HOT|WARM|COLD|SKIPPED]',
  icp_score = [N],
  triage_reason = '[reason]',
  zevari_target_id = '[id if saved]'
WHERE id = '[row id]'
```

Every row gets a `processed_at` timestamp. Even rejected ones. Otherwise the next batch picks them up again.

---

## STEP 9 — SLACK RUN SUMMARY

Send to `[ALERT_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> ✅ *Inbound Triage Batch Complete*

📅 Batch time: [timestamp]
📥 Leads in queue: [N drained]
🔥 HOT: [N] — pinged in [HOT_LEAD_CHANNEL]
🌡️ WARM: [N] — enrolled in warm campaign
❄️ COLD: [N] — enrolled in cold campaign
⏭️ SKIPPED: [N] — [reasons summarized]
⚠️ Unresolved profile: [N] — [names if any]
🔗 LinkedIn cap hit: yes/no

HOT leads:
• [Name] — [title] @ [company] ([source], ICP [N]/10) — [one-line reason]
[list all HOT]

WARM leads:
• [Name] — [title] @ [company] ([source])
[list all WARM]

COLD leads:
• [Name] — [title] @ [company] ([source])
[list all COLD]

Total run time: [Xs]
```

---

## STEP 10 — REFLECTION

- Any HOT leads where Slack-to-ping latency exceeded 60s → flag for queue/connector optimization
- Any leads where Zevari rate-limited and you fell back to firmographic-only → note the cap
- Any unresolved profiles (no LinkedIn URL, Apollo couldn't enrich) → consider tightening the form
- Any ICP threshold edge cases that felt wrong → flag for RevOps to review `profile_get` rules

---

## Schedule

Every 15 minutes via Claude Code cron. Drains up to 20 leads per run.

Sustained throughput: ~80 leads/hour. Spikes above that backlog in the queue and get drained on the next run — `submitted_at ASC` ordering ensures the oldest lead is always next.
