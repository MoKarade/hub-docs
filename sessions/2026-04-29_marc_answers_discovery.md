# Réponses de Marc — Discovery UI + scope (2026-04-29, début Session #3)

> Réponses aux 16 questions du brief [`2026-04-29_marc_redesign_request.md`](2026-04-29_marc_redesign_request.md).
> Ces réponses verrouillent les décisions pour la suite. Ne plus re-poser ces questions.

---

## Verbatim Marc

> *« je réponds aux questions du doc: 1. couleur ok, layout trop standard, pas assez elements personnels, (je ne veux pas deaster egg humour ou autre), plus danimation, drag and drop bonne idée mais plus que ca, realtime update oui, hover oui, mode focus oui, photos non, toujours mode nuit. jai garmin et google fit, oui streaming, oui social, oui gaming steam xbox, oui browser, oui github, non lectire (donne encore plus que ca, tout mon data en ligne), je veux la possibilité de supprimer tout mon data en ligne, et le reste que tu as mis.... je veux que tout se sauvegarde sur mon drive le travail qu'on fait... »*
>
> *« je reponds aux questions: 1.d 2.non 3. sombre seulelemt 4.non 5.pas de easter eggs 6. oui 7. oui tout 8. oui 9. oui 10.jai deja repondu 11. tout 12. ty, ty music, steam xbox, chrome, github, netflix disney + prime crunchyroll 13. f 14. gratuit 15. tout 16. jsp »*

---

## Réponses interprétées — question par question

### Q1 — Quand tu dis « trop IA », tu vois ça comment ?
**Réponse : d (toutes ces réponses)** — avec ces nuances importantes :
- ✅ Palette dark → **OK à garder**, c'est la bonne direction
- ❌ Layout grid SaaS → **À refaire complètement** — trop standard, manque d'éléments personnels
- ✅ Animations → **À enrichir massivement**
- ❌ Easter eggs, humour → **NON** — Marc ne veut pas de personnalité excessive
- ❌ Photos personnelles → **NON** (confirmé Q4)

**Décision verrouillée :** Le redesign UI porte essentiellement sur le **layout** (plus organique, moins SaaS-dashboard) et les **animations/interactions** (beaucoup plus riches). La palette dark reste, on ne la jette pas.

---

### Q2 — Référence concrète d'app dont tu aimes le design ?
**Réponse : non** — pas de référence spécifique.

**Impact :** On définit nous-mêmes la direction. Axe choisi :
- Dashboard **modulaire et dense** (style terminal meets personal HQ)
- Widgets qui "respirent" (spacing généreux, micro-animations au survol)
- Plus de hiérarchie visuelle entre l'important et le secondaire
- Sections clairement délimitées, pas une grille uniforme de cards identiques

---

### Q3 — Mode clair / sombre / auto ?
**Réponse : sombre seulement**

**Décision verrouillée :** Pas de bouton de thème, pas de mode clair, dark-only. Simplifie le CSS considérablement.

---

### Q4 — Photos personnelles en backgrounds/mosaïques ?
**Réponse : non**

**Décision verrouillée :** Pas de photos de Marc dans l'UI. L'identité visuelle vient du design lui-même, pas de médias personnels.

---

### Q5 — Niveau de personnalité (0 froid → 10 easter eggs) ?
**Réponse : pas d'easter eggs** → estimé **2-3/10**

**Décision verrouillée :** Professionnel, dense, interactif — mais pas de blagues cachées, pas d'émojis partout, pas de confettis. La richesse vient des **interactions**, pas du contenu fantaisiste.

---

### Q6 — Drag-and-drop pour réorganiser les widgets ?
**Réponse : oui — mais plus que ça**

**Interprétation :** Marc veut un dashboard **vraiment customisable** :
- Drag-and-drop (réordonner)
- Probablement : redimensionner les widgets (grid resizable)
- Épingler des widgets en haut
- Masquer/afficher les modules qu'il ne veut pas voir
- Persistance du layout (sauvegardé côté serveur ou localStorage)

**Tech → `dnd-kit` + layout persisté en `localStorage` (simple) ou endpoint `PATCH /v1/user/layout` (robuste).**

---

### Q7 — Animations (transitions, stagger, hover) ?
**Réponse : oui tout**

**Décision verrouillée :**
- ✅ Transitions entre pages (slide ou fade via framer-motion)
- ✅ Stagger à l'arrivée des stats (les cards apparaissent en cascade)
- ✅ Hover effects riches (lift + ombre + légère inclinaison)

---

### Q8 — Realtime : si une nouvelle transaction arrive, ça pulse sur la home ?
**Réponse : oui**

**Tech → Server-Sent Events (SSE) sur `GET /v1/events/stream` côté hub-core.** hub-frontend souscrit via `EventSource`. Chaque ingestion push un event. Plus léger qu'un WebSocket pour du unidirectionnel.

---

### Q9 — Mode focus : clic sur un widget = expand fullscreen avec plus de détails ?
**Réponse : oui**

**Décision verrouillée :** Chaque widget a un état "focus" (fullscreen overlay ou expanded sidebar). Ex : clic sur le widget Finances → modal pleine-page avec la table complète + graphiques.

---

### Q10 — Trackers santé ?
**Réponse : Garmin + Google Fit** (déjà répondu dans le premier bloc)

**Sources confirmées :**
- **Garmin** : export `.fit` via Garmin Connect (téléchargement manuel ou Garmin Health API — nécessite dev.garmin.com, OAuth)
- **Google Fit** : REST API (`fitness.googleapis.com`) — OAuth Google déjà utilisé pour Gmail à terme

---

### Q11 — Métriques santé souhaitées ?
**Réponse : tout**

**Métriques à implémenter (selon disponibilité Garmin/Google Fit) :**
- Pas/jour (step_count)
- Distance parcourue
- Fréquence cardiaque (repos + effort + variabilité)
- Sommeil (durée, phases light/deep/REM)
- Calories dépensées
- Activités sportives (course, vélo, natation, etc.)
- Hydratation (si suivi dans une app)
- Poids / IMC (si balance connectée Garmin)
- VO2max (Garmin calcule ça)
- Stress score (Garmin)
- SpO2 (si capteur dispo sur la montre)

---

### Q12 — Plateformes streaming/social à intégrer ?
**Réponse :** YouTube, YouTube Music, Steam, Xbox, Chrome, GitHub, Netflix, Disney+, Prime Video, Crunchyroll

**Sources confirmées et approche technique :**

| Plateforme | Méthode d'accès | Effort |
|---|---|---|
| **YouTube** | Takeout (watch history HTML/JSON) | ★★☆ |
| **YouTube Music** | Takeout (music-library-songs.csv, history) | ★★☆ |
| **Netflix** | Paramètres compte → "Télécharger mes données" → CSV | ★★☆ |
| **Disney+** | Pas d'API ni export standard → scraping page historique (fragile) | ★★★ |
| **Prime Video** | Alexa + Amazon Data Request → JSON export | ★★★ |
| **Crunchyroll** | Pas d'export officiel → OAuth API v2 non-publique (à creuser) | ★★★ |
| **Steam** | API publique (steamcommunity.com/profiles/ID/games + ISteamUserStats) | ★☆☆ |
| **Xbox** | Xbox Live API (xboxapi.com wrapper ou MS Graph) | ★★☆ |
| **Chrome** | Takeout (BrowserHistory.json) | ★☆☆ |
| **GitHub** | API REST publique (repos, commits, PRs, stars) — Auth optionnel | ★☆☆ |

**Pas demandé (confirmé non) :** Kindle, Pocket, Readwise, Goodreads, Spotify (non mentionné), TikTok, Instagram, Twitter/X, LinkedIn, Reddit, Facebook.

---

### Q13 — Module sécurité — quoi exactement ?
**Réponse : f = tout**

**Features confirmées :**
- ✅ **(a)** Breach check : HIBP API (email + passwords leaked) — gratuit K-Anonymity
- ✅ **(b)** Footprint Google : ce qui ressort des recherches publiques sur Marc Richard Lévis Québec
- ✅ **(c)** Inventaire de comptes : Marc renseigne manuellement + détection via breach data
- ✅ **(d)** Data brokers : qui revend son profil (Canada-spécifique = Whitepages CA, etc.)
- ✅ **(e)** Plan d'action : liens opt-out + templates emails RGPD/Loi 25 QC
- ✅ **(NOUVEAU)** **Module Suppression** : Marc veut pouvoir supprimer tout son data en ligne

**Module Suppression** (nouveau scope, demande explicite) :
- Interface de suivi des demandes de suppression par plateforme
- Statuts : à faire / envoyé / en attente / confirmé / refusé
- Templates de demande (PIPEDA + Loi 25 Québec pour Canada, RGPD pour services UE)
- Liens directs vers les pages "supprimer mon compte" de chaque service
- Tracker de réponse (date envoi, délai légal, date réponse)

---

### Q14 — Exception payante pour la sécurité ?
**Réponse : gratuit — pas d'exception**

**Décision verrouillée :** HIBP gratuit (K-Anonymity), pas d'Optery/DeleteMe. Tout le module sécurité est 100% gratuit.

---

### Q15 — Priorité des 4 chunks (UI / Santé / Streaming / Sécurité) ?
**Réponse : tout** — pas de priorisation explicite.

**Décision opérationnelle (par défaut) :**
Vu la dépendance technique, on fait dans cet ordre :
1. Refonte UI (débloque la présentation de tout le reste)
2. Deploy PC réel (sans ça, rien ne tourne)
3. Phase 0 fin (tunnel + backup)
4. Google Takeout (déjà code-complete)
5. Santé + Streaming/Gaming + Sécurité en parallèle (3 sous-équipes si on a le temps, sinon Santé → Streaming → Sécurité)

---

### Q16 — Budget en sessions ?
**Réponse : jsp** — pas de contrainte définie.

**Décision opérationnelle :** On avance session par session. Pas de deadline. On livre des incréments fonctionnels à chaque session.

---

## "Tout mon data en ligne" — scope élargi

Marc a dit : *"non lectire (donne encore plus que ca, tout mon data en ligne)"*

Au-delà de ce qui était listé, voici les sources online supplémentaires à envisager :

| Catégorie | Sources potentielles |
|---|---|
| **E-commerce** | Amazon purchases (Order History CSV), eBay |
| **Transport** | Uber/Lyft (trip history), Google Maps Timeline (déjà prévu) |
| **Finance** | PayPal (statement CSV), déjà couvert par Desjardins |
| **Communication** | WhatsApp export, iMessage (Apple), Discord export |
| **Social non coché** | Twitter/X (Takeout), Instagram (Takeout), LinkedIn (Takeout) |
| **Photos** | Google Photos (Takeout — déjà Phase 3) |
| **Documents** | Google Drive (Takeout metadata), OneDrive |
| **Calendrier** | Google Calendar (déjà Phase 5 prévue) |
| **Achats applis** | App Store / Google Play histoires d'achats |
| **Santé** | Apple Health export XML (iPhone — si applicable) |

**Approche :** On ajoute au fur et à mesure. Le principe est toujours le même : Takeout officiel ou API publique en priorité, scraping en dernier recours (fragile, CAPTCHA, ToS risqué).

---

## "Tout se sauvegarde sur mon drive"

**Bonne nouvelle : c'est déjà le cas.**

Le projet vit dans `G:\Mon disque\PERSO & LOISIRS\AUTOMATISATION\Projets\Hub perso\` qui est **Google Drive synchronisé**. Tout ce qu'on écrit ici est donc automatiquement synchonisé vers ton Google Drive et accessible depuis n'importe quel PC où tu es connecté à Google Drive.

En plus, tout le code est pushé sur **GitHub** (MoKarade) — 2ème copie indépendante.

Double sauvegarde à chaque session :
- ✅ Google Drive (synchronisation automatique fichiers locaux)
- ✅ GitHub (push explicite en fin de session)

---

## Décisions verrouillées — récap

| # | Sujet | Décision |
|---|---|---|
| D1 | Palette | Dark seulement — garder l'ink/vert actuel comme base |
| D2 | Layout | Refonte complète : moins SaaS-grid, plus modulaire/organique |
| D3 | Photos | NON dans l'UI |
| D4 | Easter eggs | NON — professionnel et riche mais pas fantaisiste |
| D5 | Interactivité | Drag-drop + resize + pin/unpin + persistance layout |
| D6 | Animations | Tout (framer-motion) : transitions, stagger, hover, realtime pulse |
| D7 | Realtime | SSE (`/v1/events/stream`) |
| D8 | Mode focus | Oui — expand/fullscreen par widget |
| D9 | Santé | Garmin + Google Fit — toutes les métriques dispo |
| D10 | Streaming | YouTube, YT Music, Netflix, Disney+, Prime, Crunchyroll |
| D11 | Gaming | Steam + Xbox |
| D12 | Browser/Dev | Chrome history + GitHub |
| D13 | Sécurité | Module complet (HIBP + footprint + inventaire + brokers + suppression) |
| D14 | Coût | Tout gratuit, aucune exception |
| D15 | Lecture | NON (Kindle, Pocket, etc.) |
| D16 | Nouveau module | **Module Suppression** : tracker des demandes de suppression de données en ligne |
