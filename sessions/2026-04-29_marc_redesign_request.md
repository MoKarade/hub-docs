# Demande de Marc — refonte UI + élargissement scope (2026-04-29, fin Session #2)

> **Verbatim Marc** (à conserver intact pour relire avec recul) :
>
> *« le front end est trop... IA, je veux que ce soit plus beau plus interactif et moins statique, je veux aussi plus de data genre la santé, mes réseaux, et tout ce que tu peux imaginer... aussi genre la sécurité de mes données, qc ou voir que mes données sur le web sont safe ou nulle part, faut que tu refasse de quoi pose moi plein de questions... rajoute dans le road map et souvient toi de ce que je viens de te dire (genre sauvegarde le aussi pour si je switch de pc) »*

## Décodage de la demande

### 1. Le frontend actuel est jugé « trop IA »

C'est le 2ᵉ feedback design de Marc — le 1er (Session #1) avait fait passer le mockup initial de violet/AI-générique vers la palette ink + vert terminal-y actuelle. Cette fois le verdict est : **on n'est pas allés assez loin**.

Ce qui reste « trop IA » dans la version actuelle :
- Palette dark uniforme (même si vert > violet)
- Layout grid SaaS-dashboard très standard
- Absence d'éléments personnels (pas de photos, pas d'illustrations)
- Animations minimales (juste pulse-slow + fade-in)
- Pas d'easter eggs / personnalité / humour

### 2. Plus beau / plus interactif / moins statique

À discuter avec Marc — voir [questions ouvertes](#questions-ouvertes-pour-marc) ci-dessous. Hypothèses :
- Plus d'animations (transitions de page, stagger, morph)
- Drag-and-drop pour réorganiser les widgets ?
- Realtime updates (push, pulse à l'arrivée d'une transaction)
- Hover effects plus riches
- Mode focus / expand sur un widget
- Possibilité d'inclure ses propres photos en backgrounds / mosaïques
- Thème jour/nuit auto

### 3. Plus de data — santé + réseaux + « tout ce que tu peux imaginer »

Élargissement majeur du scope. Sources candidates à clarifier :

| Catégorie | Sources possibles | Priorité Marc |
|---|---|---|
| **Santé** | Apple Health (iPhone), Garmin, Whoop, Oura, MyFitnessPal, Cronomètre, Strava | À demander |
| **Streaming** | Spotify, YouTube history, Netflix, Disney+ | À demander |
| **Social** | Twitter/X, Instagram, LinkedIn, Facebook, Reddit, TikTok | À demander |
| **Gaming** | Steam, PSN, Xbox | À demander |
| **Browser** | Chrome history, Firefox sync, Safari | À demander |
| **Productivité** | GitHub, Notion, Obsidian, Todoist | À demander |
| **Lecture** | Kindle highlights, Pocket, Readwise, Goodreads | À demander |

### 4. Module « Sécurité de mes données » 🆕

**Nouveau module entièrement** — mes données sur le web sont-elles safe ou pas ?

Sous-features candidates :
- **Breach check (HIBP)** : email, numéro, mots de passe leaked dans des breaches connus
- **Footprint web** : que voit Google quand on cherche `Marc Richard Lévis Quebec` ?
- **Inventaire de comptes** : quels sites Marc a un compte → état de chaque (actif, abandonné, à supprimer)
- **Score d'exposition** : tableau de bord agrégé (X breaches, Y data brokers, Z comptes oubliés)
- **Plan d'action** : opt-out automatisé (avec service type Optery — mais payant, à arbitrer vs règle 5)

### 5. Question de méta : « tu sauvegardes pour si je switch de PC »

Marc rappelle la règle 2 (« tout sauvegarder, rien à oublier »). Ce fichier (`UI_REDESIGN_BRIEF.md`) + la mise à jour de `SUITE.md` + l'entrée dans `JOURNAL.md` forment la sauvegarde — visibles depuis n'importe quel PC qui a le repo `hub-docs` cloné.

## Conflits potentiels avec les règles dures

| Règle | Risque |
|---|---|
| **Tout gratuit** | Le scan OSINT externe (data brokers) demande typiquement un service payant type Optery (~10$/mois). Alternatives gratuites : HIBP API + scraping Google + opt-out manuel. À discuter. |
| **Local d'abord** | Si on intègre Spotify/Netflix, on appelle leur API → certaines requêtes sortent du PC. Acceptable car c'est ce que Marc demande explicitement, et les API officielles (vs scraping). |
| **Stack ennuyeuse** | Drag-and-drop + animations riches → on va peut-être ajouter `framer-motion` (déjà en deps), `dnd-kit`, etc. Limites à poser. |
| **No fake data** | Pour la refonte UI on aura tendance à mocker pour itérer rapide. **Règle absolue** : tout mockup pour discussion design, mais code commité = wired sur API réelle. |

## Questions ouvertes pour Marc

> Marc a explicitement demandé qu'on lui pose plein de questions. À transférer en début de session #3.

### Direction visuelle

1. Quand tu dis « trop IA », tu vois ça plutôt comme :
   - (a) palette : trop dark/sombre, faut du clair / chaud (beige, terracotta, papier) ?
   - (b) layout : trop grid/SaaS, faut un truc plus organique (collage, scrapbook) ?
   - (c) typo : trop sans-serif modern, faut du serif / rétro / handwritten ?
   - (d) toutes ces réponses
2. Une **référence concrète** d'app ou site dont tu aimes le design ? (Linear, Notion, Apple Fitness+, Strava, Spotify Wrapped, Things 3, Anthropic.com, etc.)
3. **Mode clair / sombre / auto** ?
4. Tu acceptes que tes propres **photos** (Google Photos) soient utilisées en backgrounds, mosaïques, hero images ? Ou ça reste sobre ?
5. **Niveau de personnalité** : 0 = froid pro · 5 = mignon avec emoji · 10 = mode joueur (easter eggs, animations exagérées). Tu te situes où ?

### Interactivité

6. **Drag-and-drop** pour réorganiser les widgets de la home ?
7. **Animations** :
   - Transitions entre pages : fade ? slide ? morph ?
   - Stagger à l'arrivée des stats ?
   - Hover effects riches (lift + shadow + tilt) ?
8. **Realtime** : si hub-ingest pousse une nouvelle transaction, ça pulse / s'anime sur la home ?
9. **Mode focus** : clic sur un widget = expand fullscreen avec plus de détails ?

### Santé

10. Quels **trackers / apps** tu utilises pour ta santé ?
    - iPhone → Apple Health (export XML manuel)
    - Garmin / Whoop / Oura / Fitbit ?
    - MyFitnessPal / Cronomètre / Yuka (alimentation) ?
    - Strava / RunKeeper / Komoot (sport) ?
11. Quelles **métriques** tu veux voir : pas/jour, sommeil, fréquence cardiaque, poids, hydratation, calories, masse musculaire, tension, glycémie, autres ?

### Réseaux / streaming / autres

12. Quelles plateformes utilises-tu vraiment et veux tracker ?
    - 🎵 Spotify (écoute, top, playlists) ?
    - 📺 Netflix / Disney+ / Apple TV / Amazon Prime ?
    - ▶️ YouTube (watch history, abonnements) ?
    - 🐦 Twitter/X / Instagram / LinkedIn / Reddit / TikTok ?
    - 🎮 Steam / PSN / Xbox ?
    - 📚 Kindle / Pocket / Readwise / Goodreads ?
    - 🌐 Chrome / Firefox history (via sync) ?
    - 💼 GitHub / Notion / Obsidian / etc. ?

### Module sécurité

13. Tu veux quoi exactement :
    - (a) Check breach : ton email + tes mots de passe ont fuité où ?
    - (b) Footprint Google : ce qui ressort de tes recherches publiques
    - (c) Inventaire : quels sites tu as un compte (actif/oublié/à supprimer)
    - (d) Data brokers : qui revend ton profil (USA-centric, mais existe Canada)
    - (e) Plan d'action : opt-out semi-automatisé
    - (f) Toutes
14. Pour les services qui coûtent :
    - HIBP API (gratuit, K-Anonymity, suffit pour les emails) ✓ règle gratuit
    - Optery / DeleteMe / Privacy.com (~10-30$/mois) ✗ violerait règle gratuit
    - Tu acceptes une **exception payante** pour la sécu, ou strict sur le gratuit ?

### Priorisation et budget temps

15. Si je devais ranger 4 chunks par priorité — lequel d'abord ?
    - **Refonte UI** complète (palette, layout, anims) — ~3 sessions
    - **Santé** (Apple Health import + dashboard) — ~2 sessions
    - **Streaming/social** (Spotify + YouTube + Netflix) — ~2 sessions
    - **Sécurité** (HIBP + footprint + inventaire) — ~2 sessions
16. Tu vises **combien de sessions** au total ? On a fini #2, on peut grouper jusqu'à #6 ou pousser plus loin si tu veux.

## Décisions à prendre avant de coder quoi que ce soit

- [ ] Direction visuelle (Q1-5)
- [ ] Niveau d'interactivité acceptable (Q6-9)
- [ ] Sources santé prioritaires (Q10-11)
- [ ] Sources streaming/social prioritaires (Q12)
- [ ] Périmètre du module sécurité (Q13)
- [ ] Exception payante pour sécu ? (Q14)
- [ ] Ordre de priorité (Q15)
- [ ] Budget en sessions (Q16)

## Notes pour la prochaine session

- Cette demande **pré-empt** la priorité « Phase 0 fin / déploiement vrai PC » qui était en tête de `SUITE.md` jusqu'ici.
- Marc travaille parfois depuis un PC (`marcr`, ancien) parfois depuis un autre (`dessin14`, actuel). **Toute info volatile** doit être commitée dans hub-docs (Google Drive synchronisé) avant la fin de chaque session.
- Le mode « rien sur ce PC » qu'il a imposé en Session #2 reste : on continue à coder sans installer/lancer.
