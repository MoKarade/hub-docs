# Tutoriel : Créer une page mobile-friendly

> Cible : ajouter une nouvelle page Next.js qui marche bien sur le tel de Marc (PWA installée, écran 375-430 px).

## Règles d'or (rappel session #20)

Le `globals.css` du hub définit déjà des règles globales mobile (max-width: 1023px) :

- `button[class*="text-xs"]` → min-height 40px
- `input/select/textarea` → min-height 40px + font-size 16px (anti-zoom iOS)
- `button.rounded*` → min-height 36px
- `-webkit-tap-highlight-color` accent vert global

Donc : si tu utilises Tailwind classes standard, c'est déjà mobile-friendly. Mais y a quelques pièges à éviter.

## Patron type d'une page

```tsx
'use client'

import { Sidebar } from '@/components/sidebar'
import { HubStatus } from '@/components/hub-status'
import { Widget } from '@/components/widget'
import { Tv } from 'lucide-react'

export default function MaPage() {
  return (
    <div className="flex min-h-screen">
      <Sidebar />
      <main className="flex-1 px-4 sm:px-6 lg:px-8 pt-16 lg:pt-6 pb-6 max-w-[1400px] space-y-4">
        <header className="flex items-center justify-between flex-wrap gap-3">
          <div>
            <h1 className="text-2xl font-semibold tracking-tight flex items-center gap-2">
              <Tv size={20} className="text-accent" />
              Mon titre
            </h1>
            <p className="text-xs text-ink-400 mt-0.5">Sous-titre</p>
          </div>
          <button className="px-3 py-2 rounded-md text-xs font-semibold bg-accent/15 border border-accent/40 text-accent">
            Action
          </button>
        </header>

        {/* contenu */}

        <HubStatus />
      </main>
    </div>
  )
}
```

**Points-clés** :
- `pt-16 lg:pt-6` : reserve la place pour le hamburger mobile en haut
- `pb-6` : combiné avec `pb-[60px]` du body, laisse la place pour la bottom-nav mobile
- `flex-wrap gap-3` sur le header : les boutons passent en dessous du titre sur petit écran
- `space-y-4` : espacement vertical cohérent entre les sections

## Anti-patterns à éviter

### ❌ Tables sans `overflow-x-auto`

```tsx
{/* MAUVAIS — overflow horizontal sans scroll, table hors écran */}
<table className="w-full">
  <thead><tr><th>Date</th>...<th>Description longue</th></tr></thead>
</table>
```

✅ Solution :

```tsx
<div className="panel overflow-x-auto">
  <table className="w-full text-sm min-w-[520px]">
    <thead className="bg-ink-800/50 text-[11px] uppercase tracking-wider text-ink-400">
      <tr>
        <th className="text-left px-3 sm:px-4 py-2 font-medium">Date</th>
        <th className="text-left px-3 sm:px-4 py-2 font-medium hidden md:table-cell">Détail</th>
        <th className="text-right px-3 sm:px-4 py-2 font-medium">Montant</th>
      </tr>
    </thead>
    {/* lignes : padding sm:px-4, max-w[200px] truncate sur description */}
  </table>
</div>
```

Astuce : utilise `hidden md:table-cell` pour masquer les colonnes secondaires sur mobile et `max-w-[200px] truncate` sur les colonnes texte.

### ❌ Grid trop dense

```tsx
{/* MAUVAIS — 6 colonnes même sur 375px = 50px par cellule, illisible */}
<div className="grid grid-cols-6 gap-2">
  {/* tiles */}
</div>
```

✅ Solution :

```tsx
<div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-6 gap-2">
  {/* tiles avec min-w-0 + truncate */}
</div>
```

### ❌ Boutons icon-only sans size confortable

```tsx
{/* MAUVAIS — w-7 h-7 = 28x28px, en-dessous du seuil 44px Apple HIG */}
<button className="w-7 h-7"><X /></button>
```

✅ Solution :

```tsx
<button className="w-10 h-10 sm:w-7 sm:h-7 flex items-center justify-center" aria-label="Fermer">
  <X size={16} />
</button>
```

40px sur mobile, 28px sur desktop.

### ❌ Fixed widths qui débordent

```tsx
{/* MAUVAIS — w-48 = 192px, mais l'écran fait 360px et le panel a déjà du padding */}
<input type="text" className="w-48" />
```

✅ Solution :

```tsx
<input type="text" className="w-full sm:w-48" />
```

### ❌ Filtres sur 1 ligne flex sans wrap

```tsx
{/* MAUVAIS — 4 sélecteurs côte-à-côte sur mobile = chaos */}
<div className="flex gap-2">
  <select>...</select>
  <select>...</select>
  <select>...</select>
  <input type="date" />
</div>
```

✅ Solution :

```tsx
<div className="flex flex-col sm:flex-row sm:flex-wrap gap-2 sm:items-end">
  {/* éléments */}
</div>
```

Ou alternative pour un groupe de chips/tabs scrollable :

```tsx
<div className="tabs-scrollable">  {/* utility class globale */}
  {options.map(o => (
    <button className="px-3 py-2 rounded-md text-xs whitespace-nowrap shrink-0">
      {o.label}
    </button>
  ))}
</div>
```

### ❌ Grosses metric values sans truncate

```tsx
{/* MAUVAIS — un montant à 6 chiffres déborde sur cellule 100px */}
<div className="metric">{formatCurrency(largeValue)}</div>
```

✅ Solution :

```tsx
<div className="metric truncate">{formatCurrency(largeValue)}</div>
{/* ou réduit sur mobile */}
<div className="text-base sm:text-lg font-bold truncate">{value}</div>
```

## Patterns réutilisables du hub

### KPI strip

```tsx
<div className="grid grid-cols-2 sm:grid-cols-4 gap-2 sm:gap-3">
  <KpiTile icon={<Film size={14} />} label="Films" value={String(stats.movies)} />
  {/* ... */}
</div>
```

### Card hover-only sur desktop

```tsx
<div className="ga-card ga-card-hover p-3">
  {/* ... */}
</div>
```

`ga-card-hover` ajoute `transition-colors hover:border-ink-700` mais sur mobile il n'y a pas de hover, donc c'est juste un coût neutre.

### Modal plein écran sur mobile, centered sur desktop

```tsx
<div className="fixed inset-0 z-50 flex items-end sm:items-center justify-center p-0 sm:p-4">
  <div className="bg-ink-900 w-full sm:max-w-md rounded-t-xl sm:rounded-xl">
    {/* ... */}
  </div>
</div>
```

Sur mobile : drawer bottom-sheet (items-end, w-full, rounded-t). Sur desktop : modal centré classique.

### Form vertical sur mobile, horizontal sur desktop

```tsx
<form className="flex flex-col sm:flex-row gap-2 sm:items-end">
  <div className="flex-1">
    <label className="block text-xs mb-1">Nom</label>
    <input className="w-full bg-ink-800 border border-ink-700 rounded-md px-3 py-2 text-sm" />
  </div>
  <button className="px-4 py-2 rounded-md bg-accent/20 text-accent w-full sm:w-auto">
    Submit
  </button>
</form>
```

## Test avant push

1. **Build prod** : `npm run build` sans warning lint
2. **Mobile dev tools** : Chrome → F12 → toggle device toolbar → iPhone 13 Pro (390x844)
3. **Test physique** : ouvre `https://hubperso.com/<ta-page>` sur ton tel, vérifie :
   - Tous les boutons cliquables sans zoom
   - Tap-highlight vert visible sur tap
   - Pas de scroll horizontal global (seul les tables peuvent overflow-x-auto)
   - Le bottom-nav reste visible et la page n'est pas cachée derrière

## Liens

- `globals.css` règles mobile : ligne 110-155 `@media (max-width: 1023px)`
- Bottom nav : `components/mobile-bottom-nav.tsx`
- Exemples concrets : `app/locations/page.tsx`, `app/finances/page.tsx`, `app/streaming/page.tsx`
