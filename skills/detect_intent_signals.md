# Skill : Détection des Signaux d'Intention

## Description

Ce skill détecte les signaux d'intention sur LinkedIn : personnes ayant liké ou commenté des posts pertinents, profils ayant changé de poste récemment, entreprises ayant publié des offres d'emploi dans le domaine cible. Les leads détectés sont filtrés selon l'ICP du client et stockés avec leur signal associé.

---

## Steps

### Étape 1 — Charger la configuration client

```python
import json, yaml, os

with open('./client.yaml', 'r') as f:
    client = yaml.safe_load(f)

mots_clés = client['icp']['mots_clés']
titres_cibles = client['icp']['titres_cibles']
géographie = client['icp']['géographie']
api_key = client['bereach_api_key']

headers = {
    'Content-Type': 'application/json',
    'x-api-key': api_key
}
```

### Étape 2 — Rechercher les posts par mots-clés (signaux d'intention)

```python
import requests

leads_détectés = []

for mot_clé in mots_clés:
    response = requests.post(
        'https://api.bereach.co/search/linkedin/posts',
        headers=headers,
        json={
            'keywords': mot_clé,
            'sortBy': 'date',
            'datePosted': 'past-week',
            'count': 20,
            'start': 0
        }
    )
    posts = response.json().get('data', [])
    print(f"[SIGNAL] {len(posts)} posts trouvés pour '{mot_clé}'")
```

### Étape 3 — Récupérer les personnes ayant liké les posts

```python
for post in posts:
    post_url = post.get('url') or post.get('postUrl')
    if not post_url:
        continue

    # Récupérer les likers
    likers_resp = requests.post(
        'https://api.bereach.co/collect/linkedin/likes',
        headers=headers,
        json={
            'postUrl': post_url,
            'count': 50,
            'start': 0
        }
    )
    likers = likers_resp.json().get('data', [])

    for liker in likers:
        leads_détectés.append({
            'profile_url': liker.get('profileUrl'),
            'nom': liker.get('firstName', '') + ' ' + liker.get('lastName', ''),
            'titre': liker.get('headline', ''),
            'signal': 'like_post',
            'signal_context': post_url,
            'score_signal': 70
        })

    # Récupérer les commentateurs
    commenters_resp = requests.post(
        'https://api.bereach.co/collect/linkedin/comments',
        headers=headers,
        json={
            'postUrl': post_url,
            'count': 50,
            'start': 0
        }
    )
    commenters = commenters_resp.json().get('data', [])

    for commenter in commenters:
        leads_détectés.append({
            'profile_url': commenter.get('profileUrl'),
            'nom': commenter.get('firstName', '') + ' ' + commenter.get('lastName', ''),
            'titre': commenter.get('headline', ''),
            'signal': 'comment_post',
            'signal_context': post_url,
            'score_signal': 90
        })
```

### Étape 4 — Détecter les changements de poste récents

```python
for titre in titres_cibles:
    for geo in géographie:
        search_resp = requests.post(
            'https://api.bereach.co/search/linkedin/people',
            headers=headers,
            json={
                'keywords': titre,
                'title': titre,
                'location': geo,
                'count': 20,
                'start': 0
            }
        )
        personnes = search_resp.json().get('data', [])

        for personne in personnes:
            # Visiter le profil pour détecter un changement de poste récent
            profil_resp = requests.post(
                'https://api.bereach.co/visit/linkedin/profile',
                headers=headers,
                json={
                    'profile': personne.get('profileUrl'),
                    'includeAbout': True
                }
            )
            profil = profil_resp.json().get('data', {})

            # Vérifier si le poste actuel a moins de 3 mois
            experience = profil.get('experience', [])
            if experience:
                poste_actuel = experience[0]
                date_début = poste_actuel.get('startDate', '')
                # Logique de détection changement de poste récent
                leads_détectés.append({
                    'profile_url': personne.get('profileUrl'),
                    'nom': personne.get('firstName', '') + ' ' + personne.get('lastName', ''),
                    'titre': personne.get('headline', ''),
                    'signal': 'nouveau_poste',
                    'signal_context': date_début,
                    'score_signal': 75
                })
```

### Étape 5 — Détecter les offres d'emploi (signal de croissance)

```python
for mot_clé in mots_clés:
    jobs_resp = requests.post(
        'https://api.bereach.co/search/linkedin/jobs',
        headers=headers,
        json={
            'keywords': mot_clé,
            'datePosted': 'past-week',
            'count': 20,
            'start': 0
        }
    )
    offres = jobs_resp.json().get('data', [])

    for offre in offres:
        leads_détectés.append({
            'company_url': offre.get('companyUrl'),
            'company_name': offre.get('companyName'),
            'signal': 'offre_emploi',
            'signal_context': offre.get('jobUrl'),
            'score_signal': 65
        })
```

### Étape 6 — Filtrer par ICP et dédupliquer

```python
leads_filtrés = []
profiles_vus = set()

for lead in leads_détectés:
    profile_url = lead.get('profile_url')
    if not profile_url or profile_url in profiles_vus:
        continue

    titre = lead.get('titre', '').lower()
    # Vérifier si le titre correspond à l'ICP
    titre_match = any(t.lower() in titre for t in titres_cibles)
    if not titre_match:
        continue

    profiles_vus.add(profile_url)
    leads_filtrés.append(lead)

print(f"[RÉSULTAT] {len(leads_filtrés)} leads détectés après filtrage ICP")
```

### Étape 7 — Sauvegarder les leads détectés

```python
import json
from datetime import datetime

os.makedirs('./memory', exist_ok=True)
date_today = datetime.now().strftime('%Y-%m-%d')

with open(f'./memory/leads_{date_today}.json', 'w') as f:
    json.dump(leads_filtrés, f, ensure_ascii=False, indent=2)

print(f"[SAUVEGARDE] {len(leads_filtrés)} leads sauvegardés dans memory/leads_{date_today}.json")
```

---

## Tools

- `execute_code` — Exécution des scripts Python
- `http_request` — Appels API Bereach
- `read_file` — Lecture de client.yaml
- `write_file` — Sauvegarde des leads

---

## Examples

**Entrée :** Mots-clés `["prospection", "croissance commerciale"]`, ICP titres `["CEO", "Directeur Commercial"]`

**Sortie attendue :**
```json
[
  {
    "profile_url": "https://www.linkedin.com/in/jean-dupont",
    "nom": "Jean Dupont",
    "titre": "CEO @ StartupSaaS",
    "signal": "comment_post",
    "signal_context": "https://www.linkedin.com/posts/xxx",
    "score_signal": 90
  }
]
```
