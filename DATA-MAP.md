# 🗺️ DATA-MAP — Toutes mes données, un seul document

> **Source de vérité unique** sur où sont mes data, comment elles arrivent dans le hub, et ce que je dois exporter/importer.
>
> **Mis à jour automatiquement** par Claude à chaque ajout/connexion.

**Dernière mise à jour** : 2026-04-30

---

## 📊 Vue rapide

| Source | Statut | Méthode | Fréquence | Phase |
|---|---|---|---|---|
| 💳 Banking Desjardins (CSV) | ✅ Code OK | Manuel CSV/PDF | Mensuel | Phase 1 |
| 💳 Mastercard Desjardins | ✅ Code OK | PDF parser | Mensuel | Phase 1 |
| 📈 Disnat (placements) | ✅ Code OK | PDF parser | Mensuel | Phase 1 |
| 📍 Google Timeline | 📋 Attente data | Takeout ZIP | One-shot + delta | Phase 2 |
| 📧 Gmail | 📋 OAuth prêt | OAuth API | Auto continuous | Phase 3 |
| 📸 Google Photos | 📋 OAuth prêt | Takeout ZIP | One-shot + delta | Phase 3 |
| 📁 Google Drive | 📋 OAuth prêt | OAuth API | Auto | Phase 3 |
| 🔐 Mots de passe Google | 📋 À implémenter | Export CSV manuel | Mensuel | Phase 4 |
| 🛡️ HIBP scan masse | 📋 À implémenter | Client-side k-anonymity | À la demande | Phase 4 |
| 📅 Google Calendar | 📋 OAuth prêt | iCal/OAuth | Auto | Phase 5 |
| 💪 Garmin Connect | 📋 À implémenter | garmin-connect-py | Quotidien | Phase 5 |
| 💪 Google Fit | 📋 OAuth prêt | OAuth API | Auto | Phase 5 |
| 📄 Documents PDF | 📋 À implémenter | Watch dossier | Continuous | Phase 5 |
| 👥 Google Contacts (People) | 📋 OAuth prêt | OAuth API | Hebdo | Phase 5+ |
| ✅ Google Tasks | 📋 OAuth prêt | OAuth API | Hebdo | Phase 5+ |
| 📺 YouTube historique | 📋 OAuth prêt | OAuth API | Hebdo | Phase 5+ |
| 🏃 Strava | 📋 À implémenter | OAuth | Auto | Phase 5+ |
| 🎵 Spotify | 📋 À implémenter | OAuth | Auto | Phase 6+ |

**Légende** : ✅ Connecté · 🔄 En cours · 📋 Code prêt mais data pas encore importée

---

## 🏠 Où vit ta data (vue physique)

```
TON PC (l'autre, où tournera Docker)
│
├── C:\Users\dessin14\.hub-secrets\         ← clés age (privées, jamais Drive)
│   └── age-key.txt                          (déchiffre le vault)
│
├── Volume Docker postgres_data              ← TOUTES les données structurées
│   ├── Tables: accounts, transactions
│   ├── Tables: location_points
│   ├── Tables: oauth_tokens (chiffrés Fernet)
│   └── (futur) emails, photos, calendar_events, health_metrics, etc.
│
├── G:\...\Hub perso\inbox\                  ← drop zone des fichiers à ingérer
│   ├── desjardins/         (CSV banque)
│   ├── mastercard/         (PDF crédit)
│   ├── disnat/             (PDF placements)
│   ├── google-timeline/    (Records.json du Takeout)
│   ├── google-photos/      (Takeout ZIP)
│   ├── passwords/          (CSV export Google Password Manager)
│   └── documents/          (PDFs divers)
│
├── G:\...\Hub perso\raw_events\             ← event sourcing (immutable)
│   └── <source>/<date>/<timestamp>.json     (dump avant transformation)
│
└── G:\...\Hub perso\hub-secrets-vault.age   ← secrets chiffrés (Drive sync OK)

CLOUD (sources externes)
│
├── Google account (marc.richard4@gmail.com)
│   ├── Gmail messages              → Phase 3 OAuth gmail.readonly
│   ├── Photos library              → Phase 3 OAuth + Takeout
│   ├── Drive files                 → Phase 3 OAuth drive.readonly
│   ├── Calendar events             → Phase 5 OAuth calendar.readonly
│   ├── Fit (sommeil/activité)      → Phase 5 OAuth fitness.*.read
│   ├── Contacts (People)           → Phase 5+ OAuth contacts.readonly
│   ├── Tasks                       → Phase 5+ OAuth tasks.readonly
│   ├── YouTube history             → Phase 5+ OAuth youtube.readonly
│   ├── Maps Timeline               → Takeout uniquement (pas d'API)
│   └── Password Manager            → Export CSV manuel uniquement
│
├── Desjardins                      → CSV/PDF manuel
├── Garmin Connect                  → garmin-connect-py (scrape officieux)
└── Strava (futur)                  → OAuth Strava
```

---

## 🔌 Comment chaque source fonctionne

### 💳 Phase 1 — Banking (Desjardins) ✅ CODE OK

**Source** : compte Desjardins de Marc

**À exporter / importer** :
1. **Compte courant + épargne** : AccèsD → Téléchargement → format **CSV** → drop dans `inbox/desjardins/`
2. **Mastercard Desjardins** : AccèsD → Relevé mensuel → **PDF** → drop dans `inbox/mastercard/`
3. **Disnat (placements)** : Disnat → Relevés → **PDF** → drop dans `inbox/disnat/`

**Fréquence recommandée** : 1× par mois (le 1er, après réception relevés)

**Comment le hub traite** :
- `hub-ingest` watch les dossiers inbox
- Parser CSV/PDF → dump raw dans `raw_events/`
- POST vers `/v1/finance/transactions` (idempotent par dedup_hash SHA-256)
- Tables Postgres : `accounts`, `transactions`, `credit_card_transactions`, `investment_transactions`, `investment_positions`

**État actuel** : 470 transactions importées en test (CSV + Mastercard + Disnat). Sur l'autre PC, le code marche.

---

### 📍 Phase 2 — Localisation (Google Maps Timeline) 📋

**Source** : Google Maps Timeline (historique GPS)

**À exporter** :
1. Aller sur https://takeout.google.com
2. Tout désélectionner sauf **"Localisation"**
3. Format : **JSON**, fréquence "Une seule fois", taille max 50 GB
4. Demander → Google envoie un mail avec le ZIP (peut prendre 1-24h)
5. Décompresser → trouver `Records.json` (peut faire 100MB-1GB)
6. Drop dans `inbox/google-timeline/`

**Fréquence** : 1× tous les 3-6 mois (delta uniquement)

**Comment le hub traite** :
- Parser streamé via `ijson` (sinon OOM kill sur 1GB JSON)
- Filtre `accuracy > 100m`, sample 1 point / 30 sec
- Extraction `activity_type` (driving/walking/cycling/still)
- Conversion latE7/lngE7 → Decimal lat/lng 7 décimales
- Idempotent : hash SHA-256 sur `(timestamp_ms, latE7, lngE7)`
- Table : `location_points`

**État actuel** : code complet, attend le ZIP de Marc.

---

### 📧 Phase 3 — Gmail 📋

**Source** : compte Gmail `marc.richard4@gmail.com`

**À exporter / importer** :
- **Aucun export manuel** ! OAuth fait tout en continu.
- Page `/settings` → bouton **"Connecter Gmail"** → consent Google → tokens chiffrés en DB

**Fréquence** : sync incrémentale via History API (toutes les heures par cron)

**Scopes** : `gmail.readonly` uniquement (lecture, jamais d'envoi)

**Comment le hub traite** :
- OAuth refresh token automatique
- Worker fetch nouveaux messages depuis dernier `historyId`
- Parsing MIME (corps + pièces jointes)
- Indexation full-text Postgres + embeddings nomic
- Table : `emails` + `email_attachments`

**Privacy** : Gmail garde l'original, le hub indexe une copie locale chiffrée. Read-only OAuth = aucun impact sur ton inbox.

---

### 📸 Phase 3 — Google Photos 📋

**Source** : Google Photos (compte Marc)

⚠️ **API limitée depuis 2025** : Photos Library API ne donne accès qu'aux albums créés via app, pas ton historique complet.

**À exporter / importer** :
1. **Pour l'historique complet** : https://takeout.google.com → "Google Photos" → ZIP
2. **Pour les nouveaux** : OAuth Photos Library API (incrémental)

**Fréquence** : Takeout 1× par 6 mois + OAuth continuous

**Comment le hub traite** :
- Décompression ZIP → dossier `inbox/google-photos/<album>/`
- Génération thumbnails (sharp)
- Extraction EXIF (date, GPS, appareil photo)
- Embeddings CLIP locaux (recherche par contenu visuel)
- Table : `photos` + vecteurs pgvector

---

### 📁 Phase 3 — Google Drive 📋

**Source** : Drive complet (Mon Drive + partagés)

**À exporter / importer** :
- OAuth fait tout : `/settings` → "Connecter Drive"

**Fréquence** : sync continuous via Changes API

**Scopes** : `drive.readonly`

**Comment le hub traite** :
- Liste des fichiers + métadonnées (nom, taille, date, type, partages)
- Téléchargement local optionnel (sinon juste l'index)
- Indexation full-text pour les Docs Google + PDFs
- Table : `drive_files`

---

### 🔐 Phase 4 — Mots de passe Google + scan HIBP 📋

**Source** : Google Password Manager (https://passwords.google.com)

⚠️ **Pas d'API publique** — export CSV manuel uniquement.

**À exporter / importer** :
1. Aller sur https://passwords.google.com
2. Roue dentée (en haut à droite) → **"Exporter les mots de passe"**
3. Authentification (mot de passe Windows ou Google)
4. Télécharge `Google Passwords.csv`
5. Drop dans `inbox/passwords/`

**Fréquence** : 1× par mois (ou dès suspicion de compromission)

**Comment le hub traite** :
- Parse CSV (5 colonnes : name, url, username, password, note)
- **Tout reste côté client** : SHA-1 calculé dans le browser
- Pour chaque mdp : k-anonymity HIBP (envoie 5 chars du hash)
- Affiche table : URL · username · pwned count · date scan
- Tri compromis en premier
- **Le password lui-même n'entre JAMAIS en DB ni n'est envoyé au backend**

**Phase 5+ extension** :
- Bouton "Régénérer ce mdp" → génère 32 chars random
- Update CSV → re-import dans Google Password Manager (manuel ou Chrome ext)
- Update site (Playwright auto pour les flows standards)

---

### 📅 Phase 5 — Google Calendar 📋

**Source** : Calendar `marc.richard4@gmail.com`

**À exporter / importer** : OAuth `calendar.readonly`

**Fréquence** : sync continuous

**Comment le hub traite** :
- Liste des événements (passés + à venir)
- Cross-reference avec localisation (étais-tu vraiment à ce meeting ?)
- Cross-reference avec dépenses (resto le jour de ce date)
- Table : `calendar_events`

---

### 💪 Phase 5 — Santé (Garmin + Google Fit) 📋

**Source A : Garmin Connect** (montre Marc)
- Pas d'API officielle → bibliothèque [garmin-connect-py](https://github.com/cyberjunky/python-garminconnect)
- Login email + password (stocké dans vault)
- Fetch quotidien : sommeil, fréquence cardiaque, pas, calories, VO2max, courses GPS

**Source B : Google Fit** (smartphone)
- OAuth `fitness.*.read` (activity, body, sleep, location)
- Sync continuous

**Source C : Apple Health** (si applicable)
- Export XML manuel depuis l'app iOS
- Drop dans `inbox/health/`

**Comment le hub traite** :
- Métriques quotidiennes uniformisées (peu importe la source)
- Table : `health_metrics` + `workouts` (séances)
- Cross-reference avec finances (« le sommeil est moins bon les soirs où je dépense plus »)

---

### 📄 Phase 5 — Documents PDF 📋

**Source** : Drag-drop manuel ou watch d'un dossier

**À importer** : drop dans `inbox/documents/`

**Fréquence** : à mesure (factures, contrats, relevés)

**Comment le hub traite** :
- OCR si scan (tesseract)
- Extraction texte si PDF natif (pdfplumber)
- Classification LLM (Qwen) : `contrat | facture | relevé | impôt | autre`
- Indexation full-text Postgres
- Table : `documents`

---

### 📺 Phase 5+ — Autres Google services (Tasks, YouTube, People) 📋

OAuth déjà préparé. Activation à la demande dans `/settings`.

| Service | Scope | Use case |
|---|---|---|
| Tasks | `tasks.readonly` | Liste tes todos pour insights productivité |
| YouTube | `youtube.readonly` | Historique de visionnage → recommandations |
| People (contacts) | `contacts.readonly` | Carnet d'adresses pour cross-ref emails/calendar |

---

### 🏃 Phase 5+ — Strava 📋

**Source** : compte Strava

**À setup** :
1. Créer un app Strava : https://www.strava.com/settings/api
2. Récupérer `client_id` + `client_secret` → ajouter au vault
3. OAuth flow standard → tokens en DB chiffrés

**Comment le hub traite** :
- Activités sportives (run/bike/etc) avec GPS, fréquence cardiaque, allure
- Cross-reference avec Garmin (déduplication)

---

### 🎵 Phase 6+ — Spotify 📋

OAuth Spotify : https://developer.spotify.com/dashboard

**Use case** : insights musique, time spent, mood vs activité.

---

## 🗂️ Récap : qu'est-ce que je dois actuellement faire ?

### Aujourd'hui (rien d'urgent)
✅ Tout est en place côté code. Le hub tourne en local sur ton autre PC après `start_hub.ps1`.

### Quand tu te sens prêt (par ordre de priorité)

1. **Sur l'autre PC** : install Docker Desktop, lancer `start_hub.ps1` → backend + DB live
2. **Banking** : drop tes CSV/PDF récents dans `inbox/` → 470 transactions de test deviennent réelles
3. **Localisation** : commande un Takeout Timeline (1-24h d'attente) → drop le ZIP
4. **Gmail / Photos / Drive / Calendar** : `/settings` → click "Connecter" → consent Google
5. **Mots de passe Google** : export CSV mensuel → check compromis HIBP
6. **Santé** : credentials Garmin dans vault + connect Google Fit OAuth

---

## 🔐 Privacy & sécurité (résumé)

- **Local d'abord** : aucune donnée ne quitte ton PC sauf backups chiffrés (Drive)
- **Read-only** partout : tous les OAuth scopes sont en lecture seule (jamais d'envoi/modif côté Google)
- **Tokens chiffrés** : access_token + refresh_token via Fernet (clé dérivée de SECRET_KEY)
- **Mots de passe scan HIBP** : SHA-1 client-side + k-anonymity (le password lui-même n'est jamais envoyé)
- **Vault age** : tous les secrets externes (clés API, OAuth secrets) chiffrés sur Drive
- **Backups** : `restic` chiffré → OneDrive (Phase 0 fin)

---

## 📅 Historique des updates de ce document

| Date | Auteur | Changement |
|---|---|---|
| 2026-04-30 | Claude | Création initiale + state actuel + roadmap |
