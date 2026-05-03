---
name: sdr-linkedin-message-quality
description: Validate, score, and rewrite short French LinkedIn prospecting messages for the SDR pipeline. Use when drafting or reviewing WARM/HOT/VERY_HOT outreach, before any LinkedIn DM is sent, or when adapting a draft to the current offer and ICP. Never use for email, deliverability, domains, SPF/DKIM/DMARC, or cold-email infrastructure.
version: 1.1.0
category: social-media
intents:
  - linkedin_message_quality_check
  - linkedin_message_rewrite
  - linkedin_followup_validation
scopes:
  - linkedin:read
  - linkedin:write
  - prompt:validate
trigger:
  - manual
  - inline
advanced:
  dry_run: true
  output_schema_locked: true
  max_line_length: 140
  approved_threshold: 16
---

# SDR LinkedIn Message Quality

Validate and improve LinkedIn prospecting messages before they are sent by an SDR agent.

Apply cold-outreach copywriting principles only when they fit LinkedIn DMs: short, specific, human, low-friction, and non-pitchy.

Never send the message. This skill only reviews, scores, rejects, or rewrites.

## When to use

Use this skill when:
- drafting a new LinkedIn prospecting message
- reviewing a WARM, HOT, or VERY_HOT follow-up message
- checking a rewritten message before sending
- adapting a draft to the current offer, CTA, or prospect signal
- enforcing line count, tone, and banned-phrase rules

Do not use this skill for:
- email copy
- deliverability topics
- domain warmup
- SPF / DKIM / DMARC
- Smartlead / Instantly / cold-email infrastructure
- long-form sales sequences

## Required inputs

Use available context. If something is missing, infer conservatively.

- `score_type`: `WARM`, `HOT`, or `VERY_HOT` — determines which field to use in line 2
- `prenom`: prospect first name
- `message`: drafted message to validate
- `observation_detectee`: element from the prospect's profile or activity (title, company, sector, recent activity without intent). Used for WARM only.
- `signal_detecte`: intent signal, pain, need, or recent activity from posts. Used for HOT and VERY_HOT only.
- `OFFRE_NOM`, `OFFRE_VALEUR`, `OFFRE_CTA`: offer variables when available

**Critical distinction**:
- WARM: use `observation_detectee` — factual element from the profile or feed, never an intent signal
- HOT: use `signal_detecte` — recent activity reference (post, interaction)
- VERY_HOT: use `signal_detecte` — intent signal, pain, or need detected

Never use `signal_detecte` in a WARM draft. If the draft implies a need or intent (search, pain, problem) for a WARM prospect, it is a hard-fail.

## Output format

Always return only valid JSON matching this schema:

```json
{
  "action": "linkedin_message_quality_check",
  "score": 0,
  "approved": false,
  "score_type": "WARM|HOT|VERY_HOT",
  "line_count": 0,
  "max_line_length": 0,
  "banned_phrases_found": [],
  "issues": [],
  "corrected_message": "",
  "reason": ""
}
```

No markdown, no commentary, no bullet points outside the JSON output.

## Validation flow

1. Normalize line endings.
2. Count non-empty lines.
3. Verify the first line is exactly `Prénom,`.
4. Check line count against the score type.
5. Check line length: every line must be <= 140 characters.
6. Check for banned phrases and risky patterns.
7. Check the message is human, calm, specific, and low-friction.
8. Score it out of 20.
9. If invalid, produce one compliant `corrected_message`.
10. Revalidate the corrected message once if the caller asks for a retry.

If `score_type` is missing, infer conservatively from structure and context:
- 4 lines -> likely WARM
- 5 lines with a recent activity reference -> likely HOT
- 5 lines with an explicit intent / pain signal -> likely VERY_HOT

## Structure rules

### Recommended drafting framework: AIDA + PAS

When a prospect is ready for follow-up and a message must be drafted, use this structure as the default copy framework:

1. **Line 1**: first name only, followed by a comma.
2. **Line 2**: personalized hook — mention one precise element of the prospect's activity or a recent signal (new hire, funding, expansion, visible growth, etc.). You may also open on a problem or an ambition.
3. **Line 3**: value — one short, concrete sentence explaining how you help, ideally tied to a result or outcome seen with similar clients.
4. **Line 4**: simple question or low-friction CTA. Never ask directly for a 30-minute call.

Tone adaptation by score type:
- **WARM**: simple observation, light relevance, no overfitting.
- **HOT**: sober reference to recent activity, activity-led hook.
- **VERY_HOT**: precise intent signal and likely pain, with a more direct value bridge.

Keep the draft compact, human, and respectful. Avoid jargon and canned phrasing.

### WARM
Require exactly 4 non-empty lines:
1. First name only followed by a comma.
2. Simple observation tied to `observation_detectee` (factual, profile-based — no intent claim).
3. One short sentence with the value.
4. Simple CTA or `OFFRE_CTA`.

**WARM hard-fail**: If line 2 or 3 contains a claim of type intent, pain, or need (even mild), it is a hard-fail. WARM = observation only.

### HOT
Require exactly 5 non-empty lines:
1. First name only followed by a comma.
2. Sober reference to recent activity from `signal_detecte`.
3. Link to business context or pain (cautious).
4. One short sentence with the value.
5. Simple CTA or `OFFRE_CTA`.

**HOT rule**: `signal_detecte` must be a recent activity (post, interaction). Not an intent signal.

### VERY_HOT
Require exactly 5 non-empty lines:
1. First name only followed by a comma.
2. Precise intent signal from `signal_detecte`.
3. Likely pain stated cautiously (conditional).
4. Direct link with `OFFRE_NOM` and `OFFRE_VALEUR`.
5. Simple, direct, non-aggressive CTA.

## Style rules

The message must be:
- direct
- human
- calm
- conversational
- specific enough to feel intentional
- one idea only
- one question only
- no first-message links
- no signature
- no product feature list
- no exaggerated promise
- no aggressive pitch
- French first, unless the provided draft is clearly in another language
- no claim on sector, pain, need, or intent without MessageContext justification

**"Ton équipe" rule**: This phrasing is only allowed if the prospect's `title` explicitly indicates a managerial or directional role (Head of, VP, Director, Manager, Lead, Chief, Founder, CEO, etc.). Otherwise use "ton activité", "ton poste", or "ton rôle".

**Sales Navigator rule**: No Sales Navigator endpoint. Ever. The LinkedIn account is free. Message quality must never depend on a field available only via Sales Navigator.

Maximum line length: 140 characters.

Prefer the user's natural French accents in rewrites when present.

## Hard-fail checks

Reject the message if any of these are true:

- wrong line count for the `score_type`
- first line is not exactly first name plus comma
- any line exceeds 140 characters
- contains a link or URL
- contains a signature or sign-off block
- contains more than one question
- contains a banned phrase
- includes a pushy CTA such as booking 30 minutes or choosing a calendar time
- makes guarantees, exaggerated claims, or “done-for-you” promises that sound too strong
- sounds like a cold email instead of a short LinkedIn DM
- reads like a brochure, feature list, or generic sales pitch

## Scoring rubric

Score out of 20:
- Structure respected: 5
- Personalization relevance: 4
- Value clarity: 4
- Human and natural tone: 3
- Simple CTA: 2
- No banned phrases or risky patterns: 2

Approve only when:
- score is at least 16
- no hard-fail rule is triggered

## ICP context

The target ICP is founders, co-founders, CEOs, dirigeants, and gérants of local SEO agencies in France.

Relevant pains:
- unstable pipeline
- dependence on word of mouth
- irregular clients
- LinkedIn not generating enough opportunities
- no stable acquisition system
- lack of time for prospecting
- stress about future clients

Use these carefully. Do not accuse the prospect or assume too much.

Preferred angle: help local SEO agency founders generate qualified meetings with local businesses that need SEO through an autonomous LinkedIn prospecting system.

## Rewrite guidance

Default rewriting should follow the AIDA + PAS flow above while staying within the exact line count rules for the score type.

If the message fails, return `approved: false` and include a compliant `corrected_message`.

Use these baseline patterns when rewriting. Note: WARM uses `observation_detectee`; HOT and VERY_HOT use `signal_detecte`.

### WARM rewrite pattern

```text
{prenom},
J'ai vu {observation_detectee}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

`observation_detectee` examples: "que tu gères une agence SEO", "ton poste de Sales Director", "ton activité dans le SaaS B2B".

### HOT rewrite pattern

```text
{prenom},
J'ai vu ton activité récente autour de {signal_detecte}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

`signal_detecte` for HOT = recent activity (post, interaction). Not intent. Example: "un post sur la prospection LinkedIn".

### VERY_HOT rewrite pattern

```text
{prenom},
J'ai vu ton signal autour de {signal_detecte}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

`signal_detecte` for VERY_HOT = intent signal or pain. Example: "un post sur l'automatisation de la prospection", "une discussion sur les volume de pipeline".

Keep the final message short and natural.

## Integration rule for follow-up agents

Before sending any LinkedIn message:
1. Generate the draft message.
2. Validate it with this skill.
3. If `approved=true`, the follow-up workflow may store the approved draft in Airtable as the message awaiting human review.
4. If `approved=false`, revalidate the `corrected_message` once.
5. If it still is not approved, do not send; log the failure and leave Airtable status unchanged.
6. Never bypass Airtable `validation_statut = VALIDE` before an actual send.

This skill is also used in no-send dry-runs: the caller may simulate upstream events (for example `connection_accepted = true`) but must still stop at the Airtable validation queue and never call LinkedIn send endpoints.

## Maintenance rule

This skill is the canonical source for LinkedIn message copy rules in the SDR stack.
Orchestration skills should reference it rather than duplicating AIDA/PAS structure, banned phrases, scoring, or rewrite logic.

Do not introduce alternative signal taxonomies, confidence fields, or log schemas here. Those belong to `sdr-linkedin-b2b`; this skill only scores and rewrites the message text itself.

The `references/quality-contract.md` defines the validator's hard rules. The `references/reasoning-playbook.md` defines the authoring guidelines. Both are part of this skill's spec.

When updating:
- patch the banned phrases in `references/message-rubric.md`
- update `references/quality-contract.md` if validation rules change
- update `references/reasoning-playbook.md` if authoring guidelines change
- patch the skill only if the output schema or scoring changes
