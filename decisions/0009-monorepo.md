# ADR-0009 — Bascule en monorepo

**Date :** 2026-05-07
**Statut :** Proposée (en attente d'exécution de la migration)
**Décideurs :** Marc, Claude
**Supersede (si acceptée) :** ADR-0001

## Contexte

L'ADR-0001 (2026-04-28) a acté un découpage en **multi-repo** (7 repos : `hub-core`, `hub-frontend`, `hub-ingest`, `hub-deploy`, `hub-docs`, `app-trajets`, `app-finance`, +`hub-shared` optionnel). Les raisons étaient : modularité, releases indépendantes des apps versionnées, CI simple par repo, éviter un "monstre repo".

Une nouvelle contrainte est apparue qui n'avait pas été évaluée en 2026-04-28 :

- **Marc travaille majoritairement via Claude Code sur le web**, jamais en local (contrainte explicite, pas un confort).
- Claude Code web **bind chaque session à un seul repo** (proxy git + scope MCP figés au démarrage). Vérifié : tentative de fetch d'un autre repo retourne `502` côté proxy.
- Conséquence pratique : tout changement cross-repo (typique : modif `hub-core` qui implique mise à jour `hub-docs`) demande **N sessions séquentielles** avec re-briefing du contexte à chaque fois. Le coût opérationnel est devenu prohibitif.

## Décision

**Bascule en monorepo unique `MoKarade/hub`** contenant tous les composants en sous-dossiers :

```
hub/
├── hub-core/          (ex-MoKarade/hub-core)
├── hub-frontend/
├── hub-ingest/
├── hub-deploy/
├── docs/              (ex-MoKarade/hub-docs)
├── apps/
│   ├── trajets/       (ex-MoKarade/app-trajets)
│   └── finance/       (ex-MoKarade/app-finance)
└── shared/            (ex-MoKarade/hub-shared, si existe)
```

Les noms de sous-dossiers seront finalisés au moment de la migration selon les noms réels des repos sur GitHub.

## Pourquoi

- **Une seule session Claude Code couvre tout le projet.** C'est la motivation principale, et elle suffit à elle seule vu le volume de cross-repo dans Phase 3.
- **Refactor atomic possible** : renommer un endpoint partout en une PR.
- **Diff cross-composant lisibles** : on voit dans la même PR le changement backend + la mise à jour de doc + le test E2E frontend.
- **Setup nouveau PC trivial** : `git clone` un seul repo au lieu de 7 (rend `hub-deploy/scripts/clone_all.ps1` obsolète, ce qui est un bénéfice net).

## Trade-offs acceptés

- **Releases indépendantes des apps versionnées** : on perd les releases GitHub par repo. Mitigation : tags semver préfixés par composant (`apps/trajets-v1.2.0`, `apps/finance-v0.9.3`). Releases GitHub continuent de fonctionner avec ces tags.
- **CI plus complexe** : un workflow unique doit utiliser `paths:` pour ne builder que ce qui change. Mitigation : convention `paths: ['hub-core/**']` par job. Gérable.
- **Repo plus gros à cloner** : non-problème (Marc a la fibre, et il ne clone presque jamais).
- **Historique git interleaved** : avec `git subtree` sans squash, `git log` mélange les commits des 7 repos. Mitigation : on importe avec `--squash` (un commit par sous-arbre, historique des sources préservé via tags `import/<repo>-2026-05-07`).
- **Contradiction explicite avec ADR-0001** : assumée. ADR-0001 sera marquée *Superseded by 0009* une fois la migration exécutée et validée.

## Alternatives rejetées

- **Garder multi-repo + sessions Claude séparées en série** : c'est l'état actuel. Marc a explicitement qualifié ça de "enfer".
- **Garder multi-repo + Claude Code en local** : Marc ne peut pas (contrainte hardware/OS, pas un choix).
- **Mono-repo avec submodules** : déjà rejetée en ADR-0001 ("compromis bizarre"), toujours valable.
- **Outillage type Nx/Turborepo** : prématuré pour un projet solo. Un monorepo "plat" suffit ; on pourra ajouter un orchestrateur si la CI devient pénible.

## Conséquences

### Migration (cf. `migration/README.md` dans `hub-docs`)

1. Créer le repo vide `MoKarade/hub` sur GitHub (initialiser avec un README pour avoir une branche `main`).
2. Y poser le workflow `bootstrap-monorepo.yml` (depuis `hub-docs/migration/`).
3. Déclencher le workflow → import `--squash` de chaque repo source dans son sous-dossier.
4. Vérifier que tout est en place ; tagger les imports.
5. **Archiver** (pas supprimer) chaque repo source sur GitHub : préserve les URLs, signale l'arrêt d'activité.
6. Mettre à jour ADR-0001 → `Superseded by 0009` ; passer ADR-0009 → `Acceptée`.
7. Mettre à jour `CLAUDE.md` du nouveau repo (chemins `../../CLAUDE.md` deviennent obsolètes vu que tout est sous une racine commune).

### Post-migration

- Toute nouvelle session Claude Code se fait sur `MoKarade/hub` uniquement.
- Le repo `hub-docs` archivé reste consultable en lecture seule pour l'historique et les liens externes.
- Les CI de chaque ex-repo doivent être réécrites en un workflow unique avec `paths:` filtering.
- Convention de tags pour les apps versionnées à formaliser dans un ADR-0010 si besoin.
