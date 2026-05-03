# Skill : Qualification et Scoring des Leads

## Description

Ce skill récupère le profil LinkedIn complet de chaque lead via l'API Bereach, calcule un score ICP de 0 à 100, et prend une décision : contacter maintenant, mettre en attente, ou exclure.

---

## Steps

### Étape 1 — Charger les leads et la configuration

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

date_today = datetime.now().strftime('%Y-%m-%d')
with open(f'./memory/leads_{date_today}.json', 'r') as f:
    leads = json.load(f)

print(f"[CHARGEMENT] {len(leads)} leads à qualifier")
```

### Étape 2 — Récupérer le profil complet via Bereach

```python
import requests, time

leads_qualifiés = []

for lead in leads:
    profile_url = lead.get('profile_url')
    if not profile_url:
        continue

    # Récupérer le profil complet
    profil_resp = requests.post(
        'https://api.bereach.co/visit/linkedin/profile',
        headers=headers,
        json={
            'profile': profile_url,
            'includeAbout': True,
            'includePosts': False,
            'includeComments': False
        }
    )
    profil = profil_resp.json().get('data', {})
    lead['profil_complet'] = profil

    # Pause anti-ban
    time.sleep(2)
```

### Étape 3 — Calculer le score ICP (0-100)

```python
def calculer_score_icp(lead, client_config):
    score = 0
    raisons = []
    profil = lead.get('profil_complet', {})

    icp = client_config['icp']
    titres_cibles = [t.lower() for t in icp.get('titres_cibles', [])]
    secteurs_cibles = [s.lower() for s in icp.get('secteur', [])]
    géographies_cibles = [g.lower() for g in icp.get('géographie', [])]
    tailles_cibles = icp.get('taille_entreprise', [])
    exclusions = [e.lower() for e in icp.get('exclure', [])]

    # Titre du poste (30 points)
    titre_actuel = profil.get('headline', '').lower()
    if any(t in titre_actuel for t in titres_cibles):
        score += 30
        raisons.append('Titre correspond à l\'ICP')

    # Secteur (20 points)
    secteur = profil.get('industry', '').lower()
    if any(s in secteur for s in secteurs_cibles):
        score += 20
        raisons.append('Secteur correspond à l\'ICP')

    # Géographie (15 points)
    localisation = profil.get('location', '').lower()
    if any(g in localisation for g in géographies_cibles):
        score += 15
        raisons.append('Géographie correspond à l\'ICP')

    # Signal d'intention (25 points)
    score_signal = lead.get('score_signal', 0)
    score += int(score_signal * 0.25)
    raisons.append(f"Signal: {lead.get('signal', 'aucun')} (+{int(score_signal * 0.25)} pts)")

    # Connexions (10 points)
    nb_connexions = profil.get('connectionsCount', 0)
    if nb_connexions > 500:
        score += 10
        raisons.append('Profil influent (500+ connexions)')
    elif nb_connexions > 200:
        score += 5

    # Exclusions (-50 points)
    for exclusion in exclusions:
        if exclusion in titre_actuel:
            score -= 50
            raisons.append(f'Exclusion: {exclusion}')

    return min(max(score, 0), 100), raisons


for lead in leads:
    score, raisons = calculer_score_icp(lead, client)
    lead['score_icp'] = score
    lead['raisons_score'] = raisons
```

### Étape 4 — Décision de qualification

```python
for lead in leads:
    score = lead['score_icp']

    if score >= 70:
        lead['décision'] = 'CONTACTER'
        lead['priorité'] = 'haute'
    elif score >= 40:
        lead['décision'] = 'ATTENTE'
        lead['priorité'] = 'normale'
    else:
        lead['décision'] = 'EXCLURE'
        lead['priorité'] = 'basse'

    leads_qualifiés.append(lead)

à_contacter = [l for l in leads_qualifiés if l['décision'] == 'CONTACTER']
print(f"[QUALIFICATION] {len(à_contacter)} leads à contacter sur {len(leads_qualifiés)} analysés")
```

### Étape 5 — Sauvegarder les leads qualifiés

```python
with open(f'./memory/leads_qualifiés_{date_today}.json', 'w') as f:
    json.dump(leads_qualifiés, f, ensure_ascii=False, indent=2)

print(f"[SAUVEGARDE] Leads qualifiés sauvegardés")
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `http_request` — Appels API Bereach (`POST /visit/linkedin/profile`)
- `read_file` — Lecture des leads et de client.yaml
- `write_file` — Sauvegarde des leads qualifiés

---

## Examples

**Entrée :** Lead avec signal `comment_post` (score 90), titre "CEO @ SaaS"

**Sortie attendue :**
```json
{
  "profile_url": "https://www.linkedin.com/in/jean-dupont",
  "score_icp": 82,
  "décision": "CONTACTER",
  "priorité": "haute",
  "raisons_score": ["Titre correspond à l'ICP", "Secteur correspond à l'ICP", "Signal: comment_post (+22 pts)"]
}
```
