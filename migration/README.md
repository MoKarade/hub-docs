# Migration multi-repo → monorepo `MoKarade/hub`

Ce dossier contient le matériel pour exécuter la migration décidée dans
[`decisions/0009-monorepo.md`](../decisions/0009-monorepo.md).

> ⚠️ **Avant de commencer** : valide que tu veux vraiment basculer en monorepo
> (relire ADR-0009). Une fois que les repos sources sont archivés, le retour
> arrière coûte un peu (réactivation, mais pas de perte de données).

## Pré-requis

- [ ] Accès admin à l'org/user `MoKarade` sur GitHub (création de repo, archivage).
- [ ] Liste finalisée des repos à importer **avec leurs noms exacts** (vérifier sur https://github.com/MoKarade?tab=repositories).
- [ ] Décider du `prefix` (sous-dossier) pour chacun. Suggestion par défaut :
  | Repo source              | Prefix dans le monorepo |
  |--------------------------|-------------------------|
  | `MoKarade/hub-core`      | `hub-core/`             |
  | `MoKarade/hub-frontend`  | `hub-frontend/`         |
  | `MoKarade/hub-ingest`    | `hub-ingest/`           |
  | `MoKarade/hub-deploy`    | `hub-deploy/`           |
  | `MoKarade/hub-docs`      | `docs/`                 |
  | `MoKarade/app-trajets`   | `apps/trajets/`         |
  | `MoKarade/app-finance`   | `apps/finance/`         |

## Étapes

### 1. Créer le nouveau repo

Sur https://github.com/new :

- Owner : `MoKarade`
- Name : `hub`
- Visibility : **Private** (recommandé) ou Public selon ton choix
- ✅ **Initialize this repository with a README** (indispensable, sinon pas de branche `main` à checkout).
- ❌ Pas de `.gitignore` ni licence (on les ajoutera après import).

### 2. Vérifier que `main` n'est pas protégé

Settings → Branches → s'assurer qu'il n'y a aucune *branch protection rule* sur `main`.
(Le workflow doit pouvoir pousser librement sur `main`.)

### 3. Poser le workflow

Toujours via l'UI GitHub web :

1. Aller dans le nouveau repo `MoKarade/hub`.
2. Onglet "Add file" → "Create new file".
3. Nom : `.github/workflows/bootstrap-monorepo.yml`
4. Coller le contenu intégral de [`bootstrap-monorepo.yml`](./bootstrap-monorepo.yml).
5. Commit directement sur `main` ("Commit new file").

### 4. Déclencher l'import

1. Onglet **Actions** du repo `hub` → workflow "Bootstrap monorepo".
2. Cliquer "Run workflow".
3. Coller dans **`repos`** un JSON array correspondant à ta vraie liste, par exemple :
   ```json
   [
     {"repo":"MoKarade/hub-core",     "prefix":"hub-core",     "branch":"main"},
     {"repo":"MoKarade/hub-frontend", "prefix":"hub-frontend", "branch":"main"},
     {"repo":"MoKarade/hub-ingest",   "prefix":"hub-ingest",   "branch":"main"},
     {"repo":"MoKarade/hub-deploy",   "prefix":"hub-deploy",   "branch":"main"},
     {"repo":"MoKarade/hub-docs",     "prefix":"docs",         "branch":"main"},
     {"repo":"MoKarade/app-trajets",  "prefix":"apps/trajets", "branch":"main"},
     {"repo":"MoKarade/app-finance",  "prefix":"apps/finance", "branch":"main"}
   ]
   ```
   ⚠️ **Vérifier** :
   - Les noms de repos existent vraiment (copier-coller depuis l'URL GitHub).
   - La `branch` par défaut de chaque repo est bien `main` (sinon corriger : certains anciens repos peuvent être sur `master`).
4. Laisser `squash` à `true` (défaut).
5. "Run workflow".

### 5. Vérifier

Une fois le workflow terminé (vert) :

- [ ] Le code de chaque ex-repo apparaît bien sous son préfixe (`hub/hub-core/...`, `hub/docs/...`, etc.).
- [ ] `git tag -l 'import/*'` (en clonant le repo) liste un tag par import.
- [ ] L'historique `git log` montre N commits de merge (un par import si squash).

Si quelque chose cloche : **supprime le repo `hub` et recommence** (le workflow refuse de tourner sur un repo non vierge, par sécurité).

### 6. Archiver les repos sources

Pour chaque repo importé (`hub-core`, `hub-frontend`, ..., `hub-docs`) :

1. Aller dans Settings → Danger Zone.
2. "Archive this repository".

⚠️ **Ne PAS supprimer.** L'archivage :
- Préserve les URLs (les liens externes vers le code restent valides).
- Empêche les nouveaux commits/issues/PRs (signal clair que c'est gelé).
- Reste consultable en lecture seule.

### 7. Mettre à jour le statut des ADRs

Dans le **nouveau** repo `hub`, sous `docs/decisions/` (qui vient de l'ex-`hub-docs`) :

- `0001-multi-repo.md` : changer `**Statut :** Acceptée` → `**Statut :** Superseded by 0009`
- `0009-monorepo.md`   : changer `**Statut :** Proposée (...)` → `**Statut :** Acceptée — 2026-XX-XX`

### 8. Adapter les chemins dans `CLAUDE.md`

Le `docs/CLAUDE.md` du nouveau repo référence `../../CLAUDE.md` et `../../04_master_plan.md`.
Ces chemins relatifs deviennent invalides (le monorepo n'a plus de "parent" qui contient
le master plan). Deux options :

- **Option A** : déplacer `04_master_plan.md`, `01_risk_audit.md`, `02_phasing.md` à la racine
  de `hub/` (recommandé, ça centralise vraiment tout).
- **Option B** : mettre à jour les chemins relatifs dans `CLAUDE.md`.

### 9. Démarrer une session Claude Code web sur `MoKarade/hub`

À ce point, tu peux fermer toutes les sessions sur les anciens repos et travailler
exclusivement sur le monorepo.

## Si la migration foire en cours

Le workflow a un garde-fou : il refuse de tourner si le repo a déjà plus d'un commit.
En cas d'échec partiel :

1. Supprimer le repo `MoKarade/hub` (Settings → Danger Zone → Delete).
2. Le recréer avec README initial.
3. Reposer le workflow et relancer.

Les repos sources ne sont pas modifiés par le workflow (clone read-only), donc
aucun risque de perte de données.
