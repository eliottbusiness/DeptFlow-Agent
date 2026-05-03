---
name: detect_intent_signals
title: Detect LinkedIn Intent Signals
version: 2.1.0
category: social-media
description: >
  Detect LinkedIn buying-intent signals for an ICP-qualified prospect using BeReach v2,
  and return a compact JSON classification for downstream scoring.
intents:
  - linkedin_intent_signal_detection
  - linkedin_signal_scoring
  - linkedin_activity_analysis
scopes:
  - linkedin:read
  - search:read
trigger:
  - manual
  - inline
advanced:
  dry_run: true
  output_schema_locked: true
  max_lookback_days: 90
---

# Detect LinkedIn Intent Signals

Use this skill after ICP qualification and before final prospect scoring.

It analyzes a prospect's LinkedIn activity and nearby engagement signals to classify intent.

The skill is conservative by design: do not overclassify weak engagement as intent.

## Input

Required:
- `linkedin_url`: LinkedIn profile URL of the prospect

Optional contextual inputs when available:
- `OFFRE_MOTS_CLES`: array of offer-related keywords
- `OFFRE_CONCURRENTS`: array of competitor company/profile URLs
- `OFFRE_HASHTAGS`: array of hashtags relevant to the offer
- `linkedin_id`: normalized identifier if the caller already has it

## Output

Return JSON only:

```json
{
  "linkedin_url": "https://www.linkedin.com/in/prenom-nom",
  "level": "WARM|HOT|VERY_HOT",
  "signals_triggered": [],
  "early_exit": false,
  "timestamp": "2026-05-01T10:30:00Z"
}
```

### Signal object format

Each signal should follow this shape:

```json
{
  "type": "commentaire_post_concurrent|post_problematique_liee|commentaire_post_offre|like_post_concurrent|like_post_offre|recent_post_signal",
  "url_post": "https://www.linkedin.com/posts/...",
  "extrait": "text or comment excerpt",
  "source": "concurrent|offre|prospect",
  "confidence": "medium|high"
}
```

## Classification rules

- `WARM` = prospect is ICP-qualified, but no reliable buying-intent signal was found.
- `HOT` = relevant activity signal was found, such as recent posts about the topic, comments on offer-related content, or likes on relevant posts.
- `VERY_HOT` = strong intent signal was found, such as engagement with competitor content, a direct pain point, or a post explicitly expressing a need, switch, or urgency.

### Strong signals that should usually map to VERY_HOT

- Comment on a competitor post
- Like on a competitor post
- Post explicitly describing a pain point linked to the offer
- Post mentioning switching tools, changing provider, or looking for a solution
- Comment that signals active evaluation, budget, migration, or urgent need
- **Business transition context** — company in liquidation, bankruptcy, restructuring → prospect must rebuild pipeline
- **Funding/investment round** — prospect's company just raised or spent → budget available for tools
- **Hypergrowth signals** — hiring spree, new offices, rapid scaling → sales team needs tools to scale
- **Leadership change** — new CEO/VP Sales appointed → prospect likely evaluating/modernizing stack

### Medium signals that should usually map to HOT

- Recent post on the offer topic without explicit buying intent
- Comment on a hashtag/topic related to the offer
- Like on a topic-relevant post
- Recent repeated activity around the theme, without direct intent language

### Conservative rule

If the signal is ambiguous, keep the prospect at `WARM` or `HOT` rather than escalating to `VERY_HOT`.

## BeReach v2 endpoints to use

Verified in `~/.hermes/docs/apibereach.json`:

- `POST /collect/linkedin/posts`
- `POST /collect/linkedin/comments`
- `POST /collect/linkedin/likes`
- `POST /collect/linkedin/hashtag`
- `POST /search/linkedin/posts`

### Request shapes

`POST /collect/linkedin/posts`
```json
{
  "profileUrl": "https://www.linkedin.com/in/username",
  "count": 20,
  "returnReposts": true
}
```

`POST /collect/linkedin/comments`
```json
{
  "postUrl": "https://www.linkedin.com/posts/...",
  "count": 100
}
```

`POST /collect/linkedin/likes`
```json
{
  "postUrl": "https://www.linkedin.com/posts/...",
  "count": 100
}
```

`POST /collect/linkedin/hashtag`
```json
{
  "hashtag": "seo",
  "count": 20
}
```

`POST /search/linkedin/posts`
```json
{
  "keywords": "SEO OR agence SEO OR référencement naturel",
  "sortBy": "relevance"
}
```

## Procedure

1. Normalize the prospect URL.
2. Collect the prospect's recent posts with `POST /collect/linkedin/posts`.
3. Inspect recent post text for offer keywords, pain points, competitor names, switching language, urgency, or solution-seeking language.
4. Collect posts around relevant offer keywords and hashtags.
5. Inspect comments and likes on those posts for the prospect's profile identity.
6. Collect posts from competitor pages when competitor URLs are known.
7. Inspect comments and likes on competitor posts.
8. Stop early if a strong signal is confirmed and `level` can safely be set to `VERY_HOT`.
9. Return the final JSON.

## Matching rules

When comparing a profile from comments/likes to the prospect, treat any of these as a match:
- `profileUrl`
- `profileUrn`
- `publicIdentifier`

Prefer `profileUrl` when present.

## CRM taxonomy mapping

When the downstream CRM persists the result of this skill, normalize the classification as follows:
- `VERY_HOT` → usually `type_signal = Intention` with `sous_type_intention` set to the best matching subtype.
- `HOT` → usually `type_signal = Activité` with `sous_type_activite` set when the activity is clear enough.
- `WARM` → usually `type_signal = Contexte` or `Aucun signal récent`, depending on whether the fit comes from profile context or a complete absence of recent signal.

If the signal is ambiguous, leave the subtype empty rather than inventing one.

## Safety rules

- Do not invent intent from weak engagement.
- Do not use hidden assumptions about the prospect's business.
- Do not call any connection or message endpoint.
- Do not write to Airtable.
- Do not generate a long narrative; return JSON only.

## Integration rule

This skill is intended to run after ICP filtering in the SDR LinkedIn pipeline.
Its output feeds the final `HOT` / `VERY_HOT` scoring branch and the message personalization context.

**CRITICAL:** The orchestrating pipeline (`sdr-linkedin-b2b`) must explicitly call this skill after the ICP filter step. Without this call, the pipeline falls back to simple post-recency scoring which cannot distinguish a generic post from a pain-point post or a business-transition context signal. In this session, failing to call this skill caused a VP Sales whose company was in liquidation to remain WARM instead of escalating to VERY_HOT.

The CRM taxonomy mapping for business-context signals:
- Company liquidation/judicial recovery → `type_signal = Intention`, `sous_type_intention = Reconstitution_pipeline`
- Funding round / investment → `type_signal = Intention`, `sous_type_intention = Investissement_stacks`
- Hypergrowth / hiring → `type_signal = Contexte`, `sous_type_activite = Croissance_rapide`
- Leadership change → `type_signal = Intention`, `sous_type_intention = Changement_direction`

## Maintenance

If BeReach field names or response shapes change, update the endpoint contract here first.
