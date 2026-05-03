# Type de signal — convention CRM

Objectif:
- Normaliser la lecture de `signal_detecte` avec un champ `type_signal` simple et stable.
- Ajouter une granularité métier supplémentaire pour les signaux d’activité.

Valeurs normalisées de `type_signal`:
- `Intention`
- `Activité`
- `Contexte`
- `Aucun signal récent`

Règles de classification:
- `Intention` : le prospect exprime une douleur, une urgence, un besoin, un changement de solution, une évaluation, une migration, ou une recherche de performance directement liée à l'offre.
- `Activité` : le prospect publie ou interagit récemment sur un thème lié à l'offre, sans intention d'achat explicite.
- `Contexte` : seul le titre, la bio, ou l'expertise générale suggère une adéquation, sans activité récente exploitable.
- `Aucun signal récent` : aucun signal exploitable dans la fenêtre de lookback.

Sous-types recommandés:
- Pour `type_signal = Intention`, alimenter `sous_type_intention`.
- Pour `type_signal = Activité`, alimenter `sous_type_activite`.
- Pour `type_signal = Contexte` ou `Aucun signal récent`, laisser les sous-types vides sauf cas exceptionnel de backfill.

`signal_detecte` reste un champ descriptif libre.

Format conseillé pour enrichir le texte:
- `Intention: "...extrait..."`
- `Activité: "...extrait..."`
- `Contexte: "...extrait..."`

Conventions de backfill:
1. Garder `signal_detecte` comme champ descriptif libre.
2. Alimenter `type_signal` avec une valeur normalisée et courte.
3. Alimenter `sous_type_intention` uniquement quand `type_signal = Intention`.
4. Alimenter `sous_type_activite` uniquement quand `type_signal = Activité`.
5. Backfiller tous les prospects existants de façon cohérente avant de changer les règles de scoring ou de message.
6. Si un sous-type manque, laisser le champ vide plutôt que d’inventer une catégorie.
