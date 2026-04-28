# hub-docs — Contexte pour Claude Code

> **Avant de commencer, lis aussi :** `../../CLAUDE.md` (handoff projet global) et `~/.claude/CLAUDE.md` (profil Marc + règles).

## Rôle du repo

Documentation française complète du Personal Data Hub. Repo séparé pour évoluer indépendamment du code.

## Convention de rédaction

- **Langue : français.** Marc est francophone.
- **Format : Markdown standard** (rendu GitHub).
- **Diagrammes : Mermaid** (rendu natif sur GitHub, pas de captures d'écran).
- **Pas de captures d'écran obsolètes :** préférer du texte qui ne se périme pas.
- **ADR (Architecture Decision Records) :** 1 fichier par décision majeure dans `decisions/<NNNN>-<slug>.md`. Format court : Contexte / Décision / Pourquoi / Trade-offs / Alternatives rejetées / Conséquences.

## État actuel (2026-04-28)

✅ `01-vision.md` — pourquoi/quoi/échelle, principes
✅ `02-architecture.md` — 3 diagrammes Mermaid (vue d'ensemble, flux IA, versioning apps)
✅ `05-versioning-apps.md` — comment cohabitent v1/v2/v3
✅ `07-runbook.md` — que faire si X tombe (10 symptômes → actions)
✅ `decisions/0001-multi-repo.md`
✅ `decisions/0002-event-sourcing.md`

❌ `03-data-model.md` — TODO Phase 1 (une fois les premiers modèles SQLAlchemy posés)
❌ `04-api-contract.md` — TODO Phase 1 (à générer depuis OpenAPI ou écrire à la main)
❌ `06-security.md` — TODO Phase 0 fin (threat model détaillé)
❌ `08-rgpd.md` — TODO Phase 3 (avec data sensibles : photos, emails, biométrie)
❌ `tutorials/` — TODO au fil des features (add-data-source, deploy-new-app-version, etc.)
❌ `decisions/0003+` — au fur et à mesure des choix techniques majeurs

## Quand créer un ADR

Quand on prend une décision architecturale qui :
- Sera difficile à inverser
- Engage le projet dans une direction (vendor, framework, pattern)
- Fait débat (pas évident pourquoi on choisit X plutôt que Y)

Exemples futurs probables :
- `0003-postgres-vs-sqlite-vs-duckdb.md`
- `0004-ollama-natif-vs-docker.md`
- `0005-cloudflare-tunnel-vs-tailscale.md`
- `0006-age-sops-vs-vault.md`
- `0007-versioning-par-souschemin-vs-sousdomaine.md`

## Règles spécifiques

- ✅ Mettre à jour les docs quand on change le code (la doc périmée est pire que pas de doc)
- ✅ Préférer prose à liste, sauf pour les listes vraiment listables (steps, choix)
- ✅ Diagrammes simples, pas surchargés
- ❌ Pas de jargon non expliqué (Marc est beginner)
- ❌ Pas de captures d'écran qui se périment
- ❌ Pas de doc auto-générée poubelle (ex: doxygen verbeux non lu)

## Liens

- Master plan : `../../04_master_plan.md`
- Audit risques : `../../01_risk_audit.md`
- Phasing : `../../02_phasing.md`
