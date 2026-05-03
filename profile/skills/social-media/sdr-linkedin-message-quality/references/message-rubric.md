# Message Rubric — SDR LinkedIn Message Quality

>This reference file contains the live rubric, banned phrases, quality contract, and rewrite notes for the `sdr-linkedin-message-quality` skill.

---

## Score types: `observation_detectee` vs `signal_detecte`

This distinction is enforced in the quality contract and validated independently by the validator.

| score_type | Champ à utiliser | Contenu |
|------------|-----------------|---------|
| WARM | `observation_detectee` | Élément factuel du profil ou du fil (titre, entreprise, secteur, activité sans intention). Jamais de signal de type Intention. |
| HOT | `signal_detecte` | Référence à une activité récente (post, interaction). Sobre et factuelle. |
| VERY_HOT | `signal_detecte` | Signal d'intention précis (douleur, besoin, recherche de solution). Lien possible avec `OFFRE_NOM`. |

**Règle absolue**: Un draft WARM ne doit jamais utiliser un `signal_detecte` de type Intention. Si le générateur produit un draft WARM avec un claim qui implique un besoin ou une intention (prospection, automatisation, douleur), le validateur le reject comme "WARM avec claim d'intention injustifié".

---

## Claim rule (remplace "aucun mot inventé")

**Règle**: Aucun claim métier ou prospect non justifié.

Un claim est toute affirmation sur le prospect, son entreprise, son secteur, sa douleur, son besoin, son intention, ou son contexte.

Chaque claim du draft DOIT être directement traçable vers un élément de MessageContext. Cela inclut les affirmations implicites (ex: "le problème de pipeline" suppose une douleur non vérifiée).

**Mots de liaison autorisés** (sans ajout de fait) : et, mais, car, donc, comme, si, quand, avec, sans, pour. Ces mots sont acceptables à condition qu'ils n'introduisent aucun secteur, douleur, besoin, intention ou contexte absent de MessageContext.

**Exemples de claims acceptables** (si dans MessageContext) :
- "J'ai vu ton activité récente autour de [X]" — X est dans les 5 derniers posts
- "Ton agence fait du SEO local" — titre ou secteur le mentionne

**Exemples de claims non acceptables** (meme plausibles) :
- "Je sais que ton pipeline dépend du bouche-à-oreille" — non présent dans MessageContext
- "Comme beaucoup d'agences SEO" — claim de secteur général non vérifiable

---

## "Ton équipe" rule

Cette formulation est autorisée uniquement si le `title` du prospect contient explicitement un rôle managérial ou directionnel.

**Rôles autorisant "ton équipe"** : Head of, VP, Vice President, Director, Directrice, Manager, Responsable, Lead, Chief (CEO, CRO, CTO, CFO, CMO), Founder, Co-founder, Président, Gérant.

**Si le titre ne confirme pas une équipe** (Account Executive, BDR, SDR, Consultant, etc.) → utiliser "ton activité", "ton poste", "ton rôle", ou "ce que tu fais chez [entreprise]".

**Hard-fail** : Si "ton équipe" (ou equivalent) apparaît dans un draft pour un prospect dont le titre n'indique pas de rôle managérial → reject.

---

## Sales Navigator — Règle absolue

Aucun endpoint Sales Navigator. Jamais. Le compte LinkedIn est free. La qualité du message ne doit jamais dépendre d'une information accessible uniquement via Sales Navigator.

---

## Scoring rubric

Score out of 20:
- Structure respected: 5
- Personalization relevance: 4
- Value clarity: 4
- Human and natural tone: 3
- Simple CTA: 2
- No banned phrases or risky patterns: 2

Approve only when:
- score >= 16
- no hard-fail rule is triggered
- all claims are justified by MessageContext
- WARM uses only observation_detectee (no intention claim)
- "ton équipe" only if title confirms a managerial role

---

## Hard-fail rules

Reject if any of these are true:
- wrong line count for the declared score type
- first line is not exactly first name plus comma
- any line exceeds 140 characters
- contains a link or URL
- contains a signature or sign-off block
- contains more than one question
- contains a banned phrase
- contains a pushy CTA such as booking 30 minutes or choosing a calendar time
- makes guarantees or exaggerated claims
- reads like a cold email, brochure, or feature list
- WARM contains an intention-type claim (signal_detecte de type Intention)
- "ton équipe" / "votre équipe" used for a non-managerial title
- any claim in the draft is not traceable to an element of MessageContext

---

## Banned phrases

Never allow these phrases in the final message, in any capitalization:
- je me permets
- j'espère
- j'espère que vous allez bien
- bonne journée
- bien à vous
- sans effort manuel
- en tant que
- je suis
- nous sommes
- notre mission
- nous proposons
- je connais bien ton entreprise (si non justifié par MessageContext)
- je sais que tu cherches (si non justifié par un signal intention dans MessageContext)
- ton équipe (sauf si title confirme un rôle managérial)
- votre équipe (sauf si title confirme un rôle managérial)

---

## LinkedIn style constraints

- first line must be `Prenom,`
- one idea only
- one question only
- no first-message links
- no signature
- no aggressive pitch
- no product feature list
- max 140 characters per line
- French first for this project
- no claim on sector, pain, need, or intent without MessageContext justification

---

## Drafting framework: AIDA + PAS

1. First name only, followed by a comma.
2. Personalized hook: reference to `observation_detectee` (WARM) or `signal_detecte` (HOT/VERY_HOT).
3. Value sentence: short, concrete, tied to a result or OFFRE_VALEUR.
4. Simple question or low-friction CTA. Never ask directly for a 30-minute call.

Adapt the tone by score type:
- WARM: simple observation, light relevance, no overfitting, no intent claims.
- HOT: sober reference to recent activity, activity-led hook, no intent signal.
- VERY_HOT: precise intent signal and likely pain, with a direct value bridge and OFFRE_NOM if relevant.

---

## Rewrite patterns

### WARM — use `observation_detectee` only

```text
{prenom},
J'ai vu {observation_detectee}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

Note: `observation_detectee` = factuel uniquement. Exemples valides:
- "que tu gères une agence SEO"
- "ton poste de Sales Director"
- "ton activité dans le SaaS B2B"

### HOT — use `signal_detecte` (activité recente)

```text
{prenom},
J'ai vu ton activité récente autour de {signal_detecte}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

Note: `signal_detecte` = activité recente (un post, une interaction). Pas une douleur. Exemples valides:
- "un post sur la prospection LinkedIn"
- "une publication sur l'outbound"

### VERY_HOT — use `signal_detecte` (intention/douleur)

```text
{prenom},
J'ai vu ton signal autour de {signal_detecte}.
{OFFRE_VALEUR}
{OFFRE_CTA}
```

Note: `signal_detecte` = signal d'intention ou douleur. Exemples valides:
- "un post sur l'automatisation de la prospection"
- "une discussion sur les volume de pipeline"

---

## Quality notes

- Prefer calm and direct language.
- Do not over-personalize with unverified assumptions.
- Keep the CTA short and frictionless.
- Adapt French accents naturally when rewriting.
- If the draft is borderline, favor safety and reduce aggressiveness.
- The validator must independently verify every claim against MessageContext — never trust the generator's self-assessment.

---

## Maintenance notes

When the project evolves:
- update the banned phrase list here
- update the quality contract (`references/quality-contract.md`) if validation rules change
- update the reasoning playbook (`references/reasoning-playbook.md`) if authoring guidelines change
- patch the skill if the output schema or scoring changes
- keep the JSON output schema stable unless explicitly requested