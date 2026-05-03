---
name: sdr_linkedin_followup
description: Envoyer des messages personnalisés après acceptation des demandes de connexion
triggers:
  - cron: "0 9-18/2 * * 1-5"
model: google/gemini-2.5-flash-lite
parameters:
  - name: LIMITE_MESSAGES_JOUR
    type: int
    default: 100
    description: Limite quotidienne de messages
  - name: DELAI_ENTRE_ACTIONS_SEC
    type: int
    default: 45
    description: Délai minimum entre envois de messages
steps:
  - id: verify_limits
    description: Vérifier les limites messages BeReach
    provider: bereach
    actions:
      - type: GET
        path: /me/limits
    expect: 200
    on_error: stop

  - id: fetch_airtable
    description: Récupérer prospects avec connexion envoyée (attente 24h+)
    provider: airtable
    records:
      table: ${AIRTABLE_TABLE_NAME}
      filterByFormula: AND(
        {statut} = "CONNEXION_ENVOYEE",
        {date_connexion_envoyee} < NOW() - 86400
      )
      sort:
        - field: date_connexion_envoyee
          direction: asc
      maxRecords: ${LIMITE_MESSAGES_JOUR}
    on_error: stop

  - id: check_accepted_connections
    description: Vérifier quels prospects ont accepté via l'API conversations
    provider: bereach
    actions:
      - type: GET
        path: /conversations
        query:
          status: accepted
          since: ${24h_ago}
    expect: 200
    on_error: stop

  - id: match_accepted
    description: Identifier les connexions acceptées
    script: |
      const accepted_ids = conversations.map(c => c.linkedin_id);
      const to_message = [];
      
      for (const record of airtable_records) {
        if (accepted_ids.includes(record.fields.linkedin_id)) {
          to_message.push(record);
        }
      }
      
      return to_message;

  - id: filter_none
    description: Arrêter si aucun prospect à contacter
    condition: ${to_message.length} == 0
    on_true: stop

                              - id: generate_message
    description: Genere le message selon le score avec templates inline fixes (4/5 lignes)
    model: google/gemini-2.5-flash-lite
    script: |
      const prenom = prospect.prenom || "Jean";
      const score = prospect.score;
      const signal = prospect.signal_detecte || "";

      const TEMPLATES = {
        WARM: `Jean,
J'ai vu que tu developpes une agence SEO locale.
Je peux t'aider a stabiliser ton pipeline avec des rendez-vous qualifies via LinkedIn.
Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence SEO locale ?`,
        HOT: `Jean,
J'ai vu ton activite recente autour de prospection LinkedIn.
Ca montre que l'acquisition reste un sujet important pour ton agence.
Je peux t'aider a creer un flux regulier de rendez-vous qualifies via LinkedIn.
Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence SEO locale ?`,
        VERY_HOT: `Jean,
J'ai vu ton signal autour de pipeline instable.
Si ton pipeline depend encore beaucoup du bouche-a-oreille, ca peut vite devenir instable.
Mon systeme aide les agences SEO locales a obtenir des rendez-vous qualifies via LinkedIn.
Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence SEO locale ?`
      };

      let msg = TEMPLATES[score] || TEMPLATES.WARM;
      if (score === "HOT" || score === "VERY_HOT") {
        msg = msg.replace(/{signal_detecte}/g, signal);
      }

      return msg.trim();

  - id: send_message
    description: Envoyer le message LinkedIn
    provider: bereach
    script: |
      // Extraire profile_url depuis les champs Airtable
      const profile_url = prospect.fields.url_profil || prospect.fields.profile_url || "";
      if (!profile_url || profile_url.trim() === "") {
        console.error("[SKIP] profile_url vide pour", prospect.fields.prenom);
        return { skip: true, reason: "profile_url empty" };
      }
      // Préparer le message (déjà généré par generate_message)
      const message = prospect.fields.message || prospect.message || "";
      if (!message) {
        console.error("[SKIP] message vide pour", prospect.fields.prenom);
        return { skip: true, reason: "message empty" };
      }
      return { profile_url, message };
    actions:
      - type: POST
        path: /message/linkedin
        body:
          profile: ${profile_url}
          message: ${message}
    expect: 200
    on_error: retry
    delay: ${DELAI_ENTRE_ACTIONS_SEC}
  - id: update_airtable
    description: Mettre à jour le statut Airtable
    provider: airtable
    record:
      table: ${AIRTABLE_TABLE_NAME}
      id: ${airtable_record_id}
      fields:
        statut: MESSAGE_ENVOYE
        date_message_envoye: ${now()}
        message_envoye: ${generated_message}
        nb_messages_envoyes: ${prospect.nb_messages_envoyes + 1}

  - id: summary
    description: Résumé final
    output: |
      {
        "action": "sdr_followup",
        "linkedin_id": "${linkedin_id}",
        "prospect": "${prenom} ${nom}",
        "score": "${score}",
        "message_sent": true,
        "timestamp": "${now()}"
      }
