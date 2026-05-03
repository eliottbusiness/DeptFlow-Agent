# Skill : Envoi de Messages de Prospection (DMs)

## Description

Ce skill envoie des messages de prospection personnalisés aux prospects qui ont accepté la demande de connexion. Le message suit une structure précise : accroche basée sur le signal → problème identifié → valeur proposée → CTA doux.

---

## Steps

### Étape 1 — Identifier les connexions acceptées sans DM envoyé

```python
import json, yaml, os, requests, time, random
from datetime import datetime

with open('./client.yaml', 'r') as f:
    client = yaml.safe_load(f)

api_key = client['bereach_api_key']
headers = {
    'Content-Type': 'application/json',
    'x-api-key': api_key
}

limite_dms = client['daily_limits']['dms']  # 20 par défaut
délai_min = client['daily_limits']['délai_min_entre_actions']
délai_max = client['daily_limits']['délai_max_entre_actions']

date_today = datetime.now().strftime('%Y-%m-%d')

# Vérifier le compteur quotidien
compteur_file = f'./memory/compteur_{date_today}.json'
if os.path.exists(compteur_file):
    with open(compteur_file, 'r') as f:
        compteur = json.load(f)
else:
    compteur = {'connexions': 0, 'dms': 0, 'visites': 0}

if compteur['dms'] >= limite_dms:
    print(f"[LIMITE] Limite quotidienne de {limite_dms} DMs atteinte. Arrêt.")
    exit()
```

### Étape 2 — Vérifier les nouvelles connexions acceptées via l'inbox

```python
# Lire l'inbox pour détecter les connexions acceptées
# Endpoint : POST /chats/linkedin
chats_resp = requests.post(
    'https://api.bereach.co/chats/linkedin',
    headers=headers,
    json={'count': 50, 'start': 0}
)
conversations = chats_resp.json().get('data', [])

# Charger les leads avec connexion envoyée
leads_files = [f for f in os.listdir('./memory') if f.startswith('leads_qualifiés_')]
leads_avec_connexion = []

for leads_file in leads_files:
    with open(f'./memory/{leads_file}', 'r') as f:
        leads = json.load(f)
    leads_avec_connexion.extend([
        l for l in leads
        if l.get('connexion_envoyée') and not l.get('dm_envoyé')
    ])

print(f"[CHARGEMENT] {len(leads_avec_connexion)} leads avec connexion acceptée, sans DM")
```

### Étape 3 — Générer le message de prospection

```python
def générer_dm_prospection(lead, client_config):
    signal = lead.get('signal', 'cold')
    nom_prénom = lead.get('nom', '').split()[0] if lead.get('nom') else ''
    offre = client_config['offer']
    problème = offre['problème']
    valeur = offre['valeur']
    cta = offre['cta']

    if signal == 'comment_post':
        accroche = f"Merci d'avoir accepté ma demande, {nom_prénom} ! Votre commentaire sur ce post m'a vraiment interpellé."
    elif signal == 'like_post':
        accroche = f"Merci pour la connexion, {nom_prénom} ! J'avais remarqué votre intérêt pour ce sujet."
    elif signal == 'nouveau_poste':
        accroche = f"Merci pour la connexion, {nom_prénom} ! Encore félicitations pour votre nouveau rôle."
    elif signal == 'offre_emploi':
        accroche = f"Merci pour la connexion, {nom_prénom} ! J'avais vu que vous étiez en pleine phase de croissance."
    else:
        accroche = f"Merci pour la connexion, {nom_prénom} !"

    message = f"""{accroche}

Beaucoup de profils comme le vôtre me partagent que {problème}.

{valeur}.

{cta}"""

    return message
```

### Étape 4 — Envoyer les DMs

```python
dms_envoyés = 0

for lead in leads_avec_connexion:
    if compteur['dms'] + dms_envoyés >= limite_dms:
        print(f"[LIMITE] Limite DMs atteinte après {dms_envoyés} messages.")
        break

    profile_url = lead.get('profile_url')
    if not profile_url:
        continue

    message = générer_dm_prospection(lead, client)

    # Envoyer le DM
    # Endpoint : POST /message/linkedin
    # Utiliser 'profile' pour un premier message (pas de conversationUrn)
    response = requests.post(
        'https://api.bereach.co/message/linkedin',
        headers=headers,
        json={
            'profile': profile_url,
            'message': message
        }
    )

    if response.status_code == 200:
        lead['dm_envoyé'] = True
        lead['dm_date'] = date_today
        lead['dm_message'] = message
        dms_envoyés += 1
        print(f"[OK] DM envoyé à {lead.get('nom')} ({profile_url})")
    else:
        lead['dm_erreur'] = response.text
        print(f"[ERREUR] Échec DM {profile_url}: {response.status_code}")

    # Pause aléatoire anti-ban
    pause = random.randint(délai_min, délai_max)
    time.sleep(pause)
```

### Étape 5 — Mettre à jour les compteurs et sauvegarder

```python
compteur['dms'] += dms_envoyés

with open(compteur_file, 'w') as f:
    json.dump(compteur, f)

# Sauvegarder les leads mis à jour
for leads_file in leads_files:
    with open(f'./memory/{leads_file}', 'w') as f:
        json.dump(leads, f, ensure_ascii=False, indent=2)

print(f"[RÉSUMÉ] {dms_envoyés} DMs envoyés aujourd'hui (total: {compteur['dms']}/{limite_dms})")
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `http_request` — Appels API Bereach (`POST /message/linkedin`, `POST /chats/linkedin`)
- `read_file` — Lecture des leads et client.yaml
- `write_file` — Mise à jour des logs

---

## Examples

**Payload envoyé à Bereach (premier DM) :**
```json
{
  "profile": "https://www.linkedin.com/in/jean-dupont",
  "message": "Merci d'avoir accepté ma demande, Jean ! Votre commentaire sur ce post m'a vraiment interpellé.\n\nBeaucoup de profils comme le vôtre me partagent que leur équipe commerciale perd du temps sur la prospection manuelle.\n\nNous automatisons votre prospection LinkedIn pour générer 3x plus de RDV qualifiés.\n\nSeriez-vous ouvert à un échange de 20 minutes pour voir si ça peut s'appliquer à votre contexte ?"
}
```

**Endpoint :** `POST https://api.bereach.co/message/linkedin`
