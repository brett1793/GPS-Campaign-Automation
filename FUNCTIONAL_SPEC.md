# GPS Campaign Automation — Specification

> **Step 3 — Companion to** `GPS-Campaign-Visual-Identities-Spec.md` (Step 1) and `GPS-Campaign-Message-Library-Spec.md` (Step 2). Defines the automation that selects a campaign post each day, composes the `fb_draft` call, and queues it for admin review.

**Authored:** April 26, 2026
**Status:** Step 3 SPEC (implementation-ready)
**Siblings:** `GPS-Campaign-Visual-Identities-Spec.md` · `GPS-Campaign-Message-Library-Spec.md`

---

## 1. Purpose & posture

Steps 1 and 2 produced the *what*: the strand visual systems and the structured content library. Step 3 is the *how*: a daily-firing automation that picks the next post and queues it.

### What this automation does

Once per day at 11:00 AM ET, the system:

1. Reads the message library spec (parsed live from `~/Documentation/Specs/GPS-Campaign-Message-Library-Spec.md`)
2. Looks at the last ~30 days of campaign history to figure out which strand and archetype are next due
3. Picks an item — either canonical (stat / bill / photo / comparison) or via pointer query (Quote Bank / Mortality DB / WP article archive / TMS submissions / TMS stories)
4. Composes the `fb_draft` call with strand-specific styling, image prompt, and CTA URL
5. Records the post in `wp_gps_campaign_history` so cooldown enforcement and rotation tracking work
6. Queues the resulting draft as `pending_review` for the admin's manual posting workflow

The admin opens the social-scheduler queue, reviews the draft, downloads the rendered image, copies the caption, manually posts to Facebook, and then calls `fb_mark_published` (or clicks the "Mark Published" button in the admin UI which calls it). Every post passes through human review *and* human posting. There is no Facebook Graph API publishing path active in GPS today; even when one is added later, this automation will continue to land drafts in `pending_review` per the Step 3 decision — the only change would be the publish action moving from "manual download and post" to "click publish."

### What this automation does NOT do

- **Does not auto-publish.** All drafts land at `pending_review`. There is no Facebook Graph API publishing path active in GPS today — drafts queue for the existing manual workflow: admin reviews, downloads the image, copies the caption, posts to Facebook by hand, then calls `fb_mark_published` (status becomes `published_manually`). The automation will never bypass this manual path even if Graph API publishing is added in the future.
- **Does not generate Behind the Walls death notices automatically without consent verification.** Death notice items must have `verified=true AND consent_to_publish=true`. If no record matches, the BTW slot falls through to the counter pattern or family voice quote — never publishes a name without consent.
- **Does not fire on holidays or sensitive incident windows.** A simple skip-list is admin-controllable.
- **Does not generate AI faces for `personal_story` archetype posts.** Per the Message Library §4.6.1, the storyteller is named-by-pseudonym but never face-imagined. The automation enforces this by routing personal_story posts through a typographic-background image flow.
- **Does not create new library items.** The library is editor-curated. The automation consumes; it does not author.

### How it fits with existing GPS systems

| System | Relationship |
|---|---|
| **`fb_draft` MCP tool** | The automation's final action. It composes inputs (`topic`, `style`, `image_prompt`, `cta_link`, `source_type='ai_suggested'`, `source_ref={item_id}`) and calls `fb_draft`. No changes to `fb_draft` itself. |
| **`fb_mark_published` MCP tool** | The admin's final action. After manually posting to Facebook, admin calls `fb_mark_published(draft_id, fb_post_url)` to move the draft out of the queue and record the published URL. |
| **GPS Quote Bank** | Pointer queries call `search_quotes`. No changes to Quote Bank. |
| **GPS Mortality DB** | BTW death notices call `search_deaths` with `verified=true, consent_to_publish=true`. No changes to Mortality DB. |
| **GPS Social Sharing Toolkit** | Adjacent. The Toolkit handles per-article promotion in the first 7 days post-publish. This automation handles between-article cadence. The automation's news-tied items use a 7d cooldown to avoid stepping on the Toolkit. Both follow the same manual download-and-post pattern. |
| **Tell My Story (TMS)** | Pointer queries call `search_posts(category='community-stories')` for personal stories and `search_submissions(status='reviewed')` for raw narrative excerpts. No changes to TMS. |
| **Telegram bot** | The automation pings `send_telegram_message` on errors and on the daily completion summary. |

---

## 2. Architecture: split between MCP tool and WordPress plugin

The work splits cleanly into two components living in two places:

### 2.1 New MCP tool — `GPS:fb_suggest_campaign_topics`

Lives in the existing GPS MCP server. Stateless reads. Returns a structured suggestion.

**Responsibilities:**
- Parse the message library spec from disk
- Read recent campaign history from `wp_gps_campaign_history` (via WordPress REST or direct DB)
- Apply strand rotation, archetype rotation, item selection per Library §7
- Run pointer queries against Quote Bank / Mortality DB / WP article archive / TMS
- Apply submission redaction when source is a TMS submission
- Compose the `fb_draft` parameters
- Optionally invoke `fb_draft` directly (when called from cron) or return the composition for the caller to invoke (when called for dry-run)

**Why MCP and not pure WordPress:** Spec parsing, query orchestration across multiple data sources, and selection algorithms benefit from the MCP layer's tool ecosystem. The selection logic is also the part most likely to be tweaked over time, and iterating on Python in the MCP server is faster than WordPress plugin updates.

### 2.2 New WordPress plugin — `gps-campaign-automation`

Lives at `/var/www/wordpress/wp-content/plugins/gps-campaign-automation/` on the WordPress server.

**Responsibilities:**
- Own the `wp_gps_campaign_history` table (schema in §6)
- Provide the daily WP-Cron trigger (`gps_campaign_daily_post`)
- Provide the admin UI: campaign queue view, library browser, manual override controls
- Orchestrate the daily run: call MCP tool → receive suggestion → call `fb_draft` → record history
- Surface errors to admin Telegram + an admin notice in WordPress
- Provide a REST endpoint `/wp-json/gps/v1/campaign/recent` for the MCP tool to read history

**Why a plugin and not just MCP:** The cron schedule needs to fire reliably (WP-Cron is the established GPS pattern), the history table needs to be queryable from PHP for the admin UI, and the admin UI is a WordPress concern. Most importantly: every other GPS automation lives in a WP plugin. Consistency matters for maintenance.

### 2.3 Daily flow at a glance

```
   11:00 AM ET, every day
          │
          ▼
   WP-Cron fires `gps_campaign_daily_post`
          │
          ▼
   gps-campaign-automation plugin runs the orchestrator
          │
          ▼
   Plugin calls MCP tool fb_suggest_campaign_topics(mode="auto")
          │
          ▼
   MCP tool:
     ├── reads library spec from disk
     ├── reads wp_gps_campaign_history (last 30d)
     ├── computes strand → archetype → item
     ├── for pointer items: runs query (search_quotes, search_deaths, etc)
     ├── for personal_story: pulls article + extracts excerpt
     ├── for TMS submission: runs redaction pipeline
     └── returns composed fb_draft inputs + selection metadata
          │
          ▼
   Plugin calls GPS:fb_draft(topic, style, image_prompt, cta_link, ...)
          │
          ▼
   fb_draft creates draft with status=pending_review
          │
          ▼
   Plugin writes wp_gps_campaign_history row (item_id, draft_id, posted_at, ...)
          │
          ▼
   Plugin sends Telegram summary: "Today's queue: {strand}/{archetype} → {draft_url}"
          │
          ▼
   Admin opens social-scheduler queue, reviews draft
          │
          ▼
   Admin downloads rendered image + copies composed caption
          │
          ▼
   Admin posts to Facebook manually (FB web UI on phone or desktop)
          │
          ▼
   Admin calls fb_mark_published(draft_id, fb_post_url) — moves draft out of queue
          │
          ▼
   Plugin syncs status: wp_gps_campaign_history.status = 'published'
   (cooldown enforcement against this row continues normally)
```

---

## 3. MCP tool API: `GPS:fb_suggest_campaign_topics`

### 3.1 Signature

```python
def fb_suggest_campaign_topics(
    mode: str = "auto",
    force_strand: str = None,
    force_archetype: str = None,
    force_item_id: str = None,
    dry_run: bool = False,
    invoke_fb_draft: bool = True,
    schedule_at: str = None,
) -> dict:
    """
    Selects and composes a campaign post per the Message Library + Visual Identities specs.

    Args:
        mode: 'auto' (full automated selection) | 'force' (use force_* params)
        force_strand: 'v27' | 'etw' | 'btw' (overrides strand selection)
        force_archetype: e.g. 'stat_poster', 'bill_card', 'personal_story', 'documentary_photo'
        force_item_id: e.g. 'etw-stat-1772deaths' (overrides item selection)
        dry_run: If True, returns the composition without calling fb_draft. No history row written.
        invoke_fb_draft: If False (and dry_run=False), composes everything and writes history but
                        does not actually call fb_draft. Caller is expected to invoke separately.
        schedule_at: Optional ISO datetime. If set, passes through to fb_draft as scheduled_at.
                     Default behavior: drafts land in pending_review (unscheduled). Note that
                     Graph API auto-publishing is not active in GPS today, so scheduled_at has
                     no effect on actual publishing — admin always posts manually.
    """
```

### 3.2 Return shape

```json
{
  "ok": true,
  "selected": {
    "strand": "etw",
    "strand_weight_target": 0.45,
    "strand_weight_observed_30d": 0.42,
    "archetype": "stat_poster",
    "archetype_target_per_cycle": 3,
    "archetype_observed_last_10": 2,
    "item_id": "etw-stat-1772deaths",
    "item_source": "library_canonical",
    "pointer_result": null,
    "selection_path": "auto: strand_rotation -> archetype_rotation -> priority_weighted_random"
  },
  "composed": {
    "topic": "EtW stat poster: 1,772+ documented deaths in GDC custody since Jan 2020...",
    "style": "End the Warehouse / Mode A documentary photo, wine red #8B1A1A accent, Helvetica + JetBrains Mono",
    "image_prompt": "[Visual Identities §5 EtW prefix] stark empty concrete prison corridor in Southern US correctional facility...",
    "image_size": "square",
    "image_model": "flux-1.1-pro",
    "cta_link": "https://gps.press/submit-a-report/?utm_source=fb&utm_medium=poster&utm_campaign=etw_mortality",
    "source_type": "ai_suggested",
    "source_ref": "etw-stat-1772deaths"
  },
  "fb_draft": {
    "draft_id": 12345,
    "status": "pending_review",
    "admin_url": "https://gps.press/wp-admin/admin.php?page=fb-scheduler&draft=12345"
  },
  "history": {
    "history_id": 678,
    "wp_gps_campaign_history_row": "https://gps.press/wp-admin/admin.php?page=gps-campaign&history=678"
  },
  "diagnostics": {
    "library_items_total": 46,
    "library_items_in_cooldown": 4,
    "pointer_queries_run": [],
    "fallthroughs": [],
    "redaction_passes": []
  }
}
```

### 3.3 Error shape

```json
{
  "ok": false,
  "error_code": "ALL_STRANDS_EXHAUSTED",
  "error_message": "All three strands have only items currently in cooldown. Library may be too small or cooldown windows too aggressive.",
  "diagnostics": {
    "v27_postable": 0,
    "etw_postable": 0,
    "btw_postable": 0,
    "cooldown_floor_days": {"v27": 21, "etw": 21, "btw": 1}
  }
}
```

Error codes:

| Code | Meaning |
|---|---|
| `LIBRARY_PARSE_ERROR` | Spec file unreadable or malformed YAML in an item block |
| `ALL_STRANDS_EXHAUSTED` | No strand has a postable item; cooldowns or library too small |
| `POINTER_QUERY_FAILED` | An MCP query returned an error and no fallthrough succeeded |
| `IMAGE_GENERATION_FAILED` | Flux/OpenAI image gen failed; draft created without image, admin alerted |
| `FB_DRAFT_FAILED` | Final `fb_draft` call failed; history row not written, admin alerted |
| `INVALID_OVERRIDE` | `force_*` params reference a non-existent strand/archetype/item |
| `SUBMISSION_REDACTION_BLOCKED` | Redaction pipeline flagged a submission as un-publishable; selection falls through |

---

## 4. Selection algorithm

### 4.1 Strand rotation (Library §7.1)

The target is 45/45/10 between V2027 / EtW / BTW. The automation tracks the last 30 days of history and picks the strand with the largest negative gap (most under-represented relative to target).

**Algorithm:**

```
inputs:
    target_weights = {v27: 0.45, etw: 0.45, btw: 0.10}
    legislative_session_active = bool (computed from date — Phase 3 = Nov-Jan, Phase 4 = Jan-Mar)
    if legislative_session_active:
        target_weights[v27] = 0.60
        target_weights[etw] = 0.30
        target_weights[btw] = 0.10

    history = wp_gps_campaign_history rows from last 30 days, status IN ('pending_review','scheduled','published','published_manually')
    counts = group by strand
    total = sum(counts.values()) or 1
    observed_weights = {strand: counts.get(strand, 0) / total for strand in target_weights}

algorithm:
    gaps = {strand: target_weights[strand] - observed_weights[strand] for strand in target_weights}
    sorted_strands = sorted(gaps.items(), key=lambda x: -x[1])  # most under-represented first

    for strand, gap in sorted_strands:
        if can_produce_postable_item(strand):
            return strand

    raise ALL_STRANDS_EXHAUSTED
```

**`can_produce_postable_item(strand)`** is true if at least one item of any archetype in that strand passes cooldown filtering (see §4.4) AND, for pointer items, returns at least one fresh result.

### 4.2 Archetype rotation within strand (Library §7.2)

Once a strand is selected, the automation picks an archetype using a similar gap-based approach. Targets are per-strand, expressed as a count per 10-post cycle:

```
v27_targets = {stat_poster: 4, bill_card: 2, quote: 2, news_tied: 1, side_by_side: 1}
etw_targets = {stat_poster: 3, documentary_photo: 2, quote: 1, tms_submission: 1, personal_story: 1, comparison: 1, news_tied: 1}
btw_targets = {death_notice: 0.5, counter: 0.3, family_voice: 0.2}  # BTW totals are smaller because the strand only fires ~10% of slots
```

The automation looks at the last 10 posts *within the selected strand* and computes which archetype is most behind target.

**Special slots:**
- `side_by_side` (V2027 only): triggers on Mondays. If `auto` mode and current day is Monday and the side_by_side slot hasn't fired this week, it takes priority over the regular archetype rotation.
- `incident_report` (BTW only): never auto-fires. Only triggered manually by editor (`force_archetype='incident_report'`).
- `personal_story` (EtW only): minimum once per 10-post cycle when fresh stories exist. The rotation logic boosts personal_story sampling weight by 1.5× if no personal_story has fired in the last 10 posts and at least one fresh TMS story exists.

### 4.3 Item selection within archetype

For an archetype slot, the automation builds the candidate pool:

**Library-canonical archetypes** (`stat_poster`, `bill_card`, `documentary_photo`, `comparison`):
- Pull all items in the library matching `strand` and `archetype`
- Filter: exclude items where `last_used` is within the cooldown window (per Library §7.3)
- Weighted random by priority: high=3, medium=2, low=1

**Pointer archetypes** (`quote`, `news_tied`, `personal_story`, `tms_submission`, `death_notice`, `counter`, `family_voice`):
- Pull the matching pointer item from the library (e.g. `etw-quote-query-conditions`)
- Run its embedded query against the appropriate MCP tool
- Filter results: exclude any whose IDs are within per-result cooldown (per-quote-id, per-post-id, per-submission-id, per-death-id)
- Filter results: apply freshness window (e.g. quote freshness 180d, news_tied 21d)
- Weighted random over remaining results — use result `priority` if present (Quote Bank entries can carry priority via tags), else uniform random

If the candidate pool is empty after filtering:
1. Try the next-priority archetype in the rotation
2. If all archetypes in the strand exhaust, fall through to the next strand (return to §4.1 with current strand removed from candidates)

### 4.4 Cooldown enforcement

Cooldowns are enforced by checking `wp_gps_campaign_history` for matching previous posts.

**For library-canonical items** (cooldown is per-item-id):
```sql
SELECT MAX(posted_at) FROM wp_gps_campaign_history
WHERE item_id = ? AND status NOT IN ('failed', 'rejected')
```
If `MAX(posted_at) > NOW() - INTERVAL {cooldown_days} DAY`, item is in cooldown.

**For pointer items** (cooldown is per-result-id):
```sql
SELECT MAX(posted_at) FROM wp_gps_campaign_history
WHERE pointer_result_id = ? AND pointer_result_type = ? AND status NOT IN ('failed', 'rejected')
```
Where `pointer_result_type` is `quote_id`, `wp_post_id`, `submission_id`, or `death_record_id`.

**Default cooldowns** (override in library item):
- Stat posters, bill cards: 21d
- Photo concepts: 14d
- Quote-query results: 10–21d (per-quote-id)
- News-tied: 7d (per-post-id)
- Personal story (TMS): 21d (per-story-id)
- Submission narrative (TMS): 14d (per-submission-id)
- Death notice: 1d cooldown for the strand (so it can fire again the next day if a new death is documented)
- Counter pattern: 14d

### 4.5 Side-by-side pairing (V2027 weekly slot)

Per BT's Step 3 answer: automation pairs by topic-tag using a **topic-tag affinity table** (because the strands deliberately own different topics, literal tag overlap is sparse).

**Affinity table** — for each EtW topic-tag, V2027 topic-tags that thematically pair with it:

| EtW tag | V2027 tags it pairs with | Narrative bridge |
|---|---|---|
| `mortality` | `wrongful-conviction`, `judicial`, `iac` | "Innocent people are dying inside" |
| `overcrowding` | `wrongful-conviction`, `habeas` | "We're warehousing people we should be releasing" |
| `geriatric` | `habeas`, `wrongful-conviction`, `bills` | "Elderly people held under defective convictions" |
| `parole` | `habeas`, `iac`, `bills` | "The release mechanisms exist on paper" |
| `lifers` | `habeas`, `iac`, `wrongful-conviction` | "The longest-held are most likely to be wrongfully convicted" |
| `staffing` | `voter-support`, `legislative` | "The system can't function — voters know it" |
| `rehabilitation` | `wrongful-conviction`, `fiscal` | "We spend $1.8B on warehousing, $52 on rehabilitation, and 5% are innocent" |
| `medical-neglect` | `wrongful-conviction`, `judicial` | "People dying in custody who shouldn't have been there" |
| `violence` | `wrongful-conviction`, `judicial` | "The most vulnerable to prison violence are often innocent" |
| `gdc-response` | `prosecutor-accountability`, `accountability` | "Same denial pattern from prosecutors and from corrections" |
| `commissary` | `voter-support`, `fiscal` | "The system extracts from the people it incarcerates" |
| `solitary` | `wrongful-conviction`, `iac` | "Coercion that produces false convictions starts with isolation" |

**Pairing algorithm:**

```
1. Pick the EtW item:
   - High-priority EtW item not in cooldown
   - Prefer items whose first tag is in the affinity table (to maximize pairing options)

2. Pick the V2027 item:
   - Look up affinity_tags = affinity_table[etw_item.tags[0]]
   - Pull V2027 items whose tags intersect affinity_tags
   - Filter by cooldown
   - Weighted random by priority

3. Compose the side-by-side post:
   - Topic: "[EtW item focal] is the cost. [V2027 item focal/bill] is the fix."
   - Style: V2027 (cream/navy/gold) — V2027 anchors all side-by-side because the CTA is action-oriented
   - Image prompt: V2027 split-pane style from Visual Identities §6
   - CTA: V2027 item's CTA (legislative action — that's the point of pairing)
```

If no V2027 item shares affinity tags with the chosen EtW item, fall through to a regular V2027 stat or bill card. The automation logs the failed pairing attempt in `diagnostics`.

### 4.6 Personal story rendering rules (Message Library §4.6.1)

The `personal_story` archetype has stricter rules than other photo-led archetypes. The automation enforces these mechanically:

**Image flow:**
- **Never** call `generate_image` with a prompt that could produce a face
- The image is composed from one of N pre-built typographic backgrounds stored in WP media library
- Background pool: chainlink shadow, cellblock void, paper texture, document close-up, blurred razor-wire, abstract concrete grain
- The plugin overlays story title + pseudonym + excerpt as typography
- Composition happens in the plugin, not in the image generator

**Excerpt selection:**
- Pull the article via `get_post(post_id)`
- Try the article's Yoast meta description first (it's often the strongest 1-sentence hook)
- If meta description is missing or weak (under 12 words), fall back to first paragraph of `post_content` stripped of HTML
- Trim to 18–28 words, breaking at the nearest sentence boundary
- If trimming would lose the closing punctuation, add an em-dash continuation: "...the moment everything changed —"

**Pseudonym handling:**
- Pull author from the WordPress post — it's the pseudonym storyteller account
- Render exactly as: `— {pseudonym}` in the attribution line
- Render exactly as: `Read {pseudonym}'s story` in the CTA verb
- Never extend ("By Anonymous Storyteller from Macon"), never modify

**CTA URL:**
- The article's permalink, with UTM appended: `?utm_source=fb&utm_medium=poster&utm_campaign=etw_tms`
- Never the `tellmystory.gps.press` landing page

### 4.7 Submission redaction pipeline (Message Library §4.6.2)

For `etw-tms-submission-narrative` items, the raw submission narrative *must not* reach `fb_draft` without redaction.

**Two-pass approach:**

**Pass 1 — Rules-based:**
```python
def rules_redact(narrative_text):
    text = narrative_text

    # Strip dates (multiple formats)
    text = re.sub(r'\b\d{4}-\d{2}-\d{2}\b', '[DATE]', text)              # 2026-04-15
    text = re.sub(r'\b\d{1,2}/\d{1,2}/\d{2,4}\b', '[DATE]', text)        # 4/15/26
    text = re.sub(r'\b(January|February|...) \d{1,2},?\s*\d{0,4}\b',
                  '[DATE]', text, flags=re.IGNORECASE)                    # April 15, 2026

    # Strip facility-specific names (replace with category)
    facilities = list_facilities(active_only=True)
    for f in facilities:
        # Replace exact matches and short_names
        for variant in [f.name, f.short_name]:
            if variant and variant.lower() in text.lower():
                text = re.sub(re.escape(variant), '[FACILITY]', text, flags=re.IGNORECASE)

    # Strip person names matching common patterns
    # Conservative: only redact obvious "Officer Lastname", "Inmate Firstname Lastname",
    # "Sgt. Lastname", "Warden Lastname" patterns. Don't try to identify all proper nouns —
    # too many false positives.
    text = re.sub(r'\b(Officer|Sgt\.?|Lt\.?|Capt\.?|Warden|Deputy|CO|Inmate|Resident)\s+[A-Z][a-z]+(\s+[A-Z][a-z]+)?\b',
                  r'[\1 REDACTED]', text)

    # Strip GDC ID numbers
    text = re.sub(r'\bGDC[\s#:]*\d{6,}\b', '[GDC ID]', text, flags=re.IGNORECASE)

    return text
```

**Pass 2 — LLM verification (Dify flow `submission_redaction_check`):**

System prompt:
```
You are a privacy auditor for a journalism organization. The text below has been
through a regex-based redaction pass. Your job is to identify ANY remaining
personally-identifying information that the regex missed.

Look for:
- First or last names that the regex didn't catch
- Specific dates that slipped through (e.g. "last Tuesday")
- Facility names or specific locations within facilities
- Cell numbers, dorm names, housing unit identifiers
- Any detail that combined with public info could identify a specific person

Return JSON:
{
  "safe": true/false,
  "remaining_pii": [{"text": "...", "category": "name|date|location|other", "suggested_replacement": "..."}],
  "redacted_text": "..."
}

If safe is false, redacted_text should contain the further-redacted version.
```

If `safe=false`, the automation uses `redacted_text`. If even after this the LLM is still flagging issues (configurable threshold — default 3 remaining items), the automation flags the item as `SUBMISSION_REDACTION_BLOCKED` and falls through to the next archetype.

**Excerpt extraction (after redaction):**
- The redacted text is often longer than the 12–22 word target
- A second Dify flow (`submission_excerpt_pull`) pulls the strongest 12–22 word excerpt that:
  - Reads as a coherent standalone line
  - Doesn't end mid-clause
  - Carries emotional or factual weight, not just generic conditions
  - Doesn't reintroduce any redacted token (`[REDACTED]`, `[FACILITY]`, etc.)

**Redaction logging:**
Every redaction pass writes to `wp_gps_campaign_redaction_log`:
- Original text length
- Redacted text length
- Number of rules-pass redactions
- Number of LLM-pass additional redactions
- Final safe flag
- Submission ID

Admin can review the log to spot redaction quality issues.

---

## 5. fb_draft composition

After selection, the MCP tool composes the four key inputs to `fb_draft`:

### 5.1 Topic

Plain-English description that the `fb_draft` writer (Sonnet 4.6 via fb_post_writer AI class) rewrites for Facebook.

**Composition pattern by archetype:**

```
stat_poster:
  topic = "{strand} stat poster: {focal}{unit} — {subhead}. Cite {source.citation}.
           Frame for a Facebook audience as a single sharp number with a one-line
           interpretation. CTA: {cta.verb}."

bill_card:
  topic = "{strand} bill card for {eyebrow}: {body} {subhead}. Three pillars:
           [{pillars}]. Tone: legislative-direct, no marketing. CTA: {cta.verb}."

documentary_photo:
  topic = "{strand} documentary post: {caption_pattern.line}. Source: {caption_pattern.source_line}.
           Pair the image (see image_prompt) with a caption that lets the photo carry
           the emotional weight. Two short paragraphs max."

comparison:
  topic = "{strand} comparison: {eyebrow}. Highlight row: {highlight_row}. Frame for
           a Facebook reader who hasn't seen the dataset before. CTA: {cta.verb}."

quote (Quote Bank):
  topic = "{strand} quote post. Quote: \"{quote_text}\". Attribution: {attribution}
           ({attribution_type}). Context: {context}. Frame as testimony — let the
           quote lead, add a single sentence of context."

personal_story (TMS):
  topic = "{strand} personal story post. Featuring '{post_title}' by {pseudonym}.
           Excerpt: \"{excerpt}\". CTA verb: 'Read {pseudonym}'s story'.
           Tone: respectful — this is someone's lived experience. Frame the excerpt
           as the hook and direct readers to the full story. NO image prompt — image
           is composed from typographic background pool."

tms_submission_narrative:
  topic = "{strand} testimony post. Source close to a Georgia state prison submitted
           the following: \"{redacted_excerpt}\". Frame as anonymous testimony.
           Single sentence of context after the quote. CTA: Submit a tip."

death_notice:
  topic = "{strand} death notice. Headline: 'Another death GDC didn't tell us about.'
           Document: facility=[{facility_name|withheld}], date=[{date_of_death}],
           verification: GPS staff cross-checked. Body: GPS has documented {total_count}
           deaths since Jan 2020 — names withheld until family notification + consent.
           Tone: factual, mournful, accusatory toward GDC opacity."

side_by_side:
  topic = "{V2027 strand} side-by-side post. EtW problem: {etw_item.focal} {etw_item.subhead}.
           V2027 fix: {v27_item.eyebrow} — {v27_item.body}. Frame as a 2-panel
           problem→solution narrative. CTA: {v27_item.cta.verb}."
```

### 5.2 Style

Pulls from Visual Identities §3-4 and passes through to `fb_draft.style`. Each strand has a fixed style string the automation uses verbatim:

```python
STYLE_BY_STRAND = {
    "v27": "Vision 2027 visual system: cream #F5F1EB background, navy #0D1B2A primary, "
           "burnished gold #D4A843 accent. Georgia serif for headlines, Helvetica for body. "
           "Tone: legislative, hopeful-but-grounded, action-oriented. No marketing language.",

    "etw_mode_a": "End the Warehouse Mode A: documentary B&W photograph as full-bleed background, "
                  "wine red #8B1A1A overlay accents, white serif eyebrow, large white sans-serif focal. "
                  "Tone: documentary, somber, evidentiary.",

    "etw_mode_b": "End the Warehouse Mode B: off-white paper #fafaf8 background, wine red #8B1A1A primary, "
                  "JetBrains Mono for technical/document framing, Helvetica for narrative. Horizontal "
                  "ledger rules every 48px, diagonal stamp accents. Tone: open-records, document-evidence, "
                  "ledger-style.",

    "btw": "Behind the Walls: dark #0a0a0f background, alert red #dc2626 accents, Source Serif 4 headlines, "
           "Inter body. Tone: breaking-news, factual, identity-protective. Reserved for death notices, "
           "counter milestones, and incident reports."
}
```

The automation picks `etw_mode_a` vs `etw_mode_b` per archetype:
- `documentary_photo`, `personal_story` → Mode A
- `stat_poster`, `comparison`, `tms_submission_narrative`, `quote` → Mode B
- `news_tied` → Mode A if the article has a strong featured image, Mode B otherwise

### 5.3 Image prompt

For library-canonical photo concepts: prepend the strand prefix from Visual Identities §5, then the item's `image_prompt`.

```python
def compose_image_prompt(item, strand):
    if strand == "etw":
        prefix = "Documentary B&W photograph in the style of Sebastiao Salgado, " \
                 "35mm film grain, deep chiaroscuro, no people, journalistic aesthetic. "
    elif strand == "v27":
        prefix = "Editorial illustration in the style of vintage legislative graphics, " \
                 "muted cream and navy palette with burnished gold accents, clean composition. "
    elif strand == "btw":
        prefix = "Documentary close-up, dark monochrome with single red accent, " \
                 "newsroom photo aesthetic, no people. "
    return prefix + item.image_prompt
```

For `personal_story`: skip image generation. Plugin uses pre-rendered typographic backgrounds.

For `tms_submission_narrative`, `quote`, `bill_card`, `comparison`, `stat_poster`: skip Flux generation entirely. The plugin renders these in HTML/CSS using the templates from Visual Identities §6 and screenshots them server-side via headless Chromium (existing GPS pattern from the violence reports plugin). Faster, cheaper, deterministic.

For `news_tied`: reuse the article's existing featured image (`get_post(post_id).featured_image_url`). Pass to `fb_draft` as `image_prompt=""` and `generate_image=False`. This matches the Social Sharing Toolkit pattern.

### 5.4 CTA link

Look up the item's `cta.url_id` in the Library §6 CTA Registry. Substitute `{strand}` and any other runtime tokens. For `read-tms-story`, substitute `{post_url}` with the actual article permalink.

### 5.5 Other fb_draft parameters

```python
fb_draft_params = {
    "topic": composed_topic,
    "style": STYLE_BY_STRAND[strand_style_key],
    "cta_link": resolved_cta_url,
    "generate_image": should_generate_image,  # False for stat_poster, news_tied, personal_story
    "image_prompt": composed_image_prompt,    # Empty when generate_image=False
    "image_size": "square" if archetype in ("stat_poster", "documentary_photo", "personal_story")
                  else "landscape",
    "image_model": "flux-1.1-pro",  # Default; admin can override per-item via library notes
    "scheduled_at": "",  # Always empty. Drafts always land in pending_review per BT decision.
                         # Note: Graph API publishing isn't active in GPS today, so even setting
                         # scheduled_at would not produce auto-publishing — admin always posts manually
                         # via the existing fb_mark_published flow.
    "source_type": "ai_suggested",
    "source_ref": item_id  # e.g. "etw-stat-1772deaths" or "etw-quote-q-conditions:quote_id_3440"
}
```

---

## 6. WordPress plugin: `gps-campaign-automation`

### 6.1 Plugin layout

```
/var/www/wordpress/wp-content/plugins/gps-campaign-automation/
├── gps-campaign-automation.php          # Main plugin file, hooks, constants
├── includes/
│   ├── class-database.php                # Schema, CRUD for history table
│   ├── class-orchestrator.php            # Daily run orchestration (calls MCP)
│   ├── class-mcp-client.php              # Wrapper for fb_suggest_campaign_topics + fb_draft
│   ├── class-pairing-engine.php          # Side-by-side pairing logic (mirrors MCP for admin preview)
│   ├── class-personal-story-renderer.php # Typographic-background composition for TMS personal stories
│   ├── class-image-cache.php             # Caches generated photo concept images
│   ├── class-rest-api.php                # /wp-json/gps/v1/campaign/* endpoints
│   ├── class-admin.php                   # Admin pages
│   └── class-cron.php                    # WP-Cron registration + handlers
├── admin/
│   ├── views/
│   │   ├── dashboard.php                 # Recent posts, queue, gap analysis
│   │   ├── library-browser.php           # Browse the message library with last_used / cooldown status
│   │   ├── manual-trigger.php            # Force a strand/archetype/item for testing
│   │   ├── redaction-log.php             # Review submission redaction passes
│   │   └── settings.php                  # Cron time, kill switch, skip-list
│   ├── css/admin.css
│   └── js/admin.js
├── assets/
│   └── personal-story-backgrounds/       # 6 pre-rendered typographic backgrounds (1080x1080 PNG)
└── readme.txt
```

### 6.2 Database schema

#### `wp_gps_campaign_history` — what's been posted

| Column | Type | Description |
|---|---|---|
| `id` | BIGINT UNSIGNED AUTO_INCREMENT | Primary key |
| `posted_at` | DATETIME | When the run executed (NOT when FB published — that's on the draft) |
| `run_mode` | ENUM('auto','manual','dry_run') | How the run was triggered |
| `strand` | ENUM('v27','etw','btw') | Strand selected |
| `archetype` | VARCHAR(50) | Archetype selected (e.g. `stat_poster`, `personal_story`) |
| `item_id` | VARCHAR(100) | Library item id (e.g. `etw-stat-1772deaths`); for pointer items, the pointer query id (e.g. `etw-quote-query-conditions`) |
| `pointer_result_type` | VARCHAR(30) NULL | `quote_id` · `wp_post_id` · `submission_id` · `death_record_id` (NULL for canonical items) |
| `pointer_result_id` | BIGINT UNSIGNED NULL | The specific result returned (used for cooldown enforcement) |
| `is_side_by_side` | BOOLEAN | True if this was the V2027 weekly side-by-side slot |
| `paired_etw_item_id` | VARCHAR(100) NULL | If side_by_side, the EtW item it was paired with |
| `fb_draft_id` | BIGINT UNSIGNED NULL | The resulting fb_draft record; NULL if dry_run or fb_draft failed |
| `status` | ENUM('queued','published','failed','rejected','dry_run') | High-level outcome — synced from fb_draft on admin actions via hook |
| `composed_topic` | TEXT | The topic string passed to fb_draft (for audit) |
| `composed_image_prompt` | TEXT | The image prompt passed to fb_draft (for audit) |
| `cta_link_resolved` | VARCHAR(500) | The final CTA URL with UTM |
| `error_code` | VARCHAR(50) NULL | If status='failed', the error code |
| `error_message` | TEXT NULL | If status='failed', the message |
| `created_at` | DATETIME | Insert time |

**Indexes:**
- PRIMARY (id)
- KEY (posted_at)
- KEY (strand, archetype, posted_at)
- KEY (item_id, posted_at)
- KEY (pointer_result_type, pointer_result_id, posted_at)
- KEY (fb_draft_id)
- KEY (status)

#### `wp_gps_campaign_redaction_log` — TMS submission redaction audit

| Column | Type | Description |
|---|---|---|
| `id` | BIGINT UNSIGNED AUTO_INCREMENT | Primary key |
| `history_id` | BIGINT UNSIGNED NULL | Linked history row (NULL if redaction was attempted but item was rejected) |
| `submission_id` | BIGINT UNSIGNED | The submission run through redaction |
| `original_length` | INT | Character count of input |
| `redacted_length` | INT | Character count of final output |
| `rules_pass_count` | INT | Number of rules-based redactions performed |
| `llm_pass_count` | INT | Number of LLM-detected additional redactions |
| `safe_flag` | BOOLEAN | LLM verifier final safe flag |
| `final_excerpt` | TEXT NULL | The 12–22 word excerpt that ended up in the post (NULL if blocked) |
| `blocked` | BOOLEAN | True if redaction failed and item was rejected |
| `created_at` | DATETIME | When run |

#### `wp_gps_campaign_image_cache` — for photo concept caching

| Column | Type | Description |
|---|---|---|
| `id` | BIGINT UNSIGNED AUTO_INCREMENT | Primary key |
| `cache_key` | VARCHAR(64) UNIQUE | SHA256 of `{item_id}|{image_prompt}|{strand_prefix}` |
| `wp_media_id` | BIGINT UNSIGNED | WordPress media library ID |
| `wp_media_url` | VARCHAR(500) | Direct URL |
| `item_id` | VARCHAR(100) | Library item that generated it |
| `created_at` | DATETIME | When cached |
| `last_reused_at` | DATETIME NULL | When last reused |
| `reuse_count` | INT DEFAULT 0 | Times reused |

When the orchestrator needs an image for a photo concept, it checks the cache by key first. If hit and the cached image is < 90 days old, reuses the WP media URL. If miss or stale, generates fresh, uploads, caches.

This is the answer to BT's image budget concern from §11 Q3 — at ~$0.04/image and ~25% of slots being photo archetypes, caching the corridor / cellblock / fence / paperwork backgrounds saves real money. Most posts reuse the same 6 backgrounds.

### 6.3 Cron registration

```php
register_activation_hook(__FILE__, 'gps_campaign_activate');

function gps_campaign_activate() {
    if (!wp_next_scheduled('gps_campaign_daily_post')) {
        // Schedule for 11:00 AM ET — convert to UTC for WP cron
        $tz = new DateTimeZone('America/New_York');
        $scheduled = new DateTime('today 11:00 AM', $tz);
        if ($scheduled < new DateTime('now', $tz)) {
            $scheduled->modify('+1 day');
        }
        wp_schedule_event($scheduled->getTimestamp(), 'daily', 'gps_campaign_daily_post');
    }
}

add_action('gps_campaign_daily_post', ['GPS_Campaign_Orchestrator', 'run_daily']);
```

11:00 AM ET picks up the late-morning Facebook engagement window. Configurable in admin settings.

### 6.4 Admin pages

Menu structure under "GPS Campaign" (top-level WordPress admin menu):

```
GPS Campaign (dashicons-megaphone)
├── Dashboard         — Recent posts, gap analysis, next-fire countdown
├── Library Browser   — All 46 library items with last_used / in_cooldown status
├── Manual Trigger    — Force a strand/archetype/item right now
├── Redaction Log     — TMS submission redaction audit
└── Settings          — Cron time, kill switch, skip-list
```

#### Dashboard

```
┌──────────────────────────────────────────────────────────┐
│  Today's Post                                            │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Strand: EtW · Archetype: stat_poster              │    │
│  │ Item: etw-stat-1772deaths                         │    │
│  │ Status: pending_review · awaiting manual post     │    │
│  │ [Open in social-scheduler queue]                  │    │
│  │   → download image, copy caption, post to FB,    │    │
│  │     then click "Mark Published"                  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Strand distribution (last 30 days)                     │
│  V2027:  ████████████████░░░░░░░░  44% (target 45%)    │
│  EtW:    █████████████████░░░░░░░  47% (target 45%)    │
│  BTW:    ███░░░░░░░░░░░░░░░░░░░░░   9% (target 10%)    │
│                                                          │
│  Library health                                          │
│  Items in cooldown:    4 of 46                          │
│  Items never used:    12 of 46  ← prioritize these     │
│  Pointer queries returning empty: 0                     │
│                                                          │
│  Recent runs (last 7)                                    │
│  Apr 26 · EtW/personal_story    · Lila's story  · Posted│
│  Apr 25 · V2027/bill_card       · Bill B        · Posted│
│  Apr 24 · BTW/death_notice      · [withheld]    · Queued│
│  Apr 23 · EtW/stat_poster       · 1,772+ deaths · Posted│
│  Apr 22 · V2027/stat_poster     · 7 of 9 justices · Posted│
│  Apr 21 · EtW/comparison        · Rehab spending · Posted│
│  Apr 20 · V2027/side_by_side    · Geriatric+Bill A · Posted│
│                                                          │
│  Pending manual post: 2 drafts older than 24h           │
└──────────────────────────────────────────────────────────┘
```

#### Library Browser

For each of the 46 items, show: id, strand, archetype, priority, last_used_date, in_cooldown_until, total_uses, [Force Now] button.

Sortable by last_used (oldest first highlights stale items the editor might want to refresh).

#### Manual Trigger

Three sub-forms:

1. **Force a strand** — picks archetype + item automatically within that strand
2. **Force an item** — runs the full composition for a specific item id
3. **Dry run** — composes everything but doesn't call fb_draft, shows the proposed topic/image_prompt/CTA for inspection

#### Settings

```
Cron schedule
  Daily run time:  [11:00 AM] [America/New_York]
  Kill switch:     [ ] Pause automation entirely

Skip dates (one per line, ISO format)
  ┌─────────────────────────┐
  │ 2026-12-25              │   # Christmas
  │ 2026-07-04              │   # Independence Day
  │ 2026-11-26              │   # Thanksgiving
  └─────────────────────────┘

Cooldown overrides (advanced)
  Stat poster cooldown:        [21] days (default)
  Photo concept cooldown:      [14] days (default)
  Personal story cooldown:     [21] days (default)
  Submission narrative cooldown: [14] days (default)

Pause individual strands
  [ ] Pause V2027
  [ ] Pause EtW
  [ ] Pause BTW

Telegram notifications
  [x] Send daily summary at 11:05 AM ET
  [x] Alert on errors
  [ ] Alert on every successful queue
```

### 6.5 Status sync from fb_draft

When the admin acts on a draft in the social-scheduler queue (publishes, schedules, rejects, deletes), an existing hook in the social-scheduler plugin already fires (it's how published-tracking works today). The campaign-automation plugin listens for this hook and syncs the status to `wp_gps_campaign_history.status`:

```php
add_action('gps_fb_draft_status_changed', function($draft_id, $new_status) {
    GPS_Campaign_History::sync_status_from_draft($draft_id, $new_status);
});
```

Status mapping:
- `pending_review` → `queued`
- `scheduled` → `queued`
- `published` → `published`
- `published_manually` → `published`
- `failed` → `failed`
- (admin deletes draft) → `rejected`

**The typical lifecycle today** is `pending_review` → (admin downloads image, copies caption, posts to FB by hand, calls `fb_mark_published`) → `published_manually`. The `scheduled` and `published` statuses exist for a future Graph API integration but are not the primary path. Cooldown logic treats `published`, `published_manually`, `scheduled`, and `pending_review` identically: the post counts as queued and the underlying item enters cooldown. Only `failed` and `rejected` exempt items from cooldown.

Why this matters: the cooldown logic only enforces against `status NOT IN ('failed', 'rejected')`. If the admin rejects a draft, the underlying library item should be available for selection again immediately rather than sitting in a 21-day cooldown.

---

## 7. Failure modes and graceful degradation

### 7.1 Library parse error

If the spec file fails to parse (malformed YAML, missing required fields):
- Abort the run
- Telegram-alert admin with the parse error and line number
- No history row written
- Next day's cron will retry

### 7.2 All strands exhausted

If every strand has zero postable items (everything in cooldown OR pointer queries return empty):
- Abort the run
- Telegram-alert: "Campaign automation: no postable items today. Library may need expansion."
- Write a `dry_run` history row for diagnostics
- Next day's cron retries — most cooldowns will tick down a day

### 7.3 Pointer query failure

If a pointer query (e.g. `search_quotes`) errors out:
- Try the next archetype within the strand
- If all archetypes fail, fall through to the next strand
- Log the failed query in `diagnostics.pointer_queries_run`

### 7.4 Image generation failure

If Flux/OpenAI image gen fails:
- Create the draft anyway, with `image_prompt` populated but `image_url=null`
- The draft lands in `pending_review` with a flag — admin can manually upload an image or re-trigger generation
- Telegram-alert admin
- History row recorded with `status='queued'`

### 7.5 fb_draft fails

If the final `fb_draft` call throws:
- No draft created — nothing to review
- History row recorded with `status='failed'` and the error code/message
- Telegram-alert admin
- Item is NOT counted toward cooldown (since the post didn't actually get queued)

### 7.6 Submission redaction blocked

If the LLM verifier flags 3+ remaining PII items after the rules pass:
- Item rejected
- Redaction log row written with `blocked=true`
- Selection falls through to next archetype within the strand
- No Telegram alert (this is expected behavior, not an error) — admin reviews redaction log periodically

### 7.7 WP-Cron didn't fire

WP-Cron requires site traffic to fire. The site is live and gets traffic, but if there's a 24h+ gap with no firing:
- A separate `crontab` system cron at 11:30 AM ET checks `wp_gps_campaign_history.posted_at MAX()`. If > 25 hours old, it triggers `wp_cron` directly via WP-CLI.
- This is a belt-and-suspenders pattern already used by other GPS plugins (mortality fetcher, news aggregator).

### 7.8 Kill switch

The `Settings → Pause automation entirely` checkbox sets a single option `gps_campaign_paused = true`. The orchestrator checks this first thing on every run and exits immediately if true. No history row written. Useful for incident windows where any campaign post would feel tone-deaf.

---

## 8. Override and manual modes

### 8.1 Manual trigger from admin UI

The admin can fire the orchestrator on demand from the Manual Trigger page. Three forms:

1. **Force strand** — pick V2027 / EtW / BTW, automation handles the rest
2. **Force item** — pick a specific library item id
3. **Dry run** — picks something but doesn't actually call `fb_draft`

All manual runs are recorded in history with `run_mode='manual'` (or `dry_run`).

### 8.2 MCP tool from Termius

BT can also fire the MCP tool directly:

```
GPS:fb_suggest_campaign_topics                                        # auto
GPS:fb_suggest_campaign_topics force_strand=v27                       # force V2027
GPS:fb_suggest_campaign_topics force_item_id=etw-stat-1772deaths       # force item
GPS:fb_suggest_campaign_topics dry_run=true                            # dry run
GPS:fb_suggest_campaign_topics force_archetype=personal_story dry_run=true  # test TMS flow
```

The MCP tool calls into the WordPress plugin's REST endpoint to write the history row. So Termius and the cron path go through the same write path.

### 8.3 Skip-day list

Settings → Skip dates is a list of ISO dates. On those days, the cron fires but the orchestrator returns immediately without selecting an item. Useful for holidays where any post feels off.

---

## 9. Observability

### 9.1 Telegram daily summary

11:05 AM ET (5 min after the cron fires), the plugin sends:

```
GPS Campaign — Apr 26 2026

Today's draft:
  EtW / personal_story
  Item: etw-tms-story-tied (Lila's story)
  Status: pending_review · awaiting manual post
  Open: https://gps.press/wp-admin/admin.php?page=fb-scheduler&draft=12345
  → Download image, copy caption, post to FB, then mark published

Last 7 days mix: 3 V2027, 3 EtW, 1 BTW (target 3.15/3.15/0.7)
Library health: 4 items in cooldown, 12 never used.
Pending manual posting: 2 drafts older than 24 hours
```

### 9.2 Telegram error alerts

On any error condition, an immediate alert with the error code, message, and a link to the redaction log / admin dashboard if applicable.

### 9.3 Admin dashboard

The campaign-automation plugin's main dashboard (§6.4) shows current state. Designed to be the single page BT checks on iPhone via Safari.

### 9.4 Audit trail

Every `wp_gps_campaign_history` row carries the full composed topic and image prompt. If a draft turns out badly, BT can trace exactly what was sent to `fb_draft` — including which library item, which pointer result, and which redaction passes ran.

---

## 10. Implementation phases

### Phase 1: MCP tool stub + history table (1–2 days)
- Create `wp_gps_campaign_history` table
- Build `fb_suggest_campaign_topics` MCP tool with `dry_run=true` only
- Returns a selection but doesn't call `fb_draft`
- Tests: BT runs `dry_run` from Termius and verifies the picks make sense across 30 simulated days

### Phase 2: Plugin scaffold + manual trigger (2–3 days)
- Stand up `gps-campaign-automation` plugin
- Admin UI: Library Browser + Manual Trigger only
- Manual triggers call the MCP tool with `dry_run=true`
- BT can browse items and force-trigger to inspect proposed posts

### Phase 3: fb_draft integration (1–2 days)
- Wire MCP tool to actually call `fb_draft` when `dry_run=false`
- Plugin orchestrator writes the history row
- Status sync from fb_draft hooks (including `published_manually` after admin marks posted)
- Tests: BT manually triggers a few posts end-to-end, reviews them in the social-scheduler queue, downloads images, posts to FB by hand, calls fb_mark_published

### Phase 4: WP-Cron + system fallback cron (1 day)
- Register WP-Cron daily event
- Add system crontab fallback
- Telegram daily summary
- Settings page (kill switch, skip-list, cron time)

### Phase 5: Personal story rendering + image cache (2–3 days)
- Build the typographic background pool (6 base backgrounds, 1080×1080)
- Build `class-personal-story-renderer.php` with HTML/CSS → screenshot pipeline
- Wire `class-image-cache.php` for photo concept caching
- Tests: TMS personal story posts render correctly with pseudonym + excerpt

### Phase 6: Submission redaction (2 days)
- Build rules-pass redaction in PHP
- Build the `submission_redaction_check` Dify flow
- Build the `submission_excerpt_pull` Dify flow
- Build redaction log table + admin view
- Tests: BT runs 5–10 reviewed submissions through the pipeline and verifies redaction quality

### Phase 7: Side-by-side pairing (1–2 days)
- Implement pairing engine using affinity table from §4.5
- Add to MCP tool's archetype rotation logic
- Add Mondays-only trigger
- Tests: dry-run several Mondays to verify pairings make narrative sense

### Phase 8: Production cutover (½ day)
- Enable WP-Cron in production
- First 14 days: manual review of every queue + tighter Telegram notifications
- After 14 days: settle into routine review cadence

**Total estimate: 11–16 days of build time** spread across normal development weeks.

---

## 11. Where this lives

- **MCP tool source:** `~/GPS-MCP-Server/gps_mcp/tools/fb_suggest_campaign_topics.py` (and supporting modules in same package)
- **WordPress plugin source:** `~/GPS-Campaign-Automation/` (new git repo, GitHub: `brett1793/GPS-Campaign-Automation`)
- **Deployed to:** `/var/www/wordpress/wp-content/plugins/gps-campaign-automation/`
- **Library spec consumed:** `/home/ubuntu/Documentation/Specs/GPS-Campaign-Message-Library-Spec.md` (read live each run; no caching beyond the run itself)
- **Visual identities consumed:** `/home/ubuntu/Documentation/Specs/GPS-Campaign-Visual-Identities-Spec.md` (read once at MCP server startup; cached in process)
- **This spec:** `/home/ubuntu/Documentation/Specs/GPS-Campaign-Automation-Spec.md`

---

## 12. Provenance

- **Authored:** April 26, 2026
- **Sibling specs:** `GPS-Campaign-Visual-Identities-Spec.md` · `GPS-Campaign-Message-Library-Spec.md` · `GPS-Quote-Bank-Spec.md` · `GPS-Inmate-Lookup-Death-Consolidation-Spec.md` · `Tell-My-Story-Spec.md` · `GPS-Social-Sharing-Toolkit-Spec.md`
- **Step 3 decisions baked in (BT, April 26, 2026):**
  1. Cadence: 1 post/day at 11:00 AM ET
  2. Drafts always queue for admin review (`pending_review`); no auto-schedule
  3. Side-by-side pairing: automation pairs by topic-tag using the affinity table in §4.5
- **Recommendations applied as defaults (overridable in plugin Settings):**
  4. TMS personal-story pacing: minimum once per 10-post cycle when fresh stories exist (option A from Library §11)
  5. Image generation: aggressive caching of photo concepts via `wp_gps_campaign_image_cache`; HTML/CSS rendering for typographic archetypes (no Flux for stat posters, comparisons, bill cards)
  6. Submission redaction: rules-based first pass + Dify LLM verification pass
- **Publishing model:** Manual download-and-post via the existing GPS social-share workflow. There is no Facebook Graph API publishing path active in GPS today. Admin always: (a) reviews draft in social-scheduler queue, (b) downloads rendered image, (c) copies composed caption, (d) posts to Facebook by hand, (e) calls `fb_mark_published(draft_id, fb_post_url)` (or clicks "Mark Published" in admin UI) — status becomes `published_manually`. The automation queues; the admin always posts.

---

## 13. What's next after Step 3

Once the automation is live and stable, the natural follow-ons are:

- **Library expansion**: add more EtW stats (commissary investigation, MAS technology, family-impact), more photo concepts (yard, infirmary, intake), more comparison datasets (medical staffing vs population, mental health staffing). The library is meant to grow. The schema is designed for editor-curated additions without code changes.

- **Performance tuning**: after 30–60 days of cadence, look at FB engagement metrics per archetype. If `personal_story` outperforms `stat_poster` 4× consistently, adjust archetype rotation targets. The plugin Settings page should expose these as integers admins can tune.

- **DB migration of library**: when item count exceeds ~120 or `last_used` updates need to be transactional, migrate items from spec markdown into `wp_gps_campaign_library` (schema mirrors the Library YAML 1:1). The MCP tool gains a switch: read from spec vs. read from DB.

- **Cross-platform expansion**: the same library + selection logic feeds an Instagram automation, an X/Twitter automation, a Threads automation. The composition layer is the platform-specific piece. The selection layer (MCP tool) is platform-agnostic.

- **A/B testing within archetype**: when two stat posters target the same theme, the automation could fire one to half the audience and one to the other half (boosted post variant) and compare engagement. Out of scope for v1 but a natural extension.

- **Graph API integration (optional)**: when/if Facebook Graph API publishing becomes available (page access tokens secured, app reviewed/approved), the manual download-and-post step can be replaced by a `Publish to FB` button in the admin UI that calls the Graph API directly. The data model and queueing logic in this spec already accommodate this — the only changes needed would be: (a) implement a `fb_publish` MCP tool that posts via Graph API, (b) wire the admin UI button to call it, (c) the existing status flow (`pending_review` → `published` instead of `published_manually`) handles everything else without code changes. **Until that happens, manual download-and-post is the only publishing path.**

That's Step 4 territory — out of scope here.

---

*End of Specification*
