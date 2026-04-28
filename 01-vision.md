# 01 — Vision du Personal Data Hub

## En une phrase

Un hub privé hébergé chez Marc qui agrège toutes ses données personnelles, accessible avec une IA locale qui répond précisément à ses questions et l'alerte sur ce qui mérite son attention.

## Problème résolu

Aujourd'hui Marc a ses données dispersées : sa banque a ses transactions, Google a sa localisation et ses photos, son app finance maison a sa propre logique, son app trajets aussi. Pour une question simple comme "combien j'ai dépensé en restos pendant mon voyage à Seattle en mars", il faudrait croiser à la main 3-4 sources.

## Solution

Un point unique :
- Toutes les sources convergent vers une DB locale (PostgreSQL + pgvector).
- Une IA locale (Ollama + Qwen 2.5 14B) qui parle français et qui répond aux questions précises.
- Les apps existantes (trajets, finance) continuent d'exister mais consomment l'API du hub plutôt que leurs propres fichiers.
- Versioning multi-version live des apps pour pouvoir comparer / itérer sans casser.
- Accessible depuis n'importe où via un tunnel privé Cloudflare avec authentification Google + MFA.

## Principes directeurs

1. **Aucune fake data.** Que de la vraie. C'est tout l'intérêt du local.
2. **Local d'abord.** Les données ne quittent jamais le PC sauf pour le backup chiffré.
3. **Tout gratuit.** Pas d'abonnement, pas de service cloud payant. DuckDNS + Cloudflare Tunnel free + Ollama + GitHub free.
4. **Stack ennuyeuse.** Pas de tech expérimentale. Python + FastAPI + Postgres + Next.js + Tailwind. Stable, documenté, maintenable.
5. **Event sourcing.** Les data brutes sont stockées telles quelles, immutables. La DB de vue est reconstructible. Robuste contre les bugs et les changements de schéma.
6. **Décisions justifiées.** Chaque choix technique est tracé en ADR. Quand on revient 6 mois plus tard, on sait pourquoi.

## Ce que le hub n'est PAS

- ❌ Un produit à vendre. C'est l'outil personnel de Marc.
- ❌ Un coffre-fort. Si Marc veut chiffrer une note, il utilise un password manager.
- ❌ Un agrégateur cloud. Tout est local.
- ❌ Une plateforme multi-tenant. Single-user (extensible plus tard si besoin).
- ❌ Un service web public. Privé par défaut, accessible uniquement à Marc.

## Échelle envisagée

- 1 utilisateur (Marc)
- ~10 ans d'historique de toutes les sources confondues
- Volume estimé : ~250 GB sur disque, ~2 GB en DB (sans les médias)
- Disponibilité visée : 99 % (panne planifiée OK, panne non détectée à éviter)
- RTO (recovery time objective) : 4-8 heures en cas de catastrophe
- RPO (recovery point objective) : ≤ 24 heures de perte de data en cas de pire scenario
