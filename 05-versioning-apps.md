# 05 — Versioning des apps embarquées

## Pourquoi

Marc itère sur ses apps personnelles (trajets, finance) régulièrement. Il veut pouvoir avoir **plusieurs versions live en parallèle** pour comparer / rollback / ne pas tout casser quand il expérimente.

## Modèle adopté

Chaque app a un repo (`app-trajets`, `app-finance`) avec des **sous-dossiers par version** :

```
app-trajets/
├── README.md
├── shared/                  # code commun (client API hub)
└── versions/
    ├── v1/                  # version originale, gelée
    │   ├── Dockerfile
    │   ├── app/
    │   └── ...
    ├── v2/
    │   ├── Dockerfile
    │   ├── app/
    │   └── ...
    └── v3/                  # live actuelle
        ├── Dockerfile
        ├── app/
        └── ...
```

Chaque version :
- Est conteneurisée (1 Docker image par version)
- Tourne sur un port différent
- Est routée par Caddy via sous-chemin

## Routage Caddy

```caddyfile
@app_trajets_v1 path /apps/trajets/v1/*
handle @app_trajets_v1 {
    uri strip_prefix /apps/trajets/v1
    reverse_proxy app-trajets-v1:8011
}

@app_trajets_v2 path /apps/trajets/v2/*
handle @app_trajets_v2 {
    uri strip_prefix /apps/trajets/v2
    reverse_proxy app-trajets-v2:8012
}

# etc.
```

## Une seule version "writeuse"

Toutes les versions LISENT depuis l'API du hub-core. Mais **une seule version peut écrire** (la dernière en date, taggée "live").

Implémentation :
- Le hub-core a un endpoint `/v1/admin/active-version?app=trajets` qui dit qui est live
- Les versions non-live reçoivent un 403 si elles essaient un POST/PATCH/DELETE
- Le frontend du hub embed les versions read-only avec un badge "archive"

## Compatibilité de schéma

L'API du hub-core est **versionnée** (`/v1/...`). Les apps consomment cette API stable, pas la DB directement.

Quand le schéma DB change :
- Migration Alembic dans hub-core
- L'API `/v1` reste backward-compatible (jamais de breaking change)
- Si breaking : on bump à `/v2` et on garde `/v1` en parallèle un certain temps

Comme ça, une app v1 d'il y a 2 ans continue de marcher tant qu'elle parle à `/v1` qui existe encore.

## Création d'une nouvelle version

```powershell
cd app-trajets
$next = "v3"
cp -r versions/v2 versions/$next
# édite le code dans versions/$next
git add . && git commit -m "feat(app-trajets): bootstrap $next"
git tag app-trajets-$next
git push --tags
```

Ensuite update `docker-compose.prod.yml` dans hub-deploy pour rajouter le nouveau service `app-trajets-v3`, et le routing Caddy.

## Suppression d'une version

Tu peux freeze une version (Docker image dans GHCR avec tag), puis retirer du compose en prod. Le code reste dans le repo. Si tu veux la rejouer un jour : re-add au compose.

## UI sélecteur de version

Dans hub-frontend, sur la page d'une app embarquée :
```
Trajets [v1] [v2] [v3 ●]
```

Cliquer sur une version change l'iframe vers le bon sous-chemin. Pour les versions non-live, badge "archive" affiché.
