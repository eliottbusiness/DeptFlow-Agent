# Airtable dateTime Options — Working Format

## Problem
The Airtable Metadata API (`POST /meta/bases/{baseId}/tables` and `.../tables/{tableId}/fields`) rejects `dateTime` field creation with error:

```
422 INVALID_FIELD_TYPE_OPTIONS_FOR_CREATE
Failed schema validation: <field>.(type=dateTime).options.dateFormat must be an object,
options.dateFormat.name does not match one of the possible values,
options.timeFormat must be an object, options.timeZone is missing
```

## Root Cause
The API expects `dateFormat` and `timeFormat` to be **objects** containing BOTH a `name` key AND a `format` key — not simple strings. And `timeZone` is required.

## Working Payload Structure

```json
{
  "name": "date_creation",
  "type": "dateTime",
  "description": "Date d'ajout dans le CRM",
  "options": {
    "dateFormat": {
      "name": "iso",
      "format": "YYYY-MM-DD"
    },
    "timeFormat": {
      "name": "24hour",
      "format": "HH:mm"
    },
    "timeZone": "Europe/Paris"
  }
}
```

### Option Values

| Option | Valid `name` values | `format` pattern |
|--------|--------------------|------------------|
| `dateFormat.name` | `"iso"`, `"US"`, `"European"`, `"YYYY-MM-DD"`, `"DD/MM/YYYY"`, etc. | See Airtable docs — common: `"YYYY-MM-DD"`, `"DD/MM/YYYY"` |
| `timeFormat.name` | `"24hour"`, `"12hour"`, `"HH:mm"`, `"h:mma"`, etc. | Common: `"HH:mm"` (24h), `"h:mma"` (12h) |
| `timeZone` | IANA timezone string | `"Europe/Paris"`, `"America/New_York"`, `"UTC"`, etc. |

## What Does NOT Work

❌ `dateFormat: "YYYY-MM-DD"`  
❌ `dateFormat: {"name": "YYYY-MM-DD"}` (name and format merged)  
❌ `dateFormat: {"format": "YYYY-MM-DD"}` (missing `name`)  
❌ Omitting `timeZone` entirely  
❌ Using `timeZone: "client"` without proper format  

## Test Sequence from Session

1. Tried: `{"dateFormat": "YYYY-MM-DD", "timeFormat": "HH:mm"}` → 422 (must be objects)
2. Tried: `{"dateFormat": {"name": "YYYY-MM-DD"}, "timeFormat": {"name": "HH:mm"}}` → 422 (name values invalid)
3. Tried: `{"dateFormat": {"name": "DD/MM/YYYY"}, "timeFormat": {"name": "HH:mm"}}` → 422 (name values invalid)
4. **Succeeded:** `{"dateFormat": {"name": "iso", "format": "YYYY-MM-DD"}, "timeFormat": {"name": "24hour", "format": "HH:mm"}, "timeZone": "Europe/Paris"}` → 200

Key insight: `name` is a **style/locale identifier** (`"iso"`, `"US"`, `"European"`), while `format` is the actual strftime pattern. Both must be present.

## Base ID Extraction Pattern

If `AIRTABLE_BASE_ID` contains a full URL like:
```
https://airtable.com/appmjwbfTSI6GrO64/tblLLOlUQnjq83aeD/viww8PWEVVkMChq1S?blocks=hide
```

Extract the base ID with:
```python
import re
match = re.search(r'app([a-zA-Z0-9]+)', raw_base_id)
base_id = "app" + match.group(1)  # => "appmjwbfTSI6GrO64"
```

## References

- Official schema docs: https://airtable.com/developers/web/api/field-schema
- Options reference: https://airtable.com/developers/web/api/field-schema#options
- Discovered via iterative testing + Airtable API browser inspection (2026-05-01 session)
