# MessageContext — Schema complet

## Objectif

Ce document définit le schema du `MessageContext` passe au generateur `sdr-message-generator`. Chaque champ a une provenance definie. Le generateur ne doit jamais utiliser une donnee qui ne provient pas de ce schema.

---

## Champs obligatoires

**Le generateur ECHOUE si un de ces champs est absent ou null.**

| Champ | Type | Description | Erreur si absent |
|-------|------|-------------|-----------------|
| `prenom` | string | Prenom du prospect. No fallback. | `PRENOM_MANQUANT` |
| `score` | WARM \\| HOT \\| VERY_HOT | Niveau de score. | `INVALID_SCORE` |
| `offre_cta` | string | CTA defini dans la config client. No fallback. | `OFFRE_CTA_MANQUANT` |

---

## Champs conditionnels

| Champ | Type | Condition | Comportement si absent |
|-------|------|-----------|------------------------|
| `signal_intention` | string \\| null | Requis pour HOT/VERY_HOT si confiance >= 0.6 | Si absent pour HOT → downgrade observation_faible. Si absent pour VERY_HOT → `SIGNAL_REQUIS_POUR_VERY_HOT` |
| `observation_faible` | string \\| null | Disponible pour WARM et HOT (fallback) | Optionnel. Si absent et score=WARM → `CONTEXT_INSUFFICIENT` |
| `preuve_signal` | string \\| null | Requis si `signal_intention` est utilise | Si absent avec signal_intention → `SIGNAL_REQUIS_POUR_VERY_HOT` |
| `confiance` | float \\| null | Requis si `signal_intention` est utilise. Seuil: >= 0.6 | Si < 0.6 → signal_intention non utilisable, downgrade ou erreur |

---

## Champs optionnels

| Champ | Type | Description |
|-------|------|-------------|
| `titre` | string \\| null | Job title du prospect |
| `entreprise` | string \\| null | Nom de l'entreprise |
| `secteur_detecte` | string \\| null | Secteur depuis prospect.industry |
| `linkedin_url` | string \\| null | Pour skip detection |
| `localisation` | string \\| null | Ville/pays |
| `type_signal` | string \\| null | Intention \\| Activite \\| Contexte \\| Aucun signal recent |
| `sous_type_intention` | string \\| null | Sous-type (ex: Comparaison/benchmark) |
| `sous_type_activite` | string \\| null | Sous-type (ex: Publication thematique) |

---

## Champs client config (provenance: YAML)

| Champ | Provenance YAML | Description |
|-------|----------------|-------------|
| `offre_nom` | `offre.nom_offre` | Nom de l'offre |
| `offre_problemes` | `offre.probleme_resolu[]` | Liste des problemes resous |
| `offre_benefice` | `offre.benefice_principal` |Benefice principal |
| `offre_mots_cles` | `offre.mots_cles_intention[]` | Mots-cles intention |
| `offre_concurrents` | `offre.concurrents_ou_alternatives[]` | Concurrents |
| `icp_secteurs` | `icp.secteurs_activite[]` | Secteurs cibles |
| `icp_signaux_cles` | `icp.signaux_intention_cles[]` | Signaux cles ICP |
| `icp_titres_cibles` | `icp.titres_cibles[]` | Titres cibles |

---

## Sources autorisees (PERMITTED)

Chaque element du message doit pouvoir etre retrace a une source. Les sources permises par categorie :

```yaml
PERMITTED:
  secteurs:
    - icp_secteurs        # Secteurs definis dans l'ICP client
    - secteur_detecte     # Secteur detecte depuis prospect.industry

  problemes:
    - offre_problemes      # Problemes resous par l'offre
    - signal_intention     # Probleme mentionne dans le signal

  signaux:
    - signal_intention     # Signal d'intention (si confiance >= 0.6 et preuve)
    - observation_faible   # Observation non-intentionnelle
    - icp_signaux_cles     # Signaux cles de l'ICP

  mots_cles_valeur:
    - offre_benefice       # Benefice principal
    - offre_mots_cles     # Mots-cles de l'offre

  concurrents:
    - offre_concurrents   # Concurrents configures

  roles:
    - titre               # Titre du prospect
    - icp_titres_cibles   # Titres cibles de l'ICP

  entreprises:
    - entreprise          # Entreprise du prospect
```

---

## Contraintes LinkedIn

| Champ | Valeur | Description |
|-------|--------|-------------|
| `max_lignes` | 4 pour WARM, 5 pour HOT/VERY_HOT | Nombre de lignes |
| `max_caracteres_ligne` | 140 | Longueur max par ligne |
| `forbidden_phrases` | list[string] | Phrases interdites (depuis message-rubric.md) |

---

## Validation du schema

Avant generation, le generateur doit verifier :
1. `prenom` non null et non vide
2. `score` dans (WARM, HOT, VERY_HOT)
3. `offre_cta` non null et non vide

Si une de ces verifications echoue → erreur fatale immediate.