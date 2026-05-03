# LinkedIn Safety Rules for SDR Automation

**Critical**: Violating these leads to account restrictions or bans.

## Connection Requests

### NEVER Include a Note
- ❌ Bad: `{"linkedin_id": "123", "message": "Let's connect"}`
- ✅ Correct: `{"profile": "<bereach_profile_target>"}`
- Connection invites must remain note-free and use the internal adapter target.

### Daily Limit
- Hard cap: 30 invites/day (per BeReach doc, new accounts)
- Exceeding triggers "invite limit reached" + account review

## Messaging

### Rate Limits
- 100 messages/day max (configurable via `LIMITE_MESSAGES_JOUR`)
- Additional implicit limit: ~30 messages/hour to avoid "spammy behavior" flags

### Content Restrictions
- No promotional links in first message
- No "buy now", "free trial", "discount" in opening
- No more than 3 sequence messages without response (risk: "too many messages" block)
- Always include opt-out: "Si tu n'es pas intéressé, dis-moi et j'arrête."

## Profile Actions

Do NOT automate (will trigger ban):
- Profile updates
- Endorsements
- Following/unfollowing
- Group joins
- Post likes/comments (outside messaging)

## Detection & Mitigation

**Warning signs**:
- "Alerts" → "You're near your weekly invite limit" → stop invites for 24h
- "This member can't accept invites" → check `linkedin_id` health before sending
- Message "Not sent" → account restricted, halt all actions

**Mitigation**:
- Warm up account: 10 invites/day first week, then ramp to 30
- Rotate `user_agent` strings (handled by BeReach)
- Add 2-3 day pause after 25 consecutive invites sent

## Sources

- LinkedIn User Agreement §8.2 (Automation)
- BeReach Documentation v1.0 — Safety Guidelines
- Observed 2026-05-01 on new account `deptflow`: invites limited to 0 until manual approval
