# Project architecture map (Deptflow / Hermes)

This note captures the stable repo layout and the current source-of-truth hierarchy observed during the Deptflow repository exploration session.

## Source-of-truth order
1. `configs/clients/deptflow.yaml` — active client configuration (ICP, offer, BeReach, quotas, delivery, Hermes settings).
2. `src/sdr_ai/pipeline.py` — runtime orchestration of search → enrichment → scoring → CRM → connection send → logs/report.
3. `src/sdr_ai/scoring.py` — canonical operational scoring for REJETE / WARM / HOT.
4. `src/sdr_ai/bereach.py` — BeReach API adapter and identifier normalization.
5. `src/sdr_ai/crm.py` — persistent CRM backend (Google Sheets or local CSV in dry-run).
6. `runs/deptflow/*.json` — historical run reports and audits.
7. `profiles/deptflow.MEMORY.md` and `profiles/deptflow/skills/*/SKILL.md` — Hermes memory + operational procedures.

## Key directories and what they contain
- `configs/` — client YAMLs and examples.
- `profiles/deptflow/` — Hermes client memory and installed client skills.
- `hermes/skills/` — source skills/templates shipped with the repo.
- `src/sdr_ai/` — Python runtime, CLI, agents, quality layer, CRM, quota, notifications.
- `scripts/` — install/bootstrap/maintenance utilities.
- `runs/` — local CRM CSV, quota state, run JSON reports, audits.
- `docs/` — documentation and QA/spec notes.

## Important repo facts learned
- The active Deptflow YAML is more reliable than the memory snapshot when they differ.
- The project currently has no dedicated static file for LinkedIn outreach messages; the connection step is intentionally no-note, and the only message-related content is reporting/justification/quality support.
- `REJETE` should not be inserted into the main `PROSPECTS` CRM path.
- WARM/HOT deduplication is done via LinkedIn URL / resolved LinkedIn identifier.
- Run outputs are JSON and include structured logs, counts, and audits.

## Commonly important files to inspect first
- `README.md`
- `README_DEPTFLOW.md`
- `INSTALL_DEPTFLOW.md`
- `configs/client.example.yaml`
- `configs/clients/deptflow.yaml`
- `profiles/deptflow.MEMORY.md`
- `src/sdr_ai/config.py`
- `src/sdr_ai/models.py`
- `src/sdr_ai/scoring.py`
- `src/sdr_ai/bereach.py`
- `src/sdr_ai/pipeline.py`
- `src/sdr_ai/crm.py`
- `src/sdr_ai/quota.py`
- `src/sdr_ai/notifications.py`
- `src/sdr_ai/quality.py`
- `src/sdr_ai/agents/quality/*`
- `runs/deptflow/*.json`
- `runs/deptflow/audit_*.json`

## Session-specific note
This reference was created after a repository walkthrough that identified the architecture, source-of-truth hierarchy, and a few doc/config drift points between memory snapshots and the active YAML.