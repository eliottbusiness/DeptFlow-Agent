# Airtable target resolution for SDR LinkedIn B2B

Session-verified rule:
- Prefer `AIRTABLE_TABLE_ID` when available.
- Fallback to `AIRTABLE_TABLE_NAME` only when URL-encoded.
- Never build Airtable REST URLs with a raw, unencoded table name.

Known-good values from the session:
- `AIRTABLE_TABLE_NAME=SDR_Prospects`
- `AIRTABLE_TABLE_ID=tblVifAXWdYt0TWtL`
- `AIRTABLE_BASE_ID` accessible via Metadata API and REST read tests.

Recommended resolver pattern:
1. If `AIRTABLE_TABLE_ID` exists, use `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${AIRTABLE_TABLE_ID}`
2. Else use `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${encodeURIComponent(AIRTABLE_TABLE_NAME)}`
3. Reuse the same resolver for:
   - dedup GETs with `filterByFormula`
   - POST record creation
   - PATCH record updates

Dedup formula to keep stable:
- `filterByFormula={linkedin_id}="<linkedin_id>"`

Diagnostic clue:
- If Metadata API and REST reads work but writes still 404, the most likely issue is code still using the raw table name or a different URL path than the read path.
