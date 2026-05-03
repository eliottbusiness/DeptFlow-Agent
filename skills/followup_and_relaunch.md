# Skill : Suivi et Relances

## Description

Ce skill lit l'inbox LinkedIn via Bereach, identifie les conversations nécessitant une relance (J+3, J+7), envoie les messages de suivi, escalade vers l'humain via Telegram en cas de réponse positive, et ajoute à la blacklist en cas de refus.

---

## Steps

### Étape 1 — Lire l'inbox LinkedIn

```python
import json, yaml, os, requests, time
from datetime import datetime, timedelta

with open('./client.yaml', 'r') as f:
    client = yaml.safe_load(f)

api_key = client['bereach_api_key']
headers = {
    'Content-Type': 'application/json',
    'x-api-key': api_key
}

telegram_token = client['messaging']['telegram_bot_token']
telegram_chat_id = client['messaging']['telegram_chat_id']

# Lire l'inbox
# Endpoint : POST /chats/linkedin
chats_resp = requests.post(
    'https://api.bereach.co/chats/linkedin',
    headers=headers,
    json={'count': 50, 'start': 0}
)
conversations = chats_resp.json().get('data', [])
print(f"[INBOX] {len(conversations)} conversations trouvées")
```

### Étape 2 — Analyser les réponses

```python
mots_refus = ['non merci', 'pas intéressé', 'stop', 'désabonner', 'ne plus', 'arrêtez']
mots_intérêt = ['intéressé', 'oui', 'pourquoi pas', 'dites-moi', 'en savoir plus', 'rdv', 'appel', 'disponible']

réponses_positives = []
réponses_négatives = []

for conv in conversations:
    dernier_message = conv.get('lastMessage', {})
    texte = dernier_message.get('text', '').lower()
    expéditeur = dernier_message.get('sender', {})
    est_prospect = expéditeur.get('isProspect', False)

    if not est_prospect:
        continue  # Ignorer nos propres messages

    if any(mot in texte for mot in mots_intérêt):
        réponses_positives.append(conv)
    elif any(mot in texte for mot in mots_refus):
        réponses_négatives.append(conv)

print(f"[ANALYSE] {len(réponses_positives)} réponses positives, {len(réponses_négatives)} refus")
```

### Étape 3 — Escalader les réponses positives vers Telegram

```python
def envoyer_telegram(token, chat_id, message):
    requests.post(
        f'https://api.telegram.org/bot{token}/sendMessage',
        json={'chat_id': chat_id, 'text': message, 'parse_mode': 'HTML'}
    )

for conv in réponses_positives:
    profil = conv.get('participant', {})
    nom = profil.get('firstName', '') + ' ' + profil.get('lastName', '')
    titre = profil.get('headline', '')
    profile_url = profil.get('profileUrl', '')
    dernier_message = conv.get('lastMessage', {}).get('text', '')

    message_telegram = f"""🔥 <b>RÉPONSE POSITIVE !</b>

👤 <b>{nom}</b>
💼 {titre}
🔗 {profile_url}

💬 Message reçu :
<i>{dernier_message}</i>

➡️ À toi de jouer !"""

    envoyer_telegram(telegram_token, telegram_chat_id, message_telegram)
    print(f"[ESCALADE] Alerte Telegram envoyée pour {nom}")
```

### Étape 4 — Ajouter les refus à la blacklist

```python
blacklist_file = './memory/blacklist.json'
if os.path.exists(blacklist_file):
    with open(blacklist_file, 'r') as f:
        blacklist = json.load(f)
else:
    blacklist = []

for conv in réponses_négatives:
    profil = conv.get('participant', {})
    profile_url = profil.get('profileUrl', '')
    if profile_url and profile_url not in blacklist:
        blacklist.append(profile_url)
        print(f"[BLACKLIST] Ajouté : {profile_url}")

with open(blacklist_file, 'w') as f:
    json.dump(blacklist, f, ensure_ascii=False, indent=2)
```

### Étape 5 — Identifier les leads à relancer

```python
date_today = datetime.now()
leads_à_relancer = []

leads_files = [f for f in os.listdir('./memory') if f.startswith('leads_qualifiés_')]

for leads_file in leads_files:
    with open(f'./memory/{leads_file}', 'r') as f:
        leads = json.load(f)

    for lead in leads:
        if not lead.get('dm_envoyé'):
            continue
        if lead.get('réponse_reçue'):
            continue
        if lead.get('relances', 0) >= 2:
            continue
        if lead.get('profile_url') in blacklist:
            continue

        dm_date = datetime.strptime(lead.get('dm_date', '2000-01-01'), '%Y-%m-%d')
        jours_écoulés = (date_today - dm_date).days

        nb_relances = lead.get('relances', 0)

        if nb_relances == 0 and jours_écoulés >= 3:
            leads_à_relancer.append(lead)
        elif nb_relances == 1 and jours_écoulés >= 7:
            leads_à_relancer.append(lead)

print(f"[RELANCES] {len(leads_à_relancer)} leads à relancer")
```

### Étape 6 — Envoyer les relances

```python
for lead in leads_à_relancer:
    profile_url = lead.get('profile_url')
    nom_prénom = lead.get('nom', '').split()[0] if lead.get('nom') else ''
    nb_relances = lead.get('relances', 0)

    if nb_relances == 0:
        message_relance = f"Bonjour {nom_prénom}, je me permets de revenir vers vous. Avez-vous eu l'occasion de réfléchir à mon message précédent ? Curieux d'avoir votre retour."
    else:
        message_relance = f"Bonjour {nom_prénom}, dernier message de ma part. Si le timing n'est pas bon, pas de souci — je reste disponible si vous souhaitez en discuter plus tard."

    # Envoyer la relance
    # Endpoint : POST /message/linkedin
    # Utiliser 'profile' pour retrouver la conversation existante
    response = requests.post(
        'https://api.bereach.co/message/linkedin',
        headers=headers,
        json={
            'profile': profile_url,
            'message': message_relance
        }
    )

    if response.status_code == 200:
        lead['relances'] = nb_relances + 1
        lead['dernière_relance'] = date_today.strftime('%Y-%m-%d')
        print(f"[OK] Relance {nb_relances + 1} envoyée à {lead.get('nom')}")
    else:
        print(f"[ERREUR] Relance échouée pour {profile_url}: {response.status_code}")

    time.sleep(60)  # Pause entre relances
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `http_request` — Appels API Bereach (`POST /chats/linkedin`, `POST /message/linkedin`)
- `read_file` — Lecture des leads et client.yaml
- `write_file` — Mise à jour blacklist et logs

---

## Examples

**Lecture inbox :**
```json
POST https://api.bereach.co/chats/linkedin
{"count": 50, "start": 0}
```

**Relance J+3 :**
```json
POST https://api.bereach.co/message/linkedin
{
  "profile": "https://www.linkedin.com/in/jean-dupont",
  "message": "Bonjour Jean, je me permets de revenir vers vous..."
}
```
