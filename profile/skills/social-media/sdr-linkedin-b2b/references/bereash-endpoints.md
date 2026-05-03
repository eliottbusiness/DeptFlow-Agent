# BeReach API v2 — Endpoint Mapping & Field Formats

## Verified Endpoints (v2.0.0)

### Working Endpoints

| Method | Endpoint | Purpose | Body Format | Response Notes |
|--------|----------|---------|-------------|----------------|
| GET | `/me/limits` | Read daily quotas | — | `limits.*.daily.*` structure |
| POST | `/search/linkedin/people` | Prospect search | See section "Search Payload" | Returns `items[]`, `hasMore` |
| POST | `/connect/linkedin/profile` | Send connection | `{"profile": "url_or_urn"}` | 200/201/202 = success |
| POST | `/message/linkedin` | Send message | `{"profile": "...", "message": "..."}` | — |

### Deprecated / Removed Endpoints (404)

| Old Endpoint | New Replacement | Old Body | New Body |
|--------------|----------------|----------|----------|
| `POST /actions/linkedin/connect` | `POST /connect/linkedin/profile` | `{"linkedin_id":"..."}` | `{"profile":"..."}` |
| `POST /actions/linkedin/message` | `POST /message/linkedin` | `{"linkedin_id":"...","message":"..."}` | `{"profile":"...","message":"..."}` |
| `GET /actions/linkedin/message` | N/A (use POST) | — | — |

## Search Payload Field Formats (CRITICAL)

Fields validated via 422 error responses:

| Field | Type | Required? | Format / Example | Notes |
|-------|------|-----------|------------------|-------|
| `title` | **string** | no | `"Founder"` or `"Founder,CEO,Fondateur"` | Comma-separated OR single value. NOT an array. |
| `keywords` | **string** | no | `"SEO,agence SEO,référencement naturel"` | Comma-separated keywords. Significantly improves relevance. NOT an array. |
| `industry` | **array** | no | `["SEO","Marketing digital"]` | Array of strings. |
| `location` | **array** | no | `["France"]` | Array of strings (country or city). |
| `profileLanguage` | **array** | no | `["fr"]` | Array of language codes. |
| `count` | int | yes (default 25) | `25` | Max per page. |
| `start` | int | yes (default 0) | `0`, `25`, `50` | Pagination offset. |
| `hasMore` | boolean | — | `true` / `false` | Returned in response, not sent. |

**422 Error Examples:**
```
"title: Invalid input: expected string, received array"
"keywords: Invalid input: expected string, received array"
"location: Invalid input: expected array, received string"
```

**Takeaway:** When in doubt, send `title` and `keywords` as plain strings (comma-separated). Send `industry`, `location`, `profileLanguage` as arrays.

## Pagination Response Format

```json
{
  "items": [
    {
      "name": "John Doe",
      "headline": "Founder & CEO at ExampleCo",
      "profileUrl": "https://www.linkedin.com/in/john-doe",
      "profileUrn": "urn:li:fsd_profile:ACo...",
      "publicIdentifier": "john-doe-12345",
      "currentPositions": [{"company": "ExampleCo"}],
      "recentPosts": [...],
      "about": "...",
      "location": "Paris, France"
    }
  ],
  "hasMore": true,
  "total": 127  // optional
}
```

**Pagination logic:**
- Request `count=25`, increment `start` by 25 each page
- Continue while `hasMore === true` AND `len(all_prospects) < max_prospects` (safety cap)
- Stop on first successful connection regardless of remaining pages

## Connection Request Validation

**Endpoint:** `POST /connect/linkedin/profile`

**Strict body:**
```json
{
  "profile": "{bereach_profile_target}"
}
```

**Internal adapter contract:** `build_bereach_profile_target(prospect)`

```json
{
  "bereach_profile_target": "",
  "source": "profileUrl|url_profil|profileUrn|publicIdentifier|none",
  "valid": true,
  "error": ""
}
```

**Source priority:**
1. `profileUrl` (preferred, normalized)
2. `url_profil` (normalized)
3. `profileUrn`
4. `publicIdentifier`
5. `none` → `ERROR_MISSING_PROFILE_TARGET`

**Normalization rules for LinkedIn URLs:**
- remove query parameters
- remove trailing slash
- force the canonical `www.linkedin.com` host and keep the `https://www.linkedin.com/in/...` or `https://www.linkedin.com/pub/...` form

**Forbidden in the connection body:** `linkedin_id`, `message`, `note`, `text`, `content`.

**Operational rule:** if `valid=false`, skip the prospect and do not call BeReach.

## Message Request Validation

**Endpoint:** `POST /message/linkedin`

**Required body:**
```json
{
  "profile": "https://www.linkedin.com/in/username",
  "message": "Formatted message text with exactly 5 lines..."
}
```

**Same `build_bereach_profile_target` contract applies.** If `valid=false` → skip with ERROR log.

## Limits Response — Full Example

```json
{
  "success": true,
  "multiplier": 3,
  "limits": {
    "connection_request": {
      "daily": {
        "current": 4,
        "limit": 90,
        "remaining": 86
      }
    },
    "message": {
      "daily": {
        "current": 0,
        "limit": 300,
        "remaining": 300
      }
    }
  },
  "creditsUsed": 0,
  "retryAfter": 0,
  "_meta": { ... }
}
```

**Key:** Always parse via `limits.connection_request.daily.remaining`, never `remaining.connections`.

## Deduplication ID Priority Chain

Airtable dedup field: `linkedin_id`. Derive from prospect item:

```python
linkedin_id = (
    item.get("profileUrn") or           # "urn:li:fsd_profile:ACo..."
    item.get("publicIdentifier") or    # "john-doe-12345"
    item.get("profileUrl") or          # "https://www.linkedin.com/in/john-doe"
    None
)
```

If `linkedin_id` is `None` → log WARNING, skip dedup check (but still check Airtable via profileUrl fallback if needed).

**Airtable query formula:**
```
filterByFormula=({linkedin_id} = 'VALUE')
```
URL-encode the `linkedin_id` value.

## Keywords-Based Search Strategy (Proven)

**Problem:** Broad `title="Founder"` returns 0% ICP match (architecture, real estate, HR, etc.).

**Solution:** Add semantic filtering via `keywords` string.

**Working example (5 results with SEO relevance):**
```json
{
  "title": "Founder",
  "keywords": "SEO,agence SEO,référencement naturel,marketing local,agence marketing digital,consultant SEO,SEO local",
  "profileLanguage": ["fr"],
  "count": 25,
  "start": 0
}
```

**Result:** First hit: "Yves ATTIAS — CEO & Founder at Agence YATEO | Votre partenaire e" (agence digitale). This dramatically increases WARM/HOT yield.

**Recommendation:** Always include targeted `keywords` when ICP sectors are specific (SEO, marketing, consulting). Broad ICP (e.g., "Technology") may not need keywords.

## Diagnostic Checklist (When Things Break)

- [ ] 404 on `/search/linkedin/people`? → Verify OpenAPI `~/.hermes/docs/apibereach.json` has the path
- [ ] 422 on search? → Check field types: `title` and `keywords` must be **strings**, `industry`/`location` must be **arrays**
- [ ] Limits show 0 but quota available? → Check JSON path: use `limits.connection_request.daily.remaining`, not `remaining.connections`
- [ ] No results from search? → Try without `industry`/`location` filters first; add `keywords` to narrow
- [ ] Connection 400/422? → Verify body is exactly `{"profile": "<profileUrl>"}` with no extra fields
- [ ] Airtable 404? → Verify `AIRTABLE_BASE_ID` and `AIRTABLE_TABLE_NAME` (must be table **name**, not ID)
- [ ] Dedup not working? → Verify URL encoding of `linkedin_id` in filter formula

## Revision History

- **v1.0.0 (2026-05-01)** — Initial mapping from OpenAPI v2.0.0 and live test validation.
