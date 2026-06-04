# V-Studios Demo — Landing Page

Démo d'agence web créative. Site vitrine one-page animé, multilingue, avec formulaire multi-étapes fonctionnel. Utilisé pour présenter les capacités techniques et design d'une agence.

---

## Stack technique

| Rôle | Outil | Version |
|---|---|---|
| Framework UI | Vue 3 (Composition API, `<script setup>`) | ^3.4 |
| Build | Vite | ^5.2 |
| Animations | GSAP 3 + ScrollTrigger | ^3.12 |
| Smooth scroll | Lenis (`@studio-freight/lenis`) | ^1.0.42 |
| i18n | vue-i18n | ^9.14 |
| Styles | SCSS dans `<style lang="scss">` (global, non scoped) | via sass ^1.72 |
| Types | TypeScript | ^5.2 |

**Dev server** : `npm run dev` (port par défaut 5173)

---

## Structure des fichiers

```
src/
├── App.vue                   # Composant racine — tout le layout + animations GSAP
├── main.ts                   # Mount + plugin vue-i18n
├── components/
│   └── ContactForm.vue       # Formulaire multi-étapes autonome (3 steps + Web3Forms)
├── i18n/
│   ├── index.ts              # createI18n, locale persistée en localStorage
│   ├── fr.ts                 # Français (langue par défaut)
│   ├── en.ts                 # Anglais
│   └── es.ts                 # Espagnol
└── vite-env.d.ts
```

---

## Identité visuelle

### Palette
```scss
--color-bg:              #050505       // fond noir quasi-pur
--color-text:            #f0f0f0       // texte principal blanc cassé
--color-accent:          #00ffcc       // cyan neon (accent principal)
--color-accent-gradient: linear-gradient(135deg, #00ffcc 0%, #ff00cc 100%)  // cyan → rose
```

### Typographie
- Police : `Inter, Helvetica Neue, sans-serif`
- Titres principaux : `clamp()` responsive, `font-weight: 800`, `letter-spacing: -0.03em`
- Labels/eyebrows : `0.68-0.75rem`, `letter-spacing: 0.2-0.3em`, `text-transform: uppercase`

### Style général
- Dark mode full — fond `#050505` partout
- Glassmorphism léger sur la navbar (pill flottante)
- Cards avec `backdrop-filter` ou fond `#0a0a0a` selon la section
- Borders : `rgba(255,255,255,0.06-0.08)` subtiles
- Shadows profondes : `0 40px 80px rgba(0,0,0,0.6)`

---

## Sections — état actuel

### 1. Navbar (pill glassmorphism flottante)
- **Position** : `fixed top: 1.5rem`, centré, pill `border-radius: 100px`
- **Contenu** : Logo (dot pulsant cyan + "V-STUDIOS DEMO") | Menu centré (3 liens) | Droite : lang-switcher + CTA
- **Comportement** : `navbar--scrolled` au scroll (fond + border cyan renforcés, largeur réduite à 84%)
- **Lang-switcher** : FR / EN / ES en pill, locale persistée localStorage
- **CTA "CONTACT"** : pill cyan, flèche `↗` animée au hover, responsive (icon-only < 600px)
- **Custom cursor** : désactivé dans navbar (`cursor: none` sur `.landing-page *`)

### 2. Hero (100vh)
- **Image fond** : Unsplash `photo-1567359781514-3b964e2b04d6` (fibres abstraites neon)
  - Filtre CSS : `hue-rotate(-65deg) saturate(0.75) brightness(0.88)` → tons bleu foncé/teal
  - Zoom GSAP continu : `scale: 1.15`, `20s`, yoyo infini
  - Parallax scroll : `yPercent: 20` scrub sur ScrollTrigger
- **Overlay** : `gradient(180deg, rgba(5,5,5,0.3) → rgba(5,5,5,0.62) → rgba(5,5,5,0.97))`
- **Grain cinématique** : `hero::after` SVG noise, `opacity: 0.055`
- **Titre** : "V-STUDIOS" / "DEMO" — animation `.char` par `.char` (`stagger: 0.04`, `y: 100%`)
- **CTAs** : "Démarrer un projet" → `#contact` | "Voir nos travaux" → `#gallery`
- **Infobar** : `80+ clients` · `150+ projets` · `fondé en 2019`
- **Scroll indicator** : ligne verticale centrée, goutte cyan animée (bottom ~7rem)

### 3. Process (scroll horizontal pinné)
- **Mécanique GSAP** : section pinnée, `processCardsWrapper` translate en X pendant le scroll vertical
- **Header** : eyebrow "— Méthode" | h2 blanc "Notre Approche" | compteur "3 étapes" + "↔ scroll" à droite
- **3 cards** (`clamp(340px, 58vw, 720px)` × `62vh`) avec :
  - Badge `01/02/03` (pill cyan borderté)
  - Orb coloré unique par card : card 1 = cyan, card 2 = rose, card 3 = violet (via `::before` + `filter: blur(80px)`, amplifié au hover)
  - Titre + description (depuis i18n `process.steps[]`)
  - Watermark géant bottom-right (`opacity: 0.028`)
  - Footer tags séparés par une ligne : `['UX Research', 'Moodboard', 'Workshop']` etc.
  - Hover lift : `translateY(-10px)` + shadow renforcée

### 4. Gallery (grille éditoriale)
- **Layout** : CSS Grid 3 colonnes asymétrique (`1.3fr 0.95fr 1.2fr`, 2 rangées `38vh`)
  - Item 0 (Maison Alvarez) : col 1, span 2 rangées (portrait)
  - Item 1 (Opia Health) : col 2, rangée 1 (landscape)
  - Item 2 (Folia Magazine) : col 2, rangée 2 (landscape)
  - Item 3 (Nox Studio) : col 3, span 2 rangées (portrait)
- **Animation entrée** : GSAP `clipPath: inset(100%→0%)` curtain, stagger 0.18s
- **Overlay au hover** : gradient bottom, badge catégorie cyan, titre + année
- **Images** : Unsplash dans `galleryData[]` (pas picsum)
- **Header** : eyebrow "— Réalisations" + `h2 "Galerie Immersive"` (em en gradient) + compteur + "Voir tout ↗"

### 5. Contact (grille 2 colonnes)
- **Composant isolé** : `ContactForm.vue` — importé dans App.vue
- **Colonne gauche** : email, localisation, disponibilité, réseaux
- **Colonne droite** : formulaire 3 étapes dans une card (`background: rgba(255,255,255,0.02)`, border, border-radius 24px, gradient line top)
  - **Stepper** : cercles numérotés → ✓ rempli, ligne de progression animée
  - **Step 1** : Nom + Email (float labels, validation inline)
  - **Step 2** : 4 project type cards (2×2) + 4 budget pills (2×2), sélection highlighted cyan
  - **Step 3** : Textarea (maxlength 500, compteur char) + submit
  - **Submit** : Web3Forms (clé env `VITE_WEB3FORMS_KEY`) — simule succès si clé absente
  - **Success** : cercle ✓ cyan, animation `pop-in`
  - **Transitions** : `slide-fwd` / `slide-back` entre steps

### 6. Footer
- 3 colonnes : copyright `© 2026 V-STUDIOS DEMO` | `CRAFT SINCE 2019` | Privacy · Terms

---

## Animations GSAP — vue d'ensemble

| Animation | Déclencheur | Technique |
|---|---|---|
| Navbar entrée | `onMounted` | `fromTo y:-20→0, autoAlpha` |
| Hero title chars | `onMounted` | `from y:100%, stagger:0.04` (`.char`) |
| Hero subtitle + CTAs | Suite timeline hero | `autoAlpha + y` |
| Hero bg zoom | `onMounted` | `scale:1.15, 20s, yoyo, repeat:-1` |
| Hero bg parallax | Scroll | ScrollTrigger scrub |
| Process scroll H | Scroll | Pin + translate X scrub |
| Gallery reveal | Scroll (batch) | `clipPath inset 100%→0%` stagger |
| Text reveal global | Scroll (chaque `.reveal-text`) | `autoAlpha + y, top 60%` |
| Cursor | `mousemove` | `gsap.to(cursor, duration:0.12, power3.out)` |

---

## Custom cursor
- Ref : `cursor` dans App.vue (div fixe hors de `<main>`)
- GSAP `x/y` au `mousemove` (`duration: 0.12`)
- `.is-hovering` → expand à 46px sur tous les éléments interactifs (`a, button, .gallery__item-inner, .project-card, .budget-pill, .lang-btn, .nav-link`)
- `cursor: none` sur `.landing-page *`, sauf `input/textarea` → `cursor: text`
- Masqué sur touch (`@media (pointer: coarse)`)

---

## i18n

- 3 langues : `fr` (défaut), `en`, `es`
- Locale persistée dans `localStorage`
- Switcher dans la navbar (pill FR / EN / ES)
- `t('clé')` pour les strings simples, `tm('process.steps')` pour les arrays d'objets
- Clés de contact : étapes, types de projets, budgets, erreurs, success
- **Ne pas ajouter des traductions hardcodées** — tout passe par les fichiers i18n

---

## Variables d'environnement

```env
VITE_WEB3FORMS_KEY=xxxx   # Clé API web3forms.com pour l'envoi du formulaire (optionnel)
```
Sans clé → mode démo, simule un succès après 1.4s.

---

## Patterns importants

### Smooth scroll
Toujours utiliser `scrollTo(target)` (wrapper autour de `lenis?.scrollTo`) — jamais `window.scrollTo` directement.

### CSS
- Tout le CSS est global (non scoped) dans `<style lang="scss">`
- Exception : `ContactForm.vue` a ses propres styles globaux (classes uniques, pas de conflit)
- Convention BEM : `.bloc__element--modifier`
- Variables CSS dans `:root`

### Reveal text
Ajouter `.reveal-text` + `visibility: hidden; opacity: 0` → GSAP le gère automatiquement au scroll.

### Process steps data
Les descriptions viennent de `tm('process.steps')` (vue-i18n) → cast en `Array<{title, description}>`. Les tags sont dans `processTags[]` dans App.vue (données statiques, pas i18n).

### Gallery data
`galleryData[]` dans App.vue contient `{ img: URL Unsplash, year }`. Titres et catégories viennent de `t('gallery.projects.${index}.title')`.

---

## Ce qui reste à faire / idées potentielles

- [ ] Vraie clé Web3Forms (actuellement en mode démo)
- [ ] Preloader / intro animation
- [ ] Page de cas client (routing Vue Router)
- [ ] Section "À propos / Équipe"
- [ ] Dark/light mode toggle
- [ ] Responsive approfondi (hamburger menu mobile)
- [ ] SEO : meta tags, og:image, sitemap
- [ ] Déploiement (Vercel / Netlify)
- [ ] Animation des stats hero (counter up)

---

## Contexte du projet

Site de démonstration pour montrer les capacités d'une agence web créative. Le nom "V-Studios Demo" indique explicitement son statut de démo. Toutes les données (clients, projets, stats) sont fictives. Le formulaire simule un envoi si aucune clé API n'est configurée.
