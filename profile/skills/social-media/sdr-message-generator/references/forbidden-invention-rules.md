# Forbidden Invention Rules

## Principe fondamental

**Aucun element metier substantiel ne doit etre invente.**

Un element est considere comme invente s'il n'est pas present dans les sources
autorisees du MessageContext (PERMITTED) et qu'il n'est pas une observation
generique non substantielle.

---

## Elements metier substantiels (liste exhaustive)

Tout element de cette liste DOIT etre present dans une source PERMITTED
pour pouvoir etre utilise dans le message.

| Element | Sources permises | Interdit si absent de |
|---------|------------------|------------------------|
| Secteur (SaaS, agence SEO, retail…) | `icp_secteurs`, `secteur_detecte` | Ces 2 champs |
| Probleme client (pipeline instable, prospection manuelle…) | `offre_problemes`, `signal_intention` | Ces 2 champs |
| Benefice (rendez-vous qualifies, 20+ prospects/jour…) | `offre_benefice`, `offre_mots_cles` | Ces 2 champs |
| Signal (automatisation, outbound, inbound…) | `signal_intention`, `observation_faible`, `icp_signaux_cles` | Ces 3 champs |
| Concurrent (nom d'outil, categorie) | `offre_concurrents` | Ce champ |
| Role/titre (CEO, Head of Sales, fondateur…) | `titre`, `icp_titres_cibles` | Ces 2 champs |
| Entreprise (nom ou type) | `entreprise` | Ce champ |
| Promesse (resultat garanti, metrique) | `offre_benefice`, `offre_mots_cles` | Ces 2 champs |
| Metrique (chiffre de performance) | `offre_benefice` | Ce champ |
| Canal (LinkedIn, cold email, outbound…) | `offre_mots_cles` | Ce champ |
| Douleur (frustration explicite) | `offre_problemes`, `signal_intention` | Ces 2 champs |
| Intention (besoin formule, projet) | `signal_intention`, `icp_signaux_cles` | Ces 2 champs |

---

## Regles specifiques par type de score

### WARM — Observation faible

WARM peut uniquement utiliser des **observations verifies et non orientees achat**.

**AUTORISE pour WARM** :
- Secteur confirme (present dans `icp_secteurs` ou `secteur_detecte`)
- Role/titre verifies (present dans `titre` ou `icp_titres_cibles`)
- Entreprise (si `entreprise` non null)
- Activite recente non-intentionnelle (post de veille, partage article) — uniquement si `observation_faible` non null

**INTERDIT pour WARM** :
- signal_intention (jamais)
- Douleur ou probleme explicite
- Besoin ou frustration formulee
- Metrique ou promesse de resultat
- Concurrent ou alternative mentionnee
- Formulation qui implique un probleme non confirme

### HOT — Signal d'activite recente

HOT necessite un signal justifie (confiance >= 0.6 et preuve).

**AUTORISE pour HOT** :
- signal_intention (si confiance >= 0.6 ET preuve_signal non null)
- observation_faible (si signal_intention non disponible)
- Problematique liee au signal (si presente dans `offre_problemes` ou `signal_intention`)
- Lien business base sur le signal (sans exageration)

**INTERDIT pour HOT** :
- signal_intention avec confiance < 0.6
- signal_intention sans preuve_signal
- Promesse de resultat non presente dans `offre_benefice`
- Metrique inventee

### VERY_HOT — Signal d'intention explicite

VERY_HOT necessite un signal d'intention clair et justifie.

**AUTORISE pour VERY_HOT** :
- signal_intention fort (confiance >= 0.6, preuve non null)
- Douleur sous-jacente liee au signal (si presente dans `offre_problemes` ou `signal_intention`)
- Valeur concrete basee sur `offre_benefice`

**INTERDIT pour VERY_HOT** :
- signal_intention absent ou confiance < 0.6
- Promesse de resultat exagere
- Accusation directe ("tu as un probleme de...")

---

## Checklist anti-invention

Pour chaque element substantiel du draft, rpondre :

```
1. Cet element figure-t-il dans une source PERMITTED ?
   → OUI : Noter la source dans evidence_used
   → NON : SUPPRIMER ou REMPLACER par formulation generique

2. Cet element est-il une interpretation excessive ?
   → OUI : Reformuler ou supprimer
   → NON : APPROUVE

3. Pour WARM : cet element implique-t-il une douleur ou un besoin ?
   → OUI : SUPPRIMER (WARM = observation uniquement)
   → NON : APPROUVE
```

---

## Checklist finale (apres generation)

- [ ] Aucun secteur qui ne soit dans `icp_secteurs` OU `secteur_detecte`
- [ ] Aucun probleme qui ne soit dans `offre_problemes` OU `signal_intention`
- [ ] Aucun benefice qui ne soit dans `offre_benefice` OU `offre_mots_cles`
- [ ] Aucun concurrent qui ne soit dans `offre_concurrents`
- [ ] Aucun role/titre qui ne soit dans `titre` OU `icp_titres_cibles`
- [ ] Aucune promesse/metrique qui ne soit dans `offre_benefice`
- [ ] Aucun canal qui ne soit dans `offre_mots_cles`
- [ ] WARM ne contient aucune mention de douleur, besoin, ou signal_intention
- [ ] HOT/VERY_HOT n'utilise pas signal_intention sans preuve_signal et confiance >= 0.6
- [ ] Aucune hypothese non documentée dans `assumptions_made`

---

## Cas limites

| Cas | Regle |
|-----|-------|
| "prospects qualifies" absent de toute source | Remplacer par "contacts" (generique non substantiel) ou supprimer |
| Secteur present mais vague ("tech") | Verifier dans `secteur_detecte` ou `icp_secteurs`. Si non present = non confirme |
| Signal present mais confiance < 0.6 | HOT → downgrade observation_faible OU CONTEXT_INSUFFICIENT si rien d'autre |
| Preuve tres courte (< 10 mots) | Reduire la certitude. Ne pas sur-interpreter. |
| Observation basee uniquement sur le titre | Risque = moyen. Hypothese a documenter dans `assumptions_made`. |
| Entreprise utilisee dans le hook | Risque = faible si `entreprise` non null. Documenter. |

---

## Mots a verifer contextuellement (pas interdits par defaut)

Ces mots sont frequemment presents dans les templates contamination.
Ils ne sont PAS interdits par defaut, mais leur utilisation doit etre
justifiee par une source PERMITTED.

| Mot | Autorise si present dans |
|-----|--------------------------|
| `pipeline` | `offre_problemes` OU `signal_intention` OU `icp_signaux_cles` |
| `rendez-vous qualifies` | `offre_benefice` OU `offre_mots_cles` |
| `stabiliser` | `offre_problemes` OU `signal_intention` |
| `bouche-a-oreille` | `signal_intention` OU `preuve_signal` |
| `agence SEO` | `icp_secteurs` OU `secteur_detecte` |
| `acquisition` | `offre_mots_cles` OU `signal_intention` |
| `prospection` | `offre_mots_cles` OU `offre_problemes` |
| `automatis*` | `signal_intention` OU `observation_faible` OU `icp_signaux_cles` |

**Si le mot n'est present dans aucune de ces sources → INTERDIT.**