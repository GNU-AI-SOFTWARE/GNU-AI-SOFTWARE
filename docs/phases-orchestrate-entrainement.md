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
> - ✅ **A1b** livrée : `/orchestrate` passé en **netfs** (nœuds
>   `status`/`result` en lecture POSIX pure, canal de commandes sur la
>   racine). `make check` 5/5 hôte + VM Hurd.
> - ✅ **A3** livrée : `{"command":"history"}` et
>   `{"command":"replay","run_id":N}` sur `/orchestrate`, lecture
>   `{"select":…}` côté `/db` (contrat étendu, `make check` 4/4).
>   Replay vérifié `match:true` sur VM.
> - ✅ **B3** livrée : `/entrainement --from FILE [--resume]
>   [--cron "EXPR"]` — ingestion multi-sources idempotente (table
>   `ingest_jobs`), compteurs traités/sautés/doublons/erreurs,
>   entrée crontab + log `/root/.entrainement.log`. Vérifié sur VM :
>   `count(*)` stable au second passage.
> - ✅ **A4** livrée : score cosinus par instance persisté dans
>   `run_instances.score`, poids d'agrégation appris (EMA 0.7/0.3)
>   versionnés dans `aggregator_state`, `"topologies": "auto"`
>   (top-K de l'historique), commande `{"command":"watch"}`.
> - ✅ **B4** livrée : `/good` `/bad` notent la dernière réponse
>   `/reply` dans la table `feedback` ; `/entrainement
>   --from-feedback` ré-ingère les exemples « good » (idempotent,
>   SHA-256) et versionne le modèle `.nn` du neurone (hex) dans la
>   table `models`. Vérifié sur VM : `doublons=2` au second run,
>   modèle v1→v3.
> - 🎉 **Toutes les phases A1b/A3/B3/A4/B4 sont livrées, testées sur
>   VM Debian GNU/Hurd et poussées sur `main`** (2026-11-28).

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

---

## Spécification détaillée des phases livrées (archive)

### B3 — Multi-sources et planification de `/entrainement`

**Objectif** : ingérer des corpus entiers de façon incrémentale et
reprenables, sans passer par une session interactive.

- Contrat de commande :
  - `/entrainement --from FILE` : FILE contient une URL `hf://` ou
    `https://` par ligne (`#` = commentaire) ;
  - `/entrainement --from FILE --resume` : saute les URLs déjà en
    cache (mécanisme B2) ;
  - `/entrainement --from FILE --cron "*/30 * * * *"` : planification
    côté Hurd (entrée `crontab`), chaque run journalisé dans
    `/root/.entrainement.log` avec compteurs (traités / sautés /
    doublons / erreurs).
- Persistance de l'état d'ingestion : table `ingest_jobs` (à geler
  dans `data-base-translator`, bump de contrat) : `source_url`,
  `last_run`, `status`, `records_added` — pour éviter de re-lister
  l'arbre HF à chaque passage (cache de l'arbre 24 h).
- **Livrable** : ingestion multi-sources idempotente + cron.
- **Acceptation** : deux exécutions successives du même `--from`
  n'ajoutent aucune ligne (`select count(*)` stable) ; un fichier
  supprimé du dépôt local n'est pas re-téléchargé avec `--resume` ;
  le log cron montre les compteurs.
- **Tests** : `make check` + scénario VM (liste de 3 sources, run
  deux fois, `count(*)` identique).

### A3 — Rejouabilité : `replay` et `history` sur `/orchestrate`

**Objectif** : rejouer et auditer des exécutions depuis la base,
réseau coupé.

- Nouvelles commandes (contrat à geler, SPEC §5 phase 2+) :
  - `{"command": "replay", "run_id": N}` : lit le descripteur et les
    entrées de `runs`/`run_instances`, relance les instances avec la
    même graine (neuron `seed`), compare les sorties, répond
    `{"run_id": N, "replay_id": M, "match": true|false, "diff": [...]}` ;
  - `{"command": "history", "limit": 20}` : dernières exécutions avec
    stratégie, confiance, nb d'incidents ;
  - `{"command": "stats"}` : performances par topologie (moyenne de
    confiance, taux d'échec) calculées depuis `run_instances`.
- Lecture en phase 1 via le protocole de lignes (trivfs) ; les nœuds
  fichiers `/orchestrate/status|result|history` arrivent avec le
  passage netfs (A1, section suivante).
- **Livrable** : replay/history/stats opérationnels.
- **Acceptation** : `replay` d'un run A2 avec le réseau coupé produit
  un résultat identique (graine fixée) ; `stats` reflète les incidents
  persistés en A1.

### A1b — Passage netfs de `/orchestrate` (nœuds fichiers)

A1 a livré l'asynchrone via le protocole de lignes. Le contrat gelé
prévoit l'arborescence netfs pour le streaming :

- `/orchestrate/status`, `/orchestrate/result`, `/orchestrate/history`
  servis comme fichiers, **un curseur par lecteur** (même mécanisme
  que `/db`) : `cat /orchestrate/status` en boucle suit le run en
  direct.
- Migration trivfs → netfs dans `main-hurd.c`/`trivfs-hooks.c` ;
  le moteur (`orchestrate.c`) ne change pas.
- **Acceptation** : `watch`-like en POSIX pur (`while :; do
  cat /orchestrate/status; sleep 1; done`) montre running → done ;
  `make check` toujours vert (le REPL reste trivfs-compatible).

### A4 — Évaluateur et agrégation pondérée apprise

- Score par instance : accord inter-instances (cosinus des vecteurs de
  sortie) + cohérence avec l'historique ; champ `score` de
  `run_instances` enfin renseigné.
- Pondérations d'agrégation apprises : table `aggregator_state`
  (topologie → poids), mise à jour après chaque run à partir des
  scores, versionnée (`version` croissant).
- Sélection de topologies : le descripteur accepte
  `"topologies": "auto"` → choisit les K meilleures topologies de
  l'historique.
- **Acceptation** : sur un jeu étiqueté de 20 runs, l'agrégation
  pondérée apprise bat `majority` et `mean` uniformes (précision
  mesurée) ; `auto` sélectionne une topologie du top-3 historique.

### B4 — Boucle d'apprentissage (feedback → ré-entraînement)

- Nouveau contrat `/db` : table `feedback` (à geler) :
  `id, run_id NULLABLE, source_url, query, rating INT, created_at`.
- Côté client : `/good` / `/bad` après une réponse `/reply` ou un
  résultat d'orchestration → écrit une ligne `feedback` dans `/db`.
- `/entrainement --from-feedback` : ré-ingère les exemples notés
  `rating > 0`, ajuste les poids du neurone éphémère (`save`/`load`
  `.nn`), et stocke le modèle versionné dans `/db` (colonne
  `model_blob` ou table `models` à geler).
- **Acceptation** : après 10 feedbacks positifs sur un type de
  requête, `/reply` classe la bonne source en rang 1 sur un jeu de
  requêtes fixe (test déterministe) ; le modèle versionné est
  rechargeable (`load`) et redonne les mêmes sorties.

### Ordre conseillé — ✅ tout est livré (2026-11-28)

1. ✅ **A1b** (netfs) — streaming POSIX pur, base de `watch`.
2. ✅ **A3** (replay/history/stats) — audit + rejouabilité, réseau coupé.
3. ✅ **B3** (multi-sources + cron + `--resume`) — ingestion de corpus.
4. ✅ **A4** (scores + agrégation apprise + `topologies: "auto"`).
5. ✅ **B4** (feedback + ré-entraînement + modèle versionné).

Commits sur `main` : orchestrator `a277406`, data-base `06214ce`,
inference `8a16e84`. Tests : `make check` vert sur hôte et VM Hurd
(orchestrator 5/5, data-base 4/4, inference 6/6 sur Hurd).

### Dépendances à geler (bump de contrat `/db`)

- `ingest_jobs` (B3), `feedback` + `models` (B4),
  `aggregator_state` (A4) — schéma dans `data-base-translator`,
  tests `test_contract`/`test_pg.sh` à étendre à chaque table.
