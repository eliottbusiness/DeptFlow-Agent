# Follow-up Human Validation Workflow — SDR LinkedIn B2B

This reference defines the Airtable validation gate and send-after-validation contract. The copy rules themselves live in `sdr-linkedin-message-quality`; we avoid duplicating them here.

1. Generate and quality-check the draft message.
2. Queue the message in Airtable for manual approval.
3. Send only when Airtable `validation_statut = VALIDE`.

## Airtable fields used

New fields added to `SDR_Prospects`:

- `validation_statut` — singleSelect
  - `NON_REQUIS`
  - `A_VALIDER`
  - `VALIDE`
  - `REJETE`
  - `A_REECRIRE`
- `message_a_valider` — multilineText
- `message_quality_score` — number, precision 0
- `message_quality_issues` — multilineText
- `date_message_genere` — dateTime
- `date_validation_humaine` — dateTime
- `valide_par` — singleLineText
- `validation_notes` — multilineText
- `human_review_required` — checkbox

## Recommended Airtable view

Name: `Messages à valider`

Suggested filters:
- `validation_statut = A_VALIDER`
- `human_review_required = checked`
- `nb_messages_envoyes = 0`

Suggested visible fields:
- `prenom`
- `nom`
- `entreprise`
- `titre`
- `score`
- `signal_detecte`
- `message_a_valider`
- `message_quality_score`
- `message_quality_issues`
- `validation_statut`
- `validation_notes`
- `url_profil`

## Human validation flow

1. Alex generates the message.
2. Alex validates it with `sdr-linkedin-message-quality` (canonical copy rubric).
3. Alex writes the draft to Airtable.
4. Alex sets `validation_statut = A_VALIDER`.
5. The human receives a mobile notification if Airtable/Slack/email automation is configured.
6. The human reviews the message on mobile.
7. The human sets:
   - `VALIDE` to approve
   - `REJETE` to discard
   - `A_REECRIRE` to request another draft
8. Alex sends the LinkedIn message only when `validation_statut = VALIDE`.

## Send-after-validation branch

For the send branch:
- read prospects where `validation_statut = VALIDE`
- require `nb_messages_envoyes = 0`
- require `message_a_valider` not empty
- require `url_profil` not empty
- send via `POST /message/linkedin`
- on success:
  - set `statut = MESSAGE_ENVOYE`
  - set `date_message_envoye`
  - set `nb_messages_envoyes = 1`
  - set `human_review_required = false`

## Dry-run acceptance test

For tests, the workflow should:
- generate the draft
- run the quality skill (`sdr-linkedin-message-quality`)
- write `message_a_valider`
- set `validation_statut = A_VALIDER`
- not send any LinkedIn message

If the prospect has not accepted the connection, do not send anything and do not force a real LinkedIn call.

## Notification strategy

Recommended default: `Airtable mobile` or `Slack DM` or `email`, depending on workspace configuration.

Reason: the actual notification channel depends on the connected apps and Airtable automation settings.

## Operational rule

Never call `POST /message/linkedin` unless Airtable explicitly contains `validation_statut = VALIDE`.
