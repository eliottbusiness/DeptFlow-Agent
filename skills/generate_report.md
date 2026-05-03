# Skill : Génération de Rapport

## Description

Ce skill génère un rapport quotidien de prospection : signaux détectés, connexions envoyées, DMs envoyés, réponses reçues, leads qualifiés. Le rapport est envoyé via Telegram et sauvegardé localement. Il calcule les taux de conversion par étape du funnel.

---

## Steps

### Étape 1 — Collecter les données du jour

```python
import json, yaml, os, requests
from datetime import datetime

with open('./client.yaml', 'r') as f:
    client = yaml.safe_load(f)

telegram_token = client['messaging']['telegram_bot_token']
telegram_chat_id = client['messaging']['telegram_chat_id']

date_today = datetime.now().strftime('%Y-%m-%d')

# Charger le compteur quotidien
compteur_file = f'./memory/compteur_{date_today}.json'
if os.path.exists(compteur_file):
    with open(compteur_file, 'r') as f:
        compteur = json.load(f)
else:
    compteur = {'connexions': 0, 'dms': 0, 'visites': 0}

# Charger les leads du jour
leads_file = f'./memory/leads_qualifiés_{date_today}.json'
if os.path.exists(leads_file):
    with open(leads_file, 'r') as f:
        leads = json.load(f)
else:
    leads = []

# Charger les signaux détectés
signaux_file = f'./memory/leads_{date_today}.json'
if os.path.exists(signaux_file):
    with open(signaux_file, 'r') as f:
        signaux = json.load(f)
else:
    signaux = []
```

### Étape 2 — Calculer les métriques du funnel

```python
# Métriques
nb_signaux = len(signaux)
nb_leads_qualifiés = len([l for l in leads if l.get('décision') == 'CONTACTER'])
nb_connexions = compteur.get('connexions', 0)
nb_dms = compteur.get('dms', 0)
nb_réponses = len([l for l in leads if l.get('réponse_reçue')])
nb_positifs = len([l for l in leads if l.get('réponse_positive')])
nb_relances = len([l for l in leads if l.get('relances', 0) > 0])

# Taux de conversion
taux_qualification = round(nb_leads_qualifiés / nb_signaux * 100, 1) if nb_signaux > 0 else 0
taux_connexion = round(nb_connexions / nb_leads_qualifiés * 100, 1) if nb_leads_qualifiés > 0 else 0
taux_réponse = round(nb_réponses / nb_dms * 100, 1) if nb_dms > 0 else 0
taux_positif = round(nb_positifs / nb_réponses * 100, 1) if nb_réponses > 0 else 0

print(f"[MÉTRIQUES] Signaux: {nb_signaux} | Qualifiés: {nb_leads_qualifiés} | Connexions: {nb_connexions} | DMs: {nb_dms} | Réponses: {nb_réponses}")
```

### Étape 3 — Générer le rapport texte

```python
rapport = f"""📊 <b>RAPPORT QUOTIDIEN — {date_today}</b>

🎯 <b>Funnel de prospection :</b>
├ 🔍 Signaux détectés : <b>{nb_signaux}</b>
├ ✅ Leads qualifiés : <b>{nb_leads_qualifiés}</b> ({taux_qualification}% des signaux)
├ 🤝 Connexions envoyées : <b>{nb_connexions}</b> ({taux_connexion}% des qualifiés)
├ 💬 DMs envoyés : <b>{nb_dms}</b>
├ 📩 Réponses reçues : <b>{nb_réponses}</b> ({taux_réponse}% des DMs)
├ 🔥 Réponses positives : <b>{nb_positifs}</b> ({taux_positif}% des réponses)
└ 🔄 Relances envoyées : <b>{nb_relances}</b>

📈 <b>Taux de conversion global :</b>
{nb_signaux} signaux → {nb_positifs} opportunités ({round(nb_positifs/nb_signaux*100, 2) if nb_signaux > 0 else 0}%)

⚙️ <b>Limites utilisées :</b>
├ Connexions : {nb_connexions}/{client['daily_limits']['connexions']}
└ DMs : {nb_dms}/{client['daily_limits']['dms']}
"""
```

### Étape 4 — Envoyer le rapport via Telegram

```python
response = requests.post(
    f'https://api.telegram.org/bot{telegram_token}/sendMessage',
    json={
        'chat_id': telegram_chat_id,
        'text': rapport,
        'parse_mode': 'HTML'
    }
)

if response.status_code == 200:
    print(f"[OK] Rapport envoyé via Telegram")
else:
    print(f"[ERREUR] Échec envoi Telegram: {response.status_code}")
```

### Étape 5 — Sauvegarder le rapport

```python
os.makedirs('./reports', exist_ok=True)

rapport_data = {
    'date': date_today,
    'signaux': nb_signaux,
    'leads_qualifiés': nb_leads_qualifiés,
    'connexions': nb_connexions,
    'dms': nb_dms,
    'réponses': nb_réponses,
    'positifs': nb_positifs,
    'relances': nb_relances,
    'taux_qualification': taux_qualification,
    'taux_connexion': taux_connexion,
    'taux_réponse': taux_réponse,
    'taux_positif': taux_positif
}

with open(f'./reports/rapport_{date_today}.json', 'w') as f:
    json.dump(rapport_data, f, ensure_ascii=False, indent=2)

print(f"[SAUVEGARDE] Rapport sauvegardé dans reports/rapport_{date_today}.json")
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `read_file` — Lecture des données de prospection
- `write_file` — Sauvegarde du rapport

---

## Examples

**Rapport Telegram généré :**
```
📊 RAPPORT QUOTIDIEN — 2025-05-03

🎯 Funnel de prospection :
├ 🔍 Signaux détectés : 45
├ ✅ Leads qualifiés : 18 (40% des signaux)
├ 🤝 Connexions envoyées : 15 (83.3% des qualifiés)
├ 💬 DMs envoyés : 8
├ 📩 Réponses reçues : 3 (37.5% des DMs)
├ 🔥 Réponses positives : 1 (33.3% des réponses)
└ 🔄 Relances envoyées : 4
```
