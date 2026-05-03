# Reasoning Playbook

Ce document montre la methode de raisonnement pour chaque type de score.
**Aucun texte final n'est present dans ces exemples.**
Chaque exemple documente : l'intention rhetorique, les sources consultees,
les hypotheses interdites, et la decision du critique.

---

## Exemple A — WARM avec secteur confirme

### Contexte
```yaml
prenom: Marie
score: WARM
observation_faible: "secteur SaaS"  # from prospect.industry
titre: Head of Sales
secteur_detecte: "SaaS"
icp_secteurs: ["SaaS", "B2B Software"]
signal_intention: null
offre_cta: "Tu veux qu'on explore comment ça pourrait marcher ?"
offre_problemes: ["Prospection manuelle chronophage"]
```

### Raisonnement

```
INTENTION RHETORIQUE :
  Ligne 1 : Salutation standard avec prenom
  Ligne 2 : Observation du role/secteur — pas de douleur, pas de besoin
  Ligne 3 : Valeur generic — pas de promesse exagerée
  Ligne 4 : CTA faible friction

SOURCES AUTORISEES CONSULTEES :
  - observation_faible = "secteur SaaS" → source: secteur_detecte
  - secteur SaaS present dans icp_secteurs → AUTORISE
  - offre_cta present → UTILISABLE

HYPOTHESES POSEES (faibles) :
  H1: "Le fait que Marie soit Head of Sales dans le SaaS suffit pour une observation de role"
     Justification: titre=Head of Sales, secteur_detecte=SaaS
     Risque: FAIBLE
     Critique: APPROUVEE

HYPOTHESES INTERDITES EVITEES :
  HI1: "Le pipeline de Marie est instable"
     Raison: probleme absent de offre_problemes ET absent de signal_intention
     Decision: NON FAITE

  HI2: "Marie cherche des solutions d'acquisition"
     Raison: intention absente (signal_intention=null, confiance=null)
     Decision: NON FAITE

  HI3: "Les agencies SaaS ont des problemes de prospection"
     Raison: generalisation non autorisee sans donnee prospect
     Decision: NON FAITE

DECISION CRITIQUE :
  ligne_2 (hook): APPROUVEE
    - Observation basee sur secteur confirme (SaaS dans icp_secteurs)
    - Role Head of Sales mentionnable car titre present
    - Aucune douleur, aucun besoin, aucune intention supposee

  ligne_3 (valeur): APPROUVEE
    - Formulation generic liee a offre_problemes[0]
    - Aucun vocabulaire sectoriel специфик (pas de "SEO", "pipeline", "rendez-vous")

  ligne_4 (CTA): APPROUVEE
    - offre_cta exact, non modifie
```

---

## Exemple B — HOT avec signal_intention (confiance suffisante)

### Contexte
```yaml
prenom: Thomas
score: HOT
signal_intention: "automatisation commerciale"
preuve_signal: "Post LinkedIn 2026-04-28: 'On cherche a automatiser notre acquisition B2B...'"
confiance: 0.72
type_signal: "Intention"
offre_cta: "Tu veux qu'on explore comment ça pourrait marcher ?"
offre_problemes: ["Prospection manuelle chronophage"]
offre_mots_cles: ["acquisition", "prospection", "outbound"]
```

### Raisonnement

```
INTENTION RHETORIQUE :
  Ligne 1 : Salutation standard
  Ligne 2 : Mention du signal d'activite recente — observation, pas accusation
  Ligne 3 : Lien contexte business — pas de promesse exageree
  Ligne 4 : CTA faible friction

SOURCES AUTORISEES CONSULTEES :
  - signal_intention = "automatisation commerciale" → source: preuve_signal (post 2026-04-28)
  - confiance = 0.72 >= 0.6 → signal_intention UTILISABLE
  - offre_mots_cles contient "acquisition" → mot autorise

HYPOTHESES POSEES (faibles a moderes) :
  H1: "Le post de Thomas montre que l'acquisition B2B est un sujet pour son equipe"
     Justification: preuve_signal mentionne "acquisition B2B", offre_mots_cles contient "acquisition"
     Risque: MOYEN
     Critique: APPROUVEE avec reserve (source = preuve_signal, pas une interpretation)

  H2: "L'equipe de Thomas travail sur l'acquisition"
     Justification: titre absent, mais preuve_signal mentionne "notre equipe"
     Risque: MOYEN
     Critique: APPROUVEE (faiblement lie au signal)

HYPOTHESES INTERDITES EVITEES :
  HI1: "Thomas a un probleme de pipeline instable"
     Raison: "pipeline instable" absent de offre_problemes ET preuve_signal
     Decision: NON FAITE

  HI2: "Thomas perd des leads faute d'automatisation"
     Raison: promesse/mетрика inventee (pas dans offre_benefice ni preuve_signal)
     Decision: NON FAITE

  HI3: "L'agence SEO de Thomas manque de rendez-vous"
     Raison: secteur "agence SEO" absent de toute source
     Decision: NON FAITE

DECISION CRITIQUE :
  ligne_2 (hook): APPROUVEE
    - signal_intention base sur preuve_signal (post 2026-04-28)
    - confiance 0.72 >= 0.6 → autorise
    - phrasé en observation (pas en accusation de probleme)

  ligne_3 (contexte business): APPROUVEE
    - "acquisition" present dans offre_mots_cles → autorise
    - Lien faible au signal (pas de promesse ni de douleur)
    - Risque moyen, acceptable

  ligne_4 (CTA): APPROUVEE
    - offre_cta exact
```

---

## Exemple C — VERY_HOT avec signal_intention fort

### Contexte
```yaml
prenom: Pierre
score: VERY_HOT
signal_intention: "pipeline commercial instable"
preuve_signal: "Post 2026-04-25: 'Notre pipeline depend trop du bouche-a-oreille, faut trouver autre chose...'"
confiance: 0.85
type_signal: "Intention"
offre_cta: "Tu veux qu'on explore comment ça pourrait marcher ?"
offre_problemes: ["Prospection manuelle chronophage", "Manque de volume qualify"]
offre_benefice: "20+ prospects/jour qualifies, 15%+ taux reponse"
```

### Raisonnement

```
INTENTION RHETORIQUE :
  Ligne 1 : Salutation
  Ligne 2 : Signal d'intention explicite — plus direct que HOT
  Ligne 3 : Douleur sous-jacente — liee au signal
  Ligne 4 : Valeur concrete — benefice autorise
  Ligne 5 : CTA

SOURCES AUTORISEES CONSULTEES :
  - signal_intention = "pipeline instable" → source: preuve_signal
  - "bouche-a-oreille" present dans preuve_signal → autorise
  - offre_benefice contient "prospects qualifies" → mot autorise
  - confiance = 0.85 >= 0.6 → signal_intention UTILISABLE

HYPOTHESES POSEES (moderees) :
  H1: "La dependance au bouche-a-oreille peut devenir un frein"
     Justification: "bouche-a-oreille" present dans preuve_signal
     Risque: MODERE
     Critique: APPROUVEE (source = preuve_signal,pas une supposition)

  H2: "Un flux plus regulier de prospects qualifies serait benefique"
     Justification: "prospects qualifies" dans offre_benefice
     Risque: FAIBLE
     Critique: APPROUVEE

HYPOTHESES INTERDITES EVITEES :
  HI1: "Pierre va perdre des clients"
     Raison: promesse/métrique inventee
     Decision: NON FAITE

  HI2: "L'agence de Pierre va faire faillite"
     Raison: promesse exaggerée, aucune source
     Decision: NON FAITE

  HI3: "Pierre a un probleme de SEO local"
     Raison: secteur "SEO local" absent de toute source
     Decision: NON FAITE

DECISION CRITIQUE :
  ligne_2 (hook signal): APPROUVEE
    - signal_intention base sur preuve_signal
    - confiance 0.85 (eleve)
    -VERY_HOT autorise un hook plus direct que HOT

  ligne_3 (douleur): APPROUVEE (avec reserves)
    - "bouche-a-oreille" tire directement de preuve_signal
    - Formulation en possibilite ("peut devenir"), pas en accusation
    - Risque modere, acceptable car source == preuve_signal

  ligne_4 (valeur): APPROUVEE
    - "prospects qualifies" dans offre_benefice
    - Pas de promesse de metrique exacte (pas de "20+" dans le texte)
```

---

## Exemple D — WARM avec contexte insuffisant (ERREUR)

### Contexte
```yaml
prenom: Jean
score: WARM
observation_faible: null
signal_intention: null
titre: null
entreprise: null
secteur_detecte: null
icp_secteurs: []
offre_cta: "Tu veux qu'on en parle ?"
```

### Raisonnement

```
INTENTION RHETORIQUE :
  Ligne 1 : Salutation (prenom disponible)
  Ligne 2 : Impossible — aucune donnee pour construire une observation

IMPOSSIBILITE DE GENERER :
  Analyse ligne par ligne :
    - secteur : absent (secteur_detecte=null, icp_secteurs=[])
    - role/titre : absent (titre=null)
    - entreprise : absent (entreprise=null)
    - observation_faible : absent (null)
    - signal_intention : absent (null) mais de toute facon WARM ne peut pas l'utiliser

  Hypothese possible "J'ai vu ton profil, Jean" :
    - PROBLEME : Aucune donnee sur le profil. "J'ai vu" = invention.
    - "ton profil" = aucune specificite. Rien dans le contexte ne justifie cette phrase.
    - Raison: inventé car aucune source disponible

  Conclusion : CONTEXT_INSUFFICIENT

ERREUR RETOURNEE :
  {
    "success": false,
    "error": {
      "code": "CONTEXT_INSUFFICIENT",
      "message": "Pas assez de contexte pour generer sans inventer. Donnees disponibles: prenom=Jean, score=WARM. Donnees absentes: titre, entreprise, secteur, observation_faible, signal_intention."
    },
    "human_review_required": true
  }

REGLE APPLIQUEE :
  AUCUN FALLBACK. Pas de "Jean, j'ai vu ton profil". Pas de "Je peux t'aider".
  Le generateur echoue proprement au lieu d'inventer.
```

---

## Exemple E — HOT avec signal_intention mais confiance insuffisante

### Contexte
```yaml
prenom: Sophie
score: HOT
signal_intention: "automatisation"
preuve_signal: "Post LinkedIn (texte court, 5 mots)"
confiance: 0.45
type_signal: "Intention"
offre_cta: "Tu veux qu'on explore ?"
```

### Raisonnement

```
INTENTION RHETORIQUE :
  Score = HOT, mais signal_intention non utilisable (confiance < 0.6)

VERIFICATION signal_intention :
  confiance = 0.45 < 0.6 → signal_intention NON UTILISABLE
  reason: confiance insuffisante pour garantir la fiabilite du signal

OPTIONS :
  1. Downgrade vers observation_faible (si disponible)
     - observation_faible: null
     - Non disponible

  2. Observation faible depuis role/titre/entreprise
     - titre: null, entreprise: null
     - Non disponible

  3. Contexte insuffisant
     - Aucune donnee exploitable

ERREUR RETOURNEE :
  {
    "success": false,
    "error": {
      "code": "CONTEXT_INSUFFICIENT",
      "message": "score=HOT mais signal_intention a confiance=0.45 (< 0.6) et observation_faible=null. HOT necessite un signal justifie avec confiance >= 0.6 ou une observation_faible."
    },
    "human_review_required": true
  }

REGLE APPLIQUEE :
  HOT ne peut pas se contenter d'une observation generique sans signal.
  Si ni signal_intention (confiance >= 0.6) ni observation_faible disponible → ERREUR.
```

---

## Checklist de raisonnement (pour le generateur)

Pour chaque message genere, verifier :

```
1. SIGNAL SELECTIONNE
   [ ] signal_intention utilise ? Si oui:
       [ ] preuve_signal non null
       [ ] confiance >= 0.6
   [ ] observation_faible utilise ? Si oui:
       [ ] source disponible (secteur, role, entreprise)

2. LIGNES JUSTIFIEES
   Pour chaque ligne:
       [ ] Element substantiel present dans une source PERMITTED ?
       [ ] Si oui → noter la source dans evidence_used
       [ ] Si non → reformuler ou supprimer

3. HYPOTHESES
   [ ] Aucune hypothese non documentée
   [ ] Risque de chaque hypothese evalue (faible/moyen/eleve)
   [ ] Hypotheses a risque eleve → non approuvees

4. WARM CHECK
   [ ] score=WARM ?
       [ ] signal_intention utilise ? → ERREUR PERSONNALISATION_WARM_INCORRECTE
       [ ] Douleur explicite ? → Non autorise

5. CONTEXT INSUFFICIENT
   [ ] Si aucune donnee disponible pour lignes 2+ → ERREUR CONTEXT_INSUFFICIENT
```