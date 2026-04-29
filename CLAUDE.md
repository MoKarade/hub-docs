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

## État actuel (2026-04-29)

✅ `01-vision.md` — pourquoi/quoi/échelle, principes
✅ `02-architecture.md` — 3 diagrammes Mermaid (vue d'ensemble, flux IA, versioning apps)
✅ `03-data-model.md` — schéma DB complet (6 tables, invariants, volumes attendus)
✅ `04-api-contract.md` — 19 routes /v1/* documentées
✅ `05-versioning-apps.md` — comment cohabitent v1/v2/v3
✅ `06-security.md` — threat model, 5 acteurs, secrets, chiffrement
✅ `07-runbook.md` — que faire si X tombe (10 symptômes → actions)
✅ `decisions/0001-multi-repo.md`
✅ `decisions/0002-event-sourcing.md`
✅ `decisions/0003-postgres-pgvector.md`
✅ `decisions/0004-ollama-natif.md`
✅ `decisions/0005-cloudflare-tunnel-vs-tailscale.md`
✅ `decisions/0006-age-sops-vs-vault.md`
✅ `decisions/0007-versioning-souschemin-vs-sousdomaine.md`

❌ `08-rgpd.md` — TODO Phase 3 (avec data sensibles : photos, emails, biométrie)
❌ `tutorials/` — TODO au fil des features (add-data-source, deploy-new-app-version, etc.)
❌ `decisions/0008+` — au fur et à mesure des choix techniques majeurs

## Quand créer un ADR

Quand on prend une décision architecturale qui :
- Sera difficile à inverser
- Engage le projet dans une direction (vendor, framework, pattern)
- Fait débat (pas évident pourquoi on choisit X plutôt que Y)

Exemples futurs probables :
- `0008-pgvector-hnsw-config.md` (en Phase 3 quand on ajoutera les embeddings)
- `0009-gmail-oauth-vs-app-password.md` (Phase 3)
- `0010-photos-storage-blob-vs-thumbnails.md` (Phase 3)

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
