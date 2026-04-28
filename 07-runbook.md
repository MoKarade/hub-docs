# 07 — Runbook (que faire si X tombe en panne)

## Quick reference — diagnostic en 30 secondes

```powershell
# 1. Le hub répond-il ?
curl http://localhost:8000/v1/health

# 2. Toutes les deps OK ?
curl http://localhost:8000/v1/ready

# 3. Containers up ?
docker compose -f docker-compose.dev.yml ps

# 4. Logs récents
docker compose logs --tail=50 hub-core
```

## Symptôme → action

### "Le hub ne répond plus du tout"

1. `docker ps` → containers up ?
   - Sinon : `docker compose up -d`
2. Si toujours rien : `docker compose logs hub-core` → cherche l'erreur
3. Si DB ne démarre pas : volumes peut-être corrompus ; `docker compose down -v` (DANGER : supprime data) puis `up -d`. Heureusement le backup restic existe.

### "Le hub répond mais /v1/ready dit DB error"

1. `docker compose logs postgres` → cherche message d'erreur
2. Vérifier qu'on n'a pas viré le volume `postgres_data`
3. Si DB pas joignable depuis hub-core : check le `DATABASE_URL` dans .env

### "Le hub répond mais /v1/ready dit Ollama error"

1. Sur le host Windows : `ollama list` → modèles présents ?
2. Sinon : `ollama pull qwen2.5:14b-instruct`
3. Vérifier que Ollama écoute : `curl http://localhost:11434/api/tags`
4. Vérifier que le conteneur peut joindre `host.docker.internal` (Docker Desktop sur Windows)

### "Tout marche en local mais pas via le tunnel"

1. `cloudflared tunnel info marc-hub` → tunnel actif ?
2. Logs cloudflared : `cloudflared tunnel run marc-hub` en foreground pour voir
3. Vérifier que le DNS résout : `nslookup marc-hub.duckdns.org`
4. Vérifier Cloudflare Access n'est pas en train de me bloquer (re-auth)

### "Une ingestion a échoué (notif ntfy)"

1. Logs ingest : `docker compose logs hub-ingest`
2. Identifier le connecteur fautif
3. Replay manuellement : `docker exec -it hub-ingest python -m src.replay <connector> <date>`

### "Backup restic a échoué"

1. `restic check --repo rclone:onedrive:backups/hub`
2. Si OneDrive saturé → libérer de l'espace OU passer en B2
3. Si erreur "key not found" → ta clé age est introuvable, gros problème, check les backups de la clé

### "Mon PC a redémarré (Windows update)"

Tout devrait redémarrer auto si :
- Docker Desktop start au boot (config par défaut)
- Ollama service start au boot
- cloudflared est setup en service Windows

Pour vérifier, lance simplement `.\scripts\start_hub.ps1` qui idempotently démarre tout.

### "J'ai changé mon mot de passe Google"

Les tokens OAuth restent valides (ils sont indépendants du password). Pas d'action nécessaire SAUF si tu as activé "rotation OAuth on password change" dans Google.

### "Mon RTX 5080 fait du bruit / chauffe quand le hub tourne"

Ollama décharge le modèle après inactivité par défaut (5 min). Si tu veux qu'il décharge plus vite : `OLLAMA_KEEP_ALIVE=1m` dans son env.

### "J'ai oublié mon password Cloudflare Access / je me suis verrouillé dehors"

1. Sur le PC à la maison (en local) : tu as toujours accès via http://localhost:8000
2. Reset Cloudflare Access depuis le dashboard Zero Trust : changer la policy temporairement
3. Re-auth Google

## Procédure de restoration complète depuis backup

Si le PC a brûlé :

```powershell
# 1. Nouveau PC, install des outils prérequis (winget commands)

# 2. Récupère ta clé age depuis backup OneDrive ou clé USB
copy "F:\backup-key\hub.key" "$env:USERPROFILE\.age\hub.key"

# 3. Clone les repos
mkdir C:\hub
cd C:\hub
git clone https://github.com/MoKarade/hub-core.git
git clone https://github.com/MoKarade/hub-deploy.git
git clone https://github.com/MoKarade/hub-ingest.git
git clone https://github.com/MoKarade/hub-frontend.git

# 4. Restore depuis restic
cd hub-deploy
restic restore latest --repo rclone:onedrive:backups/hub --target C:\hub-restored

# 5. Configure .env avec les valeurs restorées
copy C:\hub-restored\.env C:\hub\hub-deploy\.env

# 6. Lance
.\scripts\start_hub.ps1

# 7. Vérifie
.\scripts\healthcheck.ps1
```

RTO target : 4-8 heures (incluant install Windows + outils).
