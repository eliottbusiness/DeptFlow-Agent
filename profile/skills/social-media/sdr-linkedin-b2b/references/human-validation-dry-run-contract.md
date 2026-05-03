# Human validation dry-run contract

Session-validated dry-run sequence for the LinkedIn follow-up workflow.

## Purpose

Verify that the follow-up branch can draft a message and queue it for human validation without sending anything to LinkedIn.

## Validated sequence

1. Load the prospect from Airtable and deduplicate by `linkedin_id`.
2. Simulate `connection_accepted = true` for test purposes.
3. Generate the draft message for the prospect score (`WARM`, `HOT`, or `VERY_HOT`).
4. Run `sdr-linkedin-message-quality` on the draft.
5. If approved, write the draft to Airtable as:
   - `message_a_valider`
   - `message_quality_score`
   - `message_quality_issues`
   - `validation_statut = A_VALIDER`
   - `human_review_required = true`
   - `date_message_genere`
6. Re-read Airtable to confirm the write.
7. Do not send a LinkedIn message.
8. Do not increment message counters or set `date_message_envoye`.

## Observed successful dry-run outcome

- Prospect: Mohamed Blaiti
- Quality score: 20/20
- Validation status: `A_VALIDER`
- Human review required: `true`
- LinkedIn send: `false`
- Cron creation: `false`

## Pitfalls

- Do not call BeReach acceptance checks during a dry-run validation test when `connection_accepted` is explicitly simulated.
- Do not let a successful draft bypass `validation_statut = VALIDE`.
- If the Airtable write fails, stop before any send step and surface the error.

## Related workflow

See `references/followup-human-validation.md` for the broader send-gate rules.