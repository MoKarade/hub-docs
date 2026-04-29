# ADR-0005 — Cloudflare Tunnel + Access (vs Tailscale Funnel)

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Marc veut accéder au hub depuis son téléphone, hors WiFi maison, sans ouvrir de port sur son routeur ISP. Question : Cloudflare Tunnel + Access, ou Tailscale Funnel, ou autre ?

## Décision

**Cloudflare Tunnel pour l'exposition + Cloudflare Access pour l'auth.**

- Cloudflare Tunnel établit une connexion **sortante** depuis le PC de Marc vers les datacenters Cloudflare.
- Cloudflare Access force une auth Google + MFA TOTP avant que la requête atteigne le tunnel.

## Pourquoi

1. **Free tier généreux.** Cloudflare Tunnel + Access sont gratuits pour usage perso (< 50 utilisateurs Access). Cohérent avec règle "tout gratuit".
2. **Aucun port ouvert.** La connexion est sortante : pas de NAT traversal, pas de port forwarding ISP. Si Marc déménage ou change de FAI, le tunnel continue de marcher.
3. **MFA TOTP obligatoire** côté Access. Pas de hack possible avec juste les creds Google.
4. **DNS gratuit via DuckDNS** + sous-domaine Cloudflare. Marc n'a pas besoin de payer un domaine.
5. **Bonus : DDoS protection automatique** (Cloudflare est devant tout).
6. **Logs et audit** : dashboard Cloudflare montre qui s'est connecté, depuis où, quand.

## Trade-offs acceptés

- **Cloudflare voit le trafic en clair.** Le tunnel termine TLS chez Cloudflare, ré-établit TLS interne. Pour Marc seul, à usage perso, accepté (cf. `06-security.md`).
- **Dépendance fournisseur.** Si Cloudflare change ses prix ou ferme le free tier, à migrer. Mitigation : Tailscale Funnel reste un fallback prêt-à-l'emploi.
- **Latence ajoutée** (~30-50 ms via Cloudflare datacenters). Acceptable pour usage UI.
- **Cloudflare Access nécessite un compte Google** (ou autre IDP). Marc a déjà un compte Google → 0 effort.

## Alternatives rejetées

### Tailscale Funnel
- ✅ Plus privé (mTLS de bout en bout, Tailscale ne voit pas le trafic)
- ❌ Pas de MFA natif sur Funnel (il faut bricoler avec Tailscale ACLs)
- ❌ Pas d'IDP intégré (pas de "login Google" pour des invités)
- ❌ Free tier limité à 3 utilisateurs (suffisant pour Marc seul, mais limitant si famille)
- 👉 Excellent fallback en cas d'abandon de Cloudflare.

### Port forwarding ISP + Let's Encrypt manuel
- ❌ Nécessite IP publique stable (Marc est probablement en CGNAT)
- ❌ Expose le PC directement à Internet
- ❌ Renouvellement de cert Let's Encrypt tous les 90 jours, scriptable mais fragile

### ngrok / serveo / localhost.run
- ❌ URLs aléatoires qui changent à chaque restart (`xxx.ngrok.app`)
- ❌ Free tier limité (debit, sessions par jour)
- ❌ Pas de MFA intégrée

### VPN classique (OpenVPN, WireGuard) self-hosted
- ❌ Toujours besoin d'un port ouvert quelque part (sauf si tunneled via Cloudflare aussi → wrap unnecessary)
- ❌ Marc devrait installer un client VPN sur chaque appareil
- ❌ Pas d'auth Google native

## Conséquences

- ✅ `hub-deploy/cloudflared/config.example.yml` à compléter en Phase 0 fin (TODO).
- ✅ `docker-compose.prod.yml` aura un service `cloudflared` qui run le tunnel.
- ✅ Setup Access policy via dashboard Cloudflare Zero Trust : email = `marc.richard4@gmail.com` + MFA TOTP obligatoire.
- ⚠️ Marc doit créer le compte Cloudflare (gratuit, ~5 min) et faire `cloudflared tunnel login` une fois.
- ⚠️ Si Marc reset son TOTP par erreur, accès local toujours OK (`http://localhost:8000`) pour reset depuis le dashboard.

## Plan B (si Cloudflare ferme le free tier ou devient hostile)

Migration vers **Tailscale Funnel + Tailscale ACL avec OIDC Google** :
1. `tailscale up --funnel` sur le PC + chaque device de Marc
2. Configurer ACLs Tailscale (admin panel) pour limiter accès
3. Utiliser OAuth Google côté hub-core directement (au lieu de Cloudflare Access)
4. RTO migration : ~4h une fois la décision prise

Le code du hub n'a pas de dépendance à Cloudflare (lit le header `Cf-Access-Authenticated-User-Email` mais peut être substitué). Migration peu coûteuse.
