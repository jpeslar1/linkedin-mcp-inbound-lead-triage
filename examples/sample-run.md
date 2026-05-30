# Sample run

A real 15-minute drain (sanitized — names changed) from a recent Tuesday afternoon batch.

## Setup

- Supabase queue had 6 unprocessed rows at run start
- Mix of sources: 1 demo request, 2 lead magnet downloads, 2 newsletter signups, 1 pricing page
- All connectors healthy

## Step 1 — queue snapshot

| id | source | email | linkedin_url | company (form) | utm_campaign |
|---|---|---|---|---|---|
| ...a4f1 | demo_request | s.harlow@oatly.com | yes | Oatly | q2-launch |
| ...c2e8 | lead_magnet_download | m.diaz@thrilling.com | yes | Thrilling | retail-playbook |
| ...d99b | newsletter_signup | (null) | yes | (null) | gemini-edition |
| ...e3c1 | lead_magnet_download | k.osei@graza.com | yes | Graza | retail-playbook |
| ...f102 | pricing_page | j.tan@swiggy.com | (null) | Swiggy | (direct) |
| ...01a7 | newsletter_signup | r.singh@gmail.com | (null) | (null) | weekly |

## Step 2 — dedupe pass

| Lead | Pipedrive | Zevari pipeline | Action |
|---|---|---|---|
| S. Harlow — Oatly | not found | not in pipeline | proceed |
| M. Diaz — Thrilling | found, no deals | not in pipeline | proceed (flag "prior contact") |
| (LinkedIn-only signup) | n/a (no email) | LinkedIn URL not in pipeline | proceed |
| K. Osei — Graza | won_deals=1 | — | **SKIPPED — existing customer** |
| J. Tan — Swiggy | not found | n/a (no LinkedIn URL) | proceed |
| R. Singh — gmail | not found | n/a | proceed (firmographic-only — no LinkedIn) |

Net: 5 proceed, 1 SKIPPED.

## Step 3 — profile resolution

| Lead | Path | Result |
|---|---|---|
| S. Harlow | C (both present) | Zevari → "VP eCommerce @ Oatly" (form said "Director eCommerce" — title was stale) |
| M. Diaz | C | Zevari → "Head of Sales @ Thrilling" |
| (newsletter, no email) | A | Zevari → "Director of Brand @ Magic Mind". Apollo backfilled email: d.westra@magicmind.com |
| J. Tan | B (no LinkedIn) | Apollo resolved → linkedin.com/in/j-tan-swiggy. Zevari → "Sr. Director of Growth @ Swiggy" |
| R. Singh | B | Apollo could not resolve. Marked "unresolved profile" — firmographic-only |

Note on S. Harlow: form said Director eCommerce. Zevari says VP eCommerce (promoted 6 weeks ago). This is exactly the failure mode this whole pipeline exists to catch — Apollo-cached enrichment would have triaged her against the old title.

## Step 4 — behavioral profile (highlights)

- **S. Harlow** — reacted to 2 of our posts in last 14 days, commented on 1 last week, authored a post yesterday on "operator-led DTC growth"
- **M. Diaz** — reacted to 1 post 6 weeks ago, no other engagement
- **D. Westra (Magic Mind)** — no engagement with our content; authored 4 posts in last 30 days about retail expansion
- **J. Tan (Swiggy)** — no engagement with our content; authored 2 posts in last 30 days about marketplace economics
- **R. Singh** — no LinkedIn, no behavioral signal

## Step 5 — ICP score

| Lead | ICP score | Hard DQ? |
|---|---|---|
| S. Harlow | 9/10 | no |
| M. Diaz | 8/10 | no |
| D. Westra | 7/10 | no |
| J. Tan | 6/10 | no |
| R. Singh | 4/10 | no |

## Step 6 — triage class

| Lead | Class | Reason |
|---|---|---|
| S. Harlow | **HOT** | demo_request + ICP 9 + 3 engagements with our content in last 14d + promotion to VP 6 weeks ago |
| M. Diaz | WARM | lead_magnet_download + ICP 8 + no acute behavioral signal |
| D. Westra | WARM | newsletter_signup + ICP 7 + authoring on relevant theme |
| J. Tan | WARM | pricing_page + ICP 6 (one notch below HOT — flagged for manual AE review) |
| R. Singh | COLD | ICP 4, no resolvable LinkedIn, gmail address |

## Step 7A — HOT routing for S. Harlow

### 7A.i — Slack ping to `#sales-hot-leads` (fired at T+38 seconds from queue read)

```
🔥 HOT inbound lead — action within 5 min

👤 Sarah Harlow — VP eCommerce @ Oatly
📧 s.harlow@oatly.com
🔗 linkedin.com/in/sarah-harlow-oatly
🏢 1,500 employees · CPG (oat dairy alternative) · Malmö, SE
📥 Source: demo_request · UTM: linkedin / q2-launch / hero-cta
🎯 ICP score: 9/10
🔥 Why HOT: demo request + promoted to VP eCommerce ~6 weeks ago + 3 engagements with our content in last 14d (reacted to your "DTC margin compression" post, commented on the "Shopify Plus vs Centra" post, reacted to the founder note Friday)

Recent activity:
• Reacted to our DTC margin post (10 days ago)
• Authored a post yesterday on "operator-led DTC growth" (3.2K views)
• Commented on a peer's Shopify Plus vs Centra post 4 days ago

Form payload notes: requested "team of 4" demo, mentioned current stack: Shopify Plus + Klaviyo + Gorgias

Pipedrive deal: (linking in thread once created)
```

### 7A.ii — Pipedrive deal

Created `"Oatly — Inbound demo_request"` in `Qualified` stage. Custom fields: `lead_source=inbound`, `icp_score=9`, `triage_class=HOT`. URL appended to the Slack ping as a thread reply.

### 7A.iii — Email to S. Harlow from AE

```
Subject: Re: your demo request for Oatly

Hey Sarah,

Saw your demo request just came through. Happy to walk through how this would work for Oatly — here's my calendar if you want to grab time this week or next: https://tidycal.com/[AE]/30-minute-meeting

Talk soon,
[AE]
```

### 7A.iv — Saved to Zevari `hot-leads-2026-05-26`, tagged `inbound` + `triage_HOT`.

## Step 7B — WARM routing

M. Diaz, D. Westra, J. Tan enrolled in `[INSTANTLY_WARM_CAMPAIGN_ID]` with custom variables (icp_score, triage_reason, source, companyRef, industryRef). All 3 saved to Zevari `warm-leads-2026-05-26`.

J. Tan flagged in summary for manual AE review — pricing page visit at ICP 6 is the edge of the band.

## Step 7C — COLD routing

R. Singh enrolled in `[INSTANTLY_COLD_CAMPAIGN_ID]`. Saved to Zevari `cold-leads-2026-05-26`.

## Step 8 — queue closed out

All 6 rows updated with `processed_at`, `triage_class`, `icp_score`, `triage_reason`, `zevari_target_id`.

## Step 9 — Slack summary posted to `#inbound-triage`

```
✅ Inbound Triage Batch Complete

📅 Batch time: 2026-05-26 14:30 ET
📥 Leads in queue: 6
🔥 HOT: 1 — pinged in #sales-hot-leads
🌡️ WARM: 3 — enrolled in warm campaign
❄️ COLD: 1 — enrolled in cold campaign
⏭️ SKIPPED: 1 — K. Osei (existing customer at Graza)
⚠️ Unresolved profile: 1 — R. Singh (Apollo couldn't resolve LinkedIn — gmail address)
🔗 LinkedIn cap hit: no

HOT leads:
• Sarah Harlow — VP eCommerce @ Oatly (demo_request, ICP 9/10) — recent promo + 3 engagements with our content in last 14d

WARM leads:
• Maria Diaz — Head of Sales @ Thrilling (lead_magnet_download)
• David Westra — Director of Brand @ Magic Mind (newsletter_signup)
• Julia Tan — Sr. Director of Growth @ Swiggy (pricing_page) — flagged for manual AE review

COLD leads:
• Ravi Singh — unresolved profile, gmail (newsletter_signup)

Total run time: 71s
```

## Step 10 — reflection notes

- HOT-ping latency was 38s from queue read to Slack post. Under the 60s target.
- S. Harlow's stale form title (Director vs current VP) is exactly the failure mode this whole pipeline exists for. Apollo-cached enrichment would have triaged her one band lower.
- J. Tan at ICP 6 + pricing_page is a class-boundary case — RevOps should look at whether pricing-page visits should auto-bump to HOT regardless of score.
- R. Singh's gmail signup couldn't be resolved by Apollo. Consider tightening the newsletter form to require a work email if we want better cold-class fidelity.
- Total run time: 71s end-to-end. Well within the 15-min interval.
