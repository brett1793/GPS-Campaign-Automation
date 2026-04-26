# GPS Campaign Automation

Daily Facebook campaign post selection and queueing for [Georgia Prisoners' Speak](https://gps.press) — pulls from the GPS Campaign Message Library, applies cadence rotation across the three campaign strands (Vision 2027 / End the Warehouse / Behind the Walls), and queues a draft for admin review every day at 11:00 AM ET.

## Status

**Spec phase.** Implementation has not yet started.

See [`FUNCTIONAL_SPEC.md`](./FUNCTIONAL_SPEC.md) for the full specification.

## Architecture (one breath)

A new MCP tool (`GPS:fb_suggest_campaign_topics`) does the thinking — parses the library spec, computes which strand and archetype are next due based on 30-day history, runs pointer queries against Quote Bank / Mortality DB / WP article archive / TMS, applies submission redaction, composes the `fb_draft` call. A new WordPress plugin (`gps-campaign-automation`) does the orchestration — owns the daily WP-Cron, the `wp_gps_campaign_history` table, the admin UI, and the call-out to MCP.

## Publishing model

There is **no Facebook Graph API publishing path** in GPS today. The automation queues drafts as `pending_review`. The admin then:

1. Opens the draft in the social-scheduler queue
2. Downloads the rendered image
3. Copies the composed caption
4. Manually posts to Facebook
5. Calls `fb_mark_published(draft_id, fb_post_url)` — status becomes `published_manually`

The automation queues; the admin always posts. This matches the existing GPS Social Sharing Toolkit pattern.

## Sibling specs

- **Step 1 — Visual Identities:** `GPS-Campaign-Visual-Identities-Spec.md` (in [GPS Documentation](https://github.com/brett1793/GPS-Documentation))
- **Step 2 — Message Library:** `GPS-Campaign-Message-Library-Spec.md`
- **Step 3 — Automation:** This repo

## Implementation phases

8 phases, 11–16 days of build time. See `FUNCTIONAL_SPEC.md` §10.

Phase 1 starts with a `dry_run`-only MCP tool stub so the picks can be validated across 30 simulated days before any post is actually composed for real.

## Authorship

This is GPS infrastructure. The GDC Accountability Project, Inc. operates under strict anonymity protocols — see `~/Documentation/Specs/Rules.md` for details.
