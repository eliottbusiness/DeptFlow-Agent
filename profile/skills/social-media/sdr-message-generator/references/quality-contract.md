# Quality Contract — sdr-message-generator → sdr-linkedin-message-quality

## Contract overview

Le generateur produit un JSON structure. Le validateur verifie la conformite
de chaque element du draft aux sources autorisees du MessageContext.

---

## Donnees transmises du generateur au validateur

Le JSON complet du generateur doit contenir :

```json
{
  "action": "sdr_generate_message",
  "success": true | false,
  "draft": "string",
  "message_context_summary": { ... },
  "evidence_used": [ "source1", "source2" ],
  "assumptions_made": [ { "hypothese": "", "risque": "", "approuvee": true|false } ],
  "forbidden_assumptions_avoided": [ { "hypothese_interdite": "", "raison": "" } ],
  "line_justifications": { "ligne_X": "justification" },
  "quality_precheck": { ... },
  "human_review_required": true | false,
  "error": null | { ... }
}
```

---

## Verifications du validateur

### 1. Structure du message

| Check | Condition | Action si echec |
|-------|-----------|-----------------|
| `line_count` == `max_lignes` | 4 pour WARM, 5 pour HOT/VERY_HOT | `approved: false` |
| `max_line_length` <= 140 | Chaque ligne <= 140 caracteres | `approved: false` |
| First line = `prenom,` | Regle LinkedIn obligatoire | `approved: false` |
| Last line ends with `?` | OFFRE_CTA doit etre une question | Warning seulement |

### 2. Banned phrases

Le validateur check les forbidden phrases definies dans `message-rubric.md` :
- `je me permets`, `j'espere`, `j'espere que vous allez bien`
- `bonne journee`, `bien a vous`
- `sans effort manuel`, `ton role de`
- `en tant que`
- Langage brochure : `Je suis`, `Nous sommes`, `Notre mission`, `nous proposons`

Si une banned phrase est trouvee → `approved: false`

### 3. Detection de contamination (nouvelles)

#### 3.1 Secteur hardcode
```
SI draft contient un secteur absent de [icp_secteurs, secteur_detecte]:
  → hard-fail: "SECTEUR_HARDCODÉ: {secteur}"
  → approved: false
```

#### 3.2 Template reuse detection
```
SI draft correspond a > 70% de similarite avec un ancien template:
  → hard-fail: "TEMPLATE_REUSE_DETECTED"
  → approved: false

Anciens templates de reference:
  - "J'ai vu que tu developpes une agence SEO locale"
  - "Je peux t'aider a stabiliser ton pipeline"
  - "bouche-a-oreille" (sauf si dans preuve_signal)
  - "stabiliser" (sauf si dans offre_problemes)
```

#### 3.3 Invented element detection
```
Pour chaque element substantiel dans le draft:
  Verifier presence dans les sources PERMITTED de MessageContext
  
SI element absent de toute source:
  → hard-fail: "ELEMENT_INVENTÉ: {element}"
  → approved: false
```

#### 3.4 WARM personalization incorrecte
```
SI score_type == "WARM":
  ET draft contient une reference a:
    - signal_intention
    - douleur explicite
    - besoin formule
    - frustration
  ALORS:
    → hard-fail: "PERSONNALISATION_WARM_INCORRECTE"
    → approved: false
```

### 4. Signal confidence check (HOT/VERY_HOT)

```
SI score_type == "HOT":
  ET evidence_used ne contient pas de signal_intention ou observation_faible:
    → warning: "HOT_SANS_SIGNAL_FORT"
    → score -= 2

SI score_type == "VERY_HOT":
  ET evidence_used ne contient pas de signal_intention:
    → hard-fail: "VERY_HOT_SANS_SIGNAL_INTENTION"
    → approved: false
```

### 5. Scoring final

Score sur 20 :

| Critere | Points |
|---------|--------|
| Structure (line_count, format) | 5 |
| Personnalisation appropriee | 4 |
| Clarity of value | 4 |
| Tone (human, natural) | 3 |
| CTA faible friction | 2 |
| Zero contamination | 2 |

**Seuil d'approbation** : score >= 16 ET zero hard-fail

---

## Decision flow

```
DRAFT RECU
    │
    ▼
1. Structure check (line_count, format)
    │
    ├─ FAIL → approved: false, reason: "STRUCTURE"
    │
    ▼
2. Banned phrases check
    │
    ├─ FAIL → approved: false, reason: "BANNED_PHRASE"
    │
    ▼
3. SECTEUR_HARDCODÉ check
    │
    ├─ FAIL → approved: false, reason: "SECTEUR_HARDCODÉ"
    │
    ▼
4. TEMPLATE_REUSE check
    │
    ├─ FAIL → approved: false, reason: "TEMPLATE_REUSE"
    │
    ▼
5. ELEMENT_INVENTÉ check
    │
    ├─ FAIL → approved: false, reason: "ELEMENT_INVENTÉ"
    │
    ▼
6. PERSONNALISATION_WARM check
    │
    ├─ FAIL → approved: false, reason: "PERSONNALISATION_WARM"
    │
    ▼
7. Scoring (0-20)
    │
    ├─ score < 16 → approved: false, reason: "SCORE_TROP_FAIBLE"
    │
    ▼
8. APPROVED
```

---

## Passage de relais vers Airtable

| Generateur output | Airtable fields |
|-------------------|-----------------|
| `success: true, draft: X` | `message_a_valider=X`, `validation_statut=A_VALIDER` |
| `success: false, error: Y` | `validation_statut=GENERATION_FAILED`, `message_a_valider=null`, `message_quality_issues=JSON.stringify(error)` |
| `score` (0-20) | `message_quality_score` |
| `issues` (array) | `message_quality_issues` |
| `human_review_required: true` | `human_review_required=true` |

---

## Critere de passage en production

Le draft passe en production (envoi LinkedIn) UNIQUEMENT si :
1. `validation_statut == VALIDE` (approuve humain)
2. `message_a_valider` non null et non vide
3. `human_review_required == false` ou `human_review_required == true` ET `validation_statut == VALIDE`

Si `validation_statut == GENERATION_FAILED` :
- `send_message` step est Sauté
- Prospect reste dans Airtable avec `GENERATION_FAILED`
- En attente de traitement manuel