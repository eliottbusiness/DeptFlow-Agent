# Reasoning Playbook — SDR LinkedIn Message Quality

>This playbook explains how to reason through a draft LinkedIn message. It shows objectives, authorized sources, risks, and forbidden traps for each line — not pre-written phrases to copy.

---

## Line-by-line reasoning

### Line 1 — Opening: `{Prenom},`

**Objective**: Capture attention naturally with just the first name and a comma.

**Authorized sources**: `prenom` field only.

**Risks**:
- Adding extra words after the comma ("Thomas, j'ai vu..." = wrong, extra content on line 1)
- Wrong capitalization of the name
- Missing the comma

**Forbidden**: Never add "cher", "salut", or any adjective before the name.

---

### Line 2 — Hook (WARM): `{observation_detectee}`

**Objective**: Show you have a legitimate, non-intrusive reason to reach out.

**Authorized sources**:
- `title` du prospect
- `company` du prospect
- `industry` si mentionné explicitement dans le profil
- Élément factuel visible dans les posts (sujet traité, expertise démontrée)
- Rien d'autre

**Reasoning pattern**:
1. Qu'est-ce que je sais de factuel sur ce prospect ? (titre, entreprise, secteur)
2. Cet élément est-il visible dans MessageContext ?
3. Puis-je le formuler sans y ajouter d'hypothèse ?

**Risks**:
- Inventer un claim de secteur non présent dans MessageContext ("agence SEO", "startup B2B")
- Interpréter une activité comme un signal d'intention (un post sur le SEO = une observation, pas une douleur)
- Utiliser "signal_detecte" dans un draft WARM

**Forbidden for WARM**: Never use signal_detecte (intention, douleur, besoin). Stick to facts only.

---

### Line 2 — Hook (HOT): Reference to `{signal_detecte}`

**Objective**: Connect to a real, specific activity or signal that shows the prospect is active on LinkedIn.

**Authorized sources**:
- Posts des 45 derniers jours (thème du post, type d'interaction)
- Signal détecté dans les activités récentes (pas une douleur, juste une activité)
- Nouveau poste ou changement de rôle visible dans le profil

**Reasoning pattern**:
1. Quel est le signal le plus récent et le plus spécifique ?
2. Ce signal est-il un fait (publication) ou une intention (recherche, douleur) ?
3. Si c'est une intention → ce n'est pas un HOT hook, c'est un VERY_HOT hook

**Risks**:
- Confusion entre activité (fait) et intention (besoin/recherche) → utiliser le bon champ
- Mentir sur le contenu d'un post qu'on n'a pas vérifié
- Transformer une simple interaction ("like") en signal d'intention fort

**Forbidden**: Don't claim a post says something specific unless you have the exact text. "J'ai vu que tu parlais de X" requires that X is actually in the post.

---

### Line 2 — Hook (VERY_HOT): Reference to `{signal_detecte}` with pain

**Objective**: Reference the specific intent signal AND cautiously name the implied problem.

**Authorized sources**:
- Signal d'intention explicite (keywords dans les posts : outbound, pipeline, automatisation, etc.)
- Douleur détectée (keywords douleur : stress, volume, manque de temps, etc.)
- Contexte business clair (secteur, modèle économique, taille)

**Reasoning pattern**:
1. Quel est le signal d'intention le plus précis ?
2. Quelle douleur probable est associée à ce signal ?
3. Puis-je nommer cette douleur sans exaggeration ? (un seul mot ou courte formulation)
4. Le lien entre signal et douleur est-il logique et justifiable par MessageContext ?

**Risks**:
- Exagérer la douleur ("ton entreprise est en crise" = trop fort)
- Inventer une douleur non présente dans les signals
- Utiliser un mot de secteur qui n'est pas dans MessageContext ("agence SEO" par exemple)

**Forbidden**: Never claim "I know your biggest problem is X" if X is not in the signal text. Keep it conditional and cautious.

---

### Line 3 — Value (all types)

**Objective**: Briefly explain what you help with, tied to a result or outcome.

**Authorized sources**:
- `OFFRE_VALEUR` (benefice principal)
- `OFFRE_NOM` (nom de l'offre — VERY_HOT only)
- Résultat vu avec des clients similaires (formulation générale acceptable)
- `probleme_resolu` de l'offre (si aligné avec le signal)

**Reasoning pattern**:
1. Quel est le benefice principal de l'offre ?
2. Ce benefice est-il aligné avec le signal/observation du prospect ?
3. Puis-je le formuler en une phrase courte, concrete, sans feature list ?

**Risks**:
- Citer des fonctionnalités ("notre IA analyse...") → trop technique, brochure
- Faire une promesse exagérée ("10x your pipeline")
- Mentions "notre solution", "notre technologie" → supprimer

**Forbidden for all**: Feature lists, "notre solution", "notre technologie", promises of specific numbers, guarantees.

---

### Line 4 (WARM) / Line 5 (HOT/VERY_HOT) — CTA

**Objective**: End with a simple, low-friction question or action.

**Authorized sources**:
- `OFFRE_CTA` (phrase CTA de l'offre)
- Question ouverte generique compatible avec le score_type
- Proposition de calendrier uniquement si le prospect est VERY_HOT avec signal fort

**Reasoning pattern**:
1. Quel CTA correspond au niveau de chaleur du prospect ?
2. Le CTA est-il proportionnel au signal ? (pas de demande de RDV pour un WARM)
3. La formulation est-elle une question unique (un seul "?") ?

**Risks**:
- Demander un RDV 30 min pour un WARM → trop aggressif
- Plusieurs questions → hard-fail
- "Est-ce que tu veux qu'on échange sur Discord / Skype" → non standard

**Forbidden**: Pushy CTAs ("book a call", "choisis un créneau", "discutons"), multiple questions, contact info in first message.

---

## Common traps to avoid

### Trap: "Ton équipe" sans role managérial

Si le prospect est "Account Executive" ou "Business Development Representative", il n'a pas d'équipe. Ne jamais écrire "ton équipe commerciale" ou "ton équipe marketing" sauf si le titre indique explicitement un role de management (Head, VP, Director, Manager, Lead, Chief, Founder).

**Fix**: "ton activité", "ton poste", "ton rôle chez [entreprise]".

### Trap: Inventer un secteur

MessageContext peut mentionner "SaaS B2B" mais le draft ne peut pas突然 claim "agence SEO" sans confirmation dans MessageContext.

**Fix**: Si le secteur n'est pas explicite, utiliser une formulation large : "ton activité", "ton contexte".

### Trap: Confondre observation et intention

Un "like" sur un post LinkedIn = activité, pas intention. Un post qui mentionne "outbound" = signal d'intention possible (à qualifier). Ne pas traiter une interaction passive comme un signal actif.

**Fix**: Pour WARM → observation factuelle. Pour HOT → activité récente. Pour VERY_HOT → intention détectée.

### Trap: Phrases finales réutilisables

Ne pas écrire une phrase qui pourrait servir telles quelles pour plusieurs prospects différents ("Je peux t'aider à créer un flux..."). Chaque message doit être spécifique au prospect.

**Fix**: Chaque phrase doit être traçable vers un élément de MessageContext. Si la phrase pourrait être copiée-collée sans changement → elle est trop générique.

---

## Validation checklist

Before accepting a draft, verify:

- [ ] Chaque claim peut être rattaché à un élément de MessageContext
- [ ] Aucune affirmation sur le secteur, la douleur, ou l'intention qui ne soit pas dans MessageContext
- [ ] "ton équipe" n'apparaît que si le titre contient un role managérial
- [ ] WARM = observation_detectee uniquement, jamais signal_detecte de type Intention
- [ ] HOT/VERY_HOT = signal_detecte
- [ ] Pas de feature list, promesse exagérée, ou garantie
- [ ] Une seule question à la fin
- [ ] Aucune URL ou lien
- [ ] Aucune signature
- [ ] Longueur <= 140 chars par ligne
- [ ] Structure de lignes conforme au score_type (4 pour WARM, 5 pour HOT/VERY_HOT)