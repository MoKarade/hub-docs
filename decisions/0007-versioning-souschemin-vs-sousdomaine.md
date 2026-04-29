# ADR-0007 — Versioning d'apps par sous-chemin Caddy (vs sous-domaines)

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Marc veut faire cohabiter plusieurs versions live de ses apps embarquées (`app-trajets`, `app-finance`). Cf. `05-versioning-apps.md`. Question : comment routent les requêtes vers la bonne version ?

Deux modèles classiques :
- **Sous-chemin** : `https://hub.example.com/apps/finance/v1/...`, `/v2/...`, `/v3/...`
- **Sous-domaine** : `https://finance-v1.hub.example.com`, `https://finance-v2...`

## Décision

**Sous-chemin via Caddy.** Une seule URL hôte, le path discrimine la version.

```caddyfile
@app_finance_v1 path /apps/finance/v1/*
handle @app_finance_v1 {
    uri strip_prefix /apps/finance/v1
    reverse_proxy app-finance-v1:8011
}

@app_finance_v2 path /apps/finance/v2/*
handle @app_finance_v2 {
    uri strip_prefix /apps/finance/v2
    reverse_proxy app-finance-v2:8012
}
```

## Pourquoi

1. **1 seul certificat TLS.** Cloudflare gère 1 cert pour `marc-hub.duckdns.org`. Avec des sous-domaines on ferait 1 cert par version (gérable mais fastidieux).
2. **1 seule policy Cloudflare Access.** Marc s'auth une fois, accède à toutes les versions. Avec des sous-domaines il faudrait dupliquer la policy ou créer un wildcard (qui est payant chez Cloudflare).
3. **Iframe simple côté hub-frontend.** `<iframe src="/apps/finance/v3/" />` ne déclenche pas de cross-origin. Avec des sous-domaines, on aurait des soucis de cookies cross-domain et de CORS.
4. **Selector de version trivial** : un dropdown change le path de l'iframe. Pas de redirection cross-domain à gérer.
5. **DNS plus simple.** 1 entrée DNS au lieu de N.
6. **Moins de surprises de Cloudflare Tunnel** : 1 tunnel vers 1 hostname, c'est plus simple à debug.

## Trade-offs acceptés

- **Couplage de l'app au préfixe.** Une app v1 doit savoir qu'elle est servie sous `/apps/finance/v1/` pour générer les liens internes correctement (sinon clic sur un lien casse). Mitigation :
  - Convention : les apps embarquées utilisent toujours des liens **relatifs** (`href="./settings"` pas `href="/settings"`).
  - Si une app legacy en absolute, on définit `<base href="/apps/finance/v1/">` dans son HTML.
  - Caddy fait `strip_prefix` pour que l'app interne se croit à la racine.
- **Cookies partagés** entre versions (même domaine). Si v1 set un cookie, v3 le voit. Acceptable pour Marc seul.
- **Logs moins clairs** : tous les hits sont sur `marc-hub.duckdns.org`. Mitigation : Caddy log en JSON avec le path complet.

## Alternatives rejetées

### Sous-domaine par version
- ❌ N certificats TLS (Cloudflare wildcard = payant, ou Let's Encrypt par sous-domaine = automatable mais lourd)
- ❌ N entrées DNS à maintenir
- ❌ N policies Cloudflare Access (ou wildcard payant)
- ❌ Cookies cross-domain à gérer
- ❌ CORS à configurer entre `hub.example.com` et `finance-v1.hub.example.com`

### Port différent par version (sans préfixe)
- ❌ `https://marc-hub.duckdns.org:8011/...` : moche, et Cloudflare Tunnel ne route pas les ports custom
- ❌ Pas exposable via un seul tunnel CF

### Header HTTP custom (`X-App-Version: v1`)
- ❌ Demande un client custom (impossible depuis un browser standard sans extension)

### Routage par query string (`?version=v1`)
- ❌ Pas standard, casse le caching navigateur
- ❌ Un user qui partage l'URL sans le `?version=` tombe sur la mauvaise version

## Conséquences

- ✅ `hub-deploy/caddy/Caddyfile` a déjà le template `@app_finance_v1`, `@app_trajets_v1` etc.
- ✅ `hub-frontend/app/apps/[app]/[version]/page.tsx` utilise un iframe vers `/apps/${app}/${version}/`.
- ✅ Convention de l'API hub-core : la version "live writeuse" est marquée par un endpoint `GET /v1/admin/active-version?app=trajets` (TODO Phase 2+). Les versions non-live reçoivent 403 sur les `POST/PATCH/DELETE`.
- ⚠️ Si une nouvelle app a besoin d'un sous-domaine pour des raisons techniques (cookies HttpOnly stricts, exemple Shopify), on revisitera.

## Pattern de migration entre versions

1. Créer `versions/v3/` à partir de `versions/v2/`
2. Build l'image Docker `app-finance-v3`
3. Ajouter le service dans `docker-compose.prod.yml` (port 8013, par exemple)
4. Ajouter le block `@app_finance_v3` dans `Caddyfile`
5. `docker compose up -d --build`
6. Tester sur `https://marc-hub.duckdns.org/apps/finance/v3/`
7. Quand stable : marquer `v3` comme writeuse via `/v1/admin/active-version`
8. v1 et v2 restent dispos en read-only avec badge "archive"

## Quand reconsidérer cette décision

- Si Marc ouvre le hub à plusieurs utilisateurs avec besoin d'isolation forte
- Si une app a besoin d'un cookie domain spécifique (ex: `Domain=app-finance.hub.example.com`)
- Si on dépasse 5+ versions par app (le Caddyfile devient verbeux — mais reste lisible)
