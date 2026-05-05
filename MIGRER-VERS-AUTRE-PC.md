# 🚚 Migrer vers l'autre PC — TL;DR

**Date du guide** : 2026-05-05 · **Estimation totale** : ~30 min côté cible

---

## 1️⃣ Sur le PC source (5 min)

```powershell
winget install 7zip.7zip   # une seule fois
cd "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-deploy\scripts"
Unblock-File .\bundle-secrets.ps1
.\bundle-secrets.ps1
```

Réponses :
- `yes` (inclure hub.db ~101 MB)
- ton **master password** (saisie cachée)

→ Output : `G:\Mon disque\...\_transfer\hub-secrets-bundle.7z` synced auto par Drive.

---

## 2️⃣ Sur le PC cible (~30 min)

### A. Cloner les repos depuis GitHub

```powershell
mkdir C:\hub
cd C:\hub
git clone https://github.com/MoKarade/hub-core
git clone https://github.com/MoKarade/hub-frontend
git clone https://github.com/MoKarade/hub-deploy
git clone https://github.com/MoKarade/hub-ingest
git clone https://github.com/MoKarade/hub-docs
```

### B. Backend Python (hub-core)

```powershell
cd C:\hub\hub-core
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
deactivate
```

(Note : on appliquera `alembic upgrade head` à l'étape D si on n'a PAS transféré hub.db.)

### C. Frontend Next.js (hub-frontend)

```powershell
cd C:\hub\hub-frontend
npm install      # ~2 min
```

### D. Restaurer les secrets depuis Drive

Vérifie que Drive a synced le bundle :

```powershell
Test-Path "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\_transfer\hub-secrets-bundle.7z"
# Doit retourner True
```

Puis :

```powershell
winget install 7zip.7zip   # une seule fois

cd "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-deploy\scripts"
Unblock-File .\restore-secrets.ps1
.\restore-secrets.ps1 -Bundle "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\_transfer\hub-secrets-bundle.7z"
```

→ Te demande ton master password → restaure tous les `.env` + `hub.db`.

⚠️ **Si tu n'as pas transféré hub.db** :

```powershell
cd C:\hub\hub-core
.venv\Scripts\python.exe -m alembic upgrade head
```

(Sinon ne pas lancer alembic, la DB transférée est déjà à jour.)

### E. Junction pour pre-push hook

```powershell
cmd /c "mklink /J C:\HubFrontend C:\hub\hub-frontend"
```

---

## 3️⃣ Démarrer la stack

### Terminal 1 — Backend

```powershell
cd C:\hub\hub-core
.venv\Scripts\python.exe -m uvicorn src.main:app --host 0.0.0.0 --port 8000
```

### Terminal 2 — Frontend

```powershell
cd C:\hub\hub-frontend
npm run dev
```

### Vérifie

Ouvre http://localhost:3000/locations → tu vois tes 13 646 visites + tous les onglets (Carte, Journée, Visites, Voyages, Stats, Mes Lieux).

---

## 4️⃣ Ollama (optionnel, pour l'IA)

Si tu veux les insights AI proactive + le chat AI, install Ollama natif Windows :

```powershell
winget install Ollama.Ollama
ollama pull qwen2.5:14b
ollama pull nomic-embed-text
```

---

## 🆘 Troubleshooting

### Le script PowerShell ne se lance pas

```
.\xxx.ps1 : n'est pas signé numériquement
```

→ `Unblock-File .\xxx.ps1` puis retente.

### Drive a pas encore synced

L'icône Drive en bas-droite Windows doit être **verte avec checkmark** (synced). Si elle tourne (sync), attends 1 min.

### npm install bloqué dans Drive

C'est normal. Tu DOIS faire `npm install` dans `C:\hub\hub-frontend` (clone local), jamais dans `G:\...\hub-frontend` (Drive streaming).

### Ports déjà occupés

```powershell
Get-Process python,node | Stop-Process -Force
# Puis relance
```

### Le bundle se déchiffre pas

Vérifie que tu as **exactement** ton master password (case-sensitive).
Le password est documenté dans la mémoire Claude `~/.claude/projects/.../memory/secrets_api_keys.md`.

---

## 📝 Plus de détails

- Doc complète : `hub-docs/sessions/setup-autre-pc.md` (annexes 1-4)
- Workflow Drive + clone local : `setup-autre-pc.md` annexe 2
- État Phase 2 + endpoints : `setup-autre-pc.md` annexe 3
- Inventaire complet des secrets : `setup-autre-pc.md` annexe 4bis
