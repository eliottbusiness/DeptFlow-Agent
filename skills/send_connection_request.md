# Skill : Envoi de Demandes de Connexion

## Description

Ce skill envoie des demandes de connexion LinkedIn personnalisées aux leads qualifiés, en respectant les limites quotidiennes définies dans client.yaml. Le message de connexion est adapté selon le signal d'intention détecté.

---

## Steps

### Étape 1 — Charger les leads qualifiés et vérifier les limites

```python
import json, yaml, os
from datetime import datetime

with open('./client.yaml', 'r') as f:
    client = yaml.safe_load(f)

api_key = client['bereach_api_key']
headers = {
    'Content-Type': 'application/json',
    'x-api-key': api_key
}

limite_connexions = client['daily_limits']['connexions']  # 25 par défaut
délai_min = client['daily_limits']['délai_min_entre_actions']  # 120s
délai_max = client['daily_limits']['délai_max_entre_actions']  # 300s

date_today = datetime.now().strftime('%Y-%m-%d')

# Charger le compteur quotidien
compteur_file = f'./memory/compteur_{date_today}.json'
if os.path.exists(compteur_file):
    with open(compteur_file, 'r') as f:
        compteur = json.load(f)
else:
    compteur = {'connexions': 0, 'dms': 0, 'visites': 0}

if compteur['connexions'] >= limite_connexions:
    print(f"[LIMITE] Limite quotidienne de {limite_connexions} connexions atteinte. Arrêt.")
    exit()

# Charger les leads à contacter
with open(f'./memory/leads_qualifiés_{date_today}.json', 'r') as f:
    leads = json.load(f)

leads_à_contacter = [l for l in leads if l.get('décision') == 'CONTACTER' and not l.get('connexion_envoyée')]
print(f"[CHARGEMENT] {len(leads_à_contacter)} leads à contacter")
```

### Étape 2 — Générer le message de connexion selon le signal

```python
def générer_message_connexion(lead, client_config):
    signal = lead.get('signal', 'cold')
    nom_prénom = lead.get('nom', '').split()[0] if lead.get('nom') else ''
    signal_context = lead.get('signal_context', '')
    offre = client_config['offer']

    if signal == 'comment_post':
        return f"Bonjour {nom_prénom}, j'ai vu votre commentaire sur ce post — votre point de vue m'a interpellé. Je travaille sur des sujets similaires, ça pourrait valoir un échange !"

    elif signal == 'like_post':
        return f"Bonjour {nom_prénom}, j'ai remarqué votre like sur ce post. On travaille sur des problématiques proches, je me permets de vous ajouter."

    elif signal == 'nouveau_poste':
        return f"Bonjour {nom_prénom}, félicitations pour votre nouveau poste ! Je me permets de vous ajouter à mon réseau."

    elif signal == 'offre_emploi':
        return f"Bonjour {nom_prénom}, j'ai vu que vous recrutez dans ce domaine — signe d'une belle croissance. Je me permets de vous ajouter."

    else:
        return None  # Pas de message pour les connexions cold
```

### Étape 3 — Envoyer les demandes de connexion

```python
import requests, time, random

connexions_envoyées = 0

for lead in leads_à_contacter:
    if compteur['connexions'] + connexions_envoyées >= limite_connexions:
        print(f"[LIMITE] Limite atteinte après {connexions_envoyées} connexions.")
        break

    profile_url = lead.get('profile_url')
    if not profile_url:
        continue

    message = générer_message_connexion(lead, client)

    # Construire le payload
    payload = {'profile': profile_url}
    if message:
        payload['message'] = message

    # Envoyer la demande de connexion
    # Endpoint : POST /connect/linkedin/profile
    response = requests.post(
        'https://api.bereach.co/connect/linkedin/profile',
        headers=headers,
        json=payload
    )

    if response.status_code == 200:
        lead['connexion_envoyée'] = True
        lead['connexion_date'] = date_today
        lead['message_connexion'] = message
        connexions_envoyées += 1
        print(f"[OK] Connexion envoyée à {lead.get('nom')} ({profile_url})")
    else:
        lead['connexion_erreur'] = response.text
        print(f"[ERREUR] Échec connexion {profile_url}: {response.status_code}")

    # Pause aléatoire anti-ban
    pause = random.randint(délai_min, délai_max)
    print(f"[PAUSE] Attente {pause}s...")
    time.sleep(pause)
```

### Étape 4 — Mettre à jour les compteurs et sauvegarder

```python
compteur['connexions'] += connexions_envoyées

with open(compteur_file, 'w') as f:
    json.dump(compteur, f)

with open(f'./memory/leads_qualifiés_{date_today}.json', 'w') as f:
    json.dump(leads, f, ensure_ascii=False, indent=2)

print(f"[RÉSUMÉ] {connexions_envoyées} connexions envoyées aujourd'hui (total: {compteur['connexions']}/{limite_connexions})")
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `http_request` — Appels API Bereach (`POST /connect/linkedin/profile`)
- `read_file` — Lecture des leads qualifiés et client.yaml
- `write_file` — Mise à jour des compteurs et logs

---

## Examples

**Entrée :** Lead avec signal `comment_post`, profil `https://www.linkedin.com/in/jean-dupont`

**Payload envoyé à Bereach :**
```json
{
  "profile": "https://www.linkedin.com/in/jean-dupont",
  "message": "Bonjour Jean, j'ai vu votre commentaire sur ce post — votre point de vue m'a interpellé. Je travaille sur des sujets similaires, ça pourrait valoir un échange !"
}
```

**Endpoint :** `POST https://api.bereach.co/connect/linkedin/profile`
