# Taxonomy, confidence, proof, and logs

Session learnings to keep the SDR LinkedIn pipeline consistent.

## Canonical label set

Normalize prospect signal labels around a small set of stable values:
- `Contexte`
- `Activité`
- `Intention`
- `Aucun signal récent`

Use these labels in:
- scoring outputs
- Airtable fields
- logs JSON
- regression tests

Avoid mixing variants such as `Taille entreprise`, `Secteur compatible`, `Seniorité OK`, or `Activité LinkedIn détectée` when a canonical label already exists. Prefer a prefix like `Contexte: ...`, `Activité: ...`, or `Intention: ...`.

## Confidence / proof contract

Recommended result fields:
- `type_signal`: one canonical taxonomy value
- `confiance`: numeric score on a 0–1 scale
- `preuve`: primary evidence snippet
- `preuves`: ordered list of evidence snippets

Rules:
- `preuve` should be the first meaningful evidence, not a synthetic label
- `preuves` should deduplicate repeated snippets
- if there is no recent activity, the proof should explicitly say so
- confidence should increase with recent activity, number of intent signals, and available proof, but remain conservative

## JSON logs

Use one structured log per action, with these keys:
- `timestamp`
- `skill`
- `action`
- `linkedin_id`
- `statut` (`SUCCESS|ERROR|SKIP`)
- `details`
- `erreur`

Recommended enrichments for future runs:
- `type_signal`
- `confiance`
- `preuve`
- `prospeсt_status` / `qualification_status` if available
- a compact `signals_count` or similar summary only if it stays stable

Keep logs human-readable, deterministic, and small enough to inspect quickly.

## Non-regression tests to add

Cover these behaviors with tests:
- canonical label normalization
- `confiance` / `preuve` population for WARM, HOT, VERY_HOT, and reject cases
- JSON log structure and required keys
- no-send dry-run behavior for follow-up drafting
- Airtable dedup before any LinkedIn action
- connection request body stays minimal and note-free

## Pitfall

Do not let orchestration and copy quality drift into two competing taxonomies. The pipeline should keep one canonical signal vocabulary and one canonical copy-quality rubric, not local variants per file.
