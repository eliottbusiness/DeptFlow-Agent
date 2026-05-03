# BeReach Diagnostic Procedure (Session-born Technique)

## When to Use
- Endpoints return 404 unexpectedly
- Limits appear as zero despite fresh quota
- New API key seems inactive
- Uncertain which endpoint version is active

## Step-by-Step

### 1. Check Raw Limits Response
```bash
curl -H "Authorization: Bearer $BEREACH_API_KEY" https://api.bereach.ai/me/limits
```
Expected structure:
```json
{
  "success": true,
  "limits": {
    "connection_request": {"daily": {"remaining": 86, ...}},
    "message": {"daily": {"remaining": 300, ...}}
  }
}
```
If `remaining.connections` or `remaining.messages` fields are missing → parsing error. Use correct paths:
- `limits.connection_request.daily.remaining`
- `limits.message.daily.remaining`

### 2. Load OpenAPI Spec
```bash
cat ~/.hermes/docs/apibereach.json | jq '.paths | keys'
```
Verify endpoint paths and HTTP methods (GET vs POST).

### 3. Cross-check Skill Endpoints
For each endpoint used in `sdr_linkedin_sourcing.md` and `sdr_linkedin_followup.md`:
- Extract path and method
- Lookup in OpenAPI `paths` object
- Confirm method (GET/POST) matches
- If not found → search for likely replacement in OpenAPI by keyword (e.g., `connect` → `/connect/linkedin/profile`, `message` → `/message/linkedin`)

### 4. Safe Live Tests (NO destructive calls)
Only test read-only endpoints:
- `GET /me/limits` (should return 200)
- `GET /search/linkedin/parameters` (should return 200 or 400 if params required)

**DO NOT TEST:**
- `/connect/linkedin/profile`
- `/message/linkedin`
- Any `/actions/linkedin/*`
- Any `/visit/`, `/collect/`, `/like/`, `/reply/`

### 5. Confirm Body Schema
Check OpenAPI `requestBody` schema for the endpoint. Common corrections:
- Old: `{"linkedin_id": "<id>"}` → New: `build_bereach_profile_target(prospect)` → `{"profile": "<bereach_profile_target>"}`
- Old: `GET /actions/linkedin/message` → New: `POST /message/linkedin` with body `{"profile": "...", "message": "..."}`

### 6. Update Skills
Apply corrections to:
- `sdr_linkedin_sourcing.md` (connection adapter + body)
- `sdr_linkedin_followup.md` (message endpoint + body + profile_url validation)

### 7. Re-test Limits Parsing
After fixing paths, re-run step 1 to confirm quotas are now read correctly.

## Quick Reference: Known Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `remaining.connections = 0` | Using wrong JSON path | Use `limits.connection_request.daily.remaining` |
| 404 on `/actions/linkedin/connect` | Deprecated endpoint | Replace with `/connect/linkedin/profile` (POST) |
| 404 on `/actions/linkedin/message` | Deprecated endpoint | Replace with `/message/linkedin` (POST) |
| `profile_url` undefined | Target not built via canonical adapter | Add `build_bereach_profile_target(prospect)` and use `bereach_profile_target` |
| Connection sent but not logged | Airtable update failure | Check Airtable API key, table name, field names |

## Files to Consult
- OpenAPI: `~/.hermes/docs/apibereach.json`
- Skill reference: `social-media/sdr-linkedin-b2b/references/bereash-endpoints.md`
- Error handling: `social-media/sdr-linkedin-b2b/references/bereash-error-handling.md`
