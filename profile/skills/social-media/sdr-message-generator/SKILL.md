# sdr-message-generator

Génère des messages LinkedIn par raisonnement structuré — sans template, sans invention.

## Principe fondamental

**Aucun élément métier substantiel ne doit être inventé.**

Sont concernés : secteur, problème, bénéfice, signal, concurrent, rôle, entreprise, promesse, métrique, canal, douleur, intention.

Tout élément du message doit être traçable jusqu'à une source dans `MessageContext`.

## Input

```yaml
prospect_record: object    # Enregistrement Airtable complet (fields.*)
scoring_result: object     # Résultat scoring (WARM/HOT/VERY_HOT + signals)
client_config: object     # Config client YAML (offre, ICP)
```

## Output

JSON complet avec :
- `success`: true | false
- `draft`: `<schema_only — no example text>`
- `evidence_used`: sources réelles utilisées
- `assumptions_made`: hypothèses faibles documentées
- `forbidden_assumptions_avoided`: hypothèses interdites évitées
- `line_justifications`: logique par ligne (pas de texte final)
- `human_review_required`: bool
- `error`: détails si échec

## Règles absolues

1. `prenom` manquant → `ERROR PRENOM_MANQUANT` (fatal, no fallback)
2. `offre_cta` manquant → `ERROR OFFRE_CTA_MANQUANT` (fatal, no fallback)
3. WARM + `signal_intention` → `ERROR PERSONNALISATION_WARM_INCORRECTE`
4. HOT sans `signal_intention` (confiance < 0.6) → downgrade observation_faible
5. VERY_HOT sans `signal_intention` → `ERROR SIGNAL_REQUIS_POUR_VERY_HOT`
6. Contexte insuffisant → `ERROR CONTEXT_INSUFFICIENT` (pas de fallback minimal)
7. Aucun élément métier hors des sources autorisées

## Types de signal

```
observation_faible (WARM uniquement) :
  - Secteur confirmé dans icp_secteurs ou secteur_detecte
  - Rôle/titre vérifiable
  - Activité non-intentionnelle (post de veille)
  - Jamais de douleur, besoin, ou intention d'achat

signal_intention (HOT/VERY_HOT uniquement) :
  - Requiert : preuve_signal non null AND confiance >= 0.6
  - Jamais pour WARM
```

## Intégration

Ce skill est appelé par `sdr_linkedin_followup.md` step `generate_message`.
Son output est validé par `sdr-linkedin-message-quality` avant write Airtable.

Si `success=false` :
- `validation_statut` dans Airtable = `GENERATION_FAILED`
- `message_a_valider` = null (vide)
- `human_review_required` = true
- `send_message` step sauté (pas d'envoi possible)

## Tests

```bash
python -m pytest tests/test_message_generator.py -v
```

13 tests unitaires requis avant passage à l'étape 2.