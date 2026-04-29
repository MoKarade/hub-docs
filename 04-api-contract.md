# 04 — Contrat d'API

> Documentation manuelle, **complémentaire** de la doc OpenAPI auto-générée à `http://localhost:8000/docs`. Cette page documente les **invariants métier** et les **garanties** que la doc machine ne peut pas exprimer.

## Conventions générales

### URL de base
- **Local dev** : `http://localhost:8000`
- **Local via Caddy** : `http://localhost/api`
- **Prod via Cloudflare Tunnel** : `https://marc-hub.duckdns.org/api`

### Versioning
- Toutes les routes métier sont préfixées par `/v1/`.
- Garantie : **`/v1` ne casse jamais.** Si on doit casser un contrat, on bump à `/v2` et on garde `/v1` en parallèle.
- Voir `decisions/0007-versioning-souschemin-vs-sousdomaine.md`.

### Format
- **Requêtes / réponses : JSON.** `Content-Type: application/json`.
- **Dates : ISO 8601** (`YYYY-MM-DD` pour les dates, `YYYY-MM-DDTHH:MM:SSZ` pour les datetimes).
- **Décimaux : strings** (Pydantic v2 sérialise les `Decimal` en string pour préserver la précision).
- **UUIDs : strings** au format canonique (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`).
- **Devises : codes ISO 3-lettres** (`CAD`, `USD`).

### Codes d'erreur
| Code | Signification |
|---|---|
| `200 OK` | Lecture réussie |
| `201 Created` | Création réussie (POST) |
| `400 Bad Request` | Erreur d'exécution SQL côté `/v1/ai/ask` |
| `404 Not Found` | Ressource inexistante |
| `422 Unprocessable Entity` | Validation Pydantic échoue (ex: `debit ET credit`, SQL refusé) |
| `503 Service Unavailable` | Dépendance externe down (Ollama unreachable) |

### Idempotence

Tous les endpoints `POST` sont **idempotents par `dedup_hash`** (SHA-256 calculé côté ingest). Si une ressource avec ce hash existe déjà, l'API renvoie `201 Created` avec la ressource existante (pas d'erreur 409).

Pour les comptes : idempotence par `(institution, account_number_masked)`.

### Auth
- **En dev local** : aucune (le hub écoute uniquement sur `localhost`).
- **En prod via tunnel** : Cloudflare Access valide un JWT signé Google + MFA TOTP avant d'atteindre Caddy. Le hub-core lit le header `Cf-Access-Authenticated-User-Email` pour l'identité.

## Catalogue des 19 routes

### Système (3)

#### `GET /`
Renvoie un descripteur basique du service.
```json
{ "name": "Personal Data Hub", "version": "0.1.0", "docs": "/docs", "health": "/v1/health" }
```

#### `GET /v1/health`
Liveness probe. Toujours 200 si le process répond.
```json
{ "status": "ok" }
```

#### `GET /v1/ready`
Readiness probe — vérifie DB + Ollama.
```json
{
  "status": "ok",   // ou "degraded"
  "checks": {
    "database": { "status": "ok" },
    "ollama": {
      "status": "ok",
      "models_available": ["qwen2.5:14b-instruct", "nomic-embed-text"],
      "configured_model": "qwen2.5:14b-instruct"
    }
  }
}
```

### Finance — Comptes (3)

#### `POST /v1/finance/accounts`
Crée un compte. Idempotent par `(institution, account_number_masked)`.

**Payload** :
```json
{
  "institution": "Desjardins",
  "account_type": "checking",   // checking | savings | credit_card | investment
  "account_number_masked": "377646-EOP",
  "nickname": "Mon courant",
  "currency": "CAD",            // ISO 3-lettres
  "is_active": true
}
```

**Réponse 201** : `AccountRead` (id, institution, …, created_at, updated_at).

#### `GET /v1/finance/accounts`
Liste tous les comptes, ordonnés par `(institution, account_number_masked)`.

**Query params** :
- `is_active` : `true` / `false` (filtre optionnel)

**Réponse 200** : `AccountRead[]`.

#### `GET /v1/finance/accounts/{account_id}`
Renvoie un compte par son UUID.

**404** si introuvable.

### Finance — Transactions courantes (2)

#### `POST /v1/finance/transactions`
Crée une transaction bancaire (compte courant ou épargne). Idempotent par `dedup_hash`.

**Payload** :
```json
{
  "account_id": "uuid-du-compte",
  "transaction_date": "2026-03-15",
  "description": "Paie /ROBOVIC INC.",
  "debit": null,
  "credit": "1850.00",          // Decimal en string
  "balance_after": "12480.55",
  "source_format": "desjardins_csv_eop",
  "source_file": "mars2026.csv",
  "source_seq_num": 7,
  "dedup_hash": "<64-char SHA-256>"
}
```

**Validation** :
- `debit XOR credit` : exactement un des deux non-null. 422 sinon.
- `debit` ou `credit >= 0`.
- `dedup_hash` exactement 64 caractères (longueur SHA-256 hex).

**404** si `account_id` n'existe pas.

#### `GET /v1/finance/transactions`
Liste les transactions, ordonnées par `transaction_date DESC, created_at DESC`.

**Query params** :
- `account_id` : UUID
- `start_date`, `end_date` : `YYYY-MM-DD` (bornes incluses)
- `limit` (défaut 100, max 1000)
- `offset`

### Finance — Cartes de crédit (2)

#### `POST /v1/finance/credit-card-transactions`
Crée une transaction Mastercard. Idempotent par `dedup_hash`.

**Payload** :
```json
{
  "account_id": "uuid",
  "card_number_masked": "5598 22** **** 5020",
  "transaction_date": "2025-12-26",
  "posting_date": "2025-12-28",
  "description": "MYIQ.COM/CA VANCOUVER BC",
  "amount": "0.70",             // Positif=achat, négatif=paiement
  "cashback_rate": "0.0050",    // Optionnel, fraction (0,50%)
  "section": "transactions_courantes",  // ou "operations_au_compte"
  "source_format": "desjardins_mastercard_pdf",
  "source_file": "2026-01-mastercard.pdf",
  "statement_date": "2026-01-08",
  "dedup_hash": "<64-char SHA-256>"
}
```

#### `GET /v1/finance/credit-card-transactions`
Filtres : `account_id`, `card_number_masked`, `section`, `start_date`, `end_date`, `statement_date`, `limit`, `offset`. Défaut `limit=200`.

### Finance — Investissement (4)

#### `POST /v1/finance/investment-transactions`
Crée une transaction Disnat (achat, vente, dividende, frais, transfert). Idempotent par `dedup_hash`.

**Payload** :
```json
{
  "account_id": "uuid",
  "sub_account_code": "5NFL7A3",
  "transaction_date": "2026-01-15",
  "settlement_date": "2026-01-17",
  "operation": "ACHAT",
  "description": "NVIDIA CORP",
  "symbol": "NVDA",
  "quantity": "10.000000",
  "unit_price": "525.430000",
  "amount": "-5254.30",         // Négatif (sortie de cash)
  "currency": "USD",
  "statement_date": "2026-01-31",
  "source_format": "desjardins_disnat_pdf",
  "source_file": "2026-01-disnat.pdf",
  "dedup_hash": "<64-char SHA-256>"
}
```

#### `GET /v1/finance/investment-transactions`
Filtres : `account_id`, `sub_account_code`, `symbol`, `operation`, `statement_date`, `limit`, `offset`. Défaut `limit=200`.

#### `POST /v1/finance/investment-positions`
Crée un snapshot mensuel de position. Idempotent par `dedup_hash`.

**Payload** :
```json
{
  "account_id": "uuid",
  "sub_account_code": "5NFL7A3",
  "statement_date": "2026-03-31",
  "description": "NVIDIA CORP",
  "symbol": "NVDA",
  "quantity": "10.000000",
  "average_unit_cost": "525.430000",
  "book_cost": "5254.30",
  "market_price": "615.20",
  "market_value": "6152.00",
  "currency": "USD",
  "portfolio_pct": "12.50",
  "source_format": "desjardins_disnat_pdf",
  "source_file": "2026-03-disnat.pdf",
  "dedup_hash": "<64-char SHA-256>"
}
```

#### `GET /v1/finance/investment-positions`
Filtres : `account_id`, `sub_account_code`, `symbol`, `statement_date`, `limit`, `offset`. Ordonnés par `statement_date DESC, market_value DESC`.

### Localisation (3)

#### `POST /v1/locations/points`
Crée un point GPS. Idempotent par `dedup_hash`.

**Payload** :
```json
{
  "timestamp_utc": "2026-01-15T14:30:00Z",
  "latitude": "46.8123456",
  "longitude": "-71.2089012",
  "accuracy_m": 25,
  "altitude_m": 50,
  "activity_type": "walking",   // still | walking | cycling | driving | ...
  "source": "google_takeout_timeline",
  "source_file": "Records.json",
  "latitude_e7": 468123456,
  "longitude_e7": -712089012,
  "dedup_hash": "<64-char SHA-256>"
}
```

**Validation** : `latitude ∈ [-90, 90]`, `longitude ∈ [-180, 180]`, `accuracy_m >= 0`.

#### `GET /v1/locations/points`
Liste les points avec filtres temporels + bbox géographique + activité.

**Query params** :
- `start`, `end` : ISO datetime UTC
- `start_date`, `end_date` : alternative `YYYY-MM-DD` (les datetimes sont `00:00:00` et `23:59:59`)
- `min_lat`, `max_lat`, `min_lng`, `max_lng` : bornes bbox (floats)
- `activity_type` : filtre activité
- `source` : filtre source
- `limit` (défaut 500, max 10000)
- `offset`

#### `GET /v1/locations/points/{point_id}`
Renvoie un point par UUID. 404 si introuvable.

### IA (2)

#### `GET /v1/ai/ping`
Smoke-test Ollama. Envoie un prompt très court et renvoie le texte généré.

**Réponse 200** :
```json
{
  "status": "ok",
  "model": "qwen2.5:14b-instruct",
  "backend": "http://host.docker.internal:11434",
  "sample_response": "Bonjour"
}
```

**503** si Ollama injoignable.

#### `POST /v1/ai/ask`
Pose une question en français au hub. Workflow LLM → SQL → exec → LLM.

**Payload** :
```json
{ "question": "Combien j'ai dépensé en restos en mars 2026 ?" }
```

**Validation** : `3 <= len(question) <= 500`.

**Workflow** :
1. **Pass 1 LLM** : Qwen 2.5 14B reçoit le schéma DB + 4 exemples few-shot + la question. Génère 1 requête SQL `SELECT` ou `WITH`.
2. **Validation SQL** : `_validate_sql()` rejette les non-SELECT, les mots-clés interdits (`INSERT|UPDATE|DELETE|DROP|TRUNCATE|ALTER|CREATE|GRANT|REVOKE|COPY|VACUUM|REINDEX|CLUSTER|EXPLAIN|ANALYZE|LOCK`), et les tables non whitelistées.
3. **Tables whitelist** : `accounts`, `transactions`, `credit_card_transactions`, `investment_transactions`, `investment_positions`. Pas `location_points` pour l'instant (Phase 2 finance only).
4. **Exécution** avec `SET LOCAL statement_timeout = 5000` (5 secondes max).
5. **Pass 2 LLM** : reformule la réponse en français, en 1-3 phrases courtes, en citant les chiffres exacts et la bonne devise (CAD par défaut, USD pour `5NFL7B1`).

**Réponse 200** :
```json
{
  "answer": "En mars 2026, tu as dépensé 374,11 $ CAD en restaurants...",
  "sql": "SELECT SUM(amount) AS total FROM credit_card_transactions WHERE ...",
  "rows": [{"total": "374.11"}],
  "row_count": 1
}
```

**Codes d'erreur** :
- `400` : exécution SQL échoue (ex: date invalide générée par le LLM, comme `2026-02-29` qui n'existe pas)
- `422` : SQL refusé par le validateur
- `503` : Ollama injoignable

**Performance** : 2 appels Ollama par question. Sur RTX 5080 + Qwen 14B chargé en VRAM, latence totale 5-10 secondes (cold start +29s pour le chargement du modèle).

## Schémas Pydantic

Les schémas sont définis dans `hub-core/src/api/v1/finance.py`, `locations.py`, `ai.py`. La doc OpenAPI à `/docs` les liste tous (cliquer sur "Schemas" en bas).

Les `*Read` modèles incluent `id`, `created_at` et tous les champs DB. Les `*Create` modèles excluent les champs auto-générés (`id`, `created_at`, `updated_at`).

## Garanties que l'API offre

- **Idempotence** : tous les `POST` sont sûrs à rejouer (cf. tableau `dedup_hash`).
- **Pas de mutation cachée** : aucun `POST` ne modifie une ressource autre que celle créée. Pas de side-effects.
- **Read-only AI** : `POST /v1/ai/ask` ne peut JAMAIS écrire en DB (validation SQL).
- **Statement timeout** : aucune requête SQL utilisateur (via `/v1/ai/ask`) ne peut bloquer plus de 5 sec la DB.
- **Pas de leak data inter-comptes** : ce sont les requêtes appelantes qui filtrent par `account_id`. **Pas de scoping serveur** car le hub est mono-utilisateur.

## Garanties que l'API n'offre PAS

- ❌ **Pas de PATCH/DELETE** sur les transactions/positions/points. Tables immutables par design (event sourcing). Pour corriger, il faut soit insérer une transaction de correction, soit purger + replay depuis raw_events.
- ❌ **Pas de transactions ACID multi-ressource** côté API. Chaque `POST` est sa propre commit. Si tu insères 100 transactions et que la 50ᵉ échoue, les 49 premières sont en DB. Mitigation : idempotence par `dedup_hash` permet de rejouer.
- ❌ **Pas de pagination par cursor** — uniquement offset. Suffisant pour les volumes attendus (~3000 transactions à 10 ans).
- ❌ **Pas de webhooks** sortants. Le hub ne notifie rien à l'extérieur (sauf ntfy en cas d'erreur d'ingest).
- ❌ **Pas de rate limiting** côté API — protection assurée par Cloudflare en amont.

## Roadmap API

### Phase 3 (à venir)
- `POST/GET /v1/emails/threads` (Gmail)
- `POST/GET /v1/emails/messages`
- `POST/GET /v1/photos` (avec embedding CLIP en pgvector)

### Phase 5 (à venir)
- `GET /v1/calendar/events`
- `GET /v1/health/records` (Apple Health)
- `GET /v1/documents`

### Endpoints utilitaires (déférés)
- `GET /v1/insights` : anomalies, doublons d'abonnements, patterns
- `GET /v1/admin/active-version?app=trajets` : pour le routing apps versionnées
- `POST /v1/admin/replay?source=desjardins&from=2026-01-01` : déclenche un replay depuis raw_events
