# Airtable schema mutation pitfalls (Hermes session note)

Session finding:
- Attempting to create a field via the Airtable Metadata API returned:
  - HTTP 403
  - `INVALID_PERMISSIONS_OR_MODEL_NOT_FOUND`
- Root cause: the available PAT had `schema.bases:read` but not `schema.bases:write`.

Implications:
- Reading base/table/field metadata works.
- Creating or updating schema objects (tables, fields, views in metadata endpoints) requires a token with `schema.bases:write` and access to the target base.

Practical fallback:
1. Verify the token scopes before attempting schema mutations.
2. If only read scope is present, stop and ask for a higher-scope PAT or perform the change manually in Airtable UI.
3. After the field exists, resume with record-level backfill using the records API.

Useful error signature:
- `403 INVALID_PERMISSIONS_OR_MODEL_NOT_FOUND` on `/v0/meta/bases/{baseId}/tables/{tableId}/fields`
