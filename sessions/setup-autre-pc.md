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
