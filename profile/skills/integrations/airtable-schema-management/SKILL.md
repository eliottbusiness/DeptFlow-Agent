---
name: airtable-schema-management
description: Creates and validates Airtable table schemas via the Metadata API. Handles field type options correctly (especially dateTime), adds missing fields without deleting existing ones, and produces a JSON summary report.
version: 1.0.0
author: Hermes Agent + Deptflow SDR
license: MIT
metadata:
  hermes:
    tags: [airtable, schema, metadata-api, crm, sdr, automation]
    homepage: https://airtable.com/developers/web/api/field-schema
    related_skills: []
---

# Airtable Schema Management

This skill automates the creation and validation of Airtable table schemas for SDR LinkedIn automation agents. It ensures the `SDR_Prospects` table has exactly the 18 required fields with correct Airtable types and options, using the Metadata API.

## When to Use

- First-time setup of the SDR_Prospects CRM table
- Validating that an existing table matches the expected schema
- Adding missing fields to an existing table without disrupting data
- Integrating Airtable as a persistence layer for agent workflows

## Prerequisites

The profile's `.env` must define:
- `AIRTABLE_API_KEY` — Personal Access Token with `schema.bases:read` and `schema.bases:write` scopes
- `AIRTABLE_BASE_ID` — Base identifier (usually an `appXXXXX` string; if stored as full URL, it will be extracted automatically)
- `AIRTABLE_TABLE_NAME` — Target table name (e.g., `SDR_Prospects`)

## Schema Definition (18 Required Fields)

| Field Name | Type | Description |
|------------|------|-------------|
| `linkedin_id` | singleLineText | Unique LinkedIn identifier |
| `prenom` | singleLineText | Prospect first name |
| `nom` | singleLineText | Prospect last name |
| `titre` | singleLineText | Current job title |
| `entreprise` | singleLineText | Company name |
| `url_profil` | url | LinkedIn profile URL |
| `localisation` | singleLineText | City, region, or country |
| `score` | singleSelect | WARM, HOT, VERY_HOT, REJETE |
| `signal_detecte` | multilineText | Detected signal description |
| `statut` | singleSelect | Pipeline state (6 options) |
| `date_creation` | dateTime | Record creation timestamp |
| `date_connexion_envoyee` | dateTime | Connection request sent |
| `date_connexion_acceptee` | dateTime | Connection accepted |
| `date_message_envoye` | dateTime | First message sent |
| `message_envoye` | multilineText | Sent message content |
| `notes` | multilineText | Manual notes |
| `source` | singleSelect | sourcing_automatique or manuel |
| `nb_tentatives_connexion` | number | Connection attempts count (integer) |
| `nb_messages_envoyes` | number | Messages sent count (integer) |

## Critical: dateTime Field Options

**Do NOT use simple strings** for `dateFormat`/`timeFormat`. The Metadata API requires nested objects:

```json
{
  "type": "dateTime",
  "description": "...",
  "options": {
    "dateFormat": {"name": "iso", "format": "YYYY-MM-DD"},
    "timeFormat": {"name": "24hour", "format": "HH:mm"},
    "timeZone": "Europe/Paris"
  }
}
```

Valid `dateFormat.name` values: `"iso"`, `"US"`, `"European"`, `"YYYY-MM-DD"`, etc.  
Valid `timeFormat.name` values: `"24hour"`, `"12hour"`, `"HH:mm"`, `"h:mma"`, etc.  
`timeZone` must be an IANA timezone string (e.g., `"Europe/Paris"`, `"America/New_York"`).

See `references/airtable-field-options.md` for the complete list of valid option values from Airtable documentation.

## Workflow

1. **Validate config** — Read `.env`, abort if any required variable missing
2. **Verify API access** — GET `/meta/bases/$BASE_ID/tables`; handle 401/403/404
3. **Resolve base ID** — Extract `appXXXXX` from URL if needed
4. **Check existing table** — Find by exact name match, record `tableId` and existing fields
5. **Compute diff** — Compare required 18 fields against existing set
6. **Create or update** — Either create full table, or add missing fields one-by-one
7. **Verify & report** — Re-fetch schema, ensure all fields present, emit JSON summary

The operation is **idempotent**: running it multiple times is safe; existing fields are never touched.

## Output

On success:
```
✨ ÉTAPE AIRTABLE — Table CRM SDR prête.
```

Plus a JSON summary printed to stdout:
```json
{
  "action": "airtable_schema_setup",
  "table_name": "SDR_Prospects",
  "table_created": true,
  "fields_created": ["linkedin_id", "prenom", ...],
  "fields_already_existing": [],
  "missing_fields_after_setup": [],
  "status": "SUCCESS",
  "details": "Table créée — 18 champ(s) ajouté(s)."
}
```

On failure: prints `❌ ERREUR — ...` and exits with code 1.

## Pitfalls

- **dateTime/date options format** — The Metadata API strictly validates. Using string values directly (e.g., `"YYYY-MM-DD"`) without wrapping in `{"name": ...}` causes 422 `INVALID_FIELD_TYPE_OPTIONS_FOR_CREATE`. Always provide both `name` and `format` keys inside `dateFormat` and `timeFormat` objects.
- **Base ID extraction** — `AIRTABLE_BASE_ID` is often a full URL. Do not pass the URL directly to API endpoints; extract the `appXXXXX` substring first.
- **Timezone required** — `dateTime` fields require a `timeZone` option. Omission causes validation error.
- **Field name uniqueness** — Airtable treats field names case-insensitively for duplicates. Use consistent snake_case.
- **singleSelect choices must be non-empty** — Provide at least one choice object.
- **number precision** — For integers, set `"precision": 0`.
- **Idempotency check mandatory** — Always check existing fields before POST; duplicate creation attempts fail with 422.
- **API rate limits** — Airtable imposes rate limits. For tables with many fields, add small delays (≈0.2s) between field-creation calls in production scripts.
- **Error inspection** — 422 responses contain the exact field and option path that failed. Use this to iteratively correct the payload.
- **Field type mapping** — Remember: `singleLineText` vs `multilineText`, `url` for links, `dateTime` for timestamps with zone, `date` for dates only.

## Related

- Airtable Metadata API reference: https://airtable.com/developers/web/api/field-schema
- Field options deep-dive: https://airtable.com/developers/web/api/field-schema#options
- Discovery pattern used in this session: iterative testing of dateTime option variants, error message pattern matching, timezone inclusion experiments, format string vs object resolution.
