# ADR-0003 — PostgreSQL + pgvector (vs SQLite + Chroma)

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Le hub a besoin de :
- Stocker des données relationnelles (transactions, comptes, positions, points GPS)
- Stocker des vecteurs d'embeddings (pour le RAG sur emails et photos en Phase 3+)
- Faire des requêtes complexes (jointures, aggrégations sur 10 ans d'historique)
- Tenir une charge de ~2 GB de DB à terme + ~600 MB d'embeddings

Question : SQLite (simple, fichier unique) + Chroma/FAISS (vector store dédié), ou PostgreSQL + pgvector (un seul moteur) ?

## Décision

**PostgreSQL 16 + pgvector dans un seul container.**

Image Docker : `pgvector/pgvector:pg16` (officielle).

## Pourquoi

1. **Un seul moteur, un seul backup, un seul réseau.** Avec SQLite + Chroma séparé, il faut backup les 2, gérer la cohérence d'écriture cross-store (un email est insérée dans SQLite mais Chroma plante → état incohérent), gérer 2 connexions différentes côté code.
2. **Postgres tient la charge.** 2 GB sur Postgres c'est rien (le seuil "trop gros" commence vers le To). SQLite tient aussi mais devient pénible au-delà de quelques centaines de Mo (verrous, write contention).
3. **Postgres a l'écosystème.** Index full-text natif, extensions (pg_trgm pour le fuzzy search, pg_stat_statements pour le profiling), partitioning si jamais on en a besoin.
4. **pgvector est mature.** Versions 0.7+ (2024) ont du HNSW indexing, performances équivalentes à FAISS pour <10M vecteurs.
5. **SQLAlchemy 2 supporte les deux** mais Postgres async via `asyncpg` est plus solide que SQLite async (qui passe par un thread pool).
6. **Disponibilité scale** : si jamais Marc veut un 2ᵉ utilisateur (foyer), Postgres est trivial à étendre. SQLite serait à migrer.

## Trade-offs acceptés

- **Container Docker en plus** vs un fichier `.db` simple. OK : Docker tourne déjà pour hub-core, ajouter Postgres = 0 effort de plus.
- **Plus de RAM** que SQLite (~50 MB de baseline pour Postgres vs ~5 MB pour SQLite). OK : RTX 5080 = 32 GB de RAM, on s'en fout.
- **Connexions TCP** vs in-process. OK : la latence loopback est négligeable (<0.1 ms).
- **Setup initial** (créer DB, user, run migrations). OK : automatisé via docker-compose + Alembic.

## Alternatives rejetées

### SQLite + Chroma
- ❌ 2 stores à backup et synchroniser
- ❌ Pas de jointures cross-store (donc dupliquer la donnée des emails dans Chroma)
- ❌ Chroma a peu de track record en prod long-terme

### DuckDB + pgvector via DuckDB-extensions
- ❌ Pas de support pgvector officiel dans DuckDB (il y a un plugin tiers `duckdb-vss` mais immature en 2026)
- ❌ DuckDB est OLAP-first, mauvais sur les écritures one-row-at-a-time (notre cas avec hub-ingest)
- ❌ Marc préfère "stack ennuyeuse" : Postgres > DuckDB pour ce projet

### Postgres + Qdrant séparé
- ❌ 2 services à maintenir
- ❌ Pas de jointure SQL entre les emails (Postgres) et leurs vecteurs (Qdrant)
- ❌ Plus de RAM, plus de complexité

### Mongo
- ❌ Pas de support pgvector équivalent natif (MongoDB Atlas Vector Search nécessite Atlas cloud, payant)
- ❌ Marc veut SQL pour le `/v1/ai/ask` (le LLM génère du SQL, pas du Mongo aggregation)

## Conséquences

- ✅ `hub-deploy/docker-compose.dev.yml` utilise `pgvector/pgvector:pg16` (déjà fait).
- ✅ `hub-core/pyproject.toml` a `psycopg[binary]>=3.2` + `asyncpg>=0.30`.
- ✅ Connection string : `postgresql+psycopg://hub:hubpass@postgres:5432/hubdb`.
- ✅ Future extension `vector` à activer dans Alembic en Phase 3 : `CREATE EXTENSION IF NOT EXISTS vector;`.
- ⚠️ Backup à faire avec `pg_dump` (pas juste copier des fichiers SQLite).
