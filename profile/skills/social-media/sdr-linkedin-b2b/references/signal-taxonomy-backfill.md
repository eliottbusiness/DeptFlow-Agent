# Signal taxonomy backfill — SDR LinkedIn B2B

Session note:
- The CRM now has a normalized `type_signal` field plus two subtype fields:
  - `sous_type_intention`
  - `sous_type_activite`

Normalized `type_signal` values:
- `Intention`
- `Activité`
- `Contexte`
- `Aucun signal récent`

Recommended subtype mapping:
- `type_signal = Intention` → populate `sous_type_intention`
- `type_signal = Activité` → populate `sous_type_activite`
- `type_signal = Contexte` or `Aucun signal récent` → leave subtype fields empty unless there is a deliberate backfill exception

Current activity subtype seed set:
- `Publication thématique`
- `Interaction sur post`
- `Veille secteur`
- `Partage de contenu`
- `Commentaire d’expertise`

Backfill rule:
- Prefer leaving a subtype blank over inventing one from weak evidence.
- Use `signal_detecte` as the descriptive text; use `type_signal` for normalized routing.

Observed session result:
- Existing `Activité` records were backfilled to `sous_type_activite = Publication thématique` as the conservative default.
