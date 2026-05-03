# DeptFlow-Agent 🚀

**Agent de prospection B2B LinkedIn automatisé** basé sur Hermes (NousResearch) et l'API Bereach.

---

## Description

DeptFlow-Agent est un système de prospection LinkedIn entièrement automatisé. Il détecte les signaux d'intention, qualifie les prospects selon votre ICP, envoie des demandes de connexion et des messages personnalisés, gère les relances, et vous alerte en temps réel via Telegram lorsqu'un prospect répond positivement.

### Architecture

```
deptflow-agent/
├── profile/
│   ├── SOUL.md              # Personnalité et règles de l'agent
│   └── config.yaml          # Configuration Hermes (LLM, outils, cron)
├── skills/
│   ├── detect_intent_signals.md    # Détection des signaux d'intention
│   ├── qualify_and_score_leads.md  # Qualification et scoring ICP
│   ├── send_connection_request.md  # Envoi des demandes de connexion
│   ├── send_prospecting_dm.md      # Envoi des DMs de prospection
│   ├── followup_and_relaunch.md    # Suivi et relances
│   └── generate_report.md          # Génération de rapports
├── context/
│   ├── prospecting_instructions.md # Instructions générales
│   ├── icp_template.md             # Template ICP (à remplir)
│   └── offer_template.md           # Template offre (à remplir)
├── client.yaml              # Configuration client (ICP, offre, limites)
├── .env.example             # Variables d'environnement (template)
└── README.md                # Ce fichier
```

---

## Prérequis

- **Hermes Agent** installé et configuré ([documentation Hermes](https://hermes.nous.ai))
- **Compte Bereach** avec clé API active ([bereach.co](https://bereach.co))
- **Bot Telegram** créé via @BotFather
- **Python 3.10+** avec les packages : `requests`, `pyyaml`
- **Compte LinkedIn** actif (le compte utilisé par Bereach)

---

## Installation

### 1. Cloner le repo

```bash
git clone https://github.com/eliottbusiness/DeptFlow-Agent.git
cd DeptFlow-Agent
```

### 2. Configurer les variables d'environnement

```bash
cp .env.example .env
# Éditer .env avec vos vraies valeurs
nano .env
```

### 3. Configurer le client

```bash
# Éditer client.yaml avec les informations de votre client
nano client.yaml
```

### 4. Créer les fichiers de contexte

```bash
# Copier et remplir les templates
cp context/icp_template.md context/icp.md
cp context/offer_template.md context/offer.md
# Éditer ces fichiers avec les informations du client
nano context/icp.md
nano context/offer.md
```

### 5. Créer les dossiers de données

```bash
mkdir -p memory reports logs
```

### 6. Lancer l'agent

```bash
hermes start --config profile/config.yaml
```

---

## Comment Dupliquer pour un Nouveau Client

1. **Forker ou copier** ce repo dans un nouveau dossier
2. **Modifier `client.yaml`** : remplir ICP, offre, limites, tokens Telegram
3. **Créer `context/icp.md`** depuis le template : définir les critères de ciblage
4. **Créer `context/offer.md`** depuis le template : définir la proposition de valeur
5. **Mettre à jour `.env`** : nouvelle clé Bereach si compte différent
6. **Tester en mode `test`** : changer `mode: test` dans client.yaml
7. **Passer en production** : changer `mode: production` et lancer l'agent

---

## Description des Skills

| Skill | Déclencheur | Description |
|-------|-------------|-------------|
| `detect_intent_signals` | Cron 8h00 (lun-ven) | Recherche posts par mots-clés, récupère likers/commentateurs, détecte changements de poste |
| `qualify_and_score_leads` | Après détection signaux | Visite les profils, calcule un score ICP 0-100, décide : contacter / attente / exclure |
| `send_connection_request` | Après qualification | Envoie les demandes de connexion avec message personnalisé selon le signal |
| `send_prospecting_dm` | Cron 12h00 (lun-ven) | Envoie les DMs aux connexions acceptées |
| `followup_and_relaunch` | Cron 12h00 et 18h00 | Lit l'inbox, escalade les réponses positives, envoie les relances J+3 et J+7 |
| `generate_report` | Cron 17h00 (vendredi) | Génère le rapport hebdomadaire et l'envoie via Telegram |

---

## Limites Recommandées LinkedIn (Anti-Ban)

| Action | Limite quotidienne recommandée | Limite absolue |
|--------|-------------------------------|----------------|
| Demandes de connexion | 20-25 | 30 |
| Messages DM | 15-20 | 25 |
| Visites de profil | 40-50 | 80 |
| Recherches | Illimité | - |

**Règles importantes :**
- Toujours espacer les actions de 2 à 5 minutes
- Ne pas automatiser plus de 4 heures consécutives
- Réduire l'activité de 50% le week-end
- En cas de restriction LinkedIn : pause de 48 heures minimum

---

## FAQ

**Q : L'agent peut-il gérer plusieurs clients simultanément ?**
R : Oui, en lançant une instance Hermes par client avec son propre `client.yaml`.

**Q : Que se passe-t-il si LinkedIn détecte l'automatisation ?**
R : L'agent respecte des limites strictes et simule un comportement humain. En cas de restriction, il s'arrête automatiquement et vous alerte via Telegram.

**Q : Comment personnaliser les messages ?**
R : Modifiez les fonctions `générer_message_connexion()` et `générer_dm_prospection()` dans les skills correspondants.

**Q : L'agent répond-il automatiquement aux prospects ?**
R : Non. En cas de réponse positive, l'agent vous alerte via Telegram et laisse la main à l'humain.

**Q : Quelle est la différence entre `profile` et `conversationUrn` dans les DMs ?**
R : Utilisez `profile` pour un premier message à un prospect. Utilisez `conversationUrn` pour répondre dans une conversation existante.

**Q : Comment tester sans risquer de spammer de vrais prospects ?**
R : Passez `mode: test` dans `client.yaml`. En mode test, les actions sont loggées mais pas exécutées.

---

## Support

Pour toute question, ouvrez une issue sur ce repo ou contactez l'équipe DeptFlow.
