---
name: saas-redesign
description: Redesigns a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar on the left, consistent spacing, and a polished professional look. Use when asked to redesign the UI, add a sidebar, or modernize the app's layout.
---

# SaaS UI Redesign Skill

Transforms this Vue 3 application from a top-navigation layout to a modern SaaS-style interface with a vertical sidebar, consistent spacing system, and a polished professional look. **All `.vue` file changes MUST be delegated to the `vue-expert` subagent.**

---

## Phase 1 — Discovery (Read-Only)

Before making any changes, read these files to understand the current layout:

1. `client/src/App.vue` — current nav structure, active-state logic, modal management, global styles
2. `client/src/main.js` — all route definitions and component imports
3. `client/src/locales/en.js` — nav label translation keys (e.g. `nav.overview`, `nav.inventory`)
4. One representative view (e.g. `client/src/views/Dashboard.vue`) — understand page-level padding and card structure

Identify:
- Every `<router-link>` in the nav and its `to` path
- The CSS class used for active state (`:class="{ active: $route.path === '...' }"`)
- Any fixed `top: 70px` offsets used for sticky FilterBar or other elements
- Global CSS variables or body-level styles declared in `App.vue`

---

## Phase 2 — Design Spec

Apply this layout and design system when implementing:

### Layout

```
┌──────────────────────────────────────────┐
│  SIDEBAR (240px fixed)  │  MAIN CONTENT  │
│  ─────────────────────  │                │
│  [Logo / App Name]      │  [FilterBar]   │
│                         │                │
│  [Nav Item — active]    │  [Page View]   │
│  [Nav Item]             │                │
│  [Nav Item]             │                │
│  [Nav Item]             │                │
│  ...                    │                │
│                         │                │
│  ─── (spacer) ────      │                │
│  [User Profile]         │                │
└──────────────────────────────────────────┘
```

- Sidebar: **240px wide**, fixed full-height, does not scroll with content
- Main content: takes remaining width (`calc(100vw - 240px)`), scrolls independently
- No top nav bar at all — remove it entirely

### Sidebar Design

```css
/* Sidebar shell */
.sidebar {
  width: 240px;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  background: #0f172a;          /* slate-900 — matches existing design system */
  border-right: 1px solid #1e293b;
  display: flex;
  flex-direction: column;
  z-index: 100;
  overflow-y: auto;
}

/* Logo / brand area */
.sidebar-header {
  padding: 24px 20px 20px;
  border-bottom: 1px solid #1e293b;
}

/* Nav section */
.sidebar-nav {
  flex: 1;
  padding: 16px 12px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

/* Individual nav item */
.sidebar-link {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  border-radius: 8px;
  color: #94a3b8;               /* slate-400 */
  font-size: 14px;
  font-weight: 500;
  text-decoration: none;
  transition: background 0.15s, color 0.15s;
}

.sidebar-link:hover {
  background: #1e293b;          /* slate-800 */
  color: #e2e8f0;
}

.sidebar-link.active {
  background: #1e3a5f;          /* blue tint */
  color: #60a5fa;               /* blue-400 */
}

/* Bottom user section */
.sidebar-footer {
  padding: 16px 12px;
  border-top: 1px solid #1e293b;
}
```

### Main Content Area

```css
.main-wrapper {
  margin-left: 240px;
  min-height: 100vh;
  background: #0f172a;
  display: flex;
  flex-direction: column;
}

.main-content {
  flex: 1;
  padding: 28px 32px;
}
```

### Spacing System

Use these consistent spacing values throughout (replace any ad-hoc pixel values):

| Token | Value | Use |
|---|---|---|
| `--space-xs` | `4px` | Badge padding, tight gaps |
| `--space-sm` | `8px` | Icon gaps, small padding |
| `--space-md` | `16px` | Card padding (small), row gaps |
| `--space-lg` | `24px` | Card padding (standard), section gaps |
| `--space-xl` | `32px` | Page padding, between sections |
| `--space-2xl` | `48px` | Between major page sections |

Add these to the `:root` block in `App.vue`'s global styles.

### Color Updates (additive — do not remove existing vars)

```css
:root {
  /* Sidebar */
  --sidebar-bg: #0f172a;
  --sidebar-border: #1e293b;
  --sidebar-link: #94a3b8;
  --sidebar-link-hover-bg: #1e293b;
  --sidebar-link-hover-text: #e2e8f0;
  --sidebar-link-active-bg: #1e3a5f;
  --sidebar-link-active-text: #60a5fa;
  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;
}
```

### Nav Icons (inline SVG, no icon library)

Add a small SVG icon beside each nav label. Use simple 16×16 paths from this set:

| Route | Icon path (16px viewBox) |
|---|---|
| `/` (Overview) | `M2 10 L8 4 L14 10 V14 H10 V11 H6 V14 H2 Z` (house) |
| `/inventory` | `M2 4h12v2H2zm0 5h12v2H2zm0 5h12v2H2z` (list) |
| `/orders` | `M3 3h10l1 4H2zm0 0v10h10V7` (box/order) |
| `/spending` | `M8 2a6 6 0 100 12A6 6 0 008 2zm0 2v4l3 2` (clock/finance) |
| `/demand` | `M2 12 L6 8 L9 10 L14 4` (trend line) |
| `/reports` | `M3 3h10v10H3zm2 2v6h2V5zm3 2v4h2V7z` (bar chart) |
| `/restocking` | `M8 2v6l3-3m-3 3L5 5M3 14h10` (restock arrow) |

Wrap each icon in:
```vue
<svg width="16" height="16" viewBox="0 0 16 16" fill="none"
     stroke="currentColor" stroke-width="1.5"
     stroke-linecap="round" stroke-linejoin="round">
  <path d="..."/>
</svg>
```

---

## Phase 3 — Implementation Steps

Execute in this order. **All `.vue` edits go through `vue-expert`.**

### Step 1 — Update `App.vue` (vue-expert)

Replace the `<header>` + `<nav class="nav-tabs">` block with a `<aside class="sidebar">` containing:

```vue
<aside class="sidebar">
  <!-- Brand -->
  <div class="sidebar-header">
    <div class="brand-name">Catalyst Components</div>
    <div class="brand-sub">Inventory Management</div>
  </div>

  <!-- Navigation -->
  <nav class="sidebar-nav">
    <router-link
      v-for="item in navItems"
      :key="item.path"
      :to="item.path"
      class="sidebar-link"
      :class="{ active: $route.path === item.path }"
    >
      <svg width="16" height="16" viewBox="0 0 16 16" fill="none"
           stroke="currentColor" stroke-width="1.5"
           stroke-linecap="round" stroke-linejoin="round">
        <path :d="item.icon"/>
      </svg>
      {{ item.label }}
    </router-link>
  </nav>

  <!-- User footer -->
  <div class="sidebar-footer">
    <!-- Move existing ProfileMenu here -->
    <ProfileMenu ... />
  </div>
</aside>

<div class="main-wrapper">
  <FilterBar v-if="showFilterBar" />
  <main class="main-content">
    <router-view />
  </main>
</div>
```

Define `navItems` as a computed or const in `setup()`:

```javascript
const navItems = [
  { path: '/',            label: t('nav.overview'),       icon: 'M2 10 L8 4 L14 10 V14 H10 V11 H6 V14 H2 Z' },
  { path: '/inventory',   label: t('nav.inventory'),      icon: 'M2 4h12v2H2zm0 5h12v2H2zm0 5h12v2H2z' },
  { path: '/orders',      label: t('nav.orders'),         icon: 'M3 3h10l1 4H2zm0 0v10h10V7' },
  { path: '/spending',    label: t('nav.finance'),        icon: 'M8 2a6 6 0 100 12A6 6 0 008 2zm0 2v4l3 2' },
  { path: '/demand',      label: t('nav.demandForecast'), icon: 'M2 12 L6 8 L9 10 L14 4' },
  { path: '/reports',     label: 'Reports',               icon: 'M3 3h10v10H3zm2 2v6h2V5zm3 2v4h2V7z' },
  { path: '/restocking',  label: 'Restocking',            icon: 'M8 2v6l3-3m-3 3L5 5M3 14h10' },
]
```

Remove the old `.top-nav` / `.nav-tabs` CSS. Update the `body` rule to `margin: 0; overflow: hidden` (the `.main-wrapper` handles scrolling).

**Important:** Any `position: sticky; top: 70px` on `FilterBar` must change to `top: 0` (there's no longer a top bar offset).

### Step 2 — Update `FilterBar.vue` (vue-expert, if it has `top: 70px`)

Change `top: 70px` → `top: 0` in its sticky positioning.

### Step 3 — Update global `<style>` in `App.vue`

Replace:
```css
/* OLD */
.top-nav { ... }
.nav-tabs { ... }
.nav-tabs a { ... }
.nav-tabs a.active { ... }
.content-wrapper { margin-top: 70px; }
```

With the sidebar CSS from Phase 2 above, plus:
```css
html, body { height: 100%; margin: 0; overflow: hidden; }
#app { display: flex; height: 100vh; }
```

### Step 4 — Verify no view-level top offsets

Scan `client/src/views/*.vue` for hardcoded `margin-top`, `padding-top`, or `top:` values that compensated for the old nav height. Remove or adjust them.

---

## Phase 4 — Polish Checklist

Before marking complete, verify these visual details:

- [ ] Sidebar is exactly 240px, full-height, does not scroll with main content
- [ ] Active nav item has blue tint background + blue text (`#60a5fa`)
- [ ] Hover state transitions smoothly (0.15s)
- [ ] Brand name visible at top of sidebar
- [ ] User/profile area pinned to bottom of sidebar
- [ ] FilterBar (if present) sits flush at the top of the main content area, no gap
- [ ] Page content has consistent `28px 32px` padding
- [ ] No horizontal scrollbar at 1280px viewport width
- [ ] All routes still work (no 404s)
- [ ] Active state correctly highlights current route (including `/restocking`)

---

## Verification

After implementation, use Playwright (`mcp__playwright__*`) to:

1. Take a screenshot of `http://localhost:3000` — confirm sidebar visible, no top nav
2. Click each nav item — confirm active highlight moves correctly
3. Resize to 1280px width — confirm no horizontal overflow
4. Navigate to `/restocking` — confirm it highlights in sidebar

```
mcp__playwright__navigate({ url: 'http://localhost:3000' })
mcp__playwright__screenshot({})
```

---

## Key Constraints

- Do NOT introduce any new npm packages (no UI libraries, no icon sets)
- Keep all existing functionality intact (modals, language switcher, tasks, filters)
- Maintain the existing color palette (`#0f172a`, `#64748b`, `#e2e8f0`)
- No emojis in the UI
- All `.vue` changes must go through the `vue-expert` subagent
