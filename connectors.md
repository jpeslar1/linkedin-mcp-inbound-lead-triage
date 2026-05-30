# Connectors

Every MCP server this prompt uses, with setup and the placeholders to replace.

## Zevari — LinkedIn MCP for Claude (REQUIRED, the core of the prompt)

**What it does:** Live LinkedIn profile lookup (`linkedin_get_profile`), company intelligence (`agents_company_intelligence`), **behavioral profile** (`agents_behavioral_profile` — the most predictive endpoint in this whole pipeline), ICP scoring (`agents_icp_score`), and lead save (`targets_save`).

**Why it's the core of this workflow:** Every other inbound triage tool reads cached Apollo / Clearbit data. That misses promotions older than the scrape lag AND misses behavioral intent entirely. Zevari reads LinkedIn live at form-submit time, so triage decisions are made off today's title, today's company, and today's engagement signal.

**Setup:** [zevari.ai](https://zevari.ai) → connect LinkedIn → copy MCP server URL into Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID

**Endpoints used:**
- `linkedin_get_profile` — live current title + company + headline (load-bearing)
- `agents_company_intelligence` — employee count, industry, recent funding, hiring signals
- `agents_behavioral_profile` — recent engagement with your content, recent posts, recent comments (load-bearing for HOT detection)
- `profile_get` — load current ICP rules + class thresholds
- `agents_icp_score` — score the lead
- `targets_save` + `targets_create_list` + `targets_get_pipeline` — save lead + dedupe check
- `leads_lookup_by_url` — dedupe check before enrichment

## Supabase — Webhook queue (REQUIRED)

**What it does:** Receives form-fill webhook payloads from your site, stores them in `[INBOUND_QUEUE_TABLE]`, and lets the prompt drain unprocessed rows on each 15-minute run.

**Why Supabase:** Clean MCP for SQL, ACID writes from the form-side webhook, easy `processed_at IS NULL` filter to drain the queue. Convex or an n8n webhook listener will also work — the prompt structure is the same.

**Setup:**
1. Create a Supabase project (or use an existing one).
2. Create the queue table — minimum schema:
   ```sql
   CREATE TABLE inbound_leads (
     id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     submitted_at timestamptz NOT NULL DEFAULT now(),
     source text NOT NULL,                    -- 'demo_request' | 'newsletter_signup' | etc.
     email text,
     linkedin_url text,
     first_name text,
     last_name text,
     company_name text,
     form_payload jsonb,
     utm_source text,
     utm_campaign text,
     utm_content text,
     processed_at timestamptz,
     triage_class text,                       -- 'HOT' | 'WARM' | 'COLD' | 'SKIPPED'
     icp_score numeric,
     triage_reason text,
     zevari_target_id text
   );

   CREATE INDEX ON inbound_leads (processed_at) WHERE processed_at IS NULL;
   ```
3. Point your form / lead magnet / pricing page webhook at a Supabase Edge Function that inserts into this table.
4. Install the Supabase MCP server in Claude Code.

**Placeholders:**
- `[SUPABASE_MCP_ID]` — your Supabase MCP server ID
- `[INBOUND_QUEUE_TABLE]` — defaults to `inbound_leads` in the prompt; change if your table is named differently

**Endpoints used:**
- `execute_sql` — `SELECT ... WHERE processed_at IS NULL` and the `UPDATE` to close out the row

## Instantly — WARM + COLD drip campaigns (REQUIRED for non-HOT routing)

**What it does:** Enrolls WARM and COLD leads in pre-built nurture campaigns. HOT leads bypass Instantly entirely — they go straight to the AE.

**Setup:** [Instantly MCP docs](https://developer.instantly.ai/mcp). Build two campaigns ahead of time:

- **Warm campaign** — 4-6 emails, value-led, mid-cycle nurture (e.g. case study → use case → social proof → soft CTA). Get the UUID.
- **Cold campaign** — long-cycle drip, monthly cadence, content-only (newsletter sends, product updates, low-pressure). Get the UUID.

**Placeholders:**
- `[INSTANTLY_MCP_ID]` — your Instantly MCP server ID
- `[INSTANTLY_WARM_CAMPAIGN_ID]` — warm nurture campaign UUID
- `[INSTANTLY_COLD_CAMPAIGN_ID]` — cold drip campaign UUID

**Endpoints used:**
- `add_leads_to_campaign_or_list_bulk` — enrollment
- `list_leads` (optional) — pre-enrollment dedupe check

## Pipedrive — CRM for HOT deal creation + dedupe (REQUIRED)

**What it does:** Two jobs: (1) dedupe — check whether the lead is already a customer or in an open deal before triaging, and (2) create a `Qualified`-stage deal for every HOT lead so the AE sees it in their pipeline view.

**Setup:** Any Pipedrive MCP. I use `mcp__pipedrive__` from the community list.

**Setup checklist before running:**
- Make sure you have a "Qualified" stage in your pipeline
- Add custom fields: `lead_source` (text), `icp_score` (numeric), `triage_class` (text), `triage_reason` (text)

**Tool prefix in prompt:** `mcp__pipedrive__`

**Endpoints used:**
- `search_contacts`, `search_organizations` — dedupe at Step 2
- `add_person`, `add_organization` — create new records for HOT leads
- `add_deal` — create the HOT deal in `Qualified` stage

## Apollo — Optional enrichment (OPTIONAL)

**What it does:** Fills missing sides — email → LinkedIn URL for email-only signups, LinkedIn URL → email for LinkedIn-only signups. Also a firmographic backup if Zevari rate-limits.

**Why optional:** If every form you run requires both `email` and `linkedin_url` you don't need Apollo. If your forms are mixed (newsletter signups are email-only, lead magnets are sometimes LinkedIn-only) Apollo is useful as a resolver.

**Setup:** Any Apollo MCP server.

**Placeholders:**
- `[APOLLO_MCP_ID]` — your Apollo MCP server ID (optional)

**Endpoints used:**
- `enrich_person` — email → LinkedIn or LinkedIn → email

## Claude in Chrome — Slack webhook (REQUIRED)

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the only reliable way to fire a Slack webhook from Claude Code).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook URL
- `[HOT_LEAD_CHANNEL]` — channel where HOT lead pings land (e.g. `#sales-hot-leads`) — cosmetic, used in the ping body
- `[ALERT_CHANNEL]` — channel where errors + run summary land — cosmetic
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) for `@`-mentions

## AE calendar link (REQUIRED for HOT email)

Not a connector — just a placeholder used in the HOT lead email.

**Placeholders:**
- `[AE_CALENDAR_LINK]` — the booking URL that goes out to HOT leads in Step 7A.iii. Use TidyCal, Cal.com, Calendly, Chili Piper, whatever your AE uses.

## WebSearch

Built into Claude Code. Used for filling gaps when Zevari's `agents_company_intelligence` is thin — press releases, funding announcements, hiring signals.

No placeholder.

## Form-side webhook (what you build outside this repo)

This repo is the consumer. The producer is whatever ships form fills into Supabase. The shape that works:

1. Form submits (any form: HubSpot, Tally, Webflow, custom)
2. Form provider fires a webhook at a Supabase Edge Function
3. Edge Function inserts a row into `[INBOUND_QUEUE_TABLE]` with `processed_at = NULL`
4. (Optional) Edge Function returns `{status: "queued"}` to the form

The Edge Function is ~30 lines of TypeScript. Not included in this repo because it's specific to your form provider.
