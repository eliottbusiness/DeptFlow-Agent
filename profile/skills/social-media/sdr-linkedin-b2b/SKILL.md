---
name: sdr-linkedin-b2b
title: SDR LinkedIn B2B Automation
version: 1.7.1
category: social-media
description: >
  Complete pipeline for sourcing, qualifying, and engaging B2B LinkedIn prospects
  with safety limits, Airtable deduplication, strict inline templates for message generation,
  and LinkedIn anti-ban rules.
intents:
  - sdr_sourcing
  - sdr_followup
  - linkedin_prospecting
  - lead_generation
scopes:
  - linkedin:write
  - linkedin:read
  - airtable:readwrite
  - gemini:generate
trigger:
  - manual
  - cron: "0 8 * * 1-5"
  - cron: "0 9-18/2 * * 1-5"
advanced:
  dry_run: true
  max_connections_per_day: 30
  max_messages_per_day: 100
  min_delay_between_actions_sec: 45
  deduplication_field: linkedin_id
  retry_policy:
    max_attempts: 3
    backoff: exponential
---

# SDR LinkedIn B2B Automation

Complete end-to-end pipeline for automated LinkedIn B2B prospecting that respects safety limits, avoids bans, and generates messages using strict inline templates plus a human approval gate before any send.

## Pipeline Overview
```
BeReach search → Prospect normalization → Deduplicate(Airtable) → Filter ICP → Score(WARM/HOT/VERY_HOT) → DebateAgent →
Connect(no-note) → Save Airtable → Follow-up after acceptance → Message quality → Human review → Send only if VALIDE → Log(Airtable)
```

**⚠️ PITFALL — IntentAnalyzer: NOT CREATED (decision 2026-05-02)**
User rejected the creation of a new `IntentAnalyzer` module. Priority is to first fix the existing scoring, DebateAgent, and company enrichment before considering intent detection. The `signaux_intention_cles` keywords in the YAML are currently used by `_heuristic_intent()` in `scoring.py` for keyword matching on post text — this is sufficient for now. The `detect_intent_signals` skill (hermes prompts only) exists but is not wired into `pipeline.py`. Do NOT create a new Python module for intent detection unless explicitly requested.

**⚠️ PITFALL — NEVER invent prospects from whole cloth (cardinal rule, 2026-05-02)**
A prospect MUST only be sourced from the actual results returned by the BeReach search API. Never generate, assume, or fabricate a prospect name, URL, or profile that does not appear in the raw search output. If the model proposes a prospect not found in the search results, that is a fabrication — reject it, use only the results you have. The rule: **if it wasn't in the search response, it doesn't exist.**

**⚠️ PITFALL — Airtable `terminal` commands must be short and simple**
When using the `terminal` tool with `curl` for Airtable:
- Use ONE simple curl command per call. Do NOT pipe through Python, use heredocs, or chain multiple curl calls.
- Always use `--max-time 8` to avoid tool timeout blocking.
- Simple working pattern: `curl -s --max-time 8 "https://api.airtable.com/v0/$BASE_ID/$TABLE?maxRecords=5" -H "Authorization: Bearer $AIRTABLE_API_KEY"`
- Complex multi-line commands (Python subprocess, heredocs, multiple pipes) timeout the `terminal` tool.
- Pipe through `python3 -m json.tool` only if the output is small and the command is simple; otherwise parse the raw JSON.
- The `execute_code` Python tool does NOT have access to `AIRTABLE_API_KEY` from the shell environment — use `terminal` with curl for all Airtable operations.

**⚠️ PITFALL — DebateAgent APPROVED everything at 100/100 (production-blocker, pending fix)**
The `DebateAgent` `_heuristic_decision()` was returning `score: 100/100 APPROVED` for ALL prospects because:
- `heuristic_mode = not(config.llm.enabled)` — with LLM disabled, heuristic mode is always active
- `_heuristic_decision()` had a "TEST MODE: all approved" comment and returned hardcoded APPROVED 100
- `_fallback_decision()` also approved any WARM/HOT prospect at score=75

Additionally, `auto_reject_threshold` (60) and `manual_review_threshold` (75) from the YAML config were never used in the heuristic path.

**Fix required (PR1, pending):** Replace `_heuristic_decision()` with real heuristic logic using the YAML thresholds:
- `score < auto_reject` → REJECTED
- `score >= manual_review AND has_intent` → APPROVED
- otherwise → NEEDS_MANUAL_REVIEW

The DebateAgent currently provides zero filtering value. Do not rely on it until the heuristic decision is fixed.

**⚠️ PITFALL — `company_size` and `industry` often missing from BeReach responses**
Apollo API enrichment is limited. Fields `industry`, `company_size`, and `location` are frequently empty. The pipeline currently marks these as `taille entreprise non renseignée (API limitée)` but still scores prospects WARM if the title matches. This makes the ICP sector/size filter largely inoperative.

**Decision needed:** Should `company_size` or `industry` missing be a REJECT trigger? Until that policy is defined, the pipeline will produce false positives.

**⚠️ PITFALL — `/visit/linkedin/profile` returns 422 even with valid API key (2026-05-02)**
The profile enrichment endpoint requires a **live BeReach browser session** (extension context). With a pure API key it always returns `HTTP 422 Unprocessable Entity` regardless of payload format (`profileUrl`, `profile`, `url`, `linkedinUrl` all tested). This means `collect_profile()` in `bereach.py` and any downstream company enrichment is **completely blocked** in server-side/headless mode.

**Impact:** `company_size`, `industry`, and `companyUrl` cannot be obtained via API. The `collect_profile()` step in the pipeline will fail with 422. Do NOT attempt this call in production without a BeReach session cookie. The `company-enrichment-workflow.md` reference is therefore **inoperative** for the current API-only setup.

**Workaround:** `company_size` and `industry` must come from the search result's `currentPositions[].company` structure (available but limited) or be inferred from the prospect's title/sector context. See `references/company-enrichment-workflow.md` for the blocked `visit_company` path — it also requires `companyUrl` which is not available without a successful `visit_profile` call.

**⚠️ PITFALL — `sortBy=recent` on `/search/linkedin/posts` returns 422**
Only `sortBy=relevance` (or omitting `sortBy` entirely) works for post search. Any attempt to sort by recent activity fails with `HTTP 422 Unprocessable Entity`. Use `sortBy=relevance` for all post searches.

**⚠️ PITFALL — `dry_run: false` must be explicitly set**
The config default is `dry_run: true`. When `dry_run: false` is set in `configs/clients/deptflow.yaml`, the pipeline writes real rows to Google Sheets (and by extension Airtable). Always verify this flag before a live run. Use `grep -n "dry_run" configs/clients/deptflow.yaml` to confirm.

**⚠️ PITFALL — CRM target is Google Sheets, not Airtable directly**
The pipeline uses `GoogleSheetsCRM` as the write target (spreadsheet ID: `1Eo_TBXCOYE5EqrmovzLvS_6zWhd3wsLDCl-kdUgVcwY`, tab `PROSPECTS`). Airtable syncs from that sheet automatically. Direct Airtable writes are NOT done by the pipeline. If the Google Sheets → Airtable sync breaks, prospects appear in Sheets but not in Airtable. Always check Sheet `PROSPECTS` tab for latest write results after a run.

**⚠️ PITFALL — Header/schemadrift (31 cols vs 21 cols)**
`CRM_COLUMNS` in `models.py`, `result_to_row()` in `crm.py`, and the Google Sheets header row must all agree. In this session: 31 Airtable fields, 26 CRM_COLUMNS entries, 21 return values from `result_to_row()`. This caused data misalignment. The check and fix procedure is in the pre-action checklist above.

## Safety & Anti-Ban Rules (NON-NEGOTIABLE)

1. **Daily Limits** (configurable via `.env`)
   - `LIMITE_CONNEXIONS_JOUR` ≤ 30
   - `LIMITE_MESSAGES_JOUR` ≤ 100
   - Check BeReach API using **correct JSON paths** (see section below)

2. **Action Delays**
   - Minimum 45 seconds between ANY LinkedIn action
   - Enforced globally via `DELAI_ENTRE_ACTIONS_SEC`

3. **Connection Requests**
   - NEVER send a note with a connection request
   - Body: `{"linkedin_id": "<id>"}` ONLY
   - No message, no text, no content field

4. **Message Content Rules**
   - The copy layer is canonical in `sdr-linkedin-message-quality`.
   - `sdr-linkedin-b2b` must not duplicate banned phrases, line-count rules, or rewrite heuristics.
   - Orchestration only needs to generate the draft, send it through quality, queue it for Airtable review, and send only after `validation_statut = VALIDE`.

5. **Account Protection**
   - If BeReach returns 429 (rate limit): wait `retryAfter × 2^(attempt-1)` then retry (max 3)
   - If 502 error: wait `30 × 2^(attempt-1)` then retry (max 3)
   - After 3 failures: log error JSON and skip prospect

## BeReach API Endpoints & Parsing (CRITICAL — v2.0.0)

### Correct Endpoints (OpenAPI v2.0.0 verified)

| Action | Method | Endpoint | Purpose | Request Body |
|--------|--------|----------|---------|--------------|
| Check limits | GET | `/me/limits` | Read daily quotas | — |
| Search prospects | POST | `/search/linkedin/people` | ICP-based prospect search | `{title: string, industry: array, location: array, count: int, start: int, keywords?: string}` |
| Send connection | POST | `/connect/linkedin/profile` | Send connection request (no note) | `{"linkedin_id": "${linkedin_id}"}` |
| Send message | POST | `/message/linkedin` | Send LinkedIn message | `{"profile": "${bereach_profile_target}", "message": "${message}"}` |

**DEPRECATED ENDPOINTS (DO NOT USE — return 404):**
- ❌ `POST /actions/linkedin/connect` (old body: `{"linkedin_id":"...`}) → use `/connect/linkedin/profile`
- ❌ `GET /actions/linkedin/message` / `POST /actions/linkedin/message` → use `POST /message/linkedin`

**Critical rules:**
- Connection body uses `{"linkedin_id": "<id>"}` only.
- The connection request body must not include `message`, `note`, `text`, `content`, or `profile`.
- Resolve and deduplicate the prospect first, then send the connection with the minimal body required by the current adapter.
- If `linkedin_id` is empty/missing → log `ERROR_MISSING_LINKEDIN_ID`, skip action, leave Airtable `statut` unchanged.

See `references/bereach-connection-target-contract.md` for the exact adapter contract and normalization rules.

For a repository-level map of the Deptflow/Hermes project, source-of-truth order, and doc drift notes, see `references/project-architecture.md`.

### Parsing BeReach Limits Response

**Response structure (real data):**

```json
{
  "success": true,
  "limits": {
    "connection_request": {
      "daily": {
        "current": 4,
        "limit": 90,
        "remaining": 86
      }
    },
    "message": {
      "daily": {
        "current": 0,
        "limit": 300,
        "remaining": 300
      }
    }
  },
  "creditsUsed": 0,
  "retryAfter": 0
}
```

**Correct JSON paths (EXACT):**
```python
connections_remaining = data["limits"]["connection_request"]["daily"]["remaining"]
connections_limit       = data["limits"]["connection_request"]["daily"]["limit"]
connections_current     = data["limits"]["connection_request"]["daily"]["current"]
messages_remaining     = data["limits"]["message"]["daily"]["remaining"]
messages_limit         = data["limits"]["message"]["daily"]["limit"]
messages_current       = data["limits"]["message"]["daily"]["current"]
```

**Common pitfall:** Old code read `remaining.connections` and `remaining.messages` — these paths **do not exist** in v2 API and return `null`. Always verify the full path.

### Search Parameters Validation

**Field formats (learned from 422 errors):**
- `title` → **string** (comma-separated values allowed, e.g. `"Founder,CEO,Fondateur"`)
- `industry` → **array of strings** (e.g. `["SEO","Marketing digital"]`)
- `location` → **array of strings** (e.g. `["France"]`)
- `keywords` → **string** (comma-separated, e.g. `"SEO,agence SEO,référencement naturel"`) — significantly improves ICP relevance
- `profileLanguage` → **array of strings** (e.g. `["fr"]`) — optional
- `count` → int (max 25 per page)
- `start` → int (pagination offset)
- `hasMore` → boolean in response (use to paginate)

**Important:** Using `keywords` string alongside `title` string filters results semantically and dramatically increases prospect ICP match rate. Always include targeted keywords when ICP is narrow (e.g., SEO agencies).

### Pagination Pattern

```python
all_prospects = []
page = 0
has_more = True
while has_more and page < max_pages and len(all_prospects) < max_prospects:
  response = POST /search/linkedin/people with {start: page*25, count: 25, ...}
  items = response.json()["items"]
  has_more = response.json().get("hasMore", False)
  all_prospects.extend(items)
  page += 1
```

**Stop conditions:**
- `hasMore == false` → no more results
- `len(all_prospects) >= max_prospects` → safety cap reached
- Connection sent → immediate stop (per rule)

### Profile URL Extraction Priority

When parsing a prospect item, derive `linkedin_id` for Airtable dedup and `bereach_profile_target` for connection body in this priority order:

```python
profile_url = item.get("profileUrl") or ""
profile_urn  = item.get("profileUrn") or ""
public_id    = item.get("publicIdentifier") or ""
linkedin_id  = profile_urn or public_id or profile_url or None
```

**Connection body uses:** the minimal `{"linkedin_id":"..."}` payload.
**Airtable `linkedin_id` uses:** the resolved CRM ID (Urn preferred, fallback to publicIdentifier, fallback to normalized profileUrl).

### API Diagnostic & Endpoint Mapping

When endpoints return 404 or limits appear as zero unexpectedly:

1. Load OpenAPI spec: `~/.hermes/docs/apibereach.json`
2. Verify endpoint exists in `paths` object
3. Check HTTP method (GET vs POST) matches OpenAPI definition
4. Test safe endpoints only: `GET /me/limits`, `GET /search/linkedin/parameters`
5. **NEVER** test destructive endpoints during diagnosis
6. Consult `references/bereash-endpoints.md` for full endpoint mapping and deprecated endpoint replacements
7. If limits parsing returns null: verify JSON path is exactly `limits.connection_request.daily.remaining` (not `remaining.connections`)
8. For LinkedIn inbox/acceptance checks, use `GET /me/linkedin/connections` with `count <= 40` per page and paginate with `start` when needed.
9. `POST /chats/linkedin/find` is a helper, not a source of truth: if it returns `found:false`, do not assume the connection was rejected; cross-check the connections list first.
10. Normalize `AIRTABLE_BASE_ID` before any Airtable call: if the env var contains a full URL, extract the `app...` base ID and use that canonical value.

See `references/bereash-endpoints.md` for complete endpoint table and body schemas, `references/followup-dry-run-acceptance.md` for the validated follow-up dry-run workflow, and `references/session-2026-05-01-run-notes.md` for the latest session-specific run evidence and guardrails.

### API Diagnostic Procedure

If endpoints return 404 or limits appear as zero unexpectedly:

1. Load OpenAPI spec: `~/.hermes/docs/apibereach.json`
2. Verify endpoint existence in `paths` object
3. Check method (GET vs POST) matches OpenAPI definition
4. Test safe endpoints only: `GET /me/limits`, `GET /search/linkedin/parameters`
5. NEVER test destructive endpoints during diagnosis
6. Consult `references/bereash-endpoints.md` for full mapping

## ICP Matching & Scoring

### ICP Criteria (all must match)
- `ICP_TITRES`: Array of job titles
- `ICP_SECTEURS`: Industry sectors
- `ICP_TAILLE_ENTREPRISE`: Company size range
- `ICP_PAYS`: Country code or name

### Scoring Logic

```
IF title/sector/size NOT in ICP → REJECTED (skip)
ELSE IF title/sector/size match ICP → WARM
  AND has LinkedIn post in last 60 days → HOT
  AND has strong signal matching OFFRE_VALEUR or OFFRE_CONCURRENTS → VERY HOT
```

**Signal detection examples:**
- Recent post about LinkedIn prospection struggles → HOT
- Post mentioning competitor tool → VERY HOT
- Comment about needing more SEO clients → HOT

## Airtable Targeting & Schema

Preferred target resolution:
1. Use `AIRTABLE_TABLE_ID` when it exists.
2. Otherwise use `AIRTABLE_TABLE_NAME`, but URL-encoded.
3. Use the same resolver for dedup GETs, POST creates, and PATCH updates.

Known-good session values:
- Table name: `SDR_Prospects`
- Table ID: `tblVifAXWdYt0TWtL`

Schema target: `SDR_Prospects` core CRM fields plus signal taxonomy extensions (`type_signal`, `sous_type_intention`, `sous_type_activite`) and the human-review fields documented below. Deduplication by `linkedin_id`.

For bounded batches, stop immediately after the requested number of new CRM records have been created; do not keep sourcing or enriching once the cap is reached. See `references/session-2026-05-02-crm-cap-and-schema.md` for the observed stop condition and schema snapshot.

Keep the signal labels and observability contract normalized. See `references/taxonomy-confidence-logs.md` for the canonical label set, the `confiance` / `preuve` contract, and the minimum JSON log keys expected by regression tests.

See `references/airtable-target-resolution.md` for the resolver rule and the exact failure mode that caused a 404 when the table existed but the code still used a non-canonical Airtable URL.

## Message Generation Strategy

### Current Approach: Inline Templates (DEFAULT — STRONGLY PREFERRED)

For WARM, HOT, and VERY_HOT scores, message generation uses **fixed inline JavaScript templates** defined directly in `sdr_linkedin_followup.md` (id: `generate_message`). These templates create the draft that is then quality-checked and queued in Airtable for human validation before any send.

For the copy framework itself, delegate to `sdr-linkedin-message-quality` and do not restate the rubric here. This skill only orchestrates the draft, validation gate, and Airtable handoff.

See `references/skill-boundaries.md` for the explicit cross-skill ownership split.

**Why inline templates?**
- 100% predictable line count
- Zero forbidden phrases
- No API latency/cost for message generation
- Easy to audit and modify
- No risk of LLM deviating from required format
- Easy to store as `message_a_valider` for approval

**Implementation:** See `sdr_linkedin_followup.md` → step `generate_message`. The script contains literal multi-line template strings with simple variable substitution (`{prenom}`, `{signal_detecte}`, `{OFFRE_CTA}`).

**Validation gate:** The draft is not sent until `validation_statut = VALIDE` in Airtable. See `references/followup-human-validation.md`.

## Human Validation Workflow

Before any LinkedIn message is sent:

1. Generate the draft message.
2. Run `sdr-linkedin-message-quality`.
3. If approved, store the message in Airtable with `validation_statut = A_VALIDER`.
4. Wait for human review on mobile.
5. Send only when the Airtable record is explicitly changed to `VALIDE`.
6. If rejected or flagged for rewrite, do not send.

Dry-run validation tests may simulate `connection_accepted = true` to exercise the internal follow-up loop without calling BeReach. In that mode, the workflow must still stop after writing the draft to Airtable and must never send a LinkedIn message.

The follow-up skill must never call `POST /message/linkedin` from the drafting branch.

See `references/human-validation-dry-run-contract.md` for the validated no-send dry-run sequence and Airtable field set.

### Canonical copy rules

Use `sdr-linkedin-message-quality` for the copy layer: AIDA + PAS structure, line counts, banned phrases, scoring, and rewrites.

This skill only orchestrates the draft, validates the Airtable gate, and prepares the send-after-validation path.

The draft generation step remains the inline template in `sdr_linkedin_followup.md` (`generate_message`).

### Profile URL Validation (SKIP Behavior)

Before sending a message, the skill must extract `profile_url` from the prospect record (Airtable field `url_profil` or `profile_url`) and build `bereach_profile_target` with the canonical adapter.

**If `bereach_profile_target` is missing or empty:**
- Log an ERROR with the prospect's name
- Set skip flag (`return { skip: true, reason: 'ERROR_MISSING_PROFILE_TARGET' }`)
- Do NOT call the BeReach `/message/linkedin` endpoint
- Leave Airtable `statut` unchanged (do not mark as attempted)
- Continue to next prospect

This prevents wasted quota and API errors.

### Airtable Human Review Fields

The `SDR_Prospects` table now includes the following follow-up review fields:

- `validation_statut`
- `message_a_valider`
- `message_quality_score`
- `message_quality_issues`
- `date_message_genere`
- `date_validation_humaine`
- `valide_par`
- `validation_notes`
- `human_review_required`

Recommended view: `Messages à valider`.

Use it to surface records where `validation_statut = A_VALIDER` and `human_review_required = true`.

**Variable source mapping:**
```javascript
const profile_url = prospect.fields.url_profil || prospect.fields.profile_url || "";
```

## Error Handling

See `references/bereash-error-handling.md` for retry policies.

## Environment Variables

See `.env` template in `templates/.env.sdr`.

### Required Keys
- `AIRTABLE_API_KEY`, `AIRTABLE_BASE_ID`, `AIRTABLE_TABLE_NAME`
- `AIRTABLE_TABLE_ID` (preferred when present)
- `GEMINI_API_KEY` (only needed if using deprecated Gemini path)
- `BEREACH_API_KEY`
- ICP variables: `ICP_TITRES`, `ICP_SECTEURS`, `ICP_TAILLE_ENTREPRISE`, `ICP_PAYS`
- OFFRE variables: `OFFRE_NOM`, `OFFRE_VALEUR`, `OFFRE_CTA`
- Limits: `LIMITE_CONNEXIONS_JOUR`, `LIMITE_MESSAGES_JOUR`, `DELAI_ENTRE_ACTIONS_SEC`

## Pre-Action Checklist

**Before ANY LinkedIn action (connection or message):**

1. [ ] Read BeReach remaining limits using correct JSON paths:
   - `limits.connection_request.daily.remaining`
   - `limits.message.daily.remaining`
2. [ ] Verify limits > 0 before any LinkedIn action
3. [ ] Check daily limits not exceeded
4. [ ] Verify delay since last action ≥ `DELAI_ENTRE_ACTIONS_SEC`
5. Resolve Airtable target first: use `AIRTABLE_TABLE_ID` when present, otherwise use URL-encoded `AIRTABLE_TABLE_NAME`
6. Airtable dedup by `linkedin_id` → skip if exists
7. Re-match ICP → skip if no match
8. Score prospect (WARM/HOT/VERY HOT)
9. Generate message using inline template (step `generate_message` in `sdr_linkedin_followup.md`)

**⚠️ PITFALL — CRM Schema Drift (CRITICAL before any config edit or run)**

The Python constant `CRM_COLUMNS`, the `result_to_row()` function return count, the Google Sheets `A:` range letter, and the actual Airtable field count MUST all agree. They frequently drift silently. A miss causes:
- truncated rows (if result_to_row returns fewer values than CRM_COLUMNS defines)
- misaligned columns (if range letter doesn't cover all columns)
- stale/missing data in Airtable fields
- dedup breakages (if `url_linkedin` is used but Airtable calls the field `url_profil`)

**Before editing the CRM schema or the config YAML, run the schema audit procedure:**
1. Read Airtable live schema via `GET /v0/meta/bases/{base_id}/tables` using `AIRTABLE_API_KEY`
2. Count the actual fields in the `SDR_Prospects` table
3. Compare with `CRM_COLUMNS` in `src/sdr_ai/models.py` (column count + names)
4. Count `result_to_row()` return values manually (each value = one line in the return list)
5. Check all `A2:U` / `A:U` references in `crm.py` → extend to `A2:AE` / `A:AE` if adding columns
6. Check all `CRM_COLUMNS.index("url_linkedin")` references → rename to `"url_profil"` if Airtable uses that name

**Reference:** See `references/crm-schema-audit.md` for the full diagnostic and the exact mismatch pattern found in this session (31 Airtable cols, 26 CRM_COLUMNS, 21 result_to_row values, 5 missing fields: `score`, `signal_detecte`, `date_message_envoye`, `message_envoye`, `nb_messages_envoyes`).

**⚠️ PITFALL — Scoring data quality policy (pending PR1)**

Current scoring has incorrect missing-data rules:
- `company_size` is treated as a hard ICP reject if absent — should be "plafond WARM" instead (absent = no HOT possible, but not a reject)
- `industry` is checked via exclusion only — should also be a soft signal when positive (sector match from activity)
- `recent_activity` absence triggers REJECTED — should trigger WARM with low confidence instead (activity is a qualification signal, not a data quality issue)

New policy (per audit 2026-05-02):
- **Bloquant**: title manquant ou titre exclu → REJECTED
- **Plafond WARM** (pas de HOT): `company_size` absent
- **Soft signal** (informatif): `industry` match ou absent
- **Signal de qualification bas** (WARM, confiance réduite): aucune activité récente

These changes are pending PR1 implementation in `scoring.py` and `debate_agent.py`.

**⚠️ PITFALL — Message quality: `observation_detectee` vs `signal_detecte` (strict separation)**

The `sdr-linkedin-message-quality` skill now enforces a strict field separation:
- WARM drafts must use `observation_detectee` (factual from profile/feed — title, company, sector, recent activity without intent). WARM drafts must NEVER contain a claim of type Intention (pain, need, search for solution) — that's a hard-fail.
- HOT/VERY_HOT drafts use `signal_detecte` (recent activity for HOT, intent/douleur for VERY_HOT)

The `sdr-linkedin-b2b` orchestration must pass the correct field to the message quality validator based on the prospect's score type. Do not pass `signal_detecte` for a WARM draft.

Additional message quality rules (from `references/quality-contract.md`):
- "ton équipe" / "votre équipe" is only allowed if the prospect's `title` contains a managerial/directional role (Head, VP, Director, Manager, Lead, Chief, Founder, CEO). Otherwise use "ton activité", "ton poste", or "ton rôle".
- No claim (sector, pain, need, intent) is acceptable unless traceable to MessageContext.
- Sales Navigator endpoints are strictly forbidden. Message quality must never depend on data available only via Sales Navigator.
10. **Resolve `linkedin_id` from prospect data (prefer Airtable `linkedin_id`, fallback to normalized BeReach identifiers when needed) before any connect action** ⭐ CRITICAL
11. **If `linkedin_id` is empty or missing: log `ERROR_MISSING_LINKEDIN_ID` with prospect name, set `skip=true`, do NOT call any BeReach endpoint, leave Airtable `statut` unchanged** ⭐ CRITICAL
12. Validate message output (line count, banned phrases, line length) before sending
13. Log action start (and record attempt in Airtable if applicable)

**Profile URL extraction priority (code pattern):**
```javascript
const profile_url = prospect.fields.url_profil || prospect.fields.profile_url || "";
const prospect_input = {
  profileUrl: item.profileUrl || "",
  url_profil: item.url_profil || "",
  profileUrn: item.profileUrn || "",
  publicIdentifier: item.publicIdentifier || "",
  linkedin_id: item.profileUrn || item.publicIdentifier || item.profileUrl || null
};
const { linkedin_id, bereach_profile_target, source, valid, error } = build_bereach_profile_target(prospect_input);
```

**Safety gate:** If `linkedin_id` is falsy (empty string, null, undefined), the action (connection or message) MUST be skipped with an ERROR log. Never call BeReach with an empty target field — it wastes quota and returns errors.

## API Diagnostic & Endpoint Mapping

When endpoints return 404 or limits appear as zero unexpectedly:

1. Load OpenAPI spec: `~/.hermes/docs/apibereach.json`
2. Verify endpoint exists in `paths` object
3. Check HTTP method (GET vs POST) matches OpenAPI definition
4. Test safe endpoints only: `GET /me/limits`, `GET /search/linkedin/parameters`
5. **NEVER** test destructive endpoints during diagnosis
6. Consult `references/bereash-endpoints.md` for full endpoint mapping and deprecated endpoint replacements
7. If limits parsing returns null: verify JSON path is exactly `limits.connection_request.daily.remaining` (not `remaining.connections`)

See `references/bereash-endpoints.md` for complete endpoint table and body schemas.
See `references/type-signal-conventions.md` for the normalized `type_signal` values and the `sous_type_intention` / `sous_type_activite` backfill rules when enriching `signal_detecte`.
See `references/company-enrichment-workflow.md` for the BeReach `company_size` recovery workflow without Sales Navigator (audit 2026-05-02 findings).
See `references/signal-taxonomy-backfill.md` for the session-safe taxonomy backfill recipe and the conservative default used for `Activité`.
See `references/regression-verification.md` for the exact local test command and regression checklist.

## Manual Testing

For regression coverage, prefer tests that assert the exact canonical labels and the log schema instead of eyeballing output.

Known-good local verification command:
```bash
python3 -m venv .venv
.venv/bin/python -m pytest -q
```
Use the venv because the system Python in this environment may not have `pytest` installed.

Set in `.env`:

```bash
SDR_DRY_RUN=true
SDR_FAKE_BEREACH=true
```

All actions logged, NO real LinkedIn calls.

For follow-up dry-runs, the skill should stop after writing `message_a_valider` and setting `validation_statut = A_VALIDER`.

## Revision History

- **v1.3.0 (2026-05-01)** — Fixed BeReach endpoint corrections: connection endpoint updated to `/connect/linkedin/profile` with `profile` body field (not `linkedin_id`); message endpoint updated to `/message/linkedin` with `profile` + `message` body. Added profile_url validation with skip behavior when empty. Enhanced API diagnostic procedure with OpenAPI verification. Added `references/bereash-endpoints.md` for endpoint mapping. Clarified limit parsing JSON paths and common pitfalls. Strengthened pre-action checklist with profile_url extraction step.
- **v1.7.1 (2026-05-02)** — Added cardinal rule: never invent prospects from whole cloth — use only results returned by BeReach search. Added Airtable `terminal` curl command guidelines: keep commands short and simple with `--max-time 8`, use `terminal` (not `execute_code`) for all Airtable operations since `execute_code` does not have access to `AIRTABLE_API_KEY` from the shell environment.
- **v1.7.0 (2026-05-02)** — Enriched scoring and logs with `confiance` / `preuve`, added regression tests for canonical labels and JSON logs, and aligned the connection body contract to the current minimal `{"linkedin_id":"..."}` format.
- **v1.7.0 (2026-05-02)** — Added CRM schema drift pitfall in pre-action checklist. Added `references/crm-schema-audit.md` with the full diagnostic procedure and the exact mismatch pattern found (31 Airtable cols, 26 CRM_COLUMNS, 21 result_to_row values, 5 missing fields). Key lesson: always verify live Airtable schema via `GET /v0/meta/bases/{base_id}/tables` — do not trust Python code constants as ground truth.
- **v1.5.0 (2026-05-01)** — Harmonized the signal taxonomy around normalized `type_signal` values, added `sous_type_activite` to the CRM taxonomy, backfilled the existing activity records, and removed duplicated copy rules from orchestration in favor of the canonical `sdr-linkedin-message-quality` skill.
- **v1.4.1 (2026-05-01)** — Added a validated no-send dry-run contract for the human validation loop. The workflow can now simulate `connection_accepted = true`, write the approved draft to Airtable (`message_a_valider`, `message_quality_score`, `message_quality_issues`, `validation_statut = A_VALIDER`, `human_review_required = true`, `date_message_genere`), and stop without sending LinkedIn or mutating message counters. See `references/human-validation-dry-run-contract.md`.
- **v1.4.0 (2026-05-01)** — Added a human validation gate before sending LinkedIn messages. Draft messages are now quality-checked, written to Airtable, marked `A_VALIDER`, and only sent once `validation_statut = VALIDE`. Added the `followup-human-validation` reference and the new Airtable review fields.
- **v1.2.0 (2026-05-01)** — Added BeReach API endpoint mapping with correct paths and deprecated endpoint warnings. Updated limit parsing to use `limits.connection_request.daily.remaining` and `limits.message.daily.remaining`. Enhanced pre-action checklist with explicit JSON path verification. Added diagnostic procedure for 404/limit-zero cases.
- **v1.1.0 (2026-05-01)** — Enforced exact inline templates for WARM/HOT/VERY_HOT. Banned phrases list expanded. Gemini generation deprecated for these scores. Line count rules made strict (±0). Added mandatory CTA question requirement.
- **v1.0.0** — Initial release with Gemini-based message generation.