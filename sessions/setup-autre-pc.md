# Reprendre le projet depuis un autre PC

Ce guide explique comment reproduire l'environnement de Marc sur un autre PC
(que ce soit un nouveau PC Windows, ou un Linux/Mac si Marc change de
plateforme un jour).

---

## Pré-requis sur le nouveau PC

```
- Git
- Docker Desktop (avec WSL2 backend sur Windows)
- Node.js (pour Claude Code)
- Ollama
- gh CLI (GitHub CLI)
- age + sops (pour les secrets quand on les ajoutera)
```

Sur Windows :
```powershell
winget install Git.Git
winget install Docker.DockerDesktop
winget install OpenJS.NodeJS.LTS
winget install Ollama.Ollama
winget install GitHub.cli
winget install FiloSottile.age
winget install Mozilla.sops
```

Reboot après Docker Desktop (WSL2).

---

## 1. Authentification GitHub

```powershell
gh auth login
# choisis github.com -> HTTPS -> oui (creds via gh) -> ouvre browser pour s'auth
```

---

## 2. Cloner les repos

```powershell
mkdir C:\hub
cd C:\hub
git clone https://github.com/MoKarade/hub-core.git
git clone https://github.com/MoKarade/hub-deploy.git
git clone https://github.com/MoKarade/hub-frontend.git
git clone https://github.com/MoKarade/hub-ingest.git
git clone https://github.com/MoKarade/hub-docs.git
```

---

## 3. Restaurer le `CLAUDE.md` global de Marc

Le fichier `~/.claude/CLAUDE.md` contient les règles de collaboration de Marc
(langue française, "Claude fait le max", no fake data, tout gratuit, etc.).

**Source de vérité** : OneDrive de Marc, dans
`C:\Users\marcr\OneDrive\Documents\Claude\Projects\Hub perso\handoff_user_claude_md.md`.

Sur le nouveau PC :
```powershell
mkdir $env:USERPROFILE\.claude -Force
# Si OneDrive est synchronise sur ce PC :
copy "C:\Users\marcr\OneDrive\Documents\Claude\Projects\Hub perso\handoff_user_claude_md.md" `
     "$env:USERPROFILE\.claude\CLAUDE.md"
# Sinon : copier manuellement le fichier (clé USB, transfert reseau, etc.)
```

---

## 4. Restaurer le contexte projet `C:\hub\CLAUDE.md`

Idem : copie depuis OneDrive ou depuis ce repo `hub-docs`.

```powershell
copy "C:\hub\hub-docs\sessions\handoff_project_claude_md.md" "C:\hub\CLAUDE.md"
# (pas encore fait : on n'a pas pousse cette copie sur hub-docs aujourd'hui)
```

> **TODO** : pousser une copie du `C:\hub\CLAUDE.md` global dans `hub-docs/sessions/` à la prochaine session, pour que ce repo seul suffise à recreer le contexte.

---

## 5. Restaurer la mémoire projet (optionnel mais recommandé)

```powershell
mkdir $env:USERPROFILE\.claude\projects\C--hub\memory -Force
copy C:\hub\hub-docs\sessions\memory\*.md `
     $env:USERPROFILE\.claude\projects\C--hub\memory\
```

Cette mémoire contient les faits projet stables (banque = Desjardins,
5 comptes spécifiques, formats CSV/PDF, etc.) que Claude utilisera comme
contexte permanent.

---

## 6. Lancer la stack

```powershell
cd C:\hub\hub-deploy

# Premier lancement : copier .env.example en .env et generer SECRET_KEY/POSTGRES_PASSWORD
copy .env.example .env
# Editer .env, regenerer les secrets via :
#   openssl rand -base64 32   # pour SECRET_KEY
#   openssl rand -hex 24      # pour POSTGRES_PASSWORD

# Pull modeles Ollama (~9 GB total)
ollama pull qwen2.5:14b-instruct
ollama pull nomic-embed-text

# Lancer
docker compose -f docker-compose.dev.yml up -d --build
```

Vérifier avec :
- http://localhost:8000/v1/health
- http://localhost:8000/v1/ready
- http://localhost:8000/docs

---

## 7. Récupérer les VRAIES données de Marc

Les données de Marc sont **dans la DB Postgres locale** sur le PC d'origine. Elles ne sont PAS dans GitHub (et c'est voulu — règle privacy).

Pour les avoir sur le nouveau PC, deux options :

**Option A (Recommandé) — Re-importer les fichiers source**

Marc re-télécharge ses CSV/PDF depuis AccèsD/Disnat et les met dans
`C:\hub\inbox\` :
- `inbox\desjardins\*.csv` (compte courant + épargne)
- `inbox\desjardins-cc\*.pdf` (carte de crédit)
- `inbox\disnat\*.pdf` (investissements)
- `inbox\google-timeline\*.json` (Google Takeout)

Puis :
```powershell
docker compose --profile ingest run --rm hub-ingest
```

Le pipeline va re-importer toutes les données.

**Option B — Backup/restore Postgres depuis le PC d'origine**

Sur le PC d'origine :
```powershell
docker exec hub-dev-postgres-1 pg_dump -U hub hubdb > C:\hub\backup-2026-04-28.sql
```

Transférer le `.sql` (clé USB, etc.) vers le nouveau PC, puis :
```powershell
docker exec -i hub-dev-postgres-1 psql -U hub hubdb < C:\hub\backup-2026-04-28.sql
```

Plus rapide qu'A mais nécessite l'accès physique aux 2 PC.

---

## 8. Reprendre Claude Code

```powershell
cd C:\hub
claude
```

Lance Claude Code. Il va automatiquement charger :
- `C:\Users\marcr\.claude\CLAUDE.md` (global)
- `C:\hub\CLAUDE.md` (projet)
- La mémoire `~/.claude/projects/C--hub/memory/`

Demande à Claude de **lire `JOURNAL.md`** au début de la nouvelle session
pour qu'il sache où on en est.

---

## Annexe — fichiers clés à connaître

| Fichier | Rôle |
|---|---|
| `~/.claude/CLAUDE.md` | Règles globales Marc (ton, langue, "Claude fait le max", no fake) |
| `C:\hub\CLAUDE.md` | Contexte projet (vision, stack, sources, principes) |
| `C:\hub\JOURNAL.md` | Journal de session (plan en cours + historique des décisions/bugs) |
| `C:\hub\01-04_*.md` | Discovery initiale (audit risques, phasing, structure repos, master plan) |
| `~/.claude/projects/C--hub/memory/` | Mémoire structurée Claude (faits projet stables) |
| `C:\hub\hub-docs\sessions\` | Transcripts de sessions passées (lecture seule, archive) |

---

## Annexe 2 — Workflow Drive + clone local (post-session #16)

À partir de la session #16, on a découvert que `node_modules` dans Google Drive en
mode streaming (Files On Demand) bloque `npm install` (>50 min sans aboutir).
Solution : **garder le code source dans Drive, mais lancer le dev frontend depuis
un clone local sur SSD**.

### Setup initial (premier démarrage sur le nouveau PC)

```powershell
# 1. Clone le hub depuis Google Drive (path local rapide)
git clone "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-frontend" C:\hub\hub-frontend

# 2. Copie le .env.local
copy "G:\Mon disque\...\hub-frontend\.env.local" "C:\hub\hub-frontend\.env.local"

# 3. Install + dev
cd C:\hub\hub-frontend
npm install
npm run dev
```

### Démarrer hub-core (backend)

Le backend tourne directement depuis Drive (Python `.venv` n'a pas le même problème
que `node_modules`).

```powershell
# Lance via le helper créé en session #16 (PID détaché, survit à la fermeture du shell)
cmd /c C:\hub\start-uvicorn.bat
```

Si `.venv` est manquant (premier démarrage) :

```powershell
cd "G:\Mon disque\...\hub-core"
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
.venv\Scripts\python.exe -m alembic upgrade head  # applique toutes les migrations
```

### Synchroniser le frontend (Drive ↔ local)

Quand tu modifies du code à un endroit, il faut sync l'autre :

```powershell
# Drive → local (récupère les commits faits dans Drive)
cd C:\hub\hub-frontend
git pull

# Local → Drive (push les commits faits en local)
cd "G:\Mon disque\...\hub-frontend"
git pull
```

(Ou utilise GitHub comme bridge : `git push` d'un côté, `git pull` de l'autre.)

### Pourquoi pas une junction symlink

Tentative en session #16 : `mklink /J node_modules C:\hub-cache\node_modules` →
échoue avec **"Des volumes NTFS locaux sont requis"** car Google Drive n'est
pas du NTFS.

---

## Annexe 3 — État Phase 2 Localisation (post-sessions #15, #17)

À jour 2026-05-04. **Phase 2 100% livrée** + features explorer XXL.

### Données ingérées

13 ans d'historique (2013 → 2026) depuis `Timeline.json` 75MB :
- 13 646 visites
- 155 495 points GPS
- 12 333 activités

### 6 onglets sur `/locations`

1. **Carte GPS** : 4 modes (visites/points/trajets/heatmap) × 4 styles tuiles (dark/voyager/satellite/topo) + clustering + click-popup avec stats lieu + plein écran + split 2 dates + heatmap year slider
2. **Journée** : date picker + timeline chronologique + Calendar events (modal détail) + Photos lightbox
3. **Visites** : liste paginée + filter sémantique + retag inline
4. **Voyages** : auto-détection avec `home_recency_months=12` + recherche texte + filter année + cards avec mini-map + auto-naming ("Lille, France · déc 2024") + photos + notes
5. **Stats** : Vue globale + Top 10 lieux + Records (streaks) + Pays/Villes drill-down + comparaison année/année (complètes uniquement)
6. **Mes Lieux** : CRUD lieux nommés + Batch geocoding live (Nominatim 1/sec) + visites HOME/WORK avec adresses

### Workers en background

- **Geocode worker** : `POST /v1/locations/geocode-batch`. Priorise les visites RÉCENTES (Quebec). ETA ~50 min pour ~3000 cellules. Cache permanent dans `location_addresses` (rejouable si DB perdue).
- **Auto-migrate** : Alembic au startup hub-core.

### Tables Phase 2

| Table | Description | Rows |
|---|---|---|
| `location_visits` | Visites sémantiques | 13 646 |
| `location_activities` | Trajets/activités | 12 333 |
| `location_points` | Points GPS bruts (path) | 155 495 |
| `location_addresses` | Cache géocodage par cellule (lat_e4, lng_e4) | ~3000 (à terme) |
| `named_places` | Lieux nommés Marc | 0+ |
| `trip_notes` | Notes voyages ancrées start_date | 0+ |

### IA conversationnelle

`/v1/ai/ask` accepte `history: [{role, content}]`. Test live :

```
Q1: Combien de fois en France en 2024 ?  →  245 jours
Q2: Et en 2023 ? (history=[Q1])           →  103 jours
```

Le LLM résout "et avant ?" automatiquement.

### Reste pour les sessions futures

- **#16 AI alertes** : cron + ntfy push (anniversaire voyage, absence prolongée HOME)
- **Polish** : skeleton states pendant le geocoder
- **Photos lightbox** : raccourcis clavier (← →) + zoom

---

## Annexe 4 — Commandes utiles

```powershell
# Démarrer la stack complète
cmd /c C:\hub\start-uvicorn.bat       # backend :8000
cd C:\hub\hub-frontend; npm run dev   # frontend :3000

# Vérifier la stack
curl http://localhost:8000/v1/health
curl http://localhost:3000/locations

# Lancer un batch de géocodage (~50min ETA)
curl -X POST http://localhost:8000/v1/locations/geocode-batch `
  -H "Content-Type: application/json" `
  --data '{"only_unknown":false,"max_cells":10000}'

# Suivre la progress
curl http://localhost:8000/v1/locations/geocode-progress

# Tuer les processes (si bloqués)
Get-Process python | Stop-Process -Force
Get-Process node | Stop-Process -Force

# Sync frontend Drive ↔ local (via GitHub)
cd C:\hub\hub-frontend; git push
cd "G:\Mon disque\...\hub-frontend"; git pull
```
