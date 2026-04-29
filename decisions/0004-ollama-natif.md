# ADR-0004 — Ollama en natif sur Windows (vs Docker)

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Le hub utilise Qwen 2.5 14B Instruct (~9 GB en VRAM) via Ollama pour l'endpoint `/v1/ai/ask`. Marc a une RTX 5080 avec 16 GB VRAM. Question : Ollama en container Docker (cohérent avec le reste de la stack) ou en service natif Windows ?

## Décision

**Ollama tourne en NATIF sur le host Windows.** Pas en Docker.

Les containers (hub-core, etc.) appellent Ollama via `http://host.docker.internal:11434`.

## Pourquoi

1. **GPU passthrough complexe sous Docker Desktop Windows.** Pour utiliser la RTX 5080 depuis un container, il faut WSL2 + nvidia-container-toolkit + drivers compatibles. Ça peut casser à chaque update Windows / WSL / Docker / drivers NVIDIA. En natif : NVIDIA gère tout via les drivers standard.
2. **Performance.** Mesuré le 2026-04-28 : Qwen 14B en natif sur RTX 5080 = **57.6 tokens/sec** après warm-up. En Docker GPU passthrough on perd typiquement 10-20 % à cause du context switch CUDA. Pour un usage interactif (IA Q&A), 57 vs ~46 tok/s est notable.
3. **Updates simples.** `winget upgrade Ollama.Ollama` + `ollama pull qwen2.5:14b-instruct`. En Docker il faut rebuild une image, gérer la version pinning, monter `/root/.ollama`, etc.
4. **Démarrage plus rapide.** Ollama natif est un service Windows qui démarre au boot. En Docker il faut attendre Docker Desktop, puis le container. Différence : ~30 sec.
5. **Cohérent avec le master plan.** Marc a explicitement budgété "Ollama natif" en discovery du 2026-04-28.

## Trade-offs acceptés

- **Pas tout en Docker.** Asymétrie : tout est containerisé sauf Ollama. Mitigation : c'est documenté partout (`02-architecture.md`, `hub-deploy/CLAUDE.md`).
- **Backup d'Ollama hors restic.** Les modèles sont dans `~/.ollama/models` côté host. Pas dans le compose volumes. Mitigation : on peut juste re-pull un modèle (ce sont des fichiers publics), donc pas de backup nécessaire.
- **Setup nouveau PC** demande un step en plus (`ollama pull qwen2.5:14b-instruct` + `nomic-embed-text`). Mitigation : automatisé dans `hub-deploy/scripts/setup_windows.ps1`.

## Alternatives rejetées

### Ollama dans un container avec GPU passthrough
- ❌ Setup fragile (dépend de WSL2 + nvidia-container-toolkit + drivers à jour)
- ❌ -10-20 % de perfs
- ❌ Risque de break à chaque MAJ Docker Desktop

### LLM via Llama.cpp Python (binding direct dans hub-core)
- ❌ Couplage fort entre hub-core et le runtime LLM. Si on swap le modèle, il faut redéployer.
- ❌ Moins flexible : Ollama gère l'unloading auto après inactivité, le multi-modèle (Qwen + nomic), le serving HTTP.

### LLM cloud (OpenAI / Anthropic / Mistral API)
- ❌ Viole la règle "local d'abord" — toutes les questions de Marc seraient envoyées à un fournisseur tiers.
- ❌ Viole la règle "tout gratuit" — usage interactif = 5-50$/mois facile.
- ❌ Latence réseau ajoutée vs local (gain de qualité existe mais pas indispensable).

## Conséquences

- ✅ `docker-compose.dev.yml` mappe `host.docker.internal:host-gateway` pour que les containers atteignent le host.
- ✅ `hub-core/src/core/config.py` a `OLLAMA_BASE_URL=http://host.docker.internal:11434` par défaut.
- ✅ `hub-deploy/scripts/setup_windows.ps1` lance `winget install Ollama.Ollama` + `ollama pull` les modèles.
- ⚠️ Pour un déploiement Linux server pur (pas Windows), il faudra revisiter — Linux peut faire du GPU Docker propre via `--gpus=all`. Pas urgent : Marc reste sur Windows pour l'instant.
- ⚠️ Si Marc change de GPU, il doit relancer `ollama pull` (Ollama recompile parfois pour optimiser le nouveau hardware).

## Métriques de référence

- **Cold start Qwen 14B** : ~29 sec (chargement modèle 9 GB en VRAM)
- **Throughput chaud** : 57.6 tokens/sec (testé 2026-04-28 sur RTX 5080)
- **Latence /v1/ai/ask** : 5-10 sec (2 appels LLM × ~3 sec chacun + exec SQL)
- **Decharge auto** : 5 min d'inactivité par défaut → modèle dégagé de la VRAM. Configurable via `OLLAMA_KEEP_ALIVE`.
