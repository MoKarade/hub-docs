# 08 — RGPD / Loi 25 / PIPEDA

## Postulat

Le hub contient **toutes les données personnelles de Marc** : c'est lui le sujet et lui le responsable du traitement. Tant que les données restent en local (PC + backup chiffré OneDrive), aucune obligation déclaratoire au sens RGPD/Loi 25 (le hub est un "usage strictement personnel ou domestique"). Mais Marc utilise aussi le hub pour **exercer ses droits face aux entreprises tierces** qui détiennent ses données : c'est l'objet de ce document.

## Cadre légal applicable

| Loi | Juridiction | Délai max réponse | Sanction max |
|---|---|---|---|
| **Loi 25** (Québec) | Entreprises faisant affaire au QC | 30 jours | Jusqu'à 25 M$ ou 4 % du CA mondial |
| **PIPEDA** (Canada) | Entreprises du secteur privé fédéral | 30 jours | Plaintes au CPVP, possible audit |
| **RGPD** (UE) | Entreprises traitant données EU | 30 jours (extensible 60 j) | Jusqu'à 20 M€ ou 4 % du CA mondial |

**Observations** :
- Les 3 lois reconnaissent les mêmes droits : accès, rectification, suppression, portabilité.
- 30 jours est le standard de fait (Loi 25 et PIPEDA sont parfois plus souples mais 30 j reste sûr).
- Si l'entreprise refuse ou ne répond pas → plainte à la **Commission d'accès à l'information (CAI)** au QC, **CPVP** au Canada fédéral, **CNIL** en France.

## Droits de Marc (à l'extérieur du hub)

1. **Droit d'accès** — Quelles données ? Origine ? À qui transmises ?
2. **Droit de rectification** — Corriger les inexactitudes
3. **Droit de suppression** ("droit à l'oubli") — Sauf obligation légale (fiscalité, comptabilité)
4. **Droit à la portabilité** — Recevoir ses données en format machine-readable
5. **Droit d'opposition** — Aux traitements (marketing, profilage)

## Outil intégré au hub : `/v1/privacy/*`

Module ajouté en Session #20 (commit `hub-core@fe0a571`).

### Modèle DB

`removal_requests` :
- `company_name`, `company_email`, `company_url`
- `request_type` ∈ {access, deletion, rectification}
- `legal_basis` ∈ {loi25, pipeda, gdpr, other}
- `status` ∈ {draft, sent, acknowledged, data_deleted, refused, expired}
- Timestamps : `created_at`, `sent_at`, `deadline_at` (auto = sent_at + 30 j), `resolved_at`

### Endpoints

- `POST /v1/privacy/requests` — Crée une demande (status=draft) + génère email FR
- `GET /v1/privacy/requests?status_filter=sent` — Liste filtrée
- `PATCH /v1/privacy/requests/{id}` — Change status (auto-calcul deadline si `draft → sent`)
- `DELETE /v1/privacy/requests/{id}`
- `GET /v1/privacy/summary` — Compteurs draft/sent/overdue/resolved/refused
- `GET /v1/privacy/templates` — Constantes (types, lois, statuses, deadline_days)

### Workflow type

1. Marc identifie une entreprise (broker, site web, app) → `POST /v1/privacy/requests`
2. App génère email FR avec spécifiques légaux selon `legal_basis × request_type`
3. Marc copie/colle ou envoie via Gmail OAuth lui-même (signature personnelle obligatoire)
4. Marc passe le status en `sent` → `deadline_at = sent_at + 30 j` automatiquement
5. Cron `privacy_reminders` (hub-ingest, daily 9h Quebec) push une notif Web Push aux jours-clés J-7 / J-3 / J-1 / J0 / J+1 / J+3 / J+7
6. Marc marque la résolution finale : `acknowledged` → `data_deleted` (ou `refused` / `expired`)

### Pourquoi pas d'envoi automatique ?

La législation (Loi 25, PIPEDA, RGPD) exige une **signature personnelle** pour authentifier la demande. Un envoi 100 % automatisé serait juridiquement fragile en cas de contestation. On facilite la rédaction et le tracking, sans s'en occuper à la place de Marc.

## Données qui méritent une attention particulière

Le hub contient ou pourra contenir des catégories sensibles :

| Catégorie | Source | Sensibilité | Mitigation |
|---|---|---|---|
| Localisation 24/7 | Google Timeline | Très haute | Local only, chiffré au repos via Postgres + age backup |
| Emails (corps + métadonnées) | Gmail OAuth | Haute | Local only, body en clair SQLite (option Fernet futur) |
| Santé (sommeil, cardio, HRV) | Garmin + Google Fit | Haute (données médicales) | Local only |
| Photos (avec EXIF GPS) | Google Photos Picker | Moyenne | Local only |
| Mots de passe testés | HIBP k-anonymity | Faible (hash SHA-1 5 premiers chars seulement) | k-anon protège l'envoi |
| OSINT email/username | Holehe + Sherlock | Moyenne | Outils tiers tournent en local |

**Règle d'or** : si une feature impliquerait l'envoi de données sensibles à un tiers (cloud LLM, service SaaS), elle est rejetée par défaut. Cf. règle 2 du `CLAUDE.md` global ("Local d'abord — les data ne quittent jamais le PC sauf backup chiffré").

## Backup chiffré

- Restic → OneDrive, daily 04h
- Master password → age + sops (jamais en clair sur disque)
- Si OneDrive est compromis : data illisible sans clé age (stockée localement + hub-secrets-vault.age sur Drive comme backup chiffré)

## Si Marc cesse d'utiliser le hub

Pour respecter ses propres droits :
1. Garder le backup chiffré 1 an minimum (au cas où une demande légale ferait surface : impôts, contestation transaction)
2. Après 1 an : `restic forget --keep-last 0 + restic prune` (ou suppression complète Onedrive + clé age)
3. Marc supprime le hub local (`docker compose down -v` + `rm -rf raw_events inbox`)

## Si une entreprise refuse une demande légitime

Plainte à déposer en ligne :
- **QC (Loi 25)** : https://www.cai.gouv.qc.ca/
- **Canada (PIPEDA)** : https://www.priv.gc.ca/
- **France (RGPD)** : https://www.cnil.fr/

L'entreprise doit alors répondre à l'autorité dans des délais bien plus courts (souvent 7 jours).

## TODO Phase future

- [ ] Templates additionnels par secteur (banques, telcos, assurances, brokers data)
- [ ] Liste pré-remplie de data brokers majeurs (Spokeo, Acxiom, BeenVerified, etc. — déjà 6 quick links dans `privacy-osint.tsx`)
- [ ] Auto-détection des entreprises qui ont des données via OSINT (Holehe → liste des sites où l'email est inscrit)
- [ ] Génération PDF à signer électroniquement
- [ ] Suivi automatique du domicile fiscal de l'entreprise pour choisir la bonne loi
