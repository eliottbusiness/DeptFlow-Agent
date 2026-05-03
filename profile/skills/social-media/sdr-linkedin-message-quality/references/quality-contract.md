# Quality Contract — SDR LinkedIn Message Quality

>This document defines the hard rules that the validator must enforce independently of any self-assessment from the generator. The validator must never trust the generator's own justification of the draft. Every claim must be traced back to MessageContext.

---

## Contract: What the validator MUST verify

### Règle 1 — Aucun claim métier non justifié

**Definition**: Un claim est toute affirmation sur le prospect, son entreprise, son secteur, sa douleur, son besoin, son intention, ou son contexte.

**Règle**: Chaque claim du draft DOIT être directement justifié par un élément présent dans `MessageContext` (signal_detecte, observation_detectee, offre, secteur connu).

**Exemples de claims acceptables** (si présents dans MessageContext):
- "J'ai vu ton activité récente autour de [X]" — si X est dans les 5 derniers posts
- "Ton agence fait du SEO local" — si l'entreprise ou le titre le mentionne
- "Le problème de pipeline est critique pour toi" — si un signal douleur est détecté

**Exemples de claims NON acceptables** (meme si plausibles):
- "Je sais que ton pipeline dépend du bouche-à-oreille" — si aucun signal ne le mentionne
- "Comme beaucoup d'agences SEO" — claim de secteur général non verifiable
- "Tu cherches à automatiser ta prospection" — si aucun signal intention ne l'indique

**Mise en oeuvre**: Le validateur lit le draft ligne par ligne. Pour chaque ligne contenant un fait sur le prospect (verbe + sujet + complement), il vérifie qu'au moins un élément de MessageContext correspond. Si un claim est détecté sans correspondance, il count comme 1 point de pénalité (hard-fail si claim exagéré).

---

## Contract: What the validator must NOT assume

- Ne jamais supposer qu'un claim non présent dans MessageContext est "vraisemblable donc acceptable"
- Les mots de liaison (et, mais, car, donc, etc.) sont autorisés s'ils n'ajoutent aucun fait, secteur, douleur, besoin, intention ou contexte nouveau
- Une hypothèse non fondée = claim injustifié

---

## Contract: Distinction observation_detectee vs signal_detecte

| Type | Source | Usage autorisé |
|------|--------|---------------|
| `observation_detectee` | Élément factuel du profil ou du fil LinkedIn (titre, entreprise, secteur, activité récente sans intention) | WARM uniquement |
| `signal_detecte` | Signal d'intention, douleur, besoin, changement, benchmark (issue des posts ou du contexte ICP) | HOT et VERY_HOT uniquement |
| Aucun signal | Prospect sans activité récente ni signal détecté | WARM avec observation factuelle seulement |

**Règle absolue**: WARM ne doit jamais utiliser un `signal_detecte` de type Intention. Si le draft WARM contient une formulation qui implique une intention (prospection, recherche de solution, automation), c'est un hard-fail "WARM avec claim d'intention".

---

## Contract: "Ton équipe" rule

**Definition**: "Ton équipe" (ou toute formulation equivalente : "votre équipe", "ton équipe marketing", "ton équipe sales", etc.) suppose que le prospect manage une équipe.

**Règle**: Cette formulation est AUTORISÉE uniquement si le `title` du prospect contient explicitement un role managérial ou directionnel :
- Head of
- VP, Vice President
- Director, Directrice
- Manager, Responsable
- Lead
- Chief (CEO, CRO, CTO, CFO, CMO)
- Founder, Co-founder
- Président
- Gérant

**Exemples acceptables**:
- "Head of Growth" → "ton équipe"
- "Sales Director" → "ton équipe"
- "CEO & Founder" → "ton équipe"

**Exemples NON acceptables** (rejet automatique):
- "Account Executive" → pas d'équipe
- "Business Development Representative" → pas d'équipe
- "Consultant SEO" → pas d'équipe
- "Growth Manager" (sans autre précision) → vérifier si "Manager" dans le titre implique une équipe; dans le doute, utiliser "ton activité" ou "ton rôle"

**Formulations neutres de remplacement** (si le titre ne confirme pas une équipe) :
- "ton activité"
- "ton rôle"
- "ton poste"
- "ce que tu fais chez [entreprise]"

---

## Contract: Sales Navigator — Règle absolue

**Règle**: Aucun endpoint Sales Navigator. Jamais.

Cela inclut :
- `/search/linkedin/sales-nav`
- `/search/linkedin/sales-nav/people`
- `/search/linkedin/sales-nav/companies`
- Tout champ disponible uniquement via Sales Navigator (company employee count exact, detailed company financials, etc.)

Le compte LinkedIn utilisé est un compte free. La qualité du message ne doit jamais dépendre d'une information accessible uniquement via Sales Navigator.

---

## Contract: Validateur vs Generateur — Independence

Le validateur fonctionne de manière complètement indépendante :

1. **Le générateur produit un draft** — il peut se fier à MessageContext mais ne doit pas inventer de claims
2. **Le validateur relit le draft** — il ne fait PAS confiance au générateur; chaque claim est vérifié contre MessageContext
3. **Si un claim du draft ne correspond à aucun élément de MessageContext** → penalité ou hard-fail selon la gravité

Le validateur ne doit pas dire "le générateur a bien fait" — il doit dire "ce claim est-il justifié par MessageContext ?"

---

## Contract: Validation steps (in order)

1. **Normaliser** : line endings, trim, collapse multi-spaces
2. **Compter les lignes** : vérifier nombre requis selon score_type
3. **Vérifier structure** : ligne 1 = Prenom + virgule
4. **Vérifier longueur** : chaque ligne <= 140 chars
5. **Vérifier "ton équipe"** : si présent, verifier que title contient un role managérial
6. **Vérifier banned phrases** : liste en vigueur
7. **Tracer chaque claim** : chaque affirmation sur le prospect doit correspondre à un élément de MessageContext
8. **Vérifier distinction observation/signal** : WARM = observation_detectee uniquement; HOT/VERY_HOT = signal_detecte
9. **Scorer** : out of 20
10. **Approuver ou rejeter** : score >= 16 ET aucun hard-fail