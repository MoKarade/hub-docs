# Journal du Personal Data Hub

> Source de vérité unique pour : (1) le plan en cours, (2) l'historique des sessions, (3) les décisions techniques prises au fil de l'eau.
>
> Convention : on append à la fin pour le journal, on edit en place pour la section "Plan en cours".
>
> Tenue à jour à **chaque session importante**. Source : règle 2 du `~/.claude/CLAUDE.md` de Marc — "Tout sauvegarder, rien à oublier".

---

## Plan en cours

### Phase actuelle : **Étape 1 — Refonte UI + nouvelles sources de data**

> Contexte : Session #2 (2026-04-29) a terminé A+C+D+E (62 fichiers, 5 repos). Marc a demandé un redesign UI majeur + élargissement du scope (santé, streaming, sécurité). Les 16 questions de discovery ont été posées et **répondues le 2026-04-29**.
>
> Les décisions sont verrouillées : voir `sessions/2026-04-29_marc_answers_discovery.md`.
> Le plan complet est dans `sessions/SUITE.md`.

**Prochaines actions (dans l'ordre) :**
1. [ ] **Étape 1 Sprint A** — système d'animations framer-motion + `Widget` conteneur + `LayoutProvider` (drag-drop + persistance)
2. [ ] **Étape 1 Sprint B** — SSE realtime + dnd-kit drag-drop + mode focus + resize widgets
3. [ ] **Étape 1 Sprint C** — reskinage complet de toutes les pages dans le nouveau layout
4. [ ] **Étape 2** — Déployer sur le vrai PC (Docker + Ollama + GPU)
5. [ ] **Étape 3** — Phase 0 fin (tunnel Cloudflare + backup restic)
6. [ ] **Étape 4** — Phase 2 fin (Marc fournit son Google Takeout)
7. [ ] **Étapes 5-7** — Santé (Garmin/Google Fit) + Streaming/Gaming + Sécurité+Suppression

---

### Phase actuelle (archive) : **Phase 2 — Localisation Google Maps Timeline**

#### Sprint 5 — Polish Phase 1 ✅ TERMINÉ

- [x] **Polish 1** — Fix Disnat ticker 1-caractère : `VISA INC CLASS-A` ne capturait pas `V` car la regex exigeait min 2 chars (`{1,10}` + `[A-Z]` = 2 min). Changé en `{0,9}` → 1 char acceptable. Vérifié : VISA correctement extrait.
- [x] **Polish 2** — Prompt AI plus précis sur la devise. Ajout `_ANSWER_SYSTEM_PROMPT` dédié pour le 2e appel LLM (formulation), qui rappelle que Marc est au Québec, devise CAD par défaut, sauf compte Disnat USD. **Plus de "euros"** dans les réponses.
- [x] **Polish 3** — Replay Disnat clean (truncate + re-import). Vérifié : 21 transactions + 25 positions, plus aucune incohérence.

**Bug LLM connu (acceptable)** : Qwen peut générer des dates invalides (`2026-02-29` alors que 2026 n'est pas bissextile). L'erreur Postgres remonte proprement via HTTP 400 avec le SQL fautif. Pas un bug du système — un fait du LLM.

---

#### Phase 2 — Localisation Google Maps Timeline (en cours)

**But** : importer l'historique de localisation de Marc depuis Google Takeout pour pouvoir répondre à des questions comme "où étais-je le 15 janvier 2026 à 14h ?", "combien de km parcourus en mars ?", etc.

- [x] 2.1 JOURNAL préparé (9 soucis anticipés)
- [x] 2.2 Modèle `LocationPoint` créé (timestamp UTC, latitude/longitude `Numeric(10,7)` pour précision ~1cm, latE7/lngE7 entiers pour idempotence, accuracy_m, altitude_m, activity_type optionnel) + migration `5044ada9f866_phase2_location_points` appliquée.
- [x] 2.3 3 endpoints `/v1/locations/*` : POST `/points` (idempotent), GET `/points` avec filtres temporels (start/end/start_date/end_date), bbox géographique (min/max_lat/lng), activity_type, source, pagination. GET `/points/{id}`.
- [x] 2.4 `ijson>=3.3.0` ajouté aux deps `hub-ingest` (streaming JSON pour les fichiers 100MB+).
- [x] 2.5 Parser `google_takeout_timeline.py` : streame `Records.json` via `ijson.items('locations.item')`, parse les timestamps (epoch ms ou ISO 8601), filtre `accuracy > 100m`, sample 1 point / 30 sec, extrait l'`activity_type` principal (par confiance), convertit latE7/lngE7 → Decimal lat/lng à 7 décimales.
- [x] 2.6 Pipeline `google_timeline.py` : workflow standard (dump raw, POST par point avec retry tenacity, move processed/). Log de progression toutes les 1000 points.
- [x] 2.7 `C:\hub\inbox\google-timeline\` créé + [README](inbox/google-timeline/README.md) qui guide Marc : comment télécharger via takeout.google.com → extraire ZIP → copier `Records.json` → lancer `docker compose --profile ingest run --rm hub-ingest`.

**Sprint Phase 2 = CODE-COMPLETE.** Marc doit fournir son JSON Takeout pour activer l'import. Tout est prêt à parser dès que le fichier arrive dans l'inbox.

**Bilan endpoints API** : 19 routes exposées sous `/v1/*` (health, finance × 7, locations × 3, ai × 2 + root).

**Soucis anticipés Phase 2 :**

1. **Volume énorme** : un export Takeout peut faire 100MB-1GB de JSON. `ijson` obligatoire pour streaming, sinon OOM kill.
2. **2 formats Takeout** : ancien `Records.json` (locations[]) et nouveau Timeline 2024+ avec `semanticSegments`. Le parser doit auto-détecter ou Marc précise.
3. **Timestamps en ms epoch ou ISO** : selon la version. Convertir tous en UTC ISO.
4. **Précision GPS variable** : indoor/tunnel = 100m+. On filtre.
5. **Idempotence** : la même position peut apparaître dans plusieurs exports successifs. Hash sur `(timestamp_ms, latE7, lngE7)`.
6. **Vie privée extrême** : c'est tout l'historique géo de Marc. Doit rester local, ne JAMAIS sortir.
7. **Sample temporel** : 1 point / 30 sec = compromis entre détail (rester debout en café = OK avec 1 point/30s) et volume (1 jour = ~2880 points max).
8. **Activity type** : Google fournit parfois (driving, walking, cycling, still). Optionnel à conserver.
9. **Marc n'a pas encore le fichier** : on code le parser/pipeline d'abord, Marc fera le download Takeout après. Code prêt à parser dès qu'il dépose le JSON.

**Livrable Phase 2 :** une fois Marc fournit son Takeout, ses 10 ans d'historique de localisation sont en DB, accessibles via `/v1/locations/points` + interrogeables via `/v1/ai/ask` ("où étais-je le X ?").

---

### Phase 1 — Banking + IA basique ✅ TERMINÉE

(Détails ci-dessous, intacts pour traçabilité.)


#### Sprint 1 — Foundation DB + API endpoints ✅ TERMINÉ

**But :** avoir les tables et les endpoints HTTP pour stocker une transaction bancaire de bout en bout.

- [x] 1.1 Audit de la structure existante de `hub-core` (`src/db/`, `src/api/v1/`, `src/core/config.py`)
- [x] 1.1b **Bonus** : ajout d'un bind mount + `uvicorn --reload` dans `docker-compose.dev.yml` → hot-reload du code sans rebuild de l'image
- [x] 1.2 Modèles SQLAlchemy `Account` + `Transaction` créés dans `hub-core/src/db/models/`, avec `DateTime(timezone=True)` pour les timestamps UTC
- [x] 1.3 Migration Alembic `c84411d473b8_phase1_finance_accounts_transactions` générée + appliquée → 3 tables en DB (`accounts`, `transactions`, `alembic_version`)
- [x] 1.4 Schémas Pydantic + 5 endpoints `/v1/finance/*` (POST/GET accounts, POST/GET transactions, GET account by id) avec idempotence par `(institution, account_number_masked)` côté account et par `dedup_hash` SHA-256 côté transaction
- [x] 1.5 10 tests curl passent : création, idempotence, filtres, validation Pydantic (debit XOR credit), 404 sur compte inexistant

**Livrable Sprint 1 :** on peut créer un compte et insérer une transaction via l'API. La validation et l'idempotence sont en place. Rien d'importé pour de vrai encore — c'est le job du Sprint 2.

#### Sprint 2 — Parser CSV compte courant + épargne ✅ TERMINÉ

- [x] 2.1 Audit du skeleton hub-ingest : structure propre (Connector ABC, dump_raw, ntfy, APScheduler), toutes les deps déjà là (httpx, tenacity, structlog).
- [x] 2.2 Mise à jour `hub-ingest/src/core/config.py` : ajout `hub_api_base_url`, `inbox_dir`, `run_mode`.
- [x] 2.3 Dockerfile hub-ingest créé (Python 3.13-slim + curl, install editable).
- [x] 2.4 hub-ingest ajouté au compose dev sous le profile `ingest` (lancé à la demande via `docker compose --profile ingest run --rm hub-ingest`). Bind mounts : src + inbox + raw_events.
- [x] 2.5 Parser `desjardins_csv.py` : dataclass `DesjardinsCsvRow` + `parse_desjardins_csv(path)` qui yield les rows. Decode en cp1252, skip lignes vides, tolère lignes mal formées (log + skip). Mappe `EOP`→`checking`, `ET1/ET2/...`→`savings`. Hash SHA-256 sur `transit|type|date|seq|description|debit|credit|balance`.
- [x] 2.6 `pipelines/desjardins.py` : classe `DesjardinsPipeline` avec workflow complet (dump raw → group par compte → ensure_account idempotent → POST txn idempotent → move to processed/ uniquement si zéro erreur). Cache local des UUIDs comptes pour limiter les POST. Retry tenacity (3 tentatives, backoff exponentiel) sur les appels API.
- [x] 2.7 `main.py` refactoré avec dispatch `run-once` vs `scheduler`. Sprint 2 utilise `run-once`.
- [x] 2.8 `C:\hub\inbox\desjardins\` créé + 3 CSV réels copiés depuis `Downloads/`. `.gitignore` préventif posé à la racine de `C:\hub\`.
- [x] 2.9 Premier run : **3 fichiers, 64 transactions, 0 erreur**. Du premier coup.
- [x] 2.10 Validation DB : 2 comptes créés (`377646-EOP` checking, `377646-ET1` savings), 64 transactions réparties (EOP : 22 jan + 21 fév + 15 mars = 58 ; ET1 : 2 + 3 + 2 = 7). Caractères accentués correctement décodés (`AccèsD`). Idempotence validée : ré-import des 3 CSV = toujours 64 transactions en DB.

**Soucis anticipés (à surveiller) :**

1. **Encodage cp1252** : déjà confirmé, à appliquer dès la lecture du fichier. Risque : caractères mal décodés sur les libellés `Acc�sD`.
2. **CSV sans header + 14 colonnes** : utiliser `csv.reader` Python (pas split) pour gérer correctement les guillemets dans les descriptions.
3. **Numéro de séquence remis à zéro chaque mois** : `(transit, account_type, date, seq)` pas globalement unique. `dedup_hash` doit inclure year+month+seq+amounts.
4. **EOP et ET1 dans le même fichier** : un CSV peut couvrir plusieurs comptes. Le parser doit splitter par `account_type`.
5. **Création auto des comptes** : si le parser voit un transit/type qu'on n'a pas en DB, le pipeline doit créer le compte (idempotent côté API).
6. **Hub-ingest pas encore conteneurisé** : pas de Dockerfile, absent du compose. À créer.
7. **Réseau Docker → hub-core** : depuis hub-ingest container, hub-core est joignable via `http://hub-core:8000` (nom du service compose), pas `localhost`.
8. **Fichier en cours d'écriture** : si Marc copie un CSV dans `inbox/`, on pourrait le lire à moitié. Mitigation Sprint 2 : on ne fait pas de watcher continu, on traite à la demande après que Marc a copié. Watcher continu = Sprint 2.5 ou plus tard.
9. **raw_events folder** : doit être bind-mount aussi dans hub-ingest pour persister entre les exécutions.
10. **Erreurs HTTP** : timeouts, 5xx → retry avec tenacity. Si dedup déjà en DB, l'API renvoie 201 idempotent (pas d'erreur).
11. **CSV avec ligne vide en début** : observé dans janv2026.csv (la 1re ligne est blanche) → skip lignes vides.
12. **Description avec `/`** : observé `Loyer/bail /9478 5045 Quebec inc` — le `/` n'a pas de sémantique pour le parser CSV, c'est juste un caractère.

**Livrable Sprint 2 :** Marc voit ses 3 mois de transactions débit/épargne dans la DB. **Premier moment où le hub devient utile.**

#### Sprint 3 — Parser PDF Mastercard ✅ TERMINÉ

- [x] 3.1 Préparation JOURNAL (plan + 12 soucis anticipés)
- [x] 3.2 `pdfplumber>=0.11.4` ajouté aux deps `hub-ingest/pyproject.toml`
- [x] 3.3 Modèle `CreditCardTransaction` (transaction_date + posting_date + amount signé + cashback_rate + card_number_masked + section + statement_date)
- [x] 3.4 Migration Alembic `cfb9d1cc7135_phase1_credit_card_transactions` appliquée → table `credit_card_transactions` créée avec 4 indexes
- [x] 3.5 2 nouveaux endpoints : `POST /v1/finance/credit-card-transactions` (idempotent par dedup_hash) et `GET` (avec filtres account_id, card, section, dates, statement_date)
- [x] 3.6 Test pdfplumber : extraction de texte propre, structure tabulaire bien préservée. Outil de debug `src/scripts/dump_pdf.py` créé pour le futur.
- [x] 3.7 Parser `desjardins_mastercard_pdf.py` : regex `_TXN_LINE_RE` capture `JJ MM JJ MM <description> [rate%] <amount>[CR]`. Heuristique année (`statement_date.year - 1` si mois > mois du relevé, sinon `statement_date.year`). Mapping section : marqueurs `DESCRIPTION DES TRANSACTIONS COURANTES`, `Opérations au compte`, `REMISES EN ARGENT` (skip).
- [x] 3.8 Pipeline `mastercard.py` calqué sur `desjardins.py`. Branché dans `main.py run-once`.
- [x] 3.9 4 PDFs copiés dans `C:\hub\inbox\desjardins-cc\` (renommés `2026-MM-mastercard.pdf`).
- [x] 3.10 **Premier run : 4 PDFs, 360 transactions parsed, 360 pushed, 0 erreur HTTP**. MAIS : 352 en DB (≠ 360) → bug détecté.
- [x] 3.11 **Bug rencontré et résolu — collisions de hash sur transactions répétées à l'identique.** Le 26 décembre 2025, Marc a 4 transactions identiques `MYIQ.COM/CA VANCOUVER BC 0,70` (2 avec cashback 2%, 2 sans). Mon `dedup_hash` initial ne distinguait pas ces lignes (pas de seq_num au PDF, et le hash incluait `(date, description, amount, section)` mais pas le cashback_rate ni un compteur). Résultat : 4 lignes du PDF → 1 ligne en DB. Diff : 8 transactions perdues sur les 4 PDFs. **Fix** : ajout d'un `occurrence_index` (compteur par tuple `(date, description, amount, section)` dans le PDF) intégré au hash sous forme `#1`, `#2`... Truncate + replay → **360/360 en DB**, idempotence revalidée. **Leçon** : pour les sources sans identifiant unique par transaction, prévoir un compteur d'occurrences dans la clé de dédup.

**Total Phase 1 hors investissement : 424 transactions de Marc importées (64 bancaires + 360 carte de crédit) sur 4 mois.**

**Soucis anticipés Sprint 3 :**

1. **Date sans année** : `06 12` (= 6 décembre). À résoudre via la date du relevé. Le relevé du 8 janvier 2026 inclut des transactions de décembre 2025 + début janvier 2026. Heuristique : si la date `MM` du PDF > `MM` de la date du relevé, on est sur l'année précédente.
2. **Suffixe `CR`** : crédits suffixés `CR` (paiements, remboursements). Le parser doit détecter et inverser le signe.
3. **Espaces dans les nombres** : `1 000,00CR`, `2 944,21`. À nettoyer (replace " " puis "," → ".").
4. **Virgule comme séparateur décimal** : standard FR/CA, à convertir.
5. **Description avec ville/province collée** : `VICTORIAVILLEQC`. On garde la chaîne complète sans tenter de parser les CC postaux.
6. **2 sections distinctes dans le PDF** : "Transactions courantes" (avec carte) et "Opérations au compte" (paiements à la carte). Format différent. Les deux doivent être importés.
7. **Section "Remises en argent"** : breakdown du cashback par catégorie. Skip pour Sprint 3 (le rate par transaction suffit).
8. **Numéro de compte ≠ numéro de carte** : sur le PDF, le compte = `5598 22** **** 5004`, mais les transactions sont attribuées à une carte précise (`5598 22** **** 5020`). Marc a confirmé : 5004 = compte, 5020 = sa carte principale. Le modèle doit distinguer.
9. **Multi-page** : 4 pages, transactions étalées. pdfplumber doit extraire toutes les pages dans l'ordre.
10. **Pas de seq_num** : pour le hash, on combine `(account_number, transaction_date, posting_date, description, amount_signed)`.
11. **Re-télécharger un relevé** : Desjardins génère le même PDF chaque fois → contenu identique → même hashes → idempotence garantie.
12. **Échec d'extraction pdfplumber** : si pdfplumber casse sur un PDF (rare mais possible), il faut un fallback ou skipper proprement avec log d'erreur.

**Livrable Sprint 3 :** transactions carte de crédit visibles + cashback rate enregistré par transaction.

#### Sprint 4 — Parser PDF Disnat ✅ TERMINÉ

- [x] 4.1 JOURNAL préparé (12 soucis anticipés)
- [x] 4.2 Modèles `InvestmentTransaction` + `InvestmentPosition` créés. Sépare bien transactions (operations + dividendes + frais + transferts) et snapshots mensuels (positions au close du mois).
- [x] 4.3 Migration `77b853f4a31d_phase1_investment_transactions_positions` appliquée.
- [x] 4.4 4 nouveaux endpoints API : POST/GET pour `investment-transactions` et `investment-positions` (idempotent par dedup_hash).
- [x] 4.5 Parser `disnat_pdf.py` : extraction texte + regex. Découpage par sous-compte (`5NFL7A3` CAD, `5NFL7B1` USD). Détection des sections via marqueurs `Activité mensuelle` et `Détails de vos actifs`. Heuristique pour parser les blocs multi-lignes (ex: `TRSF IN` continuation). Gestion des "stop words" qui ressemblent à des tickers (ETC, UCITS, SA, INC...) → re-fusionnés dans la description.
- [x] 4.6 Pipeline `disnat.py` : 1 PDF → 2 streams (transactions + positions) → 2 endpoints différents.
- [x] 4.7 `C:\hub\inbox\disnat\` + 2 PDFs réels copiés (`2026-01-disnat.pdf`, `2026-03-disnat.pdf`).
- [x] 4.8 **Run réel : 2 PDFs, 21 transactions + 25 positions, 0 erreur.**
- [x] 4.9 **2 bugs trouvés et résolus :**
  - **Bug 1 — Amount parsing greedy** : "DEPOT SIPC - CP20465 600,00" était parsé comme amount=465600 car la regex `\d{1,3}(?:\s\d{3})*,\d+` matchait "5 600,00" (5 chiffres + espace + 600 + virgule). Fix : ajout d'un lookbehind `(?:^|(?<=[^\d]))` qui empêche de matcher après un chiffre.
  - **Bug 2 — Symbol pollution par stop-words** : "GOLD BULLION SECS LTD ETC" → symbol="ETC" (faux : ETC = Exchange Traded Commodity, partie du nom). Pareil pour "AMUNDI MSCI WORLD UCITS" → symbol="UCITS". Fix : liste `_NOT_TICKERS` qui re-fusionne ces faux tickers dans la description.
- [x] 4.10 Imperfection mineure non critique : "VISA INC CLASS-A V" → le `V` (vrai ticker 1 caractère) n'est pas extrait car ma regex exige min 2 chars. La description et le montant sont corrects, juste symbol manquant. Acceptable pour MVP.

**Total Phase 1 : 64 transactions bancaires + 360 cartes de crédit + 21 transactions investissement + 25 positions = 470 enregistrements de Marc importés.**

#### Phase 1 fin — Endpoint `/v1/ai/ask` ✅ TERMINÉ

- [x] Module `src/api/v1/ai.py` avec :
  - `GET /v1/ai/ping` : smoke test Ollama (vérifie le modèle répond)
  - `POST /v1/ai/ask` : workflow complet question→SQL→exec→réponse
- [x] Garde-fou SQL : `_validate_sql()` rejette les non-SELECT, les mots-clés interdits (INSERT/UPDATE/DELETE/...), et les tables non whitelistées.
- [x] `statement_timeout = 5000ms` côté Postgres pour limiter les requêtes lentes.
- [x] Schéma DB inclus dans le prompt système + 4 exemples few-shot pour orienter Qwen.
- [x] Test réel : "Combien j'ai dépensé en restaurants en mars 2026 ?" → SQL généré valide → 374,11 CAD trouvés. Le LLM a juste dit "euros" au lieu de "CAD/$" — micro-bug de formulation, ignorable.

**Performance** : Qwen 14B chargé en VRAM, 2 appels par /v1/ai/ask (génération SQL + reformulation), latence totale ~5-10s sur RTX 5080.

**Soucis anticipés Sprint 4 :**

1. **Format radicalement différent du PDF Mastercard** : Disnat = relevé de portefeuille mensuel (pas un journal séquentiel de transactions). 6 pages.
2. **2 sous-comptes par PDF** : `5NFL7A3` (CAD) et `5NFL7B1` (USD). Chacun a sa section. Le parser doit traiter les deux séparément.
3. **3 sections par sous-compte** : Profil (intérêts/dividendes + variation encaisse), Activité mensuelle (transactions du mois), Détails de vos actifs (positions = snapshot fin de mois).
4. **Devises mixtes** : le compte CAD A3 contient aussi des actifs en USD ! (ex: AMUNDI MSCI EM ASIA UCITS ANDXF 180 unités cotées en USD). La colonne `Devise` du tableau identifie la devise du marché.
5. **Pas de transactions dans le compte USD** dans le mois où il n'y a pas eu d'opération (jan 2026) → seulement les transferts initiaux + détails. Le pipeline doit gérer "0 transaction mais N positions".
6. **Tableau "Activité mensuelle"** : colonnes `Date trans | Date règlement | Opération | Quantité | Description | Prix | Montant`. Multi-ligne possible (`TRSF IN` sur ligne séparée du nom du titre).
7. **Tableau "Détails de vos actifs"** : colonnes `Description | Symbole | Quantité | Coût unitaire moyen | Coût comptable | Prix marché | Devise | Valeur marchande | %` + indic + statut. Plus de colonnes que d'autres tableaux.
8. **Symboles boursiers manquants parfois** : ex AMUNDI MSCI WORLD UCITS n'a pas de symbole rempli, GOLD BULLION non plus. Skip ou store as null.
9. **Numéros de ligne mal alignés** : pdfplumber peut couper au milieu si le texte multi-ligne. À tester.
10. **Idempotence positions** : on stocke 1 snapshot par compte par statement_date. Re-import du même PDF = même snapshot. Clé naturelle : `(account_id, statement_date, symbol_or_description)`.
11. **Idempotence transactions** : pas de seq_num. Hash sur `(account, date_trans, date_reg, operation, quantity, description, prix, montant, occurrence)`. Mêmes leçons que Sprint 3.
12. **Statement_date au format différent** : `Au 31 janvier 2026` (pleine lettre) vs Mastercard `08 01 2026`. Regex différente.

**Livrable Sprint 4 :** valeur du portefeuille tracée mois par mois + historique des transferts/achats/ventes/dividendes/frais.

### Après Phase 1

- Phase 1 fin : endpoint `POST /v1/ai/ask` qui prend une question en français, génère SQL via Qwen, valide, exécute, formule la réponse.
- Phase 0 final (deferred) : Cloudflare Tunnel + Access + DuckDNS + vault age+sops + backup restic.
- Phase 2 : Google Maps Timeline.

---

## Décisions techniques au fil de l'eau

### 2026-04-28

- **OneDrive = source de vérité** pour les `*.md` stratégiques (CLAUDE.md, 01-04). `C:\hub\` reçoit des copies de travail. Source : Marc, choix B explicite.
- **Banque unique** : Desjardins (AccèsD + Disnat). 5 comptes, 3 formats d'export différents (CSV, PDF Mastercard, PDF Disnat).
- **Pas de service tiers payant** pour récupérer les données bancaires (Marc abandonne son ancien service payant). Cohérent avec règle 5 "tout gratuit".
- **Pas de scraping AccèsD** (auth forte + risque blocage compte). Exports manuels CSV/PDF only.
- **Disnat n'expose pas de CSV** → on parse les PDF mensuels. Plus complexe mais inévitable.

### Décisions architecturales formelles

→ ADRs dans `hub-docs/decisions/` (à créer au fil de l'eau pour les choix structurants).

---

## Journal des sessions

### Session #1 — 2026-04-28

**État de départ :**
- 5 repos clonés dans `C:\hub\` (hors OneDrive pour éviter conflits de sync)
- `~/.claude/CLAUDE.md` global déjà installé
- Aucun service jamais lancé en vrai
- Aucune feature implémentée (juste les skeletons)

**Actions de la session :**

1. **Audit cross-repos** : lecture des 5 README + 5 CLAUDE.md → compréhension globale du hub.
2. **Copie des 5 docs stratégiques** OneDrive → `C:\hub\` (`CLAUDE.md`, `01_risk_audit.md`, `02_phasing.md`, `03_repo_structure.md`, `04_master_plan.md`) pour que les chemins relatifs `../../CLAUDE.md` des sous-CLAUDE.md résolvent correctement.
3. **Phase 0 (locale) lancée et validée du premier coup :**
   - Génération SECRET_KEY + POSTGRES_PASSWORD via openssl, écriture du `.env` de hub-deploy
   - Pull Ollama : `qwen2.5:14b-instruct` (9 GB) + `nomic-embed-text` (270 MB) — en background parallèle
   - `docker compose up -d --build` : 3 conteneurs healthy (postgres pg16+pgvector, redis 7-alpine, hub-core FastAPI)
   - 10 tests automatisés passent (endpoints `/`, `/v1/health`, `/v1/ready`, `/docs`, `/openapi.json` + connexion psql directe + extension pgvector + prompt Ollama)
   - Performance Ollama : Qwen 14B sur RTX 5080 → **57.6 tokens/sec** après warm-up. Cold-start ~29s (chargement modèle en VRAM).
4. **Discovery Phase 1 banking :**
   - Marc utilise uniquement Desjardins (5 comptes, 2 portails : AccèsD et Disnat).
   - 3 formats d'export identifiés et analysés : CSV compte courant (cp1252), PDF Mastercard (texte propre), PDF Disnat (relevé portefeuille mensuel).
   - 9 fichiers réels téléchargés et inspectés : 3 CSV (janv/fév/mars 2026), 4 PDF Mastercard (janv→avril), 2 PDF Disnat (jan + mars).
   - Confirmé : pas de CSV pour Disnat → parsing PDF inévitable.

**Décisions prises pendant la session :**

- OneDrive reste source de vérité pour les `.md` stratégiques (vs `C:\hub\`).
- Méthode banque retenue : exports manuels (CSV pour AccèsD, PDF pour Mastercard et Disnat).
- Path d'implémentation Phase 1 banking : 4 sprints itératifs (DB → CSV → Mastercard → Disnat).

**À retenir pour Marc :**
- Adresse Disnat (Lévis) ≠ adresse Mastercard (Québec) → probablement un déménagement récent. Pas critique.
- "Transferts reçus" massifs en janvier sur Disnat → migration récente d'un autre courtier (probablement l'ancien service tiers payant). À confirmer.

**Suite prévue :** Sprint 1 dans cette même session.

**Sprint 1 lancé et terminé dans la foulée :**

- Audit hub-core : structure FastAPI/async/SQLAlchemy 2 propre, alembic configuré async, deps complètes (asyncpg + psycopg v3 binary).
- Décision archi mineure : ajout d'un **bind mount** `../hub-core/src:/app/src` + `--reload` dans le compose dev, pour itérer sans rebuild. Sinon chaque modif `.py` aurait nécessité ~30s de rebuild Docker.
- Décision archi mineure : `DateTime(timezone=True)` partout (vs naive). On stocke en `timestamp with time zone` dans Postgres pour éviter le silent-loss de tz info quand on écrit `datetime.now(UTC)` côté Python.
- Modèles : `Account` (institution + account_type + masked_number + currency + actif + timestamps) et `Transaction` (FK account, date, description, debit/credit/balance, métadonnées source pour audit, dedup_hash unique). Carte de crédit et investissement seront des modèles séparés en Sprint 3 et 4.
- Idempotence : `Account` dédupliqué par `(institution, account_number_masked)`, `Transaction` par `dedup_hash` (SHA-256). Les POST sont idempotents — re-jouables sans risque de doublon. Critique pour le watcher du Sprint 2.
- Validation Pydantic : `model_validator` qui interdit "debit ET credit" et "ni debit ni credit" → renvoie 422 explicite.

**Décisions techniques actées dans la session :**
- Bind mount + reload pour le dev (commit dans `hub-deploy/docker-compose.dev.yml`).
- DateTime timezone-aware partout.
- Idempotence par hash côté transaction (pas par contrainte composite naturelle), pour pouvoir étendre facilement aux autres formats (Mastercard, Disnat) qui n'ont pas de seq_num.

**Sprint 2 enchaîné dans la même session #1 :**

- **Architecture choisie** : pas de watcher continu pour Sprint 2. Mode `run-once` qui scan l'inbox + exit. Plus simple à débugger, plus rapide à itérer. Watcher continu (APScheduler ou watchdog) reporté à plus tard si nécessaire.
- **Mode "lancement à la demande"** : `docker compose --profile ingest run --rm hub-ingest`. Le service est sous un profile `ingest` exclu du `compose up` standard. Marc lance la commande quand il a copié de nouveaux CSV.
- **Bind mounts compose** : `../inbox:/var/hub/inbox` et `../raw_events:/var/hub/raw_events` côté host = `C:\hub\inbox` et `C:\hub\raw_events`. Erreur initiale `../../hub/inbox` corrigée avant le build.
- **Création auto des comptes** : le pipeline détecte un nouveau `(transit, account_type)` et POST `/v1/finance/accounts`. Idempotent côté API → pas de doublon même si on rejoue.
- **Bug rencontré et résolu pendant la session (1) — Doublon "Paie ROBOVIC" du 1er janvier** : ma transaction de test du Sprint 1 utilisait un `dedup_hash` fabriqué manuellement qui ne suivait pas la formule du parser. Quand le pipeline a importé le vrai CSV, il a calculé un hash différent → 2 entrées en DB pour la même transaction sémantique. **Résolu** en supprimant la transaction de test (DELETE WHERE dedup_hash = '1f70...'). **Leçon** : pour les tests futurs, utiliser la même formule de hash que le parser, pas inventer.
- **Bug évité — chemins bind mount** : initialement écrit `../../hub/inbox` (qui aurait pointé sur `C:\inbox` au lieu de `C:\hub\inbox`). Corrigé avant le build, identifié par relecture mentale du chemin.
- **Détails sympa observés dans les vraies data de Marc** :
  - Plus grosse opération du trimestre : 7000$ "Virement à EOP" (transfert épargne→courant le 16 mars).
  - Loyer change entre janvier (1039$ pour `9478 5045 Quebec inc`) et mars (1600$ pour `Valerie cameron`) → cohérent avec un déménagement Québec→Lévis confirmé indirectement par les adresses différentes des relevés Disnat (Lévis) vs Mastercard (Québec).
  - Caractères accentués bien décodés (cp1252 → utf-8 → Postgres) : `AccèsD`, `Intérêt sur ET`.

**Fin de session #1.** Phase 0 + Phase 1 Sprints 1 et 2 terminés. À reprendre : Sprint 3 (parser PDF Mastercard, 4 fichiers : janv/fév/mars/avril 2026).

---

### Session #2 — 2026-04-29 (nouveau PC, code-only)

**Contexte** : nouveau PC Windows (`dessin14`), pas l'ancien (`marcr`). Marc a explicitement demandé de **ne rien installer/lancer** localement — uniquement écrire le code qu'il déploiera plus tard sur son vrai PC équipé.

**Mandat** : faire les chunks A, C, D, E (tout sauf B = Phase 3).

**Livrables (62 fichiers, 5 repos)** :

- **A — Docs hub-docs** : `03-data-model.md`, `04-api-contract.md`, `06-security.md`, ADRs 0003 (postgres+pgvector), 0004 (ollama natif), 0005 (cloudflare tunnel vs tailscale), 0006 (age+sops), 0007 (versioning sous-chemin Caddy)
- **C — Frontend** : pages `/finances` (3 onglets banque/carte/invest + filtres), `/search` (chat IA persistant + SQL/rows collapsibles), `/locations` (carte react-leaflet dynamic SSR-off), home wired sur l'API live (`LiveStatCards` + `SpendingChart` recharts), `lib/api.ts` typed pour les 19 routes, `formatCurrency` defaut CAD/`fr-CA`
- **D — Tests + outillage** : ~80 tests pytest (4 parsers + 4 endpoints + SQL validator), 5 CI workflows (1/repo), `clone_all.ps1`, `replay.py` (ADR-0002)
- **E — Infra Phase 0 fin** : `docker-compose.prod.yml` (Postgres pas exposé + Caddy + cloudflared), `cloudflared/README.md` (tunnel + access TOTP), `.sops.yaml` + `init_secrets.ps1` + `decrypt_env.ps1`, vault `secrets/README.md` rewrite, restic backup (3 scripts + README)

**Validation** : reviewer agent indépendant a trouvé **7 vrais bugs**, tous fixés :
1. `backup.ps1` — `_sendNotif` appelé avant définition + Out-File UTF-16 BOM (cassait le restore psql) → réordonné + `[System.IO.File]::WriteAllText(..., UTF8Encoding $false)`
2. `app/search/page.tsx` — `useSearchParams()` casse au build Next 15 sans Suspense → wrappé `<SearchPageInner>` dans `<Suspense>`
3. `tests/test_finance_transactions.py` — comparaison string `"100.00"` casse sur SQLite (renvoie `"100"`) → `Decimal(body["debit"]) == Decimal("100.00")`
4. `docker-compose.prod.yml` — réseau implicite → ajout explicite `networks: default: name: hub_prod`
5. `hub-deploy CI` — `python` pas dispo sur Ubuntu runner → `python3` + `setup-python@v5`
6. `backup.ps1` — `--exclude '*.yaml'` excluait aussi les `.enc.yaml` (perte des secrets dans le snapshot) → `--exclude-file` ciblé `*.tmp + backup/staging`
7. `app/finances/page.tsx` — date filters envoyés à un endpoint qui ne les accepte pas → filtre côté client uniquement, commenté

**Documentation** : `sessions/2026-04-29_session2_summary.md` (récap détaillé) + `sessions/SUITE.md` (roadmap des prochaines étapes : reprise PC équipé → Phase 0 fin → Phase 2 fin → Phase 3 → Phase 4+ → Phase 5).

**Preview frontend** : 4 fichiers HTML statiques dans `_previews/` (Tailwind CDN + lucide SVG inline) reproduisent fidèlement le rendu des 4 pages. Marc peut ouvrir depuis Google Drive sans aucune install.

**Cleanup PC** : 3 fichiers locaux à `C:\Users\dessin14\.claude\` ont été supprimés en fin de session (cf. demande Marc « rien sur ce PC »). Le code projet sur Google Drive (`G:\Mon disque\...`) reste — c'est syncé partout, pas spécifique au PC.

**Fin de session #2.** Code complet pour Phases 0-2. À reprendre : redéployer sur le vrai PC + Phase 0 fin réelle (tunnel + backup) + import Takeout pour finaliser Phase 2.

---

#### Demande de fin Session #2 — Marc redesign brief

**Verbatim Marc** : *« le front end est trop... IA, je veux que ce soit plus beau plus interactif et moins statique, je veux aussi plus de data genre la santé, mes réseaux, et tout ce que tu peux imaginer... aussi genre la sécurité de mes données, qc ou voir que mes données sur le web sont safe ou nulle part »*

3 chantiers nouveaux à prioriser pour les prochaines sessions :
1. **Refonte UI** — moins « AI-générique », plus beau, plus interactif. Le 2ᵉ feedback design Marc — la palette ink + vert n'est toujours pas assez distinctive.
2. **Élargir les sources** — santé (Apple Health, Garmin?), réseaux (Spotify, YouTube, Twitter…), gaming, lecture, browser history.
3. **Module sécurité OSINT** — check breach (HIBP), footprint web, inventaire de comptes, score d'exposition.

Brief complet + 16 questions ouvertes à Marc : [`sessions/2026-04-29_marc_redesign_request.md`](sessions/2026-04-29_marc_redesign_request.md).
Roadmap mise à jour : [`sessions/SUITE.md`](sessions/SUITE.md) (Étape 0 = discovery UI ; Étape 1 = refonte ; Étape 5 = santé/réseaux/sécu).

À traiter en début de Session #3.

---

### Session #3 — 2026-04-29 (réponses discovery + plan verrouillé)

**But :** Recevoir et documenter les réponses aux 16 questions UI/scope. Verrouiller les décisions. Mettre à jour la roadmap.

**Réponses Marc (verbatim + décodage) :** [`sessions/2026-04-29_marc_answers_discovery.md`](sessions/2026-04-29_marc_answers_discovery.md)

#### Décisions verrouillées

| Décision | Valeur |
|---|---|
| Palette | Dark-only (garder ink + vert) |
| Layout | Refonte complète — moins SaaS-grid, plus modulaire/personnel |
| Photos dans UI | NON |
| Easter eggs / humour | NON |
| Animations | Tout (framer-motion) : transitions, stagger, hover lift, realtime pulse |
| Drag-drop | Oui + plus (resize, pin/unpin, persistance layout) |
| Realtime | SSE (`GET /v1/events/stream`) |
| Mode focus | Oui — widget click = expand fullscreen |
| Santé | Garmin + Google Fit — toutes les métriques disponibles |
| Streaming | YouTube, YouTube Music, Netflix, Disney+, Prime Video, Crunchyroll |
| Gaming | Steam + Xbox |
| Browser/Dev | Chrome history + GitHub |
| Sécurité | Scope complet (HIBP + footprint + inventaire + data brokers + suppression) |
| Coût | Tout gratuit — aucune exception |
| Lecture | NON (Kindle, Pocket, etc.) |
| Nouveau module | **Module Suppression** — tracker demandes de suppression données en ligne (PIPEDA + Loi 25 QC) |

#### Note importante — Google Drive

Marc a demandé que tout soit sauvegardé sur son Drive. C'est déjà le cas :
- Le projet vit dans `G:\Mon disque\...` → synchronisé automatiquement sur Google Drive
- + Tout est pushé sur GitHub (MoKarade) en fin de session

Double sauvegarde à chaque session = zéro risque de perte.

#### Roadmap mise à jour

`sessions/SUITE.md` entièrement mis à jour avec le plan concret pour les étapes 1-10.
La prochaine session commence par **Étape 1 Sprint A** (système animations + Widget conteneur + LayoutProvider).
