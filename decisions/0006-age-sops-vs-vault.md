# ADR-0006 — age + sops pour les secrets (vs Bitwarden CLI / HashiCorp Vault)

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Le hub a des secrets : `POSTGRES_PASSWORD`, `SECRET_KEY`, `CLOUDFLARE_TUNNEL_TOKEN`, `DUCKDNS_TOKEN`, futurement OAuth tokens Google, clés API ntfy, etc.

Question : où les stocker, comment les chiffrer, comment les distribuer ?

Contraintes :
- Single-user (Marc), single-machine principale
- Reproductibilité sur nouveau PC en cas de catastrophe
- Pas de service tiers payant (règle "tout gratuit")
- Secrets ne doivent JAMAIS finir en clair sur GitHub

## Décision

**age + sops.**

- `age` (par Filippo Valsorda) : chiffrement asymétrique simple, 1 fichier `~/.age/hub.key`.
- `sops` (par Mozilla) : wrapper qui chiffre les valeurs YAML/JSON tout en gardant la structure éditable.

Workflow :
1. `age-keygen -o ~/.age/hub.key` → génère une paire (publique commitée dans `.sops.yaml`, privée jamais).
2. `secrets/postgres.yaml` (en clair, **gitignored**) → `secrets/postgres.enc.yaml` (chiffré, **commitable**).
3. `SOPS_AGE_KEY_FILE=~/.age/hub.key sops --decrypt secrets/postgres.enc.yaml` pour relire.
4. La clé privée `hub.key` est sauvegardée sur **2 clés USB physiques** (jamais cloud).

## Pourquoi

1. **Gratuit** (binaires open-source, pas de service à payer).
2. **Simple** : 2 binaires, 1 clé. Pas de daemon à faire tourner (vs Vault qui demande un serveur).
3. **GitOps-friendly** : les secrets chiffrés sont commités sur GitHub. Au setup nouveau PC, `git clone` puis `sops --decrypt` suffit (avec la clé `age` de la clé USB).
4. **Audit-friendly** : `git log secrets/` montre quand chaque secret a changé.
5. **Format YAML lisible** : sops chiffre les VALEURS mais pas les clés, donc on voit dans `secrets/postgres.enc.yaml` qu'il y a `password:`, juste pas la valeur.
6. **Recommandé par les pros** : utilisé par Mozilla, Spotify, Anthropic en prod.

## Trade-offs acceptés

- **Clé `age` à protéger physiquement.** Si perdue → backups inaccessibles, secrets impossibles à déchiffrer. Mitigation : 2 clés USB chez Marc + parents, test mensuel.
- **Pas d'accès web aux secrets** (vs Bitwarden qui a une UI mobile). Acceptable : Marc édite ses secrets en CLI sur le PC, c'est rare (< 1× / mois).
- **Pas de rotation automatique.** Mitigation : pour `POSTGRES_PASSWORD` et `SECRET_KEY`, on rotate manuellement tous les 6 mois (en théorie). En pratique c'est jamais fait par les solos, et c'est ok pour ce niveau de risque.

## Alternatives rejetées

### Bitwarden CLI
- ✅ Marc a déjà un compte Bitwarden (probablement)
- ❌ Nécessite `bw login` à chaque session shell
- ❌ Pas GitOps : les secrets ne sont pas versionnés
- ❌ Si Bitwarden cloud tombe, Marc peut perdre l'accès au hub jusqu'à ce que ça remarche
- ❌ La sync automatique fait courir un risque (un device compromis = tous les secrets exposés)

### HashiCorp Vault
- ❌ Demande un serveur Vault à faire tourner et à backup
- ❌ Setup et maintenance lourds pour 1 utilisateur
- ❌ Overkill : conçu pour les boîtes avec 100+ devs et de la rotation automatique

### Variables d'env shell + `.env` non chiffré + .gitignore
- ❌ Fragile : un `git add -A` mal placé peut commit le `.env` par accident
- ❌ Pas de versioning des changements
- ❌ Si Marc clone sur un nouveau PC, il doit recréer le `.env` à la main → risque d'oubli

### Docker secrets / Kubernetes secrets
- ❌ Pas chiffré at-rest dans le cluster (sans extension type sealed-secrets)
- ❌ Marc n'utilise pas Kubernetes
- ❌ Docker secrets nécessite Swarm mode (pas standalone compose)

### 1Password / Doppler / Infisical
- ❌ Service tiers payant (en plus de la règle "tout gratuit", ça crée une dépendance)
- ❌ Free tiers existent mais limités à quelques secrets

## Conséquences

- ✅ `hub-deploy/secrets/.sops.yaml` à créer en Phase 0 fin avec la pubkey de Marc.
- ✅ `hub-deploy/secrets/README.md` documente le workflow age + sops.
- ✅ `.gitignore` blacklist `secrets/*.yaml` (sans `.enc.`).
- ✅ Pre-commit hook qui rejette `secrets/*.yaml` non chiffré (recommandé).
- ✅ Scripts `init_secrets.ps1` et `decrypt_env.ps1` à créer pour automatiser le bootstrap.
- ⚠️ Documentation explicite dans `07-runbook.md` sur "comment régénérer la clé age si perdue" (réponse : on perd les backups, dramatic; mitigation prevent par 2 USB).

## Pratiques recommandées

```powershell
# Chiffrer un nouveau secret
sops --encrypt --age $(cat ~/.age/hub.key.pub) secrets/postgres.yaml > secrets/postgres.enc.yaml

# Editer un secret chiffré (ouvre $EDITOR avec déchiffrement transparent)
sops secrets/postgres.enc.yaml

# Décrypter tous les secrets vers un .env temporaire
sops --decrypt secrets/postgres.enc.yaml > .env.local
docker compose up -d
rm .env.local  # ne JAMAIS laisser traîner
```

## Test de récupération

Tous les **6 mois** :
1. Récupérer la clé `age` depuis la clé USB chez les parents
2. Sur un PC frais (VM par exemple), `git clone` + `sops --decrypt` un secret
3. Vérifier que ça marche, que le secret est correct
4. Replace l'usage si la clé USB est corrompue
