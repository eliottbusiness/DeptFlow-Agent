# SDR_Prospects — Signal Taxonomy Notes

Session-derived convention for LinkedIn SDR work.

## Recommended 2-level taxonomy

Use `type_signal` as the broad level:
- `Intention`
- `Activité`
- `Contexte`
- `Aucun signal récent`

Use `sous_type_intention` for the precise intent when `type_signal = Intention`:
- `Diagnostic / audit`
- `Comparaison / benchmark`
- `Recherche de solution`
- `Visibilité / acquisition`
- `Préparation AI Overviews`
- `Veille stratégique IA`
- `Optimisation de contenu`
- `Recommandation / bouche-à-oreille`

## Practical mapping observed in this session

- Posts about audits, poor traffic quality, or site diagnosis → `Diagnostic / audit`
- Posts comparing channels/tools or asking which option to choose → `Comparaison / benchmark`
- Posts offering or seeking concrete help / ebook / accompaniments / solution language → `Recherche de solution`
- Posts about being found at the right moment / demand capture / acquisition visibility → `Visibilité / acquisition`
- Posts about AI Overviews readiness → `Préparation AI Overviews`
- Posts about AI regulation, strategic AI news, or platform shifts → `Veille stratégique IA`
- Posts about content structure, micro-intents, or post-performance optimization → `Optimisation de contenu`
- Posts about referrals, recommendations, or word-of-mouth programs → `Recommandation / bouche-à-oreille`

## Airtable implementation note

- `type_signal` was kept as the broad field.
- `sous_type_intention` was added as a separate single-select field for precision.
- In this session, `sous_type_intention` was only populated when `type_signal = Intention`.

## Backfill reminder

When backfilling existing prospects:
1. Normalize `score` and `signal_detecte`.
2. Set the broad `type_signal` first.
3. If broad type is `Intention`, derive `sous_type_intention` from the post/signals text.
4. Clear `sous_type_intention` when the prospect is not in the `Intention` bucket.
