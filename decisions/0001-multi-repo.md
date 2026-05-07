# ADR-0001 — Multi-repo

**Date :** 2026-04-28
**Statut :** Superseded by 0009
**Décideurs :** Marc, Claude

## Contexte

Le projet a plusieurs composants distincts : hub-core (backend), hub-frontend (UI), hub-ingest (workers), hub-deploy (infra), hub-docs (documentation), 2 apps existantes versionnées (trajets, finance). Question : un repo unique (mono-repo) ou plusieurs (multi-repo) ?

## Décision

**Multi-repo : 1 repo par composant.**

7 repos visés :
- `hub-core`
- `hub-frontend`
- `hub-ingest`
- `hub-deploy`
- `hub-docs`
- `app-trajets`
- `app-finance`
- `hub-shared` (optionnel, Phase 2+)

## Pourquoi

Marc a explicitement choisi cette option en discovery (2026-04-28). Raisons :
- Plus modulaire, chaque composant est self-contained
- Releases indépendantes pour les apps versionnées
- CI/CD plus simple par repo
- Pas de monstre repo qui devient pénible à naviguer

## Trade-offs acceptés

- **Coordination de changements cross-repo** : un changement qui touche hub-core ET hub-frontend nécessite 2 PRs au lieu d'une. Mitigation : utiliser des release tags semver pour pin.
- **Pas de refactor atomic cross-repo** : on peut pas renommer un endpoint partout d'un seul coup. Mitigation : APIs versionnées + déprécation graduelle.
- **Setup local plus complexe** : il faut cloner N repos. Mitigation : un meta-script `clone_all.ps1` dans hub-deploy.

## Alternatives rejetées

- **Mono-repo (Nx, Turborepo, etc.)** : plus simple pour démarrer mais moins clair quand on a 8+ composants. Marc préférait multi-repo.
- **Mono-repo + submodules** : compromis bizarre, complexe à maintenir.

## Conséquences

- Chaque repo a son propre README, .gitignore, CI.
- `hub-deploy/scripts/clone_all.ps1` à créer plus tard pour faciliter setup nouveau PC.
- Tag semver pour les apps versionnées qui évoluent souvent.
