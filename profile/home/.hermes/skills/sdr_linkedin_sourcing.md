---
name: sdr_linkedin_sourcing
description: Sourcing automatique de prospects LinkedIn selon l'ICP, scoring et envoi de demandes de connexion
triggers:
  - cron: "0 8 * * 1-5"
model: google/gemini-2.5-flash-lite
parameters:
  - name: LIMITE_CONNEXIONS_JOUR
    type: int
    default: 30
    description: Limite quotidienne de demandes de connexion
  - name: LIMITE_MESSAGES_JOUR
    type: int
    default: 100
    description: Limite quotidienne de messages
  - name: DELAI_ENTRE_ACTIONS_SEC
    type: int
    default: 45
    description: Délai minimum entre actions LinkedIn
steps:
  - id: verify_limits
    description: Vérifier les limites BeReach avant de sourcer
    provider: bereach
    actions:
      - type: GET
        path: /me/limits
    expect: 200
    on_error: stop

  - id: resolve_airtable_target
    description: Résoudre une cible Airtable centralisée (AIRTABLE_TABLE_ID prioritaire, sinon nom de table URL-encodé)
    script: |
      const table_id = (AIRTABLE_TABLE_ID || "").trim();
      const table_name = (AIRTABLE_TABLE_NAME || "").trim();
      const encoded_table_name = encodeURIComponent(table_name);
      const airtable_table_ref = table_id || encoded_table_name;
      const airtable_url_strategy = table_id ? "table_id" : "url_encoded_table_name";
      return { airtable_table_ref, airtable_url_strategy };

  - id: search_prospects
    description: Rechercher des prospects correspondant à l'ICP
    provider: bereach
    actions:
      - type: POST
        path: /search/linkedin/people
        body:
          titres: ${ICP_TITRES}
          secteurs: ${ICP_SECTEURS}
          taille_entreprise: ${ICP_TAILLE_ENTREPRISE}
          pays: ${ICP_PAYS}
          limite: ${LIMITE_CONNEXIONS_JOUR}
    expect: 200
    on_error: stop

  - id: dedup_airtable
    description: Déduplication via Airtable (filterByFormula sur linkedin_id, même cible Airtable centralisée)
    provider: airtable
    record:
      table: ${airtable_table_ref}
      filterByFormula: FIND("{{linkedin_id}}", {linkedin_id})
    on_error: stop

  - id: scoring
    description: Déterminer le score (REJETÉ/WARM/HOT/VERY_HOT) selon ICP
    script: |
      const titre = prospect.titre.toLowerCase();
      const secteur = prospect.secteur?.toLowerCase() || "";
      
      // REJET si hors ICP
      const titres_icp = ICP_TITRES.map(t => t.toLowerCase());
      const secteurs_icp = ICP_SECTEURS.map(s => s.toLowerCase());
      
      if (!titres_icp.some(t => titre.includes(t))) return "REJETE";
      if (!secteurs_icp.some(s => secteur.includes(s))) return "REJETE";
      
      // WARM par défaut
      let score = "WARM";
      
      // HOT si post récent (< 60 jours)
      const post_recent = prospect.posts?.some(p => {
        const age = Date.now() - new Date(p.date).getTime();
        return age < 60 * 24 * 60 * 60 * 1000;
      });
      if (post_recent) score = "HOT";
      
      // VERY HOT si signal fort (concurrents ou keywords)
      const keywords_offre = OFFRE_VALEUR.toLowerCase().split(" ");
      const keywords_concurrents = OFFRE_CONCURRENTS.map(c => c.toLowerCase());
      const bio = (prospect.bio || "").toLowerCase();
      
      if (keywords_offre.some(k => bio.includes(k)) || 
          keywords_concurrents.some(c => bio.includes(c))) {
        score = "VERY_HOT";
      }
      
      return score;

  - id: filter_rejected
    description: Arrêter si REJETÉ
    condition: ${score} == "REJETE"
    on_true: stop

  - id: enrich_profile
    description: Récupérer le profil LinkedIn complet
    provider: bereach
    actions:
      - type: GET
        path: /profile/linkedin
        query:
          linkedin_id: ${linkedin_id}
    expect: 200
    on_error: stop

  - id: build_bereach_profile_target
    description: Construire la cible BeReach canonique pour la connexion et verrouiller le contrat interne
    script: |
      function normalizeLinkedInProfileUrl(value) {
        const raw = String(value || "").trim();
        if (!raw) return "";
        if (/^urn:li:/i.test(raw)) return raw;
        if (!/^https?:\/\//i.test(raw)) return "";
        try {
          const url = new URL(raw);
          if (!/(^|\.)linkedin\.com$/i.test(url.hostname)) return "";
          url.hostname = "www.linkedin.com";
          url.search = "";
          url.hash = "";
          url.pathname = url.pathname.replace(/\/+$/, "");
          if (!/^\/in\//i.test(url.pathname) && !/^\/pub\//i.test(url.pathname)) return "";
          return url.toString();
        } catch {
          return "";
        }
      }

      function build_bereach_profile_target(prospect) {
        const profileUrlRaw = String(prospect.profileUrl || "").trim();
        const urlProfilRaw = String(prospect.url_profil || "").trim();
        const profileUrn = String(prospect.profileUrn || "").trim();
        const publicIdentifier = String(prospect.publicIdentifier || "").trim();
        const linkedin_id = profileUrn || publicIdentifier || normalizeLinkedInProfileUrl(profileUrlRaw || urlProfilRaw) || String(prospect.linkedin_id || "").trim();

        let bereach_profile_target = "";
        let source = "none";

        const normalized_profileUrl = normalizeLinkedInProfileUrl(profileUrlRaw);
        if (normalized_profileUrl) {
          bereach_profile_target = normalized_profileUrl;
          source = "profileUrl";
        } else {
          const normalized_url_profil = normalizeLinkedInProfileUrl(urlProfilRaw);
          if (normalized_url_profil) {
            bereach_profile_target = normalized_url_profil;
            source = "url_profil";
          } else if (profileUrn) {
            bereach_profile_target = profileUrn;
            source = "profileUrn";
          } else if (publicIdentifier) {
            bereach_profile_target = publicIdentifier;
            source = "publicIdentifier";
          }
        }

        if (!bereach_profile_target) {
          return {
            linkedin_id,
            bereach_profile_target: "",
            source: "none",
            valid: false,
            error: "ERROR_MISSING_PROFILE_TARGET"
          };
        }

        return {
          linkedin_id,
          bereach_profile_target,
          source,
          valid: true,
          error: ""
        };
      }

      const prospect_input = {
        profileUrl: prospect.profileUrl || "",
        url_profil: prospect.url_profil || "",
        profileUrn: prospect.profileUrn || "",
        publicIdentifier: prospect.publicIdentifier || "",
        linkedin_id: prospect.linkedin_id || ""
      };

      return build_bereach_profile_target(prospect_input);

  - id: skip_missing_bereach_target
    description: Arrêter le prospect si aucune cible BeReach valide n'a pu être construite
    condition: ${valid} == false
    on_true: stop

  - id: send_connection
    description: Envoyer la demande de connexion (sans note)
    provider: bereach
    actions:
      - type: POST
        path: /connect/linkedin/profile
        body:
          profile: ${bereach_profile_target}
    expect: 200
    on_error: retry
    delay: ${DELAI_ENTRE_ACTIONS_SEC}

  - id: save_airtable
    description: Sauvegarder le prospect dans Airtable (même cible Airtable centralisée)
    provider: airtable
    record:
      table: ${airtable_table_ref}
      fields:
        linkedin_id: ${linkedin_id}
        prenom: ${prospect.prenom}
        nom: ${prospect.nom}
        titre: ${prospect.titre}
        entreprise: ${prospect.entreprise}
        url_profil: ${prospect.url_profil}
        localisation: ${prospect.localisation}
        score: ${score}
        signal_detecte: ${signal_detecte}
        statut: CONNEXION_ENVOYEE
        date_creation: ${now()}
        date_connexion_envoyee: ${now()}
        source: sourcing_automatique
        nb_tentatives_connexion: 1
        nb_messages_envoyes: 0

  - id: summary
    description: Résumé final
    output: |
      {
        "action": "sdr_sourcing",
        "linkedin_id": "${linkedin_id}",
        "bereach_profile_target": "${bereach_profile_target}",
        "adapter_source": "${source}",
        "adapter_valid": "${valid}",
        "adapter_error": "${error}",
        "prospect": "${prenom} ${nom} - ${titre} @ ${entreprise}",
        "score": "${score}",
        "statut": "CONNEXION_ENVOYEE",
        "timestamp": "${now()}"
      }
