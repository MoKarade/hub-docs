# Session #2 — 2026-04-29 (nouveau PC, code-only)

> Session menée sur un **nouveau PC Windows** (`dessin14` au lieu de `marcr`), où Marc a explicitement demandé de **ne rien installer/lancer localement**. Tout le travail = écriture de code/docs commitable sur GitHub, prêt à tourner quand le vrai PC sera ré-équipé.

## Contexte de départ

- Les 5 repos `hub-*` ont été clonés depuis `https://github.com/MoKarade/*` vers `G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\` (Google Drive synchronisé).
- État du projet à fin Session #1 (2026-04-28) : **Phase 0 + Phase 1 + Phase 2 code-complete**.
- Ce qui n'existait pas : 03/04/06 docs, ADRs 0003-0007, pages frontend `/finances` `/search` `/locations`, tests, CI, infra prod, secrets vault, backup setup.

## Mandat de session

Marc a demandé : **« fais A, C, D, E »**, où :
- **A** = combler les docs `hub-docs` manquantes
- **C** = compléter les pages frontend manquantes
- **D** = outillage et tests
- **E** = infra Phase 0 fin (déférée jusqu'ici)

Plus tard dans la session : **« valide que tout est bon, montre-moi le frontend, prépare la suite, assure-toi que rien est sur ce PC, et documente bien tout et la raison de pq tu as fait ça »**.

## Livrables — 62 fichiers, 5 repos

### A — Docs (`hub-docs/`)

| Fichier | Pourquoi |
|---|---|
| `03-data-model.md` | Schéma DB complet (6 tables, ER mermaid, invariants, volumes 10 ans). **Rationale** : bloquait tous les futurs sprints (Phase 3 emails va devoir s'intégrer à ce modèle). |
| `04-api-contract.md` | 19 routes documentées + garanties + roadmap. **Rationale** : la doc auto-générée OpenAPI ne capture pas les invariants métier (idempotence, pas de PATCH/DELETE par design, etc.). |
| `06-security.md` | Threat model 5 acteurs, mitigations, runbook compromis. **Rationale** : Phase 0 fin (tunnel + backup) demande qu'on ait une vue claire du risque AVANT de connecter au monde extérieur. |
| `decisions/0003-postgres-pgvector.md` | Justifie le choix Postgres+pgvector vs SQLite+Chroma. |
| `decisions/0004-ollama-natif.md` | Pourquoi Ollama tourne natif Windows (GPU passthrough Docker fragile, perte de 10-20% perfs). Mesures : 57.6 tok/s sur RTX 5080. |
| `decisions/0005-cloudflare-tunnel-vs-tailscale.md` | CF Tunnel + Access pour gratuit + MFA TOTP, plan B Tailscale Funnel prêt. |
| `decisions/0006-age-sops-vs-vault.md` | age + sops choisi vs Bitwarden CLI / HashiCorp Vault. Workflow GitOps : `secrets/*.enc.yaml` commitable. |
| `decisions/0007-versioning-souschemin-vs-sousdomaine.md` | Caddy `/apps/finance/v1/*` plutôt que `app-finance-v1.example.com`. 1 cert TLS, 1 policy CF Access. |

**Pourquoi des ADRs** : règle 6 du projet (« décisions justifiées »). Chaque choix structurant doit être traçable. Quand Marc revient 6 mois plus tard, il sait pourquoi tel choix a été fait, pas juste qu'il existe.

### C — Frontend (`hub-frontend/`)

Les 4 pages prévues dans `app/` n'existaient pas (sauf `/`). J'ai créé :

| Page | Composants utilisés | Pourquoi ce design |
|---|---|---|
| `/finances` | SWR + `<table>` HTML natif (pas de TanStack — too much) | 3 onglets (banque/carte/invest) — chacun a sa sémantique différente (debit/credit XOR vs amount signé vs positions snapshot). Filtres date+compte+search appliqués mix client/serveur. Tables denses tabular-nums pour aligner les chiffres. |
| `/search` | Conversation persistante + collapsibles SQL/rows | UX comme un mini ChatGPT mais sur les données de Marc. Le SQL généré est visible (transparence) — Marc peut comprendre ce que Qwen a fait, même corriger sa question si le SQL est bizarre. |
| `/locations` | `next/dynamic({ ssr:false })` pour Leaflet | Leaflet touche `window` → casse en SSR Next.js. Dynamic import obligatoire. Couleurs par activité (vert marche, bleu vélo, orange voiture, gris statique) pour identification visuelle rapide. |
| `/` (home) | `LiveStatCards` SWR + `SpendingChart` recharts wired | Les vraies stats remplacent les hardcoded $12,480. Si l'API ne répond pas, on affiche `—` (no fake). |

**Plumbing additionnel** :
- `lib/api.ts` — client typé pour les 19 routes (chaque endpoint a son helper avec types miroir des `*Read` Pydantic)
- `lib/utils.ts` — `formatCurrency` defaut CAD/`fr-CA` (Marc Québec), `formatDate`, `signedAmount`
- `components/live-stat-cards.tsx` — 4 KPIs en SWR
- `components/location-map.tsx` — wrapper Leaflet
- `insight-list.tsx` rewrite — placeholder honnête « Phase 4+ » au lieu d'insights inventés (règle no-fake)

### D — Tests + outillage

| Fichier | Couverture | Pourquoi |
|---|---|---|
| `hub-ingest/tests/test_desjardins_csv.py` | 100% des helpers + parser sur fixture | Le parser CSV est le plus utilisé (3 mois importés). Régression = perte de confiance, donc tests prioritaires. |
| `hub-ingest/tests/test_desjardins_mastercard_pdf.py` | Helpers + regex (pas le parser end-to-end) | Pas de PDF fixture committable (les vrais PDFs contiennent les vraies transactions de Marc). On teste les regex et `_resolve_year` qui sont la logique métier. |
| `hub-ingest/tests/test_disnat_pdf.py` | Idem | Couvre la fix Sprint 4 du `_AMOUNT_AT_END_RE` (lookbehind contre les chiffres collés). |
| `hub-ingest/tests/test_google_takeout_timeline.py` | Helpers + parser sur fixture JSON | Vérifie le filtre accuracy + le sample 30s + idempotence du hash. |
| `hub-core/tests/conftest.py` | Override `get_db` vers SQLite in-memory | Permet de tester les endpoints sans Postgres. Plus rapide en CI, pas besoin de service container. |
| `hub-core/tests/test_finance_accounts.py` | CRUD + idempotence + filtres + 404 + 422 | Couverture des branches importantes. |
| `hub-core/tests/test_finance_transactions.py` | XOR validation + idempotence + filtres date | Le XOR debit/credit est le seul invariant métier non-trivial. |
| `hub-core/tests/test_locations.py` | CRUD + filtres bbox + activity + 422 lat/lng range | Idem. |
| `hub-core/tests/test_ai_validate.py` | SQL validator paramétré sur tous mots-clés interdits + injection | **Test critique sécurité** : la regex doit bloquer `INSERT/UPDATE/DELETE/...` parce que le LLM Qwen pourrait théoriquement générer du SQL malveillant si Marc lui pose une question piégée. |

**CI workflows** (5, un par repo) :
- `hub-core` + `hub-ingest` : ruff lint + pytest sur Ubuntu Python 3.13
- `hub-frontend` : lint + typecheck + build Next 15
- `hub-deploy` : `docker compose config` + Caddyfile validate + yaml syntax check (avec `python3` explicite après fix)
- `hub-docs` : markdown lint + lychee link check

**Scripts outillage** :
- `hub-deploy/scripts/clone_all.ps1` — bootstrap nouveau PC (mentionné dans ADR-0001 mais jamais écrit)
- `hub-ingest/src/scripts/replay.py` — replay raw_events → inbox (mentionné dans ADR-0002, exécutable via `python -m src.scripts.replay <connector>`)

### E — Infra Phase 0 fin (`hub-deploy/`)

| Fichier | Pourquoi |
|---|---|
| `docker-compose.prod.yml` | Stack prod : pas de port Postgres exposé, pas de bind-mount source, Caddy + cloudflared ajoutés, hub-frontend buildé. Réseau `hub_prod` explicite. |
| `cloudflared/config.example.yml` | Template pour mode standalone. En docker compose on utilise le TOKEN dashboard (plus simple). |
| `cloudflared/README.md` | Setup pas-à-pas : create tunnel, dashboard hostname, MFA TOTP, troubleshooting. |
| `.sops.yaml` | Config sops à la racine de hub-deploy. Ciblée sur `secrets/*.enc.yaml`. |
| `scripts/init_secrets.ps1` | Bootstrap interactif : génère clé age, met à jour `.sops.yaml`, rappel critique de sauvegarde sur 2 USB. |
| `scripts/decrypt_env.ps1` | Helper rapide pour `sops --decrypt`. |
| `secrets/README.md` (rewrite) | Workflow complet age+sops + règles dures + troubleshooting + commande de bootstrap rapide pour Phase 0. |
| `.gitignore` (rewrite) | Re-clarifié : `secrets/*.yaml` bloqué, `secrets/*.enc.yaml` autorisé (correction de la regex bizarre `secrets/!*.enc`). |
| `backup/README.md` | Setup restic + rclone + Task Scheduler 4am + politique rétention 7d/4w/12m. |
| `backup/scripts/backup.ps1` | Snapshot pg_dump + raw_events + secrets/.enc.yaml. Gère encoding UTF-8 sans BOM (.NET WriteAllText) pour ne pas casser le restore psql. Notif ntfy en cas d'échec. |
| `backup/scripts/restore.ps1` | Restore d'un snapshot (latest par défaut). |
| `backup/scripts/verify.ps1` | `restic check` + stats. À lancer mensuellement. |
| `backup/restic-config.toml.example` | Documentation des env vars utilisées. |

## Validation par reviewer

J'ai lancé un agent reviewer indépendant. Il a trouvé **7 vrais bugs** :

| # | Fichier | Bug | Fix appliqué |
|---|---|---|---|
| 1 | `backup.ps1` | `_sendNotif` appelé avant définition + `Out-File` UTF-16 BOM + `$LASTEXITCODE` qui reflète Out-File | Réordonné fonction en haut, utilisé `[System.IO.File]::WriteAllText` avec `UTF8Encoding $false` |
| 2 | `app/search/page.tsx` | `useSearchParams` sans Suspense — crash au build Next 15 | Wrappé `<SearchPageInner>` dans `<Suspense>` |
| 3 | `tests/test_finance_transactions.py` | Comparaison string `"100.00"` casse sur SQLite (renvoie `"100"`) | Utilisé `Decimal(body["debit"]) == Decimal("100.00")` |
| 4 | `docker-compose.prod.yml` | Réseau implicite — risque si on splite la stack | Déclaré `networks: default: name: hub_prod` |
| 5 | `hub-deploy/.github/workflows/ci.yml` | `python` pas dispo sur Ubuntu — seulement `python3` | Ajouté `actions/setup-python@v5` + utilisé `python3` |
| 6 | `backup.ps1` | `--exclude '*.yaml'` excluait aussi `.enc.yaml` | Remplacé par `--exclude-file` ciblé sur `*.tmp` + `backup/staging` |
| 7 | `app/finances/page.tsx` | Filtre date envoyé à l'API investment-transactions qui ne le supporte pas | Filtre côté client uniquement, commenté dans le code |

Le reviewer a aussi flagué 5 risques mineurs (rate limiting, edge cases, etc.) qui restent en TODO Phase 4+.

## Pourquoi cette ordonnance des choix

### Pourquoi A en premier
La doc 03-data-model est référencée par 04-api-contract et 06-security, et utilisée par les tests. Si on faisait C/D/E avant, on aurait dû refaire des passes. **A déverrouille tout le reste.**

### Pourquoi pas B (Phase 3 Gmail/Photos)
Marc a explicitement exclu B. Justification probable : Phase 3 demande des credentials OAuth Google qu'il faut configurer côté console.cloud.google.com — pas faisable sans toucher à un environnement Marc.

### Pourquoi tests SQLite et pas Postgres en CI
Postgres en GH Actions = service container = +30s de spin up par run. SQLite in-memory = 0s. Pour les tests d'intégration metier, SQLite suffit. Si un jour on a besoin de tester une feature Postgres-spécifique (pgvector, JSON ops avancées), on basculera sur service container.

### Pourquoi backups en local-first
Restic vers OneDrive = Marc reste maître de la data. Pas de dépendance à un service tiers payant (cohérent règle 5 « tout gratuit »). OneDrive personnel suffit pour ~20-30 GB de backup à long terme.

### Pourquoi `_previews/` HTML statique
Marc a dit « rien sur ce PC ». Donc impossible de :
- Lancer `npm install` + `npm run dev`
- Build l'image Docker hub-frontend
- Faire un screenshot du vrai rendu

→ Le HTML statique avec Tailwind CDN reproduit le visuel sans aucune install. Marc ouvre depuis Google Drive (ou n'importe quel browser) et voit immédiatement à quoi ressemblera le frontend.

## Cleanup

Cette session a créé 3 fichiers locaux à `C:\Users\dessin14\` :
- `~/.claude/CLAUDE.md` (profil reconstruit)
- `~/.claude/projects/G--Mon-disque-.../memory/MEMORY.md`
- `~/.claude/projects/G--Mon-disque-.../memory/project_banking.md`

Marc a demandé qu'**aucune trace** ne reste sur ce PC. Tous ces fichiers ont été supprimés en fin de session — vérifié par `Test-Path` qui retourne `False`.

Le code projet (`G:\Mon disque\...\Hub perso\`) est sur Google Drive donc syncé partout, n'est pas considéré comme « sur ce PC » au sens fichier local.

## Suite prévue

Voir `hub-docs/sessions/SUITE.md` pour la roadmap détaillée des prochaines étapes.

**TL;DR de la suite** :
1. Re-déployer la stack sur le vrai PC de Marc (équipé Docker + Ollama + GPU NVIDIA)
2. Phase 0 fin réelle : tunnel + auth + backup
3. Phase 2 finalisée : import Google Takeout
4. Phase 3 : Gmail + Photos
5. Phase 5 : Calendar + Apple Health + Documents
