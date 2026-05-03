# Session 2026-05-02 Run Learnings — SDR Pipeline Live Run

## Run metadata
- Date: 2026-05-02
- Pipeline: `SDRPipeline.run_daily(limit=10, send_connections=False)`
- Duration: 132 seconds
- Config: `dry_run: false` (real CRM writes)
- CRM target: Google Sheets `PROSPECTS` tab (ID: `1Eo_TBXCOYE5EqrmovzLvS_6zWhd3wsLDCl-kdUgVcwY`)
- Airtable sync: automatique depuis Google Sheets

## Results

| Metric | Value |
|--------|-------|
| found_count | 20 |
| qualified_count | 2 |
| warm_count | 2 |
| hot_count | 0 |
| rejected_count | 8 |
| added_to_crm | 2 |
| deduplicated | 0 |
| connections_sent | 0 |
| errors | notification Telegram (404 token, non-blocking) |

## Prospects in CRM

### Prospect 1: Jean-Emmanuel Boucher
- Titre: VP Sales @ PowiDian
- Signal: post 2026-04-20宣告 liquidation judiciaire → recherche d'emploi
- System score: WARM (confiance 0.7)
- Subagent verdict: VERY_HOT (business-context signal = reconstruction pipeline)
- **Gap: `detect_intent_signals` not called → missed VERY_HOT escalation**
- Message drafted: "Bonjour Jean-Emmanuel, j'ai suivi ton parcours chez PowiDian. Les transitions après une période difficile sont souvent l'occasion de repenser ses outils. Si tu explores des solutions pour optimiser ta prospection LinkedIn sur ton prochain défi, je serais ravi d'échanger. Je te souhaite bonne continuation dans ta recherche."

### Prospect 2: Bernard LETELLIER
- Titre: Head of Sales & Business Growth @ Ultima Displays France
- Signal: post 2026-04-10 promotion produit displays (double matière/double impact)
- System score: WARM (confiance 0.7)
- Subagent verdict: WARM (titre OK, secteur displays hors ICP SaaS/Tech)
- **Gap: sector data always empty (Apollo API limited) → ICP sector filter inoperative**
- Message drafted: "Bonjour Bernard, j'ai vu votre actualité sur Ultima Displays — belle positionnement sur le marché des solutions d'exposition. Vous êtes Head of Sales, j'imagine que la gestion d'un pipeline prospection multicanal est un enjeu quotidien. Nous accompagnons des équipes commerciales SaaS/B2B sur l'automatisation LinkedIn pour générer des rendez-vous qualifiés. Serait-il pertinent d'en échanger brièvement ?"

## Critical failures

### Failure 1: `detect_intent_signals` not in pipeline.py
**File**: `src/sdr_ai/pipeline.py` (SDRPipeline.run_daily)
**Problem**: The skill `detect_intent_signals` exists and describes itself as part of the scoring pipeline, but `pipeline.py` never calls it. Scoring is based on post-recency only (< 45 days = HOT, else WARM). No semantic analysis.
**Evidence**: Jean-Emmanuel Boucher (liquidation = strong business-context signal) scored WARM. Subagent with live LinkedIn analysis scored VERY_HOT.
**Fix**: Add `detect_intent_signals` call between ICP filter and final WARM/HOT/VERY_HOT scoring.

### Failure 2: DebateAgent approves everything at 100/100
**Problem**: `DebateAgent` in this run returned `score: 100/100 APPROVED` for both prospects, including the liquidation scenario and non-ICP sector. No quality filtering value.
**Likely root cause**: `enable_llm_judge: false` in YAML → falls back to trivially-approving rules.
**Fix**: Fix or disable DebateAgent before relying on it for filtering.

### Failure 3: Company size / sector always empty
**Problem**: Apollo API limited → `industry` and `company_size` fields always empty. Pipeline marks "taille entreprise non renseignée (API limitée)" but still passes as WARM if title matches.
**Impact**: Bernard LETELLIER passed ICP checks despite displays sector being out of ICP scope.
**Fix needed**: Define policy — should `company_size` missing = REJECT trigger? Add `enrichment_confidence` field to track data quality.

### Failure 4: Header/schema drift (already documented)
- 31 Airtable fields in live schema
- `CRM_COLUMNS` in `models.py`: 26 entries
- `result_to_row()`: 21 return values
- `url_linkedin` used in code but Airtable field is `url_profil`
- **Status**: Fixed mid-session (31-col header written, code patched)

## Subagent analysis method
```
delegate_task with role=leaf, tasks=[two prospect analyses], toolsets=["web"]
```
- Both subagents ran in parallel (~46 seconds total)
- Used live LinkedIn page visits to get post content
- Generated natural French connection messages (3-4 lines, no pitch)

## Priority actions for next session

| Priority | Action |
|----------|--------|
| HIGH | Integrate `detect_intent_signals` into `pipeline.py` between ICP filter and scoring |
| HIGH | Fix or disable DebateAgent |
| MEDIUM | Define policy: missing `company_size` = REJECT? |
| MEDIUM | Add schema coherence check at pipeline startup |
| LOW | Verify Google Sheets → Airtable sync after each run |

## Conversation notes
User feedback on audits: "Ok pour 1 et 2, pour le 3 j'ai compté 31 colonnes en réalité corrige. 4 et 5 ok, je te laisse continuer avec la phase corrective stp"
→ Point 3 was about column count discrepancy. User corrected to 31, which matched the fix already applied.