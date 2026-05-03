# Session 2026-05-02 — CRM cap + Airtable schema observations

## Bounded batch behavior
- When the operator asks to stop after N prospects are in the CRM, stop as soon as `N` new Airtable records have been created.
- Do not continue sourcing, enrichment, or messaging after the cap is reached.
- Maintain the usual pre-action checks for each prospect until the cap is hit: limits, Airtable dedupe by `linkedin_id`, ICP fit, score, and delay between actions.

## Verified Airtable schema details
The following fields existed and accepted writes in the current `SDR_Prospects` table:
- `linkedin_id`
- `prenom`
- `nom`
- `titre`
- `entreprise`
- `url_profil`
- `localisation`
- `score` with single-select values: `WARM`, `HOT`, `VERY_HOT`, `REJETE`
- `type_signal`
- `sous_type_intention`
- `sous_type_activite`
- `signal_detecte`
- `statut`
- `date_creation`
- `notes`
- `source`
- `nb_tentatives_connexion`
- `nb_messages_envoyes`
- `validation_statut` with single-select values: `NON_REQUIS`, `A_VALIDER`, `VALIDE`, `REJETE`, `A_REECRIRE`
- `human_review_required` checkbox

## Useful operational note
- A GET filter on `linkedin_id` is the right dedupe check before create.
- A successful create can be followed by a GET verification to confirm the record exists and the filter key matches exactly.
