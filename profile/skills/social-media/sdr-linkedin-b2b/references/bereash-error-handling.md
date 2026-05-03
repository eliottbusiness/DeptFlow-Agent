# BeReach API Error Handling for SDR LinkedIn

**Provider**: BeReach LinkedIn automation API  
**Profile**: `deptflow`  
**Date**: 2026-05-01

## Rate Limit (429)

**Trigger**: `{"error": {"status": 429, "message": "rate limit exceeded"}}`

**Response**:
```python
retry_after = response.headers.get('retryAfter', 60)  # seconds
wait_time = retry_after * (2 ** (attempt - 1))  # exponential backoff
```

**Policy**:
- Max attempts: 3
- Wait: `retryAfter × 2^(attempt-1)` seconds
- After 3 attempts: log error JSON with `linkedin_id`, `action`, `status: "ERROR"`, then skip

**Example log entry**:
```json
{
  "timestamp": "2026-05-01T12:30:00Z",
  "skill": "sdr-linkedin-b2b",
  "action": "connection_request",
  "linkedin_id": "ABC123",
  "statut": "ERROR",
  "erreur": "429 Too Many Requests — retryAfter: 60, attempt 2/3"
}
```

## Bad Gateway (502)

**Trigger**: HTTP 502 from BeReach

**Policy**:
- Max attempts: 3
- Wait: `30 × 2^(attempt-1)` seconds
- Same logging as 429

## Other 5xx Errors (500, 503, 504)

**Policy**:
- Wait 60 seconds
- Retry up to 3 times with exponential backoff
- If persistent: log and skip

## Validation Errors (400)

**Trigger**: Invalid request (e.g., missing `linkedin_id`, malformed body)

**Policy**:
- Do NOT retry
- Log error with `details`
- Skip prospect (data issue, not transient)

**Example**:
```json
{
  "timestamp": "...",
  "skill": "sdr-linkedin-b2b",
  "action": "connection_request",
  "linkedin_id": null,
  "statut": "SKIP",
  "details": "LinkedIn ID missing — check sourcing output"
}
```

## Pre-Call Guardrails

Before ANY BeReach call:

1. Read current limits:
   ```python
   limits = bereach.get_limits()  # {remaining_connections, remaining_messages}
   ```
2. If `remaining_connections == 0` → STOP sourcing entirely (per rules)
3. If `remaining_messages == 0` → STOP follow-up (but can still connect)
4. Check `DELAI_ENTRE_ACTIONS_SEC` elapsed since last action → wait if not

## Limit Reached Behavior

**Connections limit reached** (`remaining.connections == 0`):
- Sourcing: halt immediately, log `"connexions_envoyees": <limit>` in daily summary
- Follow-up: can still send messages to already-connected prospects (uses `remaining.messages`)

**Messages limit reached**:
- Follow-up: halt, log `"messages_envoyes": <limit>`
- Connections: can still send invites (separate pool)

## New Account Quirk

Observed 2026-05-01: new BeReach accounts start with `remaining.connections = 0` and `remaining.messages = 0`. The limits increment only AFTER first manual approval from BeReach team.

**Workaround**: During testing, use `SDR_FAKE_BEREACH=true` to mock limits at 1000.

## Retry Code Snippet

```python
import time, json

def call_bereach_with_retry(action_func, linkedin_id, max_attempts=3):
    for attempt in range(1, max_attempts + 1):
        try:
            return action_func()
        except BeReachError as e:
            if e.status == 429:
                wait = e.retry_after * (2 ** (attempt - 1))
                log("WARN", f"429 — waiting {wait}s (attempt {attempt}/{max_attempts})")
                time.sleep(wait)
            elif e.status == 502:
                wait = 30 * (2 ** (attempt - 1))
                log("WARN", f"502 — waiting {wait}s")
                time.sleep(wait)
            elif 500 <= e.status < 600:
                wait = 60 * (2 ** (attempt - 1))
                log("WARN", f"{e.status} — waiting {wait}s")
                time.sleep(wait)
            else:
                log("ERROR", f"Non-retryable error: {e}")
                raise
    log("ERROR", f"Max retries exceeded for {linkedin_id}")
    return None
```

## Monitoring

Check BeReach health endpoint every 30 min:
```bash
curl -s $BEREACH_API_URL/health | jq
```

Alert if `remaining.connections < 5` or `remaining.messages < 20`.
