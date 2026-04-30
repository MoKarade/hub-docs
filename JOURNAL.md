# Journal du Personal Data Hub

> Source de vérité unique pour : (1) le plan en cours, (2) l'historique des sessions, (3) les décisions techniques prises au fil de l'eau.
>
> Convention : on append à la fin pour le journal, on edit en place pour la section "Plan en cours".
>
> Tenue à jour à **chaque session importante**. Source : règle 2 du `~/.claude/CLAUDE.md` de Marc — "Tout sauvegarder, rien à oublier".

---

## Plan en cours

### Phase actuelle : **Étape 1 — Refonte UI + nouvelles sources de data**

> Contexte : Session #2 (2026-04-29) a terminé A+C+D+E (62 fichiers, 5 repos). Marc a demandé un redesign UI majeur + élargissement du scope (santé, streaming, sécurité). Les 16 questions de discovery ont été posées et **répondues le 2026-04-29**.
>
> Les décisions sont verrouillées : voir `sessions/2026-04-29_marc_answers_discovery.md`.
> Le plan complet est dans `sessions/SUITE.md`.

**Prochaines actions (dans l'ordre) :**
1. [x] **Étape 1 Sprint A** — système d'animations framer-motion + `Widget` conteneur + `LayoutProvider` ✅ CODE LIVRÉ + bugs corrigés + **CI 100% vert** (hub-frontend #7, hub-core #5, hub-ingest #4)
2. [x] **Étape 1 Sprint B** — SSE realtime + dnd-kit drag-drop + mode focus + resize widgets ✅ CODE LIVRÉ (hub-frontend `473aa32`, hub-core déjà poussé)
3. [x] **Étape 1 Sprint C** — reskinage Google Analytics dark ✅ CODE LIVRÉ + **TESTÉ LOCALEMENT** (5 commits b857b53→eb95328). Brief design dans `sessions/sprint-c-design-brief.md`. Frontend accessible sur http://localhost:3000.
4. [x] **Étape 1 Sprint D** — 8 pages stubs (`/emails`, `/photos`, `/calendar`, `/documents`, `/health`, `/insights`, `/settings`, `/system/health`) + mobile responsive (sidebar hamburger) + PWA (manifest + InstallPrompt) ✅ CODE LIVRÉ (`b71044e`)
5. [x] **Étape 1 Sprint E** — Fix erreurs (`/apps/*` 404) + 404 page custom + scripts `launch-app.ps1` + `install-desktop-app.ps1` (vraie app desktop avec raccourci bureau + menu Démarrer) ✅ LIVRÉ (`83795cb`, `f9f4428`)
6. [ ] **Étape 1 Sprint B+** — SSE realtime + dnd-kit advanced (deferred jusqu'au déploiement)
6. [ ] **Étape 2** — Déployer sur le vrai PC (Docker + Ollama + GPU)
6. [ ] **Étape 3** — Phase 0 fin (tunnel Cloudflare + backup restic)
7. [ ] **Étape 4** — Phase 2 fin (Marc fournit son Google Takeout)
8. [ ] **Étapes 5-7** — Santé (Garmin/Google Fit) + Streaming/Gaming + Sécurité+Suppression

---

### Phase actuelle (archive) : **Phase 2 — Localisation Google Maps Timeline**

#### Sprint 5 — Polish Phase 1 ✅ TERMINÉ

- [x] **Polish 1** — Fix Disnat ticker 1-caractère : `VISA INC CLASS-A` ne capturait pas `V` car la regex exigeait min 2 chars (`{1,10}` + `[A-Z]` = 2 min). Changé en `{0,9}` → 1 char acceptable. Vérifié : VISA correctement extrait.
- [x] **Polish 2** — Prompt AI plus précis sur la devise. Ajout `_ANSWER_SYSTEM_PROMPT` dédié pour le 2e appel LLM (formulation), qui rappelle que Marc est au Québec, devise CAD par défaut, sauf compte Disnat USD. **Plus de "euros"** dans les réponses.
- [x] **Polish 3** — Replay Disnat clean (truncate + re-import). Vérifié : 21 transactions + 25 positions, plus aucune incohérence.

**Bug LLM connu (acceptable)** : Qwen peut générer des dates invalides (`2026-02-29` alors que 2026 n'est pas bissextile). L'erreur Postgres remonte proprement via HTTP 400 avec le SQL fautif. Pas un bug du système — un fait du LLM.

---

#### Phase 2 — Localisation Google Maps Timeline (en cours)

**But** : importer l'historique de localisation de Marc depuis Google Takeout pour pouvoir répondre à des questions comme "où étais-je le 15 janvier 2026 à 14h ?", "combien de km parcourus en mars ?", etc.

- [x] 2.1 JOURNAL préparé (9 soucis anticipés)
- [x] 2.2 Modèle `LocationPoint` créé (timestamp UTC, latitude/longitude `Numeric(10,7)` pour précision ~1cm, latE7/lngE7 entiers pour idempotence, accuracy_m, altitude_m, activity_type optionnel) + migration `5044ada9f866_phase2_location_points` appliquée.
- [x] 2.3 3 endpoints `/v1/locations/*` : POST `/points` (idempotent), GET `/points` avec filtres temporels (start/end/start_date/end_date), bbox géographique (min/max_lat/lng), activity_type, source, pagination. GET `/points/{id}`.
- [x] 2.4 `ijson>=3.3.0` ajouté aux deps `hub-ingest` (streaming JSON pour les fichiers 100MB+).
- [x] 2.5 Parser `google_takeout_timeline.py` : streame `Records.json` via `ijson.items('locations.item')`, parse les timestamps (epoch ms ou ISO 8601), filtre `accuracy > 100m`, sample 1 point / 30 sec, extrait l'`activity_type` principal (par confiance), convertit latE7/lngE7 → Decimal lat/lng à 7 décimales.
- [x] 2.6 Pipeline `google_timeline.py` : workflow standard (dump raw, POST par point avec retry tenacity, move processed/). Log de progression toutes les 1000 points.
- [x] 2.7 `C:\hub\inbox\google-timeline\` créé + [README](inbox/google-timeline/README.md) qui guide Marc : comment télécharger via takeout.google.com → extraire ZIP → copier `Records.json` → lancer `docker compose --profile ingest run --rm hub-ingest`.

**Sprint Phase 2 = CODE-COMPLETE.** Marc doit fournir son JSON Takeout pour activer l'import. Tout est prêt à parser dès que le fichier arrive dans l'inbox.

**Bilan endpoints API** : 19 routes exposées sous `/v1/*` (health, finance × 7, locations × 3, ai × 2 + root).

**Soucis anticipés Phase 2 :**

1. **Volume énorme** : un export Takeout peut faire 100MB-1GB de JSON. `ijson` obligatoire pour streaming, sinon OOM kill.
2. **2 formats Takeout** : ancien `Records.json` (locations[]) et nouveau Timeline 2024+ avec `semanticSegments`. Le parser doit auto-détecter ou Marc précise.
3. **Timestamps en ms epoch ou ISO** : selon la version. Convertir tous en UTC ISO.
4. **Précision GPS variable** : indoor/tunnel = 100m+. On filtre.
5. **Idempotence** : la même position peut apparaître dans plusieurs exports successifs. Hash sur `(timestamp_ms, latE7, lngE7)`.
6. **Vie privée extrême** : c'est tout l'historique géo de Marc. Doit rester local, ne JAMAIS sortir.
7. **Sample temporel** : 1 point / 30 sec = compromis entre détail (rester debout en café = OK avec 1 point/30s) et volume (1 jour = ~2880 points max).
8. **Activity type** : Google fournit parfois (driving, walking, cycling, still). Optionnel à conserver.
9. **Marc n'a pas encore le fichier** : on code le parser/pipeline d'abord, Marc fera le download Takeout après. Code prêt à parser dès qu'il dépose le JSON.

**Livrable Phase 2 :** une fois Marc fournit son Takeout, ses 10 ans d'historique de localisation sont en DB, accessibles via `/v1/locations/points` + interrogeables via `/v1/ai/ask` ("où étais-je le X ?").

---

## Session #11 — Fix erreurs + App desktop installable (2026-04-30)

**But :** Marc demande "il y a des erreurs, corrige tout" + "j'aimerais avoir genre un app sur mon pc que je puisse ouvrir et avoir tout, pareil une app sur mon tel".

**Travail effectué :**

### 1. Correction des erreurs ✅

**Bug :** `/apps/trajets` et `/apps/finance` (liens dans la sidebar) → 404 noir générique de Next.js.

**Solutions :**
- Créé `app/apps/trajets/page.tsx` (ComingSoon Phase 2+, leaflet.heat heatmap)
- Créé `app/apps/finance/page.tsx` (ComingSoon Phase 2+, recharts avancés)
- Créé `app/not-found.tsx` (404 design Sprint C avec icône MapPinOff, sidebar préservée, boutons "Retour dashboard" + "Rechercher")
- Fix lint : apostrophes échappées avec `&apos;` dans 404 page

**Audit complet** : 14 routes valides retournent 200, `/notexisting` retourne 404 custom propre. Build prod passe sans erreur (16 routes au total avec /_not-found et /apps/*).

Commit : [hub-frontend@83795cb](https://github.com/MoKarade/hub-frontend/commit/83795cb)

### 2. App desktop installable ✅

**Stratégie pragmatique** : pas besoin de Tauri ou Electron — Chrome `--app=URL` ouvre déjà une fenêtre standalone.

**`hub-deploy/scripts/launch-app.ps1`** :
- Démarre Ollama daemon si pas actif
- Démarre Docker stack (postgres + hub-core) si Docker installé
- Démarre frontend Next.js (cherche dans `C:\HubFrontend` puis fallback Drive)
- Ouvre Chrome (ou Edge fallback) en `--app=http://localhost:3000` (vraie fenêtre app, sans barre URL)
- Fallback gracieux si Docker absent (frontend marche, états d'erreur sur les pages data)

**`hub-deploy/scripts/install-desktop-app.ps1`** :
- Génère icône `hub-perso.ico` (256x256 gradient vert + lettre H)
- Crée raccourci `Hub perso.lnk` sur le bureau
- Crée entrée dans menu Démarrer (`Programs\Hub perso\`)
- Le raccourci lance launch-app.ps1 en `WindowStyle Hidden`

**Test sur ce PC** : raccourci créé sur OneDrive Bureau (`C:\Users\dessin14\Marc Richard\OneDrive - ROBOVIC\Bureau\Hub perso.lnk`) + entrée menu Démarrer ✅.

### 3. Doc PWA mobile + desktop ✅

**`hub-deploy/docs/INSTALL-AS-APP.md`** :
- **Option A** : App Desktop Windows (launch-app + install-desktop-app)
- **Option B** : PWA install (Chrome/Edge "Installer" + iOS/Android "Add to Home Screen")
- **Option C** : Tauri natif (futur, requiert Rust toolchain)
- Comparatif des 3 + troubleshooting (icônes, HTTPS, iOS Safari obligatoire)

Le manifest.json + InstallPrompt component (Sprint D) rendent déjà la PWA installable depuis n'importe quel browser supporté.

### 4. Bug PowerShell rencontré (résolu)

**Premier write de install-desktop-app.ps1** : caractères Unicode (`—`, `'`, `é`) faisaient parser-error en PowerShell 5.1 (encodage cp1252 au lieu d'UTF-8).

**Fix** : réécrit en ASCII strict, plus d'em-dashes, plus d'apostrophes courbes. Validation via `[Parser]::ParseFile()` pour confirmer 0 erreurs.

Mêmes erreurs initiales pour launch-app.ps1, fixées de la même façon.

### 5. Bug launch-app.ps1 v1 (résolu)

Marc a reporté "ça marche pas pour le pc". Diagnostic en lançant manuellement :

```
[*] Verification frontend...
  Demarrage frontend depuis C:\HubFrontend...
  Attente du frontend...
  (1/30) (2/30) ... (30/30) en attente de http://localhost:3000...
  [X] Frontend timeout. Verifie les logs.
```

**Cause** : `Start-Process node "C:\...\next\dist\bin\next" dev` créait un process zombie qui n'écoutait pas. Le fichier `dist/bin/next` n'a pas de shebang Windows, donc node ne sait pas l'exécuter en tant que script.

**Fix** : utiliser le wrapper `.bin\next.cmd` via `cmd.exe /c` :
```powershell
Start-Process -FilePath "cmd.exe" `
  -ArgumentList "/c", "`"$nextCmd`"", "dev" `
  -WorkingDirectory $frontendDir -WindowStyle Hidden
```

**Validé** : le frontend démarre maintenant en 4s, Chrome s'ouvre en mode app standalone.

Commit : [hub-deploy@5a24531](https://github.com/MoKarade/hub-deploy/commit/5a24531)

### Commits & push

- [hub-frontend@83795cb](https://github.com/MoKarade/hub-frontend/commit/83795cb) — fix routes /apps/* + 404 custom
- [hub-deploy@f9f4428](https://github.com/MoKarade/hub-deploy/commit/f9f4428) — launch-app + install-desktop-app + INSTALL-AS-APP.md

### Pour Marc — comment utiliser

**Sur ce PC (test)** :
1. Double-clique sur "Hub perso" sur ton bureau (déjà installé) → tout démarre + Chrome ouvre l'app
2. Sans Docker installé, le frontend marche mais affiche "Failed to fetch" sur les pages data (normal)

**Sur l'autre PC (le vrai)** :
1. Clone les repos
2. `npm install` dans `hub-frontend` (workaround `C:\HubFrontend` si chemin Drive)
3. Install Docker Desktop + lancer
4. `cd hub-deploy && .\scripts\install-desktop-app.ps1` → raccourci bureau + menu Démarrer
5. Double-clic sur l'icône → l'app démarre tout et s'ouvre comme une vraie app

**Sur ton téléphone (iOS/Android)** :
1. Connecter au même Wi-Fi (ou setup Cloudflare Tunnel pour accès externe)
2. Ouvrir `http://<IP-PC>:3000` dans Chrome/Safari
3. Menu → "Add to Home Screen" / "Installer l'app"
4. Tap sur l'icône → app fullscreen comme une vraie app

---

## Session #10 — Sprint D : Pages stubs + Mobile + PWA (2026-04-30)

**But :** Continuer pendant que Marc setup Docker sur l'autre PC. Faire tout ce qui peut être fait sans data.

**Travail effectué :**

### 1. Pages stubs (8 nouvelles pages) ✅

Composant réutilisable `ComingSoon` créé (`components/coming-soon.tsx`) avec :
- Icône hero + badge phase + ETA
- Description + sources prévues + capabilities
- Footer hint vers JOURNAL.md
- Style Google Analytics (`.ga-card`, `.metric-label`)

**8 nouvelles pages :**

| Route | Phase | Sources prévues |
|---|---|---|
| `/emails` | Phase 3 | Gmail API, OAuth 2.0, IMAP fallback |
| `/photos` | Phase 3 | Google Photos Takeout, CLIP ViT-B/32, EXIF |
| `/calendar` | Phase 5 | iCal Google, Apple Calendar, Outlook |
| `/documents` | Phase 5 | inbox PDFs, pdfplumber, OCR tesseract, classification LLM |
| `/health` | Phase 5 | Garmin Connect, Apple Health, Google Fit, FIT files |
| `/insights` | Phase 4+ | Cross-référencement banking + locations + santé + calendar |
| `/settings` | Phase 1+ | localStorage, Postgres configs, age+sops |
| `/system/health` | Phase 1 | Live SWR `/v1/ready` (refresh 5s) |

La page `/system/health` est interactive : elle affiche l'état de chaque service (Postgres, Ollama, Hub-core, Redis, Cloudflare Tunnel) avec des dots colorés `data-positive`/`data-negative`/unknown et liste les endpoints API.

### 2. Mobile responsive ✅

**Sidebar :**
- Hamburger button (icon `Menu`) fixed top-3 left-3 z-40 sur `<lg`
- Drawer fullscreen overlay z-50 avec backdrop ink-950/70 cliquable
- Auto-close sur navigation (`pathname` change)
- Auto-close sur Esc + bouton X interne
- Lock `body.overflow = hidden` quand ouvert
- Force `isCollapsed = false` quand `mobileOpen` (UX cohérent)
- Toggle "Réduire" caché sur mobile (`hidden lg:flex`)

**Layout :**
- Toutes les pages : `flex-1 px-4 sm:px-6 lg:px-8 pt-16 lg:pt-6 pb-6`
  - `pt-16` mobile pour laisser place au hamburger button
- WidgetGrid : `grid-cols-1 sm:grid-cols-2 md:grid-cols-3`
- WidgetSize cycles adaptés : sm/md = col-1, lg = col-1 sm:col-2, xl/full = col-1 sm:col-2 md:col-3
- FocusModal déjà responsive (`inset-4 md:inset-8 lg:inset-12`)

### 3. PWA ✅

**Fichiers ajoutés :**
- `public/icon.svg` — Logo SVG 512x512 (gradient accent vert + boxes lucide)
- `public/manifest.json` — Manifest complet avec name, short_name, theme_color (#0f1419), background_color (#0a0e14), shortcuts (Recherche, Finances, Localisation), display: standalone

**Layout enrichi (`app/layout.tsx`) :**
```ts
export const metadata: Metadata = {
  manifest: '/manifest.json',
  applicationName: 'Hub perso',
  appleWebApp: { capable: true, title: 'Hub perso', statusBarStyle: 'black-translucent' },
  icons: { icon: '/icon.svg', apple: '/icon.svg', shortcut: '/icon.svg' },
  formatDetection: { telephone: false },
}
export const viewport: Viewport = {
  themeColor: '#0f1419',
  width: 'device-width',
  initialScale: 1,
  maximumScale: 5,
  userScalable: true,
  colorScheme: 'dark',
}
```

**InstallPrompt component (`components/install-prompt.tsx`) :**
- Capture `beforeinstallprompt` event (Chrome/Edge/Brave)
- Bannière bottom-right avec icône Download, titre, description, boutons Installer/Plus tard/X
- Snooze 7 jours via `localStorage['hub-install-snooze-until']`
- Auto-hide si déjà installée (`window.matchMedia('(display-mode: standalone)')`)
- Auto-hide après install confirmé (`appinstalled` event)

### 4. Build prod validé ✅

```
Route (app)                   Size      First Load JS
/                            127 kB     301 kB
/calendar                    1.55 kB    140 kB
/documents                   1.56 kB    140 kB
/emails                      1.61 kB    140 kB
/finances                    3.64 kB    142 kB
/health                      1.62 kB    140 kB
/insights                    1.67 kB    140 kB
/locations                   3.42 kB    142 kB
/photos                      1.59 kB    140 kB
/search                      2.77 kB    141 kB
/settings                    1.62 kB    140 kB
/system/health               1.95 kB    141 kB
```

13 routes au total. Shared bundle 102 kB. Tout statique sauf `/`.

### Commit & push

- **hub-frontend** : `b71044e` — feat(sprint-d): 8 pages stubs + mobile responsive + PWA
- 19 files changed, 814 insertions(+), 28 deletions(-)

### Bug rencontré (résolu)

Premier test mobile : sidebar prenait tout l'écran (w=1664) avec position static. Cause : Tailwind n'avait pas généré les nouvelles classes `lg:*` car le dev server avait été lancé avant la copie des fichiers à `C:\HubFrontend`. Solution : kill + restart node, Tailwind a regénéré le CSS au boot.

### Next steps

Quand Docker installé chez Marc :
- Setup hub-core stack (`./scripts/start_hub.ps1`)
- Tester les vraies données dans le frontend
- Commencer Phase 1+ : streaming SSE realtime widgets
- Phase 0 fin : Cloudflare Tunnel + DuckDNS + backup restic

---

## Session #9 — Sprint C polish + Sprint B validation + Ollama (2026-04-30)

**But :** Faire tout ce qui peut être fait sans Marc présent (fixes bugs Sprint C, validation Sprint B, prép infra).

**Travail effectué :**

1. **Fix react-leaflet (page Localisation)** ✅
   - Bug : page `/locations` crashait avec "client-side exception" car react-leaflet@4.2.1 demande peer react ^18 (incompat React 19)
   - Solution : upgrade `react-leaflet` 4.2.1 → 5.0.0 (support React 19 natif)
   - Commit : `2fe2622` poussé sur `MoKarade/hub-frontend`
   - Validé visuellement : carte OpenStreetMap Québec/Lévis avec filtres date + activité

2. **Validation Sprint B (déjà code-livré, jamais testé)** ✅
   - **Sidebar collapse** : toggle 264px ↔ 60px fonctionne, persistance localStorage OK, tooltips Radix au hover en mode réduit
   - **Mode focus** : click bouton Maximize2 sur widget → ouvre `FocusModal` plein écran avec backdrop blur, fermeture via X
   - **Drag handles dnd-kit** : visibles (`GripVertical`), câblés via `cloneElement` dans `WidgetGrid` → `SortableItem`
   - **Cycle de taille** : boutons S/M/L/XL/↔ visibles au hover sur chaque widget

3. **Ollama installé via winget** ✅
   - Version 0.22.0
   - Daemon tourne sur :11434
   - Pull modèles en cours : `nomic-embed-text` (~270 MB) en background

4. **Docker Desktop : ÉCHEC install via winget** ❌
   - Exit code 4294967291 (besoin admin + WSL2 enabled + reboot)
   - Marc devra installer manuellement : https://docs.docker.com/desktop/install/windows-install/
   - Sans Docker → pas de Postgres → pas de hub-core → pas de vraies données dans le frontend

5. **Préparation infra** ✅
   - `.env` créé dans `hub-deploy/` depuis `.env.example`
   - SECRET_KEY généré (32 bytes random base64)
   - POSTGRES_PASSWORD aléatoire généré
   - Prêt pour `docker compose up` une fois Docker installé

**Conclusion :**
Sprint C est maintenant **100% fonctionnel sur les 4 pages** (Dashboard, Finances, Recherche, Localisation). Sprint B validé visuellement (drag-drop, focus, sidebar). Frontend prêt pour intégration backend dès que Docker est installé.

**Bloquant restant :**
- **Docker Desktop installation manuelle requise** (admin + WSL2 + reboot Windows)
  - Lien : https://docs.docker.com/desktop/install/windows-install/
  - Après install : `cd hub-deploy && docker compose -f docker-compose.dev.yml up -d`
  - Puis pull qwen2.5:14b dans Ollama (~9 GB)

**Validations supplémentaires :**
- ✅ `npm run build` → 6 pages compilées sans erreur
  - Fix tsconfig.json : `"types": ["node"]` (sinon erreur "Cannot find type definition file for 'd3-color'" via recharts)
  - Commit : `0bb68f7`
- ✅ `qwen2.5:14b-instruct` pull en cours (~9 GB, ~3 min) en background

**Quand Marc revient (bootstrap simple) :**

```powershell
# 1. Installer Docker Desktop (UNE FOIS)
#    Lien : https://docs.docker.com/desktop/install/windows-install/
#    Nécessite : admin + WSL2 + reboot Windows

# 2. Démarrer Docker Desktop (icône systray)

# 3. Lancer la stack — tout est déjà câblé
cd "G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\hub-deploy"
.\scripts\start_hub.ps1
# → vérifie Docker + Ollama + pulls modèles + docker compose up + healthcheck

# 4. Lancer le frontend (en parallèle dans un autre terminal)
cd C:\HubFrontend
npm run dev
# → http://localhost:3000

# Tout est prêt. .env hub-deploy contient déjà SECRET_KEY + POSTGRES_PASSWORD
# générés. Ollama daemon tourne déjà sur :11434.
```

**État final session #9 :**
- Frontend : 100% fonctionnel sur 4 pages, build prod OK
- Sprint A/B/C : complets et validés visuellement
- Backend : prêt à lancer dès Docker installé
- Ollama : installé + daemon up + nomic-embed-text ready
- Modèle qwen2.5:14b : en cours de pull (~3 min)

---

## Session #8 — Testing Sprint C + Frontend Launch (2026-04-29, nouveau PC)

**But :** Lancer le frontend localement sur le nouveau PC (dessin14) et valider que Sprint C (Google Analytics redesign) fonctionne end-to-end.

**Travail effectué :**

1. **Setup npm sur nouveau PC :**
   - Créé `.env.local` depuis `.env.example` ✅
   - Problème : `npm install` échoue sur chemins Google Drive (`G:\Mon disque\PERSO & LOISIRS\...`) avec git-bash (corruption de fichiers, espaces dans le chemin). ❌
   - Solution : copié projet entier vers `C:\HubFrontend` (sans espaces) + `npm install --legacy-peer-deps` réussi ✅

2. **Frontend running :**
   - `npm run dev` lancé → Next.js 15.5.15 écoute sur http://localhost:3000 ✅
   - Page d'accueil charge correctement ✅
   - Navigation sidebar fonctionne ✅

3. **Sprint C validé sur 3 pages :**
   - **Dashboard** : Dark mode épuré, palette ink-900, icônes lucide-react systématiques, semantic coloring (vert accent), widgets Google Analytics style (KPI strip avec .metric/.metric-label/.metric-delta)
   - **Finances** : KPI strip avec DÉBITS rouge (data-negative), CRÉDITS vert (data-positive), tabs (Banque/CC/Invest), filtres, table transactions en state vide gracieux
   - **Recherche IA** : Input "Demande à ton hub", conversation histoire, SQL visible par défaut (Sprint C4), error handling élégant (icône AlertCircle, red border subtle)

4. **Known issues (mineurs, déferred) :**
   - Page `/locations` : erreur client-side (react-leaflet probable incompatibilité, ou dynamique import SSR issue)
   - Pas de données : hub-core n'est pas en cours (pas de Docker sur ce PC encore). API calls retournent 404/Failed to fetch.
   - Sidebar collapse button : bouton trouvé mais collapse animation n'est pas visuelle (ou état non persiste UI, à investiguer)

5. **Documentation :**
   - Sprint C brief était déjà dans `sessions/sprint-c-design-brief.md` ✅
   - Code déjà pushé (b857b53→eb95328) lors session #7 ✅
   - Rien à committer aujourd'hui (juste test + documentation)

**Conclusion :**
Sprint C est **visuellement complète et fonctionnelle** pour le happy path. Le design Google Analytics est validé : dark mode épuré, sémantic coloring, data-dense layout, icônes ubiquitaires, animations subtiles. Prêt pour Sprint B+ (realtime SSE) une fois les fixes mineurs (react-leaflet) sont résolus.

**Blockers résumés :**
1. npm sur chemin Google Drive = changer le workflow (toujours copier vers C:\ ou symlink)
2. Hub-core non déployé sur ce PC = pas de vraies données pour tester l'IA/API
3. Locations page erreur = à debugger (probablement dynamic import react-leaflet + SSR hydration)

**Next steps (quand Marc veut continuer) :**
- [ ] Docker Desktop setup sur nouveau PC + hub-core container ⟹ données réelles + requêtes IA
- [ ] Fix react-leaflet page (SSR/dynamic import)
- [ ] Sprint B+ (SSE realtime + dnd-kit drag-drop)
- [ ] Persist sidebar collapse state en localStorage (vérifié code existe, check hydration)

---

### Phase 1 — Banking + IA basique ✅ TERMINÉE

(Détails ci-dessous, intacts pour traçabilité.)


#### Sprint 1 — Foundation DB + API endpoints ✅ TERMINÉ

**But :** avoir les tables et les endpoints HTTP pour stocker une transaction bancaire de bout en bout.

- [x] 1.1 Audit de la structure existante de `hub-core` (`src/db/`, `src/api/v1/`, `src/core/config.py`)
- [x] 1.1b **Bonus** : ajout d'un bind mount + `uvicorn --reload` dans `docker-compose.dev.yml` → hot-reload du code sans rebuild de l'image
- [x] 1.2 Modèles SQLAlchemy `Account` + `Transaction` créés dans `hub-core/src/db/models/`, avec `DateTime(timezone=True)` pour les timestamps UTC
- [x] 1.3 Migration Alembic `c84411d473b8_phase1_finance_accounts_transactions` générée + appliquée → 3 tables en DB (`accounts`, `transactions`, `alembic_version`)
- [x] 1.4 Schémas Pydantic + 5 endpoints `/v1/finance/*` (POST/GET accounts, POST/GET transactions, GET account by id) avec idempotence par `(institution, account_number_masked)` côté account et par `dedup_hash` SHA-256 côté transaction
- [x] 1.5 10 tests curl passent : création, idempotence, filtres, validation Pydantic (debit XOR credit), 404 sur compte inexistant

**Livrable Sprint 1 :** on peut créer un compte et insérer une transaction via l'API. La validation et l'idempotence sont en place. Rien d'importé pour de vrai encore — c'est le job du Sprint 2.

#### Sprint 2 — Parser CSV compte courant + épargne ✅ TERMINÉ

- [x] 2.1 Audit du skeleton hub-ingest : structure propre (Connector ABC, dump_raw, ntfy, APScheduler), toutes les deps déjà là (httpx, tenacity, structlog).
- [x] 2.2 Mise à jour `hub-ingest/src/core/config.py` : ajout `hub_api_base_url`, `inbox_dir`, `run_mode`.
- [x] 2.3 Dockerfile hub-ingest créé (Python 3.13-slim + curl, install editable).
- [x] 2.4 hub-ingest ajouté au compose dev sous le profile `ingest` (lancé à la demande via `docker compose --profile ingest run --rm hub-ingest`). Bind mounts : src + inbox + raw_events.
- [x] 2.5 Parser `desjardins_csv.py` : dataclass `DesjardinsCsvRow` + `parse_desjardins_csv(path)` qui yield les rows. Decode en cp1252, skip lignes vides, tolère lignes mal formées (log + skip). Mappe `EOP`→`checking`, `ET1/ET2/...`→`savings`. Hash SHA-256 sur `transit|type|date|seq|description|debit|credit|balance`.
- [x] 2.6 `pipelines/desjardins.py` : classe `DesjardinsPipeline` avec workflow complet (dump raw → group par compte → ensure_account idempotent → POST txn idempotent → move to processed/ uniquement si zéro erreur). Cache local des UUIDs comptes pour limiter les POST. Retry tenacity (3 tentatives, backoff exponentiel) sur les appels API.
- [x] 2.7 `main.py` refactoré avec dispatch `run-once` vs `scheduler`. Sprint 2 utilise `run-once`.
- [x] 2.8 `C:\hub\inbox\desjardins\` créé + 3 CSV réels copiés depuis `Downloads/`. `.gitignore` préventif posé à la racine de `C:\hub\`.
- [x] 2.9 Premier run : **3 fichiers, 64 transactions, 0 erreur**. Du premier coup.
- [x] 2.10 Validation DB : 2 comptes créés (`377646-EOP` checking, `377646-ET1` savings), 64 transactions réparties (EOP : 22 jan + 21 fév + 15 mars = 58 ; ET1 : 2 + 3 + 2 = 7). Caractères accentués correctement décodés (`AccèsD`). Idempotence validée : ré-import des 3 CSV = toujours 64 transactions en DB.

**Soucis anticipés (à surveiller) :**

1. **Encodage cp1252** : déjà confirmé, à appliquer dès la lecture du fichier. Risque : caractères mal décodés sur les libellés `Acc�sD`.
2. **CSV sans header + 14 colonnes** : utiliser `csv.reader` Python (pas split) pour gérer correctement les guillemets dans les descriptions.
3. **Numéro de séquence remis à zéro chaque mois** : `(transit, account_type, date, seq)` pas globalement unique. `dedup_hash` doit inclure year+month+seq+amounts.
4. **EOP et ET1 dans le même fichier** : un CSV peut couvrir plusieurs comptes. Le parser doit splitter par `account_type`.
5. **Création auto des comptes** : si le parser voit un transit/type qu'on n'a pas en DB, le pipeline doit créer le compte (idempotent côté API).
6. **Hub-ingest pas encore conteneurisé** : pas de Dockerfile, absent du compose. À créer.
7. **Réseau Docker → hub-core** : depuis hub-ingest container, hub-core est joignable via `http://hub-core:8000` (nom du service compose), pas `localhost`.
8. **Fichier en cours d'écriture** : si Marc copie un CSV dans `inbox/`, on pourrait le lire à moitié. Mitigation Sprint 2 : on ne fait pas de watcher continu, on traite à la demande après que Marc a copié. Watcher continu = Sprint 2.5 ou plus tard.
9. **raw_events folder** : doit être bind-mount aussi dans hub-ingest pour persister entre les exécutions.
10. **Erreurs HTTP** : timeouts, 5xx → retry avec tenacity. Si dedup déjà en DB, l'API renvoie 201 idempotent (pas d'erreur).
11. **CSV avec ligne vide en début** : observé dans janv2026.csv (la 1re ligne est blanche) → skip lignes vides.
12. **Description avec `/`** : observé `Loyer/bail /9478 5045 Quebec inc` — le `/` n'a pas de sémantique pour le parser CSV, c'est juste un caractère.

**Livrable Sprint 2 :** Marc voit ses 3 mois de transactions débit/épargne dans la DB. **Premier moment où le hub devient utile.**

#### Sprint 3 — Parser PDF Mastercard ✅ TERMINÉ

- [x] 3.1 Préparation JOURNAL (plan + 12 soucis anticipés)
- [x] 3.2 `pdfplumber>=0.11.4` ajouté aux deps `hub-ingest/pyproject.toml`
- [x] 3.3 Modèle `CreditCardTransaction` (transaction_date + posting_date + amount signé + cashback_rate + card_number_masked + section + statement_date)
- [x] 3.4 Migration Alembic `cfb9d1cc7135_phase1_credit_card_transactions` appliquée → table `credit_card_transactions` créée avec 4 indexes
- [x] 3.5 2 nouveaux endpoints : `POST /v1/finance/credit-card-transactions` (idempotent par dedup_hash) et `GET` (avec filtres account_id, card, section, dates, statement_date)
- [x] 3.6 Test pdfplumber : extraction de texte propre, structure tabulaire bien préservée. Outil de debug `src/scripts/dump_pdf.py` créé pour le futur.
- [x] 3.7 Parser `desjardins_mastercard_pdf.py` : regex `_TXN_LINE_RE` capture `JJ MM JJ MM <description> [rate%] <amount>[CR]`. Heuristique année (`statement_date.year - 1` si mois > mois du relevé, sinon `statement_date.year`). Mapping section : marqueurs `DESCRIPTION DES TRANSACTIONS COURANTES`, `Opérations au compte`, `REMISES EN ARGENT` (skip).
- [x] 3.8 Pipeline `mastercard.py` calqué sur `desjardins.py`. Branché dans `main.py run-once`.
- [x] 3.9 4 PDFs copiés dans `C:\hub\inbox\desjardins-cc\` (renommés `2026-MM-mastercard.pdf`).
- [x] 3.10 **Premier run : 4 PDFs, 360 transactions parsed, 360 pushed, 0 erreur HTTP**. MAIS : 352 en DB (≠ 360) → bug détecté.
- [x] 3.11 **Bug rencontré et résolu — collisions de hash sur transactions répétées à l'identique.** Le 26 décembre 2025, Marc a 4 transactions identiques `MYIQ.COM/CA VANCOUVER BC 0,70` (2 avec cashback 2%, 2 sans). Mon `dedup_hash` initial ne distinguait pas ces lignes (pas de seq_num au PDF, et le hash incluait `(date, description, amount, section)` mais pas le cashback_rate ni un compteur). Résultat : 4 lignes du PDF → 1 ligne en DB. Diff : 8 transactions perdues sur les 4 PDFs. **Fix** : ajout d'un `occurrence_index` (compteur par tuple `(date, description, amount, section)` dans le PDF) intégré au hash sous forme `#1`, `#2`... Truncate + replay → **360/360 en DB**, idempotence revalidée. **Leçon** : pour les sources sans identifiant unique par transaction, prévoir un compteur d'occurrences dans la clé de dédup.

**Total Phase 1 hors investissement : 424 transactions de Marc importées (64 bancaires + 360 carte de crédit) sur 4 mois.**

**Soucis anticipés Sprint 3 :**

1. **Date sans année** : `06 12` (= 6 décembre). À résoudre via la date du relevé. Le relevé du 8 janvier 2026 inclut des transactions de décembre 2025 + début janvier 2026. Heuristique : si la date `MM` du PDF > `MM` de la date du relevé, on est sur l'année précédente.
2. **Suffixe `CR`** : crédits suffixés `CR` (paiements, remboursements). Le parser doit détecter et inverser le signe.
3. **Espaces dans les nombres** : `1 000,00CR`, `2 944,21`. À nettoyer (replace " " puis "," → ".").
4. **Virgule comme séparateur décimal** : standard FR/CA, à convertir.
5. **Description avec ville/province collée** : `VICTORIAVILLEQC`. On garde la chaîne complète sans tenter de parser les CC postaux.
6. **2 sections distinctes dans le PDF** : "Transactions courantes" (avec carte) et "Opérations au compte" (paiements à la carte). Format différent. Les deux doivent être importés.
7. **Section "Remises en argent"** : breakdown du cashback par catégorie. Skip pour Sprint 3 (le rate par transaction suffit).
8. **Numéro de compte ≠ numéro de carte** : sur le PDF, le compte = `5598 22** **** 5004`, mais les transactions sont attribuées à une carte précise (`5598 22** **** 5020`). Marc a confirmé : 5004 = compte, 5020 = sa carte principale. Le modèle doit distinguer.
9. **Multi-page** : 4 pages, transactions étalées. pdfplumber doit extraire toutes les pages dans l'ordre.
10. **Pas de seq_num** : pour le hash, on combine `(account_number, transaction_date, posting_date, description, amount_signed)`.
11. **Re-télécharger un relevé** : Desjardins génère le même PDF chaque fois → contenu identique → même hashes → idempotence garantie.
12. **Échec d'extraction pdfplumber** : si pdfplumber casse sur un PDF (rare mais possible), il faut un fallback ou skipper proprement avec log d'erreur.

**Livrable Sprint 3 :** transactions carte de crédit visibles + cashback rate enregistré par transaction.

#### Sprint 4 — Parser PDF Disnat ✅ TERMINÉ

- [x] 4.1 JOURNAL préparé (12 soucis anticipés)
- [x] 4.2 Modèles `InvestmentTransaction` + `InvestmentPosition` créés. Sépare bien transactions (operations + dividendes + frais + transferts) et snapshots mensuels (positions au close du mois).
- [x] 4.3 Migration `77b853f4a31d_phase1_investment_transactions_positions` appliquée.
- [x] 4.4 4 nouveaux endpoints API : POST/GET pour `investment-transactions` et `investment-positions` (idempotent par dedup_hash).
- [x] 4.5 Parser `disnat_pdf.py` : extraction texte + regex. Découpage par sous-compte (`5NFL7A3` CAD, `5NFL7B1` USD). Détection des sections via marqueurs `Activité mensuelle` et `Détails de vos actifs`. Heuristique pour parser les blocs multi-lignes (ex: `TRSF IN` continuation). Gestion des "stop words" qui ressemblent à des tickers (ETC, UCITS, SA, INC...) → re-fusionnés dans la description.
- [x] 4.6 Pipeline `disnat.py` : 1 PDF → 2 streams (transactions + positions) → 2 endpoints différents.
- [x] 4.7 `C:\hub\inbox\disnat\` + 2 PDFs réels copiés (`2026-01-disnat.pdf`, `2026-03-disnat.pdf`).
- [x] 4.8 **Run réel : 2 PDFs, 21 transactions + 25 positions, 0 erreur.**
- [x] 4.9 **2 bugs trouvés et résolus :**
  - **Bug 1 — Amount parsing greedy** : "DEPOT SIPC - CP20465 600,00" était parsé comme amount=465600 car la regex `\d{1,3}(?:\s\d{3})*,\d+` matchait "5 600,00" (5 chiffres + espace + 600 + virgule). Fix : ajout d'un lookbehind `(?:^|(?<=[^\d]))` qui empêche de matcher après un chiffre.
  - **Bug 2 — Symbol pollution par stop-words** : "GOLD BULLION SECS LTD ETC" → symbol="ETC" (faux : ETC = Exchange Traded Commodity, partie du nom). Pareil pour "AMUNDI MSCI WORLD UCITS" → symbol="UCITS". Fix : liste `_NOT_TICKERS` qui re-fusionne ces faux tickers dans la description.
- [x] 4.10 Imperfection mineure non critique : "VISA INC CLASS-A V" → le `V` (vrai ticker 1 caractère) n'est pas extrait car ma regex exige min 2 chars. La description et le montant sont corrects, juste symbol manquant. Acceptable pour MVP.

**Total Phase 1 : 64 transactions bancaires + 360 cartes de crédit + 21 transactions investissement + 25 positions = 470 enregistrements de Marc importés.**

#### Phase 1 fin — Endpoint `/v1/ai/ask` ✅ TERMINÉ

- [x] Module `src/api/v1/ai.py` avec :
  - `GET /v1/ai/ping` : smoke test Ollama (vérifie le modèle répond)
  - `POST /v1/ai/ask` : workflow complet question→SQL→exec→réponse
- [x] Garde-fou SQL : `_validate_sql()` rejette les non-SELECT, les mots-clés interdits (INSERT/UPDATE/DELETE/...), et les tables non whitelistées.
- [x] `statement_timeout = 5000ms` côté Postgres pour limiter les requêtes lentes.
- [x] Schéma DB inclus dans le prompt système + 4 exemples few-shot pour orienter Qwen.
- [x] Test réel : "Combien j'ai dépensé en restaurants en mars 2026 ?" → SQL généré valide → 374,11 CAD trouvés. Le LLM a juste dit "euros" au lieu de "CAD/$" — micro-bug de formulation, ignorable.

**Performance** : Qwen 14B chargé en VRAM, 2 appels par /v1/ai/ask (génération SQL + reformulation), latence totale ~5-10s sur RTX 5080.

**Soucis anticipés Sprint 4 :**

1. **Format radicalement différent du PDF Mastercard** : Disnat = relevé de portefeuille mensuel (pas un journal séquentiel de transactions). 6 pages.
2. **2 sous-comptes par PDF** : `5NFL7A3` (CAD) et `5NFL7B1` (USD). Chacun a sa section. Le parser doit traiter les deux séparément.
3. **3 sections par sous-compte** : Profil (intérêts/dividendes + variation encaisse), Activité mensuelle (transactions du mois), Détails de vos actifs (positions = snapshot fin de mois).
4. **Devises mixtes** : le compte CAD A3 contient aussi des actifs en USD ! (ex: AMUNDI MSCI EM ASIA UCITS ANDXF 180 unités cotées en USD). La colonne `Devise` du tableau identifie la devise du marché.
5. **Pas de transactions dans le compte USD** dans le mois où il n'y a pas eu d'opération (jan 2026) → seulement les transferts initiaux + détails. Le pipeline doit gérer "0 transaction mais N positions".
6. **Tableau "Activité mensuelle"** : colonnes `Date trans | Date règlement | Opération | Quantité | Description | Prix | Montant`. Multi-ligne possible (`TRSF IN` sur ligne séparée du nom du titre).
7. **Tableau "Détails de vos actifs"** : colonnes `Description | Symbole | Quantité | Coût unitaire moyen | Coût comptable | Prix marché | Devise | Valeur marchande | %` + indic + statut. Plus de colonnes que d'autres tableaux.
8. **Symboles boursiers manquants parfois** : ex AMUNDI MSCI WORLD UCITS n'a pas de symbole rempli, GOLD BULLION non plus. Skip ou store as null.
9. **Numéros de ligne mal alignés** : pdfplumber peut couper au milieu si le texte multi-ligne. À tester.
10. **Idempotence positions** : on stocke 1 snapshot par compte par statement_date. Re-import du même PDF = même snapshot. Clé naturelle : `(account_id, statement_date, symbol_or_description)`.
11. **Idempotence transactions** : pas de seq_num. Hash sur `(account, date_trans, date_reg, operation, quantity, description, prix, montant, occurrence)`. Mêmes leçons que Sprint 3.
12. **Statement_date au format différent** : `Au 31 janvier 2026` (pleine lettre) vs Mastercard `08 01 2026`. Regex différente.

**Livrable Sprint 4 :** valeur du portefeuille tracée mois par mois + historique des transferts/achats/ventes/dividendes/frais.

### Après Phase 1

- Phase 1 fin : endpoint `POST /v1/ai/ask` qui prend une question en français, génère SQL via Qwen, valide, exécute, formule la réponse.
- Phase 0 final (deferred) : Cloudflare Tunnel + Access + DuckDNS + vault age+sops + backup restic.
- Phase 2 : Google Maps Timeline.

---

## Décisions techniques au fil de l'eau

### 2026-04-28

- **OneDrive = source de vérité** pour les `*.md` stratégiques (CLAUDE.md, 01-04). `C:\hub\` reçoit des copies de travail. Source : Marc, choix B explicite.
- **Banque unique** : Desjardins (AccèsD + Disnat). 5 comptes, 3 formats d'export différents (CSV, PDF Mastercard, PDF Disnat).
- **Pas de service tiers payant** pour récupérer les données bancaires (Marc abandonne son ancien service payant). Cohérent avec règle 5 "tout gratuit".
- **Pas de scraping AccèsD** (auth forte + risque blocage compte). Exports manuels CSV/PDF only.
- **Disnat n'expose pas de CSV** → on parse les PDF mensuels. Plus complexe mais inévitable.

### Décisions architecturales formelles

→ ADRs dans `hub-docs/decisions/` (à créer au fil de l'eau pour les choix structurants).

---

## Journal des sessions

### Session #1 — 2026-04-28

**État de départ :**
- 5 repos clonés dans `C:\hub\` (hors OneDrive pour éviter conflits de sync)
- `~/.claude/CLAUDE.md` global déjà installé
- Aucun service jamais lancé en vrai
- Aucune feature implémentée (juste les skeletons)

**Actions de la session :**

1. **Audit cross-repos** : lecture des 5 README + 5 CLAUDE.md → compréhension globale du hub.
2. **Copie des 5 docs stratégiques** OneDrive → `C:\hub\` (`CLAUDE.md`, `01_risk_audit.md`, `02_phasing.md`, `03_repo_structure.md`, `04_master_plan.md`) pour que les chemins relatifs `../../CLAUDE.md` des sous-CLAUDE.md résolvent correctement.
3. **Phase 0 (locale) lancée et validée du premier coup :**
   - Génération SECRET_KEY + POSTGRES_PASSWORD via openssl, écriture du `.env` de hub-deploy
   - Pull Ollama : `qwen2.5:14b-instruct` (9 GB) + `nomic-embed-text` (270 MB) — en background parallèle
   - `docker compose up -d --build` : 3 conteneurs healthy (postgres pg16+pgvector, redis 7-alpine, hub-core FastAPI)
   - 10 tests automatisés passent (endpoints `/`, `/v1/health`, `/v1/ready`, `/docs`, `/openapi.json` + connexion psql directe + extension pgvector + prompt Ollama)
   - Performance Ollama : Qwen 14B sur RTX 5080 → **57.6 tokens/sec** après warm-up. Cold-start ~29s (chargement modèle en VRAM).
4. **Discovery Phase 1 banking :**
   - Marc utilise uniquement Desjardins (5 comptes, 2 portails : AccèsD et Disnat).
   - 3 formats d'export identifiés et analysés : CSV compte courant (cp1252), PDF Mastercard (texte propre), PDF Disnat (relevé portefeuille mensuel).
   - 9 fichiers réels téléchargés et inspectés : 3 CSV (janv/fév/mars 2026), 4 PDF Mastercard (janv→avril), 2 PDF Disnat (jan + mars).
   - Confirmé : pas de CSV pour Disnat → parsing PDF inévitable.

**Décisions prises pendant la session :**

- OneDrive reste source de vérité pour les `.md` stratégiques (vs `C:\hub\`).
- Méthode banque retenue : exports manuels (CSV pour AccèsD, PDF pour Mastercard et Disnat).
- Path d'implémentation Phase 1 banking : 4 sprints itératifs (DB → CSV → Mastercard → Disnat).

**À retenir pour Marc :**
- Adresse Disnat (Lévis) ≠ adresse Mastercard (Québec) → probablement un déménagement récent. Pas critique.
- "Transferts reçus" massifs en janvier sur Disnat → migration récente d'un autre courtier (probablement l'ancien service tiers payant). À confirmer.

**Suite prévue :** Sprint 1 dans cette même session.

**Sprint 1 lancé et terminé dans la foulée :**

- Audit hub-core : structure FastAPI/async/SQLAlchemy 2 propre, alembic configuré async, deps complètes (asyncpg + psycopg v3 binary).
- Décision archi mineure : ajout d'un **bind mount** `../hub-core/src:/app/src` + `--reload` dans le compose dev, pour itérer sans rebuild. Sinon chaque modif `.py` aurait nécessité ~30s de rebuild Docker.
- Décision archi mineure : `DateTime(timezone=True)` partout (vs naive). On stocke en `timestamp with time zone` dans Postgres pour éviter le silent-loss de tz info quand on écrit `datetime.now(UTC)` côté Python.
- Modèles : `Account` (institution + account_type + masked_number + currency + actif + timestamps) et `Transaction` (FK account, date, description, debit/credit/balance, métadonnées source pour audit, dedup_hash unique). Carte de crédit et investissement seront des modèles séparés en Sprint 3 et 4.
- Idempotence : `Account` dédupliqué par `(institution, account_number_masked)`, `Transaction` par `dedup_hash` (SHA-256). Les POST sont idempotents — re-jouables sans risque de doublon. Critique pour le watcher du Sprint 2.
- Validation Pydantic : `model_validator` qui interdit "debit ET credit" et "ni debit ni credit" → renvoie 422 explicite.

**Décisions techniques actées dans la session :**
- Bind mount + reload pour le dev (commit dans `hub-deploy/docker-compose.dev.yml`).
- DateTime timezone-aware partout.
- Idempotence par hash côté transaction (pas par contrainte composite naturelle), pour pouvoir étendre facilement aux autres formats (Mastercard, Disnat) qui n'ont pas de seq_num.

**Sprint 2 enchaîné dans la même session #1 :**

- **Architecture choisie** : pas de watcher continu pour Sprint 2. Mode `run-once` qui scan l'inbox + exit. Plus simple à débugger, plus rapide à itérer. Watcher continu (APScheduler ou watchdog) reporté à plus tard si nécessaire.
- **Mode "lancement à la demande"** : `docker compose --profile ingest run --rm hub-ingest`. Le service est sous un profile `ingest` exclu du `compose up` standard. Marc lance la commande quand il a copié de nouveaux CSV.
- **Bind mounts compose** : `../inbox:/var/hub/inbox` et `../raw_events:/var/hub/raw_events` côté host = `C:\hub\inbox` et `C:\hub\raw_events`. Erreur initiale `../../hub/inbox` corrigée avant le build.
- **Création auto des comptes** : le pipeline détecte un nouveau `(transit, account_type)` et POST `/v1/finance/accounts`. Idempotent côté API → pas de doublon même si on rejoue.
- **Bug rencontré et résolu pendant la session (1) — Doublon "Paie ROBOVIC" du 1er janvier** : ma transaction de test du Sprint 1 utilisait un `dedup_hash` fabriqué manuellement qui ne suivait pas la formule du parser. Quand le pipeline a importé le vrai CSV, il a calculé un hash différent → 2 entrées en DB pour la même transaction sémantique. **Résolu** en supprimant la transaction de test (DELETE WHERE dedup_hash = '1f70...'). **Leçon** : pour les tests futurs, utiliser la même formule de hash que le parser, pas inventer.
- **Bug évité — chemins bind mount** : initialement écrit `../../hub/inbox` (qui aurait pointé sur `C:\inbox` au lieu de `C:\hub\inbox`). Corrigé avant le build, identifié par relecture mentale du chemin.
- **Détails sympa observés dans les vraies data de Marc** :
  - Plus grosse opération du trimestre : 7000$ "Virement à EOP" (transfert épargne→courant le 16 mars).
  - Loyer change entre janvier (1039$ pour `9478 5045 Quebec inc`) et mars (1600$ pour `Valerie cameron`) → cohérent avec un déménagement Québec→Lévis confirmé indirectement par les adresses différentes des relevés Disnat (Lévis) vs Mastercard (Québec).
  - Caractères accentués bien décodés (cp1252 → utf-8 → Postgres) : `AccèsD`, `Intérêt sur ET`.

**Fin de session #1.** Phase 0 + Phase 1 Sprints 1 et 2 terminés. À reprendre : Sprint 3 (parser PDF Mastercard, 4 fichiers : janv/fév/mars/avril 2026).

---

### Session #2 — 2026-04-29 (nouveau PC, code-only)

**Contexte** : nouveau PC Windows (`dessin14`), pas l'ancien (`marcr`). Marc a explicitement demandé de **ne rien installer/lancer** localement — uniquement écrire le code qu'il déploiera plus tard sur son vrai PC équipé.

**Mandat** : faire les chunks A, C, D, E (tout sauf B = Phase 3).

**Livrables (62 fichiers, 5 repos)** :

- **A — Docs hub-docs** : `03-data-model.md`, `04-api-contract.md`, `06-security.md`, ADRs 0003 (postgres+pgvector), 0004 (ollama natif), 0005 (cloudflare tunnel vs tailscale), 0006 (age+sops), 0007 (versioning sous-chemin Caddy)
- **C — Frontend** : pages `/finances` (3 onglets banque/carte/invest + filtres), `/search` (chat IA persistant + SQL/rows collapsibles), `/locations` (carte react-leaflet dynamic SSR-off), home wired sur l'API live (`LiveStatCards` + `SpendingChart` recharts), `lib/api.ts` typed pour les 19 routes, `formatCurrency` defaut CAD/`fr-CA`
- **D — Tests + outillage** : ~80 tests pytest (4 parsers + 4 endpoints + SQL validator), 5 CI workflows (1/repo), `clone_all.ps1`, `replay.py` (ADR-0002)
- **E — Infra Phase 0 fin** : `docker-compose.prod.yml` (Postgres pas exposé + Caddy + cloudflared), `cloudflared/README.md` (tunnel + access TOTP), `.sops.yaml` + `init_secrets.ps1` + `decrypt_env.ps1`, vault `secrets/README.md` rewrite, restic backup (3 scripts + README)

**Validation** : reviewer agent indépendant a trouvé **7 vrais bugs**, tous fixés :
1. `backup.ps1` — `_sendNotif` appelé avant définition + Out-File UTF-16 BOM (cassait le restore psql) → réordonné + `[System.IO.File]::WriteAllText(..., UTF8Encoding $false)`
2. `app/search/page.tsx` — `useSearchParams()` casse au build Next 15 sans Suspense → wrappé `<SearchPageInner>` dans `<Suspense>`
3. `tests/test_finance_transactions.py` — comparaison string `"100.00"` casse sur SQLite (renvoie `"100"`) → `Decimal(body["debit"]) == Decimal("100.00")`
4. `docker-compose.prod.yml` — réseau implicite → ajout explicite `networks: default: name: hub_prod`
5. `hub-deploy CI` — `python` pas dispo sur Ubuntu runner → `python3` + `setup-python@v5`
6. `backup.ps1` — `--exclude '*.yaml'` excluait aussi les `.enc.yaml` (perte des secrets dans le snapshot) → `--exclude-file` ciblé `*.tmp + backup/staging`
7. `app/finances/page.tsx` — date filters envoyés à un endpoint qui ne les accepte pas → filtre côté client uniquement, commenté

**Documentation** : `sessions/2026-04-29_session2_summary.md` (récap détaillé) + `sessions/SUITE.md` (roadmap des prochaines étapes : reprise PC équipé → Phase 0 fin → Phase 2 fin → Phase 3 → Phase 4+ → Phase 5).

**Preview frontend** : 4 fichiers HTML statiques dans `_previews/` (Tailwind CDN + lucide SVG inline) reproduisent fidèlement le rendu des 4 pages. Marc peut ouvrir depuis Google Drive sans aucune install.

**Cleanup PC** : 3 fichiers locaux à `C:\Users\dessin14\.claude\` ont été supprimés en fin de session (cf. demande Marc « rien sur ce PC »). Le code projet sur Google Drive (`G:\Mon disque\...`) reste — c'est syncé partout, pas spécifique au PC.

**Fin de session #2.** Code complet pour Phases 0-2. À reprendre : redéployer sur le vrai PC + Phase 0 fin réelle (tunnel + backup) + import Takeout pour finaliser Phase 2.

---

#### Demande de fin Session #2 — Marc redesign brief

**Verbatim Marc** : *« le front end est trop... IA, je veux que ce soit plus beau plus interactif et moins statique, je veux aussi plus de data genre la santé, mes réseaux, et tout ce que tu peux imaginer... aussi genre la sécurité de mes données, qc ou voir que mes données sur le web sont safe ou nulle part »*

3 chantiers nouveaux à prioriser pour les prochaines sessions :
1. **Refonte UI** — moins « AI-générique », plus beau, plus interactif. Le 2ᵉ feedback design Marc — la palette ink + vert n'est toujours pas assez distinctive.
2. **Élargir les sources** — santé (Apple Health, Garmin?), réseaux (Spotify, YouTube, Twitter…), gaming, lecture, browser history.
3. **Module sécurité OSINT** — check breach (HIBP), footprint web, inventaire de comptes, score d'exposition.

Brief complet + 16 questions ouvertes à Marc : [`sessions/2026-04-29_marc_redesign_request.md`](sessions/2026-04-29_marc_redesign_request.md).
Roadmap mise à jour : [`sessions/SUITE.md`](sessions/SUITE.md) (Étape 0 = discovery UI ; Étape 1 = refonte ; Étape 5 = santé/réseaux/sécu).

À traiter en début de Session #3.

---

### Session #3 — 2026-04-29 (réponses discovery + plan verrouillé)

**But :** Recevoir et documenter les réponses aux 16 questions UI/scope. Verrouiller les décisions. Mettre à jour la roadmap.

**Réponses Marc (verbatim + décodage) :** [`sessions/2026-04-29_marc_answers_discovery.md`](sessions/2026-04-29_marc_answers_discovery.md)

#### Décisions verrouillées

| Décision | Valeur |
|---|---|
| Palette | Dark-only (garder ink + vert) |
| Layout | Refonte complète — moins SaaS-grid, plus modulaire/personnel |
| Photos dans UI | NON |
| Easter eggs / humour | NON |
| Animations | Tout (framer-motion) : transitions, stagger, hover lift, realtime pulse |
| Drag-drop | Oui + plus (resize, pin/unpin, persistance layout) |
| Realtime | SSE (`GET /v1/events/stream`) |
| Mode focus | Oui — widget click = expand fullscreen |
| Santé | Garmin + Google Fit — toutes les métriques disponibles |
| Streaming | YouTube, YouTube Music, Netflix, Disney+, Prime Video, Crunchyroll |
| Gaming | Steam + Xbox |
| Browser/Dev | Chrome history + GitHub |
| Sécurité | Scope complet (HIBP + footprint + inventaire + data brokers + suppression) |
| Coût | Tout gratuit — aucune exception |
| Lecture | NON (Kindle, Pocket, etc.) |
| Nouveau module | **Module Suppression** — tracker demandes de suppression données en ligne (PIPEDA + Loi 25 QC) |

#### Note importante — Google Drive

Marc a demandé que tout soit sauvegardé sur son Drive. C'est déjà le cas :
- Le projet vit dans `G:\Mon disque\...` → synchronisé automatiquement sur Google Drive
- + Tout est pushé sur GitHub (MoKarade) en fin de session

Double sauvegarde à chaque session = zéro risque de perte.

#### Roadmap mise à jour

`sessions/SUITE.md` entièrement mis à jour avec le plan concret pour les étapes 1-10.
La prochaine session commence par **Étape 1 Sprint A** (système animations + Widget conteneur + LayoutProvider).

---

### Session #4 — 2026-04-29 (Sprint A livré + CI fixé)

**But :** Sprint A UI redesign + corriger les run fails GitHub Actions signalés par Marc.

#### Sprint A — Code livré

Nouveaux fichiers hub-frontend :
- `lib/motion.ts` — variants framer-motion partagés (fadeIn, stagger, staggerItem, pageTransition, scaleIn)
- `lib/layout-context.tsx` — LayoutProvider + useLayout hook (taille, pin, visible, ordre, localStorage)
- `components/widget.tsx` — conteneur universel avec header drag handle, pin, size cycle, focus
- `components/focus-modal.tsx` — overlay plein écran AnimatePresence
- `components/providers.tsx` — wrapper client pour AnimatePresence + LayoutProvider
- `app/template.tsx` — transition de page Next.js App Router

Fichiers modifiés : `app/layout.tsx`, `app/page.tsx`, `components/live-stat-cards.tsx`, `tailwind.config.ts`, `app/globals.css`

#### CI fixes (run fails GitHub Actions)

**Cause racine 1 :** `actions/setup-node` avec `cache: 'npm'` → requiert un `package-lock.json` absent (npm install jamais tourné). Erreur : "Dependencies lock file is not found".

**Cause racine 2 :** `npm ci` → même problème (nécessite le lockfile).

**Cause racine 3 :** `React.ReactNode` et `React.ComponentType` utilisés sans `import React` dans 9 fichiers → erreur TypeScript strict.

**Fixes appliqués :**
- CI : `cache: 'npm'` supprimé de `actions/setup-node`
- CI : `npm ci` → `npm install --legacy-peer-deps`
- CI : `.eslintrc.json` ajouté + `ESLINT_USE_FLAT_CONFIG: 'false'` env var
- TypeScript : `React.ReactNode/ComponentType` → imports nommés dans 9 fichiers
- Bug #2 Sprint A : `AnimatePresence` + Fragment → deux `AnimatePresence` séparés
- Bug #3 Sprint A : race condition localStorage → `hasHydrated` ref
- Bug #4 Sprint A : `AnimatePresence` absent dans `providers.tsx` → ajouté
- Bug #5 Sprint A : `new Date()` Server Component → `export const dynamic = 'force-dynamic'`

**Commits poussés :** e09e904 + 52ce5a7 sur hub-frontend/main.

> **Note :** CI encore partiellement rouge après Session #4 (run #4 = 52ce5a7 échoue encore). La suite du débugging CI se fait en Session #5.

---

### Session #5 — 2026-04-29 (CI 100% vert sur les 3 repos)

**But :** Terminer de corriger tous les runs GitHub Actions en échec. Marc a vu les runs rouges et a demandé de tout passer au vert.

**Résultat :** 3/3 repos verts sur leur dernier commit.

| Repo | Run final | Commit | Statut |
|---|---|---|---|
| hub-frontend | #7 | `7f132f6` | ✅ 2m 4s |
| hub-core | #5 | `8477fcc` | ✅ 28s |
| hub-ingest | #4 | `2569fd2` | ✅ 41s |

#### Corrections hub-frontend (runs #5, #6, #7)

**Run #5 → fix apostrophe + links internes**
- `L'endpoint` non échappé dans du texte JSX → `L&apos;endpoint` (`insight-list.tsx:18`)
- 6 `<a href="/...">` internes → `<Link href="...">` dans `app/page.tsx` (règle `next/core-web-vitals`)
- Commit : `7be230a`

**Run #6 → retry ECONNRESET npmjs.org**
- `npm install` échouait aléatoirement avec `ECONNRESET` depuis les runners GitHub (problème réseau transitoire vers le registry npm)
- Fix : boucle bash 3 tentatives avec `exit 0`/`exit 1` explicites + `fetch-retries 5` dans la config npm
- Commit : `bc96548`

**Run #7 → comparaison string/number TypeScript**
- `balance > 0` → TypeScript strict erreur : `balance_after` est sérialisé en `string | null` par FastAPI (PostgreSQL `NUMERIC` → JSON string). Comparaison `string > 0` silencieuse mais fausse
- Fix : `parseFloat(balance) > 0` dans `components/live-stat-cards.tsx`
- Commit : `7f132f6` ✅

#### Corrections hub-core (runs #2, #3, #4, #5)

**Run #2 → ruff E501 (lignes longues)**
- 7 fichiers avec des lignes > 100 caractères → reformattés
- Commit : `d1ec3d0`

**Run #3 → ruff B008 + bugs _validate_sql + ruff UP043/N806/I001**
- `B008` : 25 violations FastAPI légitimes (`Query(...)`, `Depends(...)` comme default args) → `ignore = ["B008"]`
- `I001` : ruff traitait `src` comme third-party → `known-first-party = ["src"]`
- `UP043` : `AsyncGenerator[X, None]` → `AsyncGenerator[X]` dans `session.py` et `conftest.py`
- `N806` : `SessionLocal` (PascalCase dans une fonction) → renommé `session_factory` + `ignore = ["N806"]`
- **Bug _validate_sql #1** : `WITH recent AS (...) SELECT * FROM recent` → erreur "table non autorisée: recent" — les CTEs n'étaient pas extraits. Fix : extraction des noms CTE via `re.finditer(r"\bWITH\s+(\w+)\s+AS\b")` + union dans `allowed`
- **Bug _validate_sql #2** : `INSERT INTO foo` levait "doit commencer par SELECT" au lieu de "mot-clé interdit". Fix : check forbidden keywords EN PREMIER, avant le check SELECT/WITH
- Commit : `c2ac9fe`

**Run #4 → ruff format --check**
- 6 fichiers hub-core non conformes au formatter ruff → `ruff format` appliqué localement (Python 3.14 + ruff installés sur la machine)
- Commit : `b9c6f14`

**Run #5 → tests qui testaient l'ancien comportement buggy**
- Après le fix d'ordre des checks, `UPDATE accounts SET nickname = 'x'` et `VACUUM` (qui sont dans la liste des mots-clés interdits) levaient désormais "interdit" au lieu de "SELECT" → 2 tests de `TestInvalidNonSelect` cassaient
- Fix : tests mis à jour avec `SET search_path = public` et `pg_sleep(1)` — des SQL non-SELECT qui ne sont PAS dans la liste interdite
- Commit : `8477fcc` ✅

#### Corrections hub-ingest (runs #2, #3, #4)

**Run #2 → ruff E741 + E501 + datetime.utcnow**
- `E741` : variable `l` ambiguë dans 3 list comprehensions de `disnat_pdf.py` → renommée `line`
- `E501` : lignes longues reformattées
- `datetime.utcnow()` déprécié Python 3.12+ dans `connectors/base.py` → `datetime.now(UTC)`
- Commit : `1083645`

**Run #3 → ruff format --check**
- 7 fichiers non conformes : `disnat_pdf.py`, `disnat.py`, `mastercard.py`, `dump_pdf_tables.py`, `replay.py`, `test_disnat_pdf.py`, `test_google_takeout_timeline.py`
- `ruff format` appliqué localement
- Commit : `db03081`

**Run #4 → fixture CSV date format**
- `tests/fixtures/desjardins_minimal.csv` utilisait `2026-01-01` (tirets) mais `_parse_date()` fait `raw.split("/")` et attend des slashes (`YYYY/MM/DD` = format réel Desjardins)
- Toutes les lignes du CSV levaient `ValueError` silencieusement attrapée → 0 lignes parsées → 5 tests `IndexError` (`test_parses_all_valid_rows`, `test_eop_debit_row`, `test_eop_credit_row`, `test_accents_decoded_via_cp1252`, `test_savings_row_separated`)
- Fix : dates corrigées en `2026/01/01` dans le fixture
- Commit : `2569fd2` ✅

#### Bugs réels corrigés (impact prod, pas seulement CI)

1. **`_validate_sql` CTE** : les requêtes `WITH xxx AS (...) SELECT * FROM xxx` étaient rejetées en prod avec "table non autorisée: xxx". Correctement réparé.
2. **`_validate_sql` ordre** : `INSERT INTO ...` levait le mauvais message d'erreur en prod ("doit commencer par SELECT" au lieu de "mot-clé interdit"). Réparé.
3. **`balance > 0` TypeScript** : comparaison silencieuse string/number dans le dashboard → résultat toujours faux si balance = chaîne positive. Réparé.

#### Audit qualité annexe (4 agents parallèles)

En parallèle du debug CI, 4 agents d'audit ont scanné les 3 repos pour trouver d'autres bugs potentiels. Résultat : les bugs ci-dessus + confirmation que le reste du code est sain.

**Commits de cette session (hub-frontend) :** `7be230a`, `bc96548`, `7f132f6`
**Commits de cette session (hub-core) :** `d1ec3d0`, `c2ac9fe`, `b9c6f14`, `8477fcc`
**Commits de cette session (hub-ingest) :** `1083645`, `db03081`, `2569fd2`

**Fin de session #5.** Tout vert. Prochaine étape : Sprint B (SSE realtime + drag-drop + focus mode).

---

### Session #6 — 2026-04-29 (Sprint B livré)

**But :** Implémenter Sprint B — SSE realtime + dnd-kit drag-drop + persistance de l'ordre des widgets.

**Résultat :** Sprint B code-complete et poussé.

| Repo | Commit | Description |
|---|---|---|
| hub-frontend | `473aa32` | 7 fichiers (3 nouveaux + 4 modifiés) |
| hub-core | déjà poussé (session #5) | `events.py` + `finance.py` + `locations.py` modifiés |

#### hub-core (poussé en session #5, activé en Sprint B)

- **`src/api/v1/events.py`** (NOUVEAU) : broadcaster asyncio.Queue. Une Queue par client SSE connecté, heartbeat 30s via `asyncio.TimeoutError`, nettoyage auto des clients déconnectés (`QueueFull` → dead set). `GET /v1/events/stream` retourne `StreamingResponse` avec `media_type=text/event-stream`.
- **`src/api/v1/finance.py`** : `await broadcast("new_transaction", {...})` après chaque `create_transaction` et `create_credit_card_transaction`.
- **`src/api/v1/locations.py`** : `await broadcast("new_location", {...})` après chaque `create_location_point`.
- **`src/api/v1/__init__.py`** : `router.include_router(events.router)` ajouté.

#### hub-frontend (Sprint B)

**Nouveaux fichiers :**

- **`lib/use-event-source.ts`** : hook SSE. Connecte `EventSource` à `GET /v1/events/stream`, écoute les events nommés (`connected`, `new_transaction`, `new_location`, `stats_update`). Reconnexion auto après 5s. `status: SseStatus` exposé. `cancelledRef` pour éviter les setState après démontage.

- **`components/widget-grid.tsx`** : grille sortable dnd-kit. `DndContext` + `SortableContext` (rectSortingStrategy) en 3 colonnes CSS. `SortableItem` : appelle `useSortable({ id })`, applique `CSS.Transform`, injecte `dragListeners` + `dragAttributes` dans le Widget via `React.cloneElement`. Col-span automatique selon `getWidget(id).size` (sm/md=1, lg=2, xl/full=3). PointerSensor avec `distance:8` pour éviter les faux déclenchements au scroll.

- **`components/dashboard-grid.tsx`** : shell client de la home. Utilise `useEventSource(SSE_URL)`, déclenche `pulseFinances=true` pendant 2s sur chaque event `new_transaction`, passe `pulse={pulseFinances}` au widget finances. Contient les 7 widgets (ai-search, finances-overview, insights, spending-chart, locations, health, apps) passés à `WidgetGrid`.

**Fichiers modifiés :**

- **`lib/layout-context.tsx`** : action `REORDER` ajoutée (type + reducer case — réassigne `order` par index). `reorder(ids)` et `getSortedIds(knownIds)` ajoutés au Provider et au context value. `getSortedIds` trie : pinned en premier, puis par `order`.

- **`components/widget.tsx`** : props `dragListeners` et `dragAttributes` ajoutés. GripVertical enveloppé dans un `<span>` qui reçoit les deux spreads + `touch-none` pour mobile.

- **`app/page.tsx`** : simplifié en Server Component pur (salutation + header + `<DashboardGrid />` + `<HubStatus />`). Toute la logique widgets et SSE est déléguée à `dashboard-grid.tsx`.

- **`package.json`** : dépendances dnd-kit ajoutées (`@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`).

#### Design decisions de ce sprint

- **`React.cloneElement`** pour injecter les drag listeners : évite de coupler Widget à dnd-kit directement. Widget reste réutilisable hors contexte drag-drop.
- **`distance:8` PointerSensor** : seuil minimal pour ne pas activer le drag lors des clics sur les boutons du header.
- **`DashboardGrid` client component** : isoler le hook SSE (`useEventSource`) de `page.tsx` Server Component, sans perdre les bénéfices SSR du shell.
- **`getSortedIds` + `reorder`** : l'ordre de tri est toujours recalculé côté `LayoutContext`, pas dupliqué dans `WidgetGrid`.

**Fin de session #6.** Sprint B livré. Prochaine étape : Sprint C (reskinage de toutes les pages dans le nouveau layout).

---

### Session #7 — 2026-04-29 (Sprint C livré — refonte UI Google Analytics dark)

**Contexte :** Marc a vu le mockup Sprint A et a dit "il faut beaucoup retravailler le frontend". Discovery session de 18 questions/réponses → brief design détaillé sauvegardé dans `sessions/sprint-c-design-brief.md`.

**Direction visuelle verrouillée :**
- Style Google Analytics dark mode (data-dense mais chirurgical)
- Plus épuré, hiérarchie typographique forte
- Vert `#5cdb95` UNIQUEMENT pour valeurs positives, plus blanc/neutre par défaut (Marc : "trop crypto bro")
- Sidebar collapsible (icône+texte ↔ icône seule)
- Mobile + desktop responsive
- Progressive disclosure : épuré en surface, riche en profondeur

**Sprint C livré en 5 sous-sprints :**

| Sous-sprint | Commit | Description |
|---|---|---|
| C1 — Fondations | `b857b53` | Palette `data` Tailwind + utilitaires CSS GA + sidebar collapsible |
| C2 — Dashboard | `da45084` | KPI strip prominent + RecentTransactions + layout 3-col |
| C3 — Finances | `6dfc430` | SummaryRow GA-style + couleurs sémantiques `data-positive`/`data-negative` |
| C4 — Search | `eb95328` | SQL généré visible par défaut (plus dans `<details>` fermé) |
| C5 — Locations | `eb95328` | StatTile GA-style |

#### C1 — Fondations design (commit `b857b53`)

- **`tailwind.config.ts`** : ajout namespace `data` (`positive` `#34a853`, `negative` `#ea4335`, `neutral` `#e6ecf2`, `muted` `#8b95a3`). Vert/rouge Google moins saturés que l'ancien accent.
- **`globals.css`** : retrait des gradients `radial-gradient` du body (trop "designer", GA est plat). Ajout utilitaires `.metric` `.metric-lg` `.metric-label` `.metric-delta` `.ga-card` `.ga-card-hover` `.section-title` + classes data sémantiques.
- **`sidebar.tsx`** : refonte collapsible avec `localStorage('hub-sidebar-collapsed-v1')`. Mode étendu (264 px, icône+label+sections titrées) ↔ mode réduit (60 px, icône seule, tooltip Radix au hover, séparateur subtil entre sections). Toggle en bas avec `PanelLeftClose`/`PanelLeftOpen` icons.

#### C2 — Dashboard rework (commit `da45084`)

- **`live-stat-cards.tsx`** : KPI strip refondu Google Analytics. Plus de vert sur valeurs neutres — vert UNIQUEMENT si solde positif (sémantique). Hiérarchie typo forte (`.metric-lg` + `.metric-label`). Bordures subtiles (`.ga-card`). Stagger animation conservé, retrait du `whileHover` lift.
- **`recent-transactions.tsx`** (NOUVEAU) : table compacte mergeant transactions de compte courant + carte de crédit, triées par date desc. Ligne par transaction avec icône direction (`ArrowDownLeft` crédit / `ArrowUpRight` débit), date MM-DD compacte, description tronquée, montant aligné à droite avec couleur sémantique (`data-positive` si crédit). Skeleton loader si pas de data. `noPadding` pour pleine largeur.
- **`dashboard-grid.tsx`** : nouveau layout 5-rows :
  - Row 1 : KPI strip (full, hors WidgetGrid pour prominence GA)
  - Row 2 : ai-search (full)
  - Row 3 : spending-chart (lg=2col) + insights (sm=1col)
  - Row 4 : recent-transactions (lg=2col) + locations (sm=1col)
  - Row 5 : health (full)
  - Row 6 : apps (full)
  - Pulse SSE migré du finances widget vers recent-transactions (plus pertinent).

#### C3 — Finances semantic colors (commit `6dfc430`)

- `SummaryRow` refonte GA : `.ga-card` + `.metric` + `.metric-label`.
- 5 emplacements convertis : SummaryRow banking + CC, table banking amount cell, table CC amount cell, table investments amount cell. `text-accent`/`text-danger` → `data-positive`/`data-negative` sémantiques.

#### C4 — Search SQL visible (commit `eb95328`)

- Le bloc SQL généré était dans `<details>` fermé. Maintenant en `.ga-card` ouvert par défaut, header `.metric-label` avec icône Code2.
- Le résultat brut reste collapsable (verbeux, optionnel).
- Retrait du gradient `bg-gradient-to-br` du panel input → simple `.ga-card`.
- `ai-search-card` : retrait du gradient designer + ring-accent du panel d'autour.

#### C5 — Locations GA-style (commit `eb95328`)

- `StatTile` refonte : `.ga-card` + `.metric` + `.metric-label`.
- "Marche" passe de `text-accent` → `data-positive` (sémantique : activité saine).
- Heatmap différée (nécessite `leaflet.heat` plugin, hors scope C5 minimal).

#### Fichiers touchés en Sprint C

```
hub-frontend/
├── app/
│   ├── finances/page.tsx       (C3)
│   ├── locations/page.tsx      (C5)
│   ├── search/page.tsx         (C4)
│   └── globals.css             (C1)
├── components/
│   ├── ai-search-card.tsx      (C4)
│   ├── dashboard-grid.tsx      (C2)
│   ├── live-stat-cards.tsx     (C2)
│   ├── recent-transactions.tsx (C2 — NOUVEAU)
│   └── sidebar.tsx             (C1)
└── tailwind.config.ts          (C1)
```

#### Reste à faire (post-Sprint C)

- Mobile responsive (sidebar hamburger)
- Heatmap location (leaflet.heat)
- Donut chart spending breakdown (recharts PieChart) — necessite categorisation NLP des descriptions
- Mode focus pleine page sur les pages secondaires (actuellement seulement sur les widgets de la home)
- Insights réels (Phase 4+) — pour l'instant placeholder

**Fin de session #7.** Sprint C livré sur tous les fronts (5/5). Frontend nettement plus Google-Analytics maintenant. Prochaine étape : déploiement réel sur le PC (Étape 2 : Docker + Ollama + GPU) ou ajustements UI selon le retour de Marc en voyant la vraie app tourner.
