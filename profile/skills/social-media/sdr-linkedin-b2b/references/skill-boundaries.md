# Skill Boundaries — SDR LinkedIn B2B

This note prevents duplicated guidance across the SDR LinkedIn skill set.

## Canonical ownership

- `sdr-linkedin-message-quality` is the canonical source for copy structure, banned phrases, scoring, and rewrite rules.
- `detect_intent_signals` is the canonical source for LinkedIn intent-signal extraction and signal classification.
- `sdr-linkedin-b2b` is the orchestration skill: sourcing, deduplication, ICP filtering, intent-signal handoff, scoring flow, Airtable handoff, validation gate, and send gating.
- `sdr-linkedin-b2b` should not restate AIDA/PAS or the message rubric; it should only point to the canonical copy skill.

## What not to duplicate in `sdr-linkedin-b2b`

- AIDA / PAS copy rules
- line-by-line message templates
- banned phrase lists
- scoring rubric details
- rewrite heuristics

## What `sdr-linkedin-b2b` may reference

- It may say that drafting should follow the canonical quality skill.
- It may mention the required line count and validation gate at a high level.
- It should link to the human validation workflow instead of restating the full copy rubric.

## Maintenance rule

When the copy rubric changes, update `sdr-linkedin-message-quality` first, then only adjust `sdr-linkedin-b2b` pointers if the orchestration changes.