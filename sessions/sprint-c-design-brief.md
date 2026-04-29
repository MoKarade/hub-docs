# Sprint C — Design Brief (2026-04-29)

Source : session discovery avec Marc (18 questions/réponses).

## Style directeur : Google Analytics dark mode

- Dense mais chirurgical — données au premier plan, zéro déco superflue
- Hiérarchie typographique forte (taille + poids + couleur, pas d'ornements)
- Cards propres, borders subtiles, pas de shadow lourde
- Couleurs neutres pour la donnée neutre, couleur seulement pour signal

## Palette révisée

- Chiffres/données neutres → `text-ink-100` (blanc cassé), PAS de vert par défaut
- Vert `#5cdb95` → UNIQUEMENT pour valeurs positives (gain, solde positif, statut ok)
- Rouge/orange → valeurs négatives, alertes
- Tout le reste : gris `ink-400` à `ink-600`

## Navigation

- Sidebar collapsible : mode étendu (icône + texte) ↔ mode réduit (icône seule)
- Responsive : hamburger mobile, sidebar fixe desktop
- Status bar visible en permanence en bas

## Dashboard home

- Plus de widgets, mais chaque widget épuré en surface
- Progressive disclosure : surface propre → click → détails
- Layout fixe bien pensé (pas de drag-drop prioritaire)
- Style Google Analytics : métriques clés en haut, graphes en dessous

## Page /finances

- Vue résumé par défaut : KPIs + donut/camembert par catégorie
- Comptes séparés (EOP, Mastercard, Disnat) + vue consolidée (toggle)
- Table transactions accessible en cliquant "voir tout"
- SQL IA visible directement dans les réponses

## Page /locations

- Heatmap des zones fréquentées (vue densité)
- Vue chronologique trajets avec filtres (date, type activité)
- Toggle entre les deux modes

## Page /search

- SQL généré visible directement dans la réponse IA (pas caché)

## Mobile

- Support téléphone + desktop
- Sidebar hamburger sur mobile

## Philosophie

"Épuré en surface, riche en profondeur."
Chaque layer de click révèle plus d'information.
Jamais de vide inutile, jamais de surcharge visuelle au premier regard.
