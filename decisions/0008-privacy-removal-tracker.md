# ADR-0008 — Module privacy : tracker manuel + cron de relance

**Date :** 2026-05-06
**Statut :** Acceptée
**Décideurs :** Marc, Claude

## Contexte

Marc voulait un "module Suppression PIPEDA/Loi 25 QC" en Phase 6 pour exercer ses droits face aux entreprises qui détiennent ses données. La question : automatiser entièrement (rédiger + envoyer + suivre + plaindre) ou laisser Marc dans la boucle ?

## Décision

**Tracker manuel (CRUD + génération template) + cron de relance par notif Web Push.**

Concrètement :
1. Marc crée la demande via UI (`/v1/privacy/requests`) en saisissant entreprise + type + loi
2. Backend génère subject + body FR avec spécifiques légaux (30 j deadline, demande standard)
3. Marc copie/colle ou envoie via Gmail OAuth lui-même
4. Marc passe le status à `sent` → `deadline_at = sent_at + 30 j` calculé auto
5. Cron `privacy_reminders` (hub-ingest, daily 9h Quebec) push notif aux jours-clés J-7 / J-3 / J-1 / J0 / J+1 / J+3 / J+7
6. Marc marque la résolution finale

## Pourquoi pas d'automatisation totale

1. **Légal** : la signature personnelle est exigée pour authentifier la demande (Loi 25 art. 32, RGPD art. 12). Un email envoyé par un bot via OAuth signé "Marc Richard" mais sans intervention humaine serait juridiquement fragile en cas de contestation.
2. **Anti-spam** : si on automatisait l'envoi en batch (ex : toutes les data brokers du marché), on risquerait d'être marqué comme spam et perdre l'effet légal des demandes.
3. **Réflexion utile** : créer la demande à la main force Marc à se demander "ai-je vraiment intérêt à supprimer mes données chez X ?" avant d'envoyer.
4. **Règle "Marc est cyclothymique" du profil** : on ne fait pas à sa place. On donne le levier, il décide.

## Trade-offs acceptés

- **Plus de friction côté Marc** : 4 clics au lieu de 0. Mitigation : génération auto du template, copie 1-clic, summary visible en permanence dans `/settings → Loi 25`.
- **Risque d'oubli** : Marc peut créer une demande draft et oublier. Mitigation : la cron `privacy_reminders` ne déclenche que sur status=sent — donc une draft oubliée reste inerte. Possible évolution : alerte sur drafts > 7j.
- **Pas de plainte CAI auto** : si l'entreprise ne répond pas, Marc doit aller manuellement sur cai.gouv.qc.ca. Mitigation : le body du reminder J+1/J+3/J+7 mentionne "Plainte CAI possible" en clair.

## Alternatives rejetées

### A. Automation 100 % via Gmail OAuth

- Fonctionne techniquement (déjà OAuth Gmail dispo)
- Rejet : risque légal + risque spam + Marc perd la maîtrise de QUI il contacte

### B. Envoi via SMTP générique (pas Gmail)

- Permettrait d'éviter le quota Gmail
- Rejet : signature non vérifiable, pas de DKIM/SPF sans config sérieuse → mail end up in spam

### C. Pas de cron, juste un dashboard "à faire"

- Plus simple
- Rejet : Marc oublie. Le cron transforme en notif active, ce qui est l'intention de "l'app envoie les notifs elle-même" (cf. session #20 demande explicite Marc).

### D. Cron envoie l'email lui-même au lieu de notifier Marc

- Tentant
- Rejet : même raison que A. Le cron rappelle, Marc agit.

## Conséquences

- Module utilisable dès aujourd'hui (CRUD + summary + template + cron prêts)
- Liste des data brokers majeurs déjà disponible dans `components/privacy-osint.tsx` (6 quick links Spokeo / Acxiom / BeenVerified / etc.) — Marc clique → page d'opt-out → soumet via leur formulaire
- Évolution naturelle : pré-remplir une liste de companies à viser via Holehe (qui détecte sur quels sites un email est inscrit)
- Pas de coût opérationnel : tout local, pas d'API tierce nécessaire

## Suivi

- Module livré en commits `hub-core@fe0a571` + `hub-frontend@aa03011` + `hub-ingest@0088a0c`
- Doc complète : `hub-docs/08-rgpd.md`
- Tests live à venir quand Marc créera sa première vraie demande
