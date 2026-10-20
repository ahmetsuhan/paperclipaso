---
name: design-guide
description: >
  Paperclip UI design conventions. Use when creating or modifying UI
  components, adding frontend pages or features, styling UI elements, or
  updating the /design-guide showcase page.
---

# Paperclip Design Guide

Paperclip's UI is a professional-grade control plane — dense, keyboard-driven, dark-themed by default. Every pixel earns its place.

`DESIGN.md` at the repo root is the source of truth; where this guide differs,
follow `DESIGN.md` and its token-only rule, and run `pnpm check:token-gates`
before committing `ui/` changes. If the `frontend-design` or
`web-design-guidelines` skills are installed, they complement this guide.

---

## 1. Design Principles

- **Dense but scannable.** Maximum information without clicks to reveal. Whitespace separates, not pads.
- **Keyboard-first.** Global shortcuts (Cmd+K, C, [, ]). Power users rarely touch the mouse.
- **Contextual, not modal.** Inline editing over dialog boxes. Dropdowns over page navigations.
- **Dark theme default.** Neutral grays (OKLCH), not pure black. Accent colors for status/priority only. Text is the primary visual element.
- **Component-driven.** Prefer reusable components that capture style conventions. Build at the right abstraction — not too granular, not too monolithic.

---

## 2. Tech Stack

- **React 19** + **TypeScript** + **Vite**
- **Tailwind CSS v4** with CSS variables (OKLCH color space)
- **shadcn/ui** (new-york style, neutral base, CSS variables enabled)
- **Radix UI** primitives (accessibility, focus management)
- **Lucide React** icons (16px nav, 14px inline)
- **class-variance-authority** (CVA) for component variants
- **clsx + tailwind-merge** via `cn()` utility

Config: `ui/components.json` (aliases: `@/components`, `@/components/ui`, `@/lib`, `@/hooks`)

---

## 3. Design Tokens

All tokens defined as CSS variables in `ui/src/index.css`. Both light and dark themes use OKLCH.

### Colors

Use semantic token names, never raw color values:

| Token | Usage |
|-------|-------|
| `--background` / `--foreground` | Page background and primary text |
| `--card` / `--card-foreground` | Card surfaces |
| `--primary` / `--primary-foreground` | Primary actions, emphasis |
| `--secondary` / `--secondary-foreground` | Secondary surfaces |
| `--muted` / `--muted-foreground` | Subdued text, labels |
| `--accent` / `--accent-foreground` | Hover states, active nav items |
| `--destructive` | Destructive actions |
| `--border` | All borders |
| `--ring` | Focus rings |
| `--sidebar-*` | Sidebar-specific variants |
| `--chart-1` through `--chart-5` | Data visualization |

### Radius

Single `--radius` variable (0.625rem) with derived sizes:

- `rounded-sm` — small inputs, pills
- `rounded-md` — buttons, inputs, small components
- `rounded-lg` — cards, dialogs
- `rounded-xl` — card containers, large components
- `rounded-full` — badges, avatars, status dots

### Shadows

Minimal shadows: `shadow-xs` (outline buttons), `shadow-sm` (cards). No heavy shadows.

---

## 4. Typography Scale

Use these exact patterns — do not invent new ones:

| Pattern | Classes | Usage |
|---------|---------|-------|
| Page title | `text-xl font-bold` | Top of pages |
| Section title | `text-lg font-semibold` | Major sections |
| Section heading | `text-sm font-semibold text-muted-foreground uppercase tracking-wide` | Section headers in design guide, sidebar |
| Card title | `text-sm font-medium` or `text-sm font-semibold` | Card headers, list item titles |
| Body | `text-sm` | Default body text |
| Muted | `text-sm text-muted-foreground` | Descriptions, secondary text |
| Tiny label | `text-xs text-muted-foreground` | Metadata, timestamps, property labels |
| Mono identifier | `text-xs font-mono text-muted-foreground` | Issue keys (PAP-001), CSS vars |
| Large stat | `text-2xl font-bold` | Dashboard metric values |
| Code/log | `font-mono text-xs` | Log output, code snippets |

---

## 5. Status & Priority Systems

### Status Colors (consistent across all entities)

Defined in `StatusBadge.tsx` and `StatusIcon.tsx`:

| Status | Color | Entity types |
|--------|-------|-------------|
| active, achieved, completed, succeeded, approved, done | Green shades | Agents, goals, issues, approvals |
| running | Cyan | Agents |
| paused | Orange | Agents |
| idle, pending | Yellow | Agents, approvals |
| failed, error, rejected, blocked | Red shades | Runs, agents, approvals, issues |
| archived, planned, backlog, cancelled | Neutral gray | Various |
| todo | Blue | Issues |
| in_progress | Indigo | Issues |
| in_review | Violet | Issues |

### Priority Icons

Defined in `PriorityIcon.tsx`: critical (red/AlertTriangle), high (orange/ArrowUp), medium (yellow/Minus), low (blue/ArrowDown).

### Agent Status Dots

Inline colored dots: running (cyan, animate-pulse), active (green), paused (yellow), error (red), offline (neutral).

---

## 6. Component Hierarchy

Three tiers:

1. **shadcn/ui primitives** (`ui/src/components/ui/`) — Button, Card, Input, Badge, Dialog, Tabs, etc. Do not modify these directly; extend via composition.
2. **Custom composites** (`ui/src/components/`) — StatusBadge, EntityRow, MetricCard, etc. These capture Paperclip-specific design language.
3. **Page components** (`ui/src/pages/`) — Compose primitives and composites into full views.

**See [references/component-index.md](references/component-index.md) for the complete component inventory with usage guidance.**

### When to Create a New Component

Create a reusable component when:
- The same visual pattern appears in 2+ places
- The pattern has interactive behavior (status changing, inline editing)
- The pattern encodes domain logic (status colors, priority icons)

Do NOT create a component for:
- One-off layouts specific to a single page
- Simple className combinations (use Tailwind directly)
- Thin wrappers that add no semantic value

---

## 7–9. Composition, Interactive, and Layout Patterns

Read [references/patterns.md](references/patterns.md) when building list rows,
grouped lists, property rows, metric grids, budget bars, comment threads, cost
tables, log viewers, hover/focus/disabled states, inline editing, popover
selectors, or anything touching the three-zone `Layout.tsx` shell.

---

## 10. The /design-guide Page

**Location:** `ui/src/pages/DesignGuide.tsx`
**Route:** `/design-guide`

This is the living showcase of every component and pattern in the app. It is the source of truth for how things look.

### Rules

1. **When you add a new reusable component, you MUST add it to the design guide page.** Show all variants, sizes, and states.
2. **When you modify an existing component's API, update its design guide section.**
3. **When you add a new composition pattern, add a section demonstrating it.**
4. Follow the existing structure: `<Section title="...">` wrapper with `<SubSection>` for grouping.
5. Keep sections ordered logically: foundational (colors, typography) first, then primitives, then composites, then patterns.

### Adding a New Section

```tsx
<Section title="My New Component">
  <SubSection title="Variants">
    {/* Show all variants */}
  </SubSection>
  <SubSection title="Sizes">
    {/* Show all sizes */}
  </SubSection>
  <SubSection title="States">
    {/* Show interactive/disabled states */}
  </SubSection>
</Section>
```

---

## 11. Component Index

When you create a new reusable component, add it to
[references/component-index.md](references/component-index.md) and the
/design-guide page, following the file conventions below.

---

## 12. File Conventions

- **shadcn primitives:** `ui/src/components/ui/{component}.tsx` — lowercase, kebab-case
- **Custom components:** `ui/src/components/{ComponentName}.tsx` — PascalCase
- **Pages:** `ui/src/pages/{PageName}.tsx` — PascalCase
- **Utilities:** `ui/src/lib/{name}.ts`
- **Hooks:** `ui/src/hooks/{useName}.ts`
- **API modules:** `ui/src/api/{entity}.ts`
- **Context providers:** `ui/src/context/{Name}Context.tsx`

All components use `cn()` from `@/lib/utils` for className merging. All components use CVA for variant definitions when they have multiple visual variants.

---

## 13. Common Mistakes to Avoid

- Using raw hex/rgb colors instead of CSS variable tokens
- Creating ad-hoc typography styles instead of using the established scale
- Hardcoding status colors instead of using StatusBadge/StatusIcon
- Building one-off styled elements when a reusable component exists
- Adding components without updating the design guide page
- Using `shadow-md` or heavier — keep shadows minimal (xs, sm only)
- Using `rounded-2xl` or larger — max is `rounded-xl` (except `rounded-full` for pills)
- Forgetting dark mode — always use semantic tokens, never hardcode light/dark values
