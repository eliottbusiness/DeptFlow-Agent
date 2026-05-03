# Regression verification checklist

Use this checklist after changes to labels, confidence/proof, or JSON logs.

## Local test command

```bash
python3 -m venv .venv
.venv/bin/python -m pytest -q
```

The system Python in this environment may not include `pytest`, so the local venv is the reliable path.

## Expected test coverage

Prefer at least one regression test for each of these areas:
- canonical label normalization
- `type_signal` / `sous_type_intention` / `sous_type_activite`
- `confiance` / `preuve` population
- enriched run log JSON fields
- BeReach prospect normalization
- no-send dry-run behavior for follow-up drafting

## What to inspect manually

- run payload contains structured `logs`
- each log entry keeps the canonical keys: `timestamp`, `skill`, `action`, `linkedin_id`, `statut`, `details`, `erreur`
- enriched fields are stable and deterministic enough for regression tests
- connection request bodies stay minimal and note-free

## Common pitfall

If a test seems to fail because `pytest` is missing, do not switch to ad hoc manual verification. Create the venv and run the test suite there.
