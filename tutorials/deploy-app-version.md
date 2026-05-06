# Tutoriel : Déployer une nouvelle version d'app embarquée

> Cible : shipper une v2 de `app-trajets`, `app-finance`, ou toute autre app versionnée embarquée dans le hub.

## Contexte

Les "apps" sont des sous-applications versionnées qui vivent à `/apps/<app>/<version>` dans le hub. Elles sont embarquées via iframe pour permettre des déploiements indépendants. Décision : ADR-0007 (versioning sous-chemin vs sous-domaine).

Le frontend principal (Next.js) sert juste un selector de version + iframe. Chaque version d'app est un build statique (HTML/JS) servi par le hub-frontend ou un static server séparé.

## Cas type : v1 → v2 d'app-trajets

### Étape 1 — Build la nouvelle version

```powershell
cd C:\hub\app-trajets
npm install
npm run build
# → produit ./dist/v2/ avec index.html + JS/CSS bundle
```

Convention : le build doit produire un dossier `dist/<version>/` avec un `index.html` standalone.

### Étape 2 — Copier le build dans hub-frontend

Le hub-frontend sert les apps via le route `/apps/[app]/[version]`. Copy le build :

```powershell
$dest = "C:\hub\hub-frontend\public\apps\trajets\v2"
Remove-Item -Recurse -Force $dest -ErrorAction SilentlyContinue
Copy-Item -Recurse "C:\hub\app-trajets\dist\v2" $dest
```

Vérifie qu'on a `public/apps/trajets/v2/index.html`.

### Étape 3 — Enregistrer la version dans le manifest

Le hub-frontend connaît les versions disponibles via un manifest JSON. Édite `hub-frontend/public/apps/manifest.json` :

```json
{
  "trajets": {
    "name": "Trajets",
    "description": "Carte des déplacements GPS",
    "versions": [
      { "id": "v1", "label": "v1.0", "released_at": "2026-04-30", "deprecated": false },
      { "id": "v2", "label": "v2.0", "released_at": "2026-05-06", "deprecated": false }
    ],
    "default_version": "v2"
  }
}
```

Le selector dans `/apps/trajets` lit ce manifest et propose v1 ou v2 (et marque v1 comme legacy si `deprecated: true`).

### Étape 4 — Smoke test local

```powershell
cd C:\hub\hub-frontend
npm run build  # rebuild Next.js avec le nouveau public/
npm run start
# → Va sur http://localhost:3000/apps/trajets/v2
```

Vérifie :
- L'iframe charge bien le `index.html` de la v2
- Le selector de version montre v1 + v2
- v1 reste accessible via `/apps/trajets/v1` (pas de breakage)

### Étape 5 — Commit + push

```powershell
cd C:\hub\hub-frontend
git add public/apps/trajets/v2 public/apps/manifest.json
git commit -m "feat(apps/trajets): release v2 — <feature ou fix>"
git push
```

Le pre-push hook lance lint + build prod + tests. S'ils passent, push autorisé.

### Étape 6 — Restart prod

Si Marc a déjà la PWA installée, il faut juste reload la page (pas de réinstall). Le manifest est cache-busted via le hash de build Next.js.

```powershell
# Stop ancien prod server
Stop-Process -Id (Get-NetTCPConnection -LocalPort 3000).OwningProcess
# Restart avec nouveau build
cd C:\hub\hub-frontend
npm run start
```

## Si la nouvelle version casse l'ancienne API

1. **Garde l'ancienne version** dans `public/apps/trajets/v1/` — Marc peut basculer si bug
2. **Marque `deprecated: true`** dans le manifest mais ne supprime pas
3. **Crée une migration** dans hub-core si la nouvelle version a besoin de nouvelles tables / colonnes
4. **Versionne l'API** : si v2 a besoin d'un nouveau format de payload, crée `/v2/...` côté backend, garde `/v1/...` qui sert les anciens

## Quand supprimer une vieille version

Après 2-3 versions stables et au moins 3 mois sans bug retour :

```powershell
Remove-Item -Recurse -Force C:\hub\hub-frontend\public\apps\trajets\v1
# Update manifest.json : retire l'entry v1
```

Garde le commit Git history pour pouvoir restaurer si besoin.

## Conventions

- `vN` ou `vN.M` (pas `version-1` ou `2026-04-30`)
- Tag le commit Git : `git tag app-trajets-v2.0` puis `git push --tags`
- Changelog dans `app-trajets/CHANGELOG.md` (qu'est-ce qui a changé entre v1 → v2)
- Tests E2E pour la nouvelle version dans `app-trajets/tests/v2/`

## Si l'app a un backend dédié

Certaines apps peuvent vouloir leur propre namespace API : `/apps-api/trajets/v2/*` au lieu de `/v1/...`. Pas implémenté pour l'instant — toutes les apps utilisent le hub-core principal. Si besoin futur, ADR-0009 documentera le pattern.

## Liens

- ADR-0007 versioning sous-chemin : `decisions/0007-versioning-souschemin-vs-sousdomaine.md`
- Hub-frontend route apps : `hub-frontend/app/apps/[app]/[version]/page.tsx`
- Manifest : `hub-frontend/public/apps/manifest.json`
