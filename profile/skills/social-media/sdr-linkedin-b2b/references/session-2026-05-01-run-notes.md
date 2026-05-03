# Session notes — 2026-05-01

Concise carry-forward notes from the latest LinkedIn SDR session.

## Validated workflow behavior
- Follow-up remains strict dry-run until human validation explicitly sets `validation_statut = VALIDE`.
- The drafting branch writes the proposed message to Airtable for review (`message_a_valider`, `message_quality_score`, `message_quality_issues`, `human_review_required = true`) and must not send LinkedIn messages.
- Send gates should also require a non-empty `url_profil` / `profile_url`, `nb_messages_envoyes = 0`, and a passing quality score.

## Representative run result
- Real sourcing-to-CRM run completed successfully with:
  - 50 search calls
  - 100 prospects found
  - 64 after deduplication
  - 16 rejected by ICP gate
  - 18 WARM
  - 2 HOT
  - 10 VERY_HOT
  - 30 CRM records created
  - 0 connections sent
  - 0 messages sent
  - 0 errors
- The run confirmed that sourcing-only mode should stop before any LinkedIn send action.

## Intent-signal examples observed
- `WARM`: no recent post in the last 60 days
- `HOT`: recent post with an active topic or operational pain
- `VERY_HOT`: recent post with explicit pain, urgency, switch signal, or competitor/tool mention

## Practical reminders
- Deduplicate by `linkedin_id` before any action.
- Keep the canonical copy rules in `sdr-linkedin-message-quality`.
- Keep intent detection in `detect_intent_signals`.
- Use this file as a session-specific evidence note, not as the source of truth for policy.
