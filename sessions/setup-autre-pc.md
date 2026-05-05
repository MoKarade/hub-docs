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

## Annexe 4bis — 🔐 Transférer les secrets sur l'autre PC

**Inventaire des secrets** (post-session #18, 2026-05-05) :

| Fichier | Taille | Contenu | Critique ? |
|---|---|---|---|
| `hub-core/.env` | 1 KB | SECRET_KEY, OAuth client, Ollama URL, DATABASE_URL | OUI |
| `hub-deploy/.env` | 1 KB | Postgres, Cloudflare tunnel, DuckDNS, OAuth | OUI |
| `hub-frontend/.env.local` | 0.3 KB | Maps API, NEXT_PUBLIC_HUB_API_URL | OUI |
| `hub-core/hub.db` | 101 MB | OAuth tokens chiffrés, 13k visites, 470 transactions, emails sync | TRÈS oui (perte = re-sync de tout) |
| `age-key-BACKUP.enc` | 208 B | Backup chiffré clé age (master password requis) | OUI |
| `~/.claude/.../memory/secrets_api_keys.md` | 4 KB | Index des credentials (Claude memory) | NON (peut être recréé manuellement) |

### 🚚 Workflow ultra-simple : bundle chiffré + Drive sync

Marc a Google Drive sur ses 2 PCs → on s'en sert comme bridge naturel.
**Pas besoin de Gmail / USB / WeTransfer.**

#### Sur le PC source

```powershell
# 1. Install 7-Zip si jamais fait (une seule fois)
winget install 7zip.7zip

# 2. Lance le bundle (te demande le master password)
cd "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-deploy\scripts"
.\bundle-secrets.ps1
```

**Output** : `G:\Mon disque\...\_transfer\hub-secrets-bundle.7z`
- Format `.7z` AES-256 + `-mhe=on` (header chiffré : sans password, on ne voit pas les noms)
- Avec `hub.db` : ~30 MB. Sans : <5 KB.
- Drive sync automatiquement vers l'autre PC en 5-30s.
- Le dossier `_transfer/` est gitignored (pas de risque de push GitHub).

#### Sur le PC cible

**1. Cloner tous les repos depuis GitHub** (5 repos):

```powershell
mkdir C:\hub
cd C:\hub
git clone https://github.com/MoKarade/hub-core
git clone https://github.com/MoKarade/hub-frontend
git clone https://github.com/MoKarade/hub-deploy
git clone https://github.com/MoKarade/hub-ingest
git clone https://github.com/MoKarade/hub-docs
```

**2. Setup backend Python** :

```powershell
cd C:\hub\hub-core
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
.venv\Scripts\python.exe -m alembic upgrade head
```

**3. Setup frontend Next.js** :

```powershell
cd C:\hub\hub-frontend
npm install      # ~2 min sur SSD local
# .env.local sera restauré à l'étape suivante
```

**4. Restaurer les secrets depuis Drive** :

```powershell
# Drive doit avoir synced le bundle avant cette étape
winget install 7zip.7zip   # si jamais fait

cd "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-deploy\scripts"

# IMPORTANT : Unblock-File car Drive marque les fichiers comme "downloaded"
Unblock-File .\restore-secrets.ps1
Unblock-File .\bundle-secrets.ps1

.\restore-secrets.ps1 -Bundle "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\_transfer\hub-secrets-bundle.7z"
```

→ Te demande ton master password → décrypte → restaure :
- `.env` → `C:\hub\hub-core\.env` + `C:\hub\hub-deploy\.env`
- `.env.local` → `C:\hub\hub-frontend\.env.local`
- `hub.db` → `C:\hub\hub-core\hub.db` (toutes tes données préservées)
- `age-key-BACKUP.enc` → racine

**5. Si tu transfères depuis Drive vers `C:\hub\` (split workflow recommandé)** :

Le hub-frontend doit aussi être copié dans `C:\hub\hub-frontend\` (pas seulement le clone GitHub) car les `node_modules` doivent être sur disque local (Drive streaming = npm install bloqué).

```powershell
cd C:\hub\hub-frontend
copy "G:\Mon disque\...\hub-frontend\.env.local" .env.local
npm run dev   # Ready in 3.4s
```

**6. Lancer le backend** :

```powershell
cmd /c C:\hub\start-uvicorn.bat
# Ou si C:\hub\start-uvicorn.bat n'existe pas (premier démarrage) :
cd C:\hub\hub-core
.venv\Scripts\python.exe -m uvicorn src.main:app --host 0.0.0.0 --port 8000
```

**7. Junction pour le pre-push hook** (validation CI locale avant git push) :

```powershell
cmd /c "mklink /J C:\HubFrontend C:\hub\hub-frontend"
```

### Si le script échoue

| Erreur | Solution |
|---|---|
| `Le fichier ... n'est pas signé numériquement` | `Unblock-File .\bundle-secrets.ps1` puis relance |
| `7zip non trouve` | `winget install 7zip.7zip` |
| `Duplicate filename on disk: .env` | (déjà fixé en #18) → `git pull` dans hub-deploy |
| `Param incorrect` (7z) | (déjà fixé) → `git pull` dans hub-deploy |
| Bundle Drive pas encore synced | Attend 30s, vérifie l'icône Drive (vert = synced) |

### Master password

Documenté dans `~/.claude/.../memory/secrets_api_keys.md` côté Claude.
Ne JAMAIS l'écrire dans le chat ou commiter en clair.

### Ce que tu PERDS sans `hub.db`

Si tu transfères uniquement les `.env` (pas la DB) :
- ❌ 13 646 visites + 12 333 activités + 155 495 points GPS Phase 2
- ❌ Cache géocodage (`location_addresses`, ~3000 cellules)
- ❌ OAuth tokens (re-OAuth Gmail/Photos/Calendar/Drive/Fit/People/Tasks/YouTube)
- ❌ Tokens Garmin chiffrés (re-connecter)
- ❌ Emails Gmail synchronisés
- ❌ 470 transactions Desjardins
- ❌ Lieux nommés + notes voyage
- ❌ Insights / streaks / records calculés

Tout est **rejouable** depuis `raw_events/` + `inbox/`, mais ça prendra des heures de re-sync. **Le plus simple : transférer `hub.db` aussi** (le script te le demande).

### Workflow alternative : copy direct (sans 7-Zip)

Si tu n'as pas 7-Zip et pas envie d'installer :

```powershell
# Copie manuelle dans Drive
mkdir "G:\Mon disque\...\_transfer\plain"
copy hub-core\.env       "G:\Mon disque\...\_transfer\plain\hub-core.env"
copy hub-deploy\.env     "G:\Mon disque\...\_transfer\plain\hub-deploy.env"
copy hub-frontend\.env.local "G:\Mon disque\...\_transfer\plain\hub-frontend.env.local"
copy hub-core\hub.db     "G:\Mon disque\...\_transfer\plain\hub.db"

# Sur l'autre PC, copie inverse
```

⚠️ **Moins safe** : les .env contiennent des credentials en clair. Drive est privé mais si quelqu'un accède à ton compte Google, il les voit. Avec le bundle .7z chiffré, même Drive admin ne peut pas les lire.

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
