---
name: Banque et comptes Marc
description: Banque unique de Marc + structure de ses comptes pour le Personal Data Hub. Utilisée pour orienter l'implémentation du connecteur banque dans hub-ingest.
type: project
originSessionId: 6ecd3435-015a-4c30-9bd6-82daeff7fcaa
---
Marc utilise **uniquement Desjardins** (banque coopérative québécoise, portail = AccèsD). Comptes :

- 1 compte courant débit / opérations (`EOP` dans les CSV) — CAD
- 1 compte épargne à terme (`ET1` dans les CSV) — CAD
- 1 compte de carte de crédit Desjardins Remises Mastercard. Numéro de compte = `****5004` (admin), carte physique principale de Marc = `****5020`. Pas de carte secondaire connue.
- 1 compte actions CAD chez Disnat / Desjardins Courtage en ligne (sous-compte `5NFL7A3`)
- 1 compte actions USD chez Disnat (sous-compte `5NFL7B1`)
- Numéro client Disnat : `5NFL7`

**Devises gérées : CAD + USD**

**Formats d'export :**
- Compte courant + épargne : CSV brut, encodage **cp1252** (caractères accentués cassés en UTF-8), 14 colonnes sans header. Clé naturelle : `(transit, account_type, date, seq_num)`.
- Carte de crédit : **PDF** (Desjardins n'expose pas de CSV pour les relevés Mastercard). Texte propre, dates `JJ MM` sans année (à déduire de la date du relevé), crédits suffixés `CR`. Cashback rate par transaction (`0,50 %` / `2,00 %`) pré-classifie les catégories.
- Disnat : **PDF mensuel de portefeuille** (relevés `5NFL7_ETATCOMPTE_<YYYY-MM-DD>.pdf`). Contient 3 sections par sous-compte : sommaire, activité mensuelle (transactions), détails des actifs (positions snapshot fin de mois). À voir avec Marc si Disnat expose aussi un export CSV de l'historique des opérations (plus simple à parser). Sinon parser PDF avec pdfplumber. Modèles DB séparés nécessaires : `investment_transaction` + `investment_position` (snapshot mensuel).

**Méthode d'extraction abandonnée :** Marc a utilisé un service tiers payant pour extraire ses données, mais veut **arrêter** car payant. Ne JAMAIS proposer de service tiers payant (Plaid, Mint, Tiller, MX, etc.) — viole la règle 5 (tout gratuit) du CLAUDE.md global.

**Why:** Marc a explicitement dit le 2026-04-28 qu'il abandonnait son service tiers payant. C'est cohérent avec sa règle "tout gratuit". Toute solution proposée pour Phase 1 banking doit être 100% gratuite.

**How to apply:** Pour le connecteur banque dans hub-ingest, partir sur des exports CSV/OFX manuels depuis AccèsD + Disnat (gratuits, supportés par Desjardins). Pas de scraping (AccèsD a auth forte + MFA, fragile et risque blocage compte). Pas d'API officielle Desjardins (n'existe pas pour les particuliers).
