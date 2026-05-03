# Instructions Générales de Prospection

## Workflow Global Bout en Bout

```
[8h00] Détection signaux d'intention
    ↓
[8h30] Qualification et scoring des leads
    ↓
[9h00] Envoi des demandes de connexion (max 25/jour)
    ↓
[12h00] Vérification inbox + envoi DMs aux connexions acceptées
    ↓
[18h00] Vérification inbox + relances J+3 et J+7
    ↓
[17h00 vendredi] Rapport hebdomadaire
```

---

## Règles de Sécurité LinkedIn (Anti-Ban)

### Limites absolues à ne jamais dépasser
- **25 connexions/jour** maximum
- **20 DMs/jour** maximum
- **50 visites de profil/jour** maximum
- **Pause de 2 à 5 minutes** entre chaque action
- **Ne jamais automatiser** plus de 4 heures consécutives

### Comportement humain à simuler
- Varier les horaires d'action (ne pas toujours agir à la même heure)
- Alterner entre visites de profil, connexions et messages
- Ne pas envoyer des messages identiques à la suite
- Respecter les week-ends (réduire l'activité de 50%)
- Si détection de rate limit : pause de 2 heures minimum

### Signaux d'alerte à surveiller
- Taux d'acceptation des connexions < 20% → revoir le message de connexion
- Taux de réponse aux DMs < 5% → revoir le message de prospection
- Erreurs API répétées → vérifier le statut du compte LinkedIn

---

## Priorité des Signaux (du plus fort au plus faible)

| Rang | Signal | Score | Action recommandée |
|------|--------|-------|-------------------|
| 1 | Commentaire sur post pertinent | 90 | Connexion + message personnalisé immédiat |
| 2 | Like sur post pertinent | 70 | Connexion avec note personnalisée |
| 3 | Changement de poste récent (< 3 mois) | 75 | Connexion de félicitations |
| 4 | Offre d'emploi publiée dans le domaine | 65 | Connexion avec angle croissance |
| 5 | Profil ICP sans signal | 30 | Connexion standard, DM différé |

---

## Instructions de Personnalisation des Messages

### Règles générales
- **Toujours** mentionner le signal détecté dans le premier message
- **Jamais** de pitch commercial dans le message de connexion
- **Toujours** poser une question ouverte en fin de DM
- **Jamais** de liste à puces dans les messages LinkedIn
- **Maximum** 3-4 phrases par message

### Structure du DM de prospection
1. **Accroche** : référence au signal (post commenté, changement de poste, etc.)
2. **Problème** : identifier une douleur probable liée au contexte
3. **Valeur** : ce que vous apportez (sans détailler)
4. **CTA doux** : question ouverte, pas de demande de RDV directe

### Exemples de CTAs efficaces
- "Est-ce que ça fait sens pour vous ?"
- "Curieux d'avoir votre avis sur ce sujet."
- "Seriez-vous ouvert à un échange rapide ?"
- "Est-ce que c'est un sujet qui vous parle en ce moment ?"

### Exemples de CTAs à éviter
- "Avez-vous 30 minutes pour un appel ?"
- "Je vous propose une démo gratuite."
- "Cliquez ici pour voir notre offre."
- "Répondez-moi pour en savoir plus."

---

## Gestion des Réponses

### Réponse positive
1. Escalader immédiatement vers l'humain via Telegram
2. Ne pas répondre automatiquement
3. Marquer le lead comme `réponse_positive: true`

### Réponse négative / refus
1. Ajouter à la blacklist immédiatement
2. Ne plus jamais contacter ce profil
3. Répondre poliment si demande explicite : "Bien compris, bonne continuation !"

### Pas de réponse
1. Attendre J+3 avant première relance
2. Attendre J+7 avant deuxième relance
3. Maximum 2 relances, puis archiver le lead
