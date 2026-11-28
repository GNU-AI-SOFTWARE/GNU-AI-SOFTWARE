# Prochaines phases — `/orchestrate` et `/entrainement`

> **État d'avancement (2026-11-28, testé sur VM Debian GNU/Hurd)** :
> - ✅ **A1** livrée : run asynchrone (`{"run_id":N,"state":"submitted"}`),
>   superviseur (1 retry + incidents persistés), `busy` si run actif.
> - ✅ **B1** livrée : `/entrainement` monte un neurone éphémère sur
>   `/llm1` quand aucun `/llmN` ne répond, et le démonte après.
> - ✅ **A2** livrée : commandes `fetch` / `fetch_run` de l'orchestrateur
>   (httpfs → archive `training_data` avec SHA-256 → tokenisation 64
>   valeurs → run). La table `incidents` est désormais inscriptible via
>   le protocole `/db` (contrat data-base-translator étendu + tests).
> - ✅ **B2** livré : `/hf sync` marque les fichiers tronqués (> fenêtre
>   httpfs 64 Kio) avec `http_status=206` au lieu d'un 200 trompeur ;
>   `/hf sync --resume <hf://…> <dir>` saute les URLs déjà en cache
>   (idempotence vérifiée : 133 → 133 lignes au second passage).
> - 🧠 **Système multi-neuronal 1M** : `MAX_NEURONS_PER_LAYER` plafonné
>   à 65535 (uint16_t, rejet propre au-delà), système de
>   **16 instances × `64,65535` = 1 049 584 neurones** via `/orchestrate`
>   (run done, 16/16 instances OK, ~17 Mo/instance).
> - ⏳ B3/A3/A4/B4 : reste (voir sections ci-dessous).

Proposition de suite, en continuité des `PLAN.md` existants
(`orchestrator-translator` : phases 0–1 livrées et vérifiées sur
Debian GNU/Hurd en VM ; `inference-translator` : phases 0–3 livrées,
`/entrainement` encore au stade script manuel).

Chaque phase a un livrable et un critère d'acceptation mesurables.

---

## A. `/orchestrate` — prochaines phases

### A1 (ex-phase 2 du PLAN) — Superviseur et exécution asynchrone

- Le `write` d'un descripteur rend la main **immédiatement** (run
  asynchrone) ; l'exécution continue en arrière-plan.
- Nœuds `netfs` (contrat gelé en phase 0, SPEC §5 « phase 2+ ») :
  `/orchestrate/status` (type 2, streaming) et
  `/orchestrate/result` (type 3), un curseur par lecteur.
- Timeout de non-réponse par instance, redémarrage, persistance de
  chaque incident dans la table `incidents`.
- File d'attente de descripteurs + exécution parallèle de plusieurs
  tâches, limite de ressources (instances simultanées max).
- **Livrable** : `/orchestrate` en netfs, supervisor opérationnel.
- **Acceptation** : `cat /orchestrate/status` reflète l'état pendant
  un run ; une instance tuée mid-run est remplacée, l'incident est
  dans `incidents`, le résultat agrégé est rendu malgré tout ;
  `make check` vert sur Hurd (VM QEMU).

### A2 (ex-phase 3) — Acquisition réseau côté orchestrateur

- Lecture de `/inference/request` (type 1), extraction des URLs.
- Montage dynamique de `httpfs` sur chaque URL, lecture de
  `content`/`status`, politique documentée pour les non-200
  (réessai / saut / abandon).
- Archivage **avant** usage : chaque contenu va dans
  `training_data` (URL, statut HTTP, contenu, SHA-256) via `/db`.
- Tokenisation minimale du contenu en vecteur d'entrée (normalisation,
  limite 64 valeurs, contrat neuron §2).
- **Livrable** : chaîne complète prompt → URL → contenu → N neurones
  → résultat agrégé, pilotée par `/orchestrate` seul.
- **Acceptation** : démo bout en bout avec une URL 200 et une URL
  404 volontaire ; les deux traces sont dans `training_data`.

### A3 (ex-phase 4) — Rejouabilité

- `replay <run_id>` : rejoue une exécution depuis les données
  archivées, réseau coupé, et compare les sorties à l'historique.
- `history` : performances par topologie depuis `run_instances`.
- **Acceptation** : `replay` d'un run de A2 sans réseau produit le
  même résultat agrégé (graine fixée côté neuron).

### A4 (ex-phase 5/7) — Évaluateur et boucle d'amélioration

- Score par instance (accord inter-instances), agrégation pondérée
  apprise, persistée et versionnée dans PostgreSQL.
- Sélection automatique de topologies à partir de l'historique.
- `watch` sur `/orchestrate` : état des instances, compteurs,
  incidents récents — lecture POSIX pure.
- **Acceptation** : sur jeu étiqueté, l'agrégation pondérée bat la
  moyenne uniforme ; `watch` répond sans état global partagé.

---

## B. `/entrainement` — prochaines phases

### B1 — Autonomie du neurone (corrige le blocage constaté)

Aujourd'hui `/entrainement` échoue avec « aucun neurone monté
(/llm1..N) » si l'orchestrateur n'est pas actif. Le rendre autonome :

- si aucun `/llm<N>` n'est monté, monter un neurone éphémère sur
  `/llm1` (topologie par défaut `2,4,1`), le démonter en sortie ;
- ou option `--neuron-none` : ingestion directe dans `/db` sans
  passage par le neurone (équivalent à `/hf sync`).
- **Acceptation** : `/entrainement hf://…` fonctionne hors
  orchestrateur, en batch, sur la VM Hurd.

### B2 — Ingestion robuste

- Découper le contenu en extraits (taille max, chevauchement),
  passer chaque extrait par le neurone (entrée → sortie), écrire
  `{entrée, sortie}` dans `training_data` **et** le cache NDJSON.
- Sauter proprement les contenus binaires ou tronqués à la fenêtre
  httpfs de 64 Kio (les marquer `http_status=206/truncated` plutôt
  que d'ingérer du bruit).
- Dédup par checksum (déjà garanti côté `/db`) + compteur
  `duplicates` remonté dans la sortie de commande.
- Reprise sur erreur : un fichier en échec ne bloque pas la suite,
  il est journalisé et resumable (`/entrainement --resume`).
- **Acceptation** : `/hf sync` d'un dépôt mixte (texte + binaire)
  ingère 100 % des fichiers texte, marque les binaires, 0 doublon
  au second passage.

### B3 — Multi-sources et planification

- `/entrainement` accepte une liste de sources (fichier de URLs
  `hf://`/`https://`, une par ligne) : ingestion incrémentale
  planifiée (cron côté Hurd), avec `/watch`-style progression.
- Priorité aux sources jamais ingérées (anti-doublon par checksum).
- **Acceptation** : deux runs successifs sur la même liste n'ajoutent
  aucune ligne (idempotence vérifiée par `select count(*)`).

### B4 — Boucle d'apprentissage (lien phase A4 / evaluator)

- Noter les réponses (`/reply`, résultats d'orchestration) :
  feedback utilisateur stocké dans `/db` (table `feedback` à geler).
- `/entrainement --from-feedback` : ré-ingère les exemples notés
  positivement, ajuste les poids du neurone (`save`/`load` du
  modèle `.nn`), versionne le modèle dans `/db`.
- **Acceptation** : après N feedbacks positifs sur un type de
  requête, `/reply` classe les bonnes sources en tête (mesurable
  sur un jeu de requêtes fixe).

---

## Ordre proposé

1. **A1** (asynchrone + status/result netfs) — débloque le streaming
   côté client (`/watch`, `/submit` suivi de `/status`).
2. **B1** — corrige le blocage `/entrainement` hors orchestrateur.
3. **A2 + B2** — acquisition réseau propre des deux côtés.
4. **B3, A3** — idempotence et rejoueabilité.
5. **A4 + B4** — boucle d'amélioration mesurable.

## Dépendances

- A1/A2 dépendent de la phase 1 d'`inference-translator` (faite) et
  du schéma PostgreSQL gelé (fait).
- B4 dépend de A4 (scores) et d'un schéma `feedback` à geler dans
  `data-base-translator` (bump de version du contrat `/db`).
