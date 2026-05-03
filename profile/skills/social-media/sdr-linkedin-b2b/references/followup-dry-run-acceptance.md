# Follow-up Dry-Run Acceptance Check — Session Notes

Validated during a strict dry-run for Mohamed Blaiti.

## Key findings

### 1) BeReach limits
- `GET /me/limits` worked and the correct path remained:
  - `limits.message.daily.remaining`
- The dry-run should stop immediately if `messages_remaining <= 0`.

### 2) Airtable lookup
- The follow-up read should target prospects with:
  - `statut = CONNEXION_ENVOYEE`
  - `nb_messages_envoyes = 0`
- If `AIRTABLE_BASE_ID` is stored as a full Airtable URL, extract the canonical `app...` base ID before building the REST URL.
- Prefer `AIRTABLE_TABLE_ID` when present; it avoids name/encoding mistakes.

### 3) Acceptance detection
- `POST /chats/linkedin/find` is useful, but not sufficient alone.
- `found:false` does not prove the connection is absent; it only means that endpoint did not resolve the conversation.
- The safer confirmation path is to inspect `GET /me/linkedin/connections` first, then use chat lookup as a secondary signal.
- BeReach enforces `count <= 40` on `/me/linkedin/connections`; larger values return 422.
- Paginate via `start` in chunks of 40 when checking for a specific contact.

### 4) Dry-run message generation
- If and only if the connection is confirmed accepted, generate the WARM message locally only.
- Do not send the message in dry-run mode.
- Validate all message rules before returning a preview:
  - starts with `Mohamed,`
  - exactly 4 non-empty lines
  - no banned phrases
  - ends with the CTA question

## Practical fallback order
1. Read limits
2. Read Airtable candidates
3. Check `GET /me/linkedin/connections` with pagination (`count<=40`)
4. Check `POST /chats/linkedin/find`
5. Generate local WARM preview only if accepted is confirmed
