# CRM Schema Audit — DeptFlow SDR

Canonical procedure for verifying and fixing CRM schema drift between Python code and live Airtable.

## The Problem Pattern

A CRM schema drift happens when:
- `CRM_COLUMNS` (Python constant in `models.py`) has N columns
- `result_to_row()` (in `crm.py`) returns M values
- Google Sheets `A:X` range letter corresponds to K columns
- Airtable live schema has L fields

These four numbers frequently diverge silently. When they do:
- Some Airtable columns never get written (gap between M and N)
- The spreadsheet range truncates data or leaves blank columns (K ≠ N)
- Column names in code don't match field names in Airtable (e.g. `url_linkedin` vs `url_profil`)

**Result from this session:**
- Airtable live: **31 fields**
- `CRM_COLUMNS`: **26 columns** (5 missing: `score`, `signal_detecte`, `date_message_envoye`, `message_envoye`, `nb_messages_envoyes`)
- `result_to_row()`: **21 values** (5 slots were hardcoded empty strings `""`)
- Spreadsheet range: `A:U` (21 cols) instead of `A:AE` (31 cols)
- Field name: `url_linkedin` in code, `url_profil` in Airtable

## Verification Procedure

### Step 1 — Read Live Airtable Schema

```bash
BASE_ID="appXXXXXXXX"
curl -s "https://api.airtable.com/v0/meta/bases/${BASE_ID}/tables" \
  -H "Authorization: Bearer ${AIRTABLE_API_KEY}" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for t in data['tables']:
    if 'SDR_Prospects' in t['name']:
        fields = t['fields']
        print(f'Table: {t[\"name\"]} | ID: {t[\"id\"]}')
        print(f'Field count: {len(fields)}')
        for i, f in enumerate(fields, 1):
            print(f'  {i:2d}. {f[\"name\"]} ({f[\"type\"]})')
"
```

### Step 2 — Count Python Artifacts

```python
# In src/sdr_ai/models.py — count CRM_COLUMNS
import re
with open("src/sdr_ai/models.py") as f:
    models = f.read()
cols_match = re.search(r'CRM_COLUMNS: list\[str\] = \[(.*?)\]', models, re.DOTALL)
if cols_match:
    cols = re.findall(r'"([^"]+)"', cols_match.group(1))
    print(f"CRM_COLUMNS: {len(cols)} columns")

# In src/sdr_ai/crm.py — count result_to_row return values
with open("src/sdr_ai/crm.py") as f:
    crm = f.read()
row_match = re.search(r'def result_to_row\(result.*?\).*?return \[(.*?)\]', crm, re.DOTALL)
if row_match:
    items = [l.strip() for l in row_match.group(1).split('\n') if l.strip()]
    print(f"result_to_row: {len(items)} values")
```

### Step 3 — Check Spreadsheet Range References

```bash
grep -n "A2:U\|A:U\|A2:AE\|A:AE" src/sdr_ai/crm.py
grep -n "url_linkedin\|url_profil" src/sdr_ai/crm.py src/sdr_ai/models.py
```

## Fix Checklist

When the audit reveals drift, apply in this order:

1. **Update `CRM_COLUMNS`** in `models.py` — add missing field names to match Airtable
2. **Update `result_to_row()`** in `crm.py` — ensure it returns exactly `len(CRM_COLUMNS)` values, in the same order
3. **Extend spreadsheet ranges** — `A2:U` → `A2:AE` and `A:U` → `A:AE` for writes
4. **Fix field name mismatches** — `CRM_COLUMNS.index("url_linkedin")` → `CRM_COLUMNS.index("url_profil")` (and all callers)
5. **Verify the 31-col match** — run the inline Python check (see session 2026-05-02)

## Inline Verification Script

```python
from src.sdr_ai.models import CRM_COLUMNS
from src.sdr_ai.crm import result_to_row
# Build a dummy ScoringResult and verify len(row) == len(CRM_COLUMNS)
```

If `len(row) != len(CRM_COLUMNS)`: the schema is broken — do not run any CRM write operations.

## Airtable Metadata API vs Schema Write Permissions

The `AIRTABLE_API_KEY` in this environment has `schema.bases:read` but **NOT** `schema.bases:write`. This means:
- Reading the live schema: works via `GET /v0/meta/bases/{base_id}/tables`
- Creating/modifying fields via API: returns 403 — must be done manually in the Airtable UI
- Record CRUD (create, read, update, delete): works fine

When adding missing columns, create them manually in Airtable first, then update `CRM_COLUMNS` and `result_to_row()` in code.

## Related Files

- `src/sdr_ai/models.py` — `CRM_COLUMNS` constant
- `src/sdr_ai/crm.py` — `result_to_row()`, `append_results()`, `update_connection_sent()`, `LocalCSVCRM`
- `src/sdr_ai/pipeline.py` — calls `append_results()` and checks return value