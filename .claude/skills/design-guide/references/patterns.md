# Paperclip UI patterns

Companion to `../SKILL.md`. Class names below show established conventions;
where a literal palette or bracket class conflicts with the token-only rule in
`DESIGN.md`, use the semantic token instead.

## 7. Composition Patterns

These patterns describe how components work together. They may not be their own component, but they must be used consistently across the app.

### Entity Row with Status + Priority

The standard list item for issues and similar entities:

```tsx
<EntityRow
  leading={<><StatusIcon status="in_progress" /><PriorityIcon priority="high" /></>}
  identifier="PAP-001"
  title="Implement authentication flow"
  subtitle="Assigned to Agent Alpha"
  trailing={<StatusBadge status="in_progress" />}
  onClick={() => {}}
/>
```

Leading slot always: StatusIcon first, then PriorityIcon. Trailing slot: StatusBadge or timestamp.

### Grouped List

Issues grouped by status header + entity rows:

```tsx
<div className="flex items-center gap-2 px-4 py-2 bg-muted/50 rounded-t-md">
  <StatusIcon status="in_progress" />
  <span className="text-sm font-medium">In Progress</span>
  <span className="text-xs text-muted-foreground ml-1">2</span>
</div>
<div className="border border-border rounded-b-md">
  <EntityRow ... />
  <EntityRow ... />
</div>
```

### Property Row

Key-value pairs in properties panels:

```tsx
<div className="flex items-center justify-between py-1.5">
  <span className="text-xs text-muted-foreground">Status</span>
  <StatusBadge status="active" />
</div>
```

Label is always `text-xs text-muted-foreground`, value on the right. Wrap in a container with `space-y-1`.

### Metric Card Grid

Dashboard metrics in a responsive grid:

```tsx
<div className="grid md:grid-cols-2 xl:grid-cols-4 gap-4">
  <MetricCard icon={Bot} value={12} label="Active Agents" description="+3 this week" />
  ...
</div>
```

### Progress Bar (Budget)

Color by threshold: green (<60%), yellow (60-85%), red (>85%):

```tsx
<div className="w-full h-2 bg-muted rounded-full overflow-hidden">
  <div className="h-full rounded-full bg-green-400" style={{ width: `${pct}%` }} />
</div>
```

### Comment Thread

Author header (name + timestamp) then body, in bordered cards with `space-y-3`. Add comment textarea + button below.

### Cost Table

Standard `<table>` with `text-xs`, header row with `bg-accent/20`, `font-mono` for numeric values.

### Log Viewer

`bg-neutral-950 rounded-lg p-3 font-mono text-xs` container. Color lines by level: default (foreground), WARN (yellow-400), ERROR (red-400), SYS (blue-300). Include live indicator dot when streaming.

---

## 8. Interactive Patterns

### Hover States

- Entity rows: `hover:bg-accent/50`
- Nav items: `hover:bg-accent/50 hover:text-accent-foreground`
- Active nav: `bg-accent text-accent-foreground`

### Focus

`focus-visible:ring-ring focus-visible:ring-[3px]` — standard Tailwind focus-visible ring.

### Disabled

`disabled:opacity-50 disabled:pointer-events-none`

### Inline Editing

Use `InlineEditor` component — click text to edit, Enter saves, Escape cancels.

### Popover Selectors

StatusIcon and PriorityIcon use Radix Popover for inline selection. Follow this pattern for any clickable property that opens a picker.

---

## 9. Layout System

Three-zone layout defined in `Layout.tsx`:

```
┌──────────┬──────────────────────────────┬──────────────────────┐
│ Sidebar  │  Breadcrumb bar              │                      │
│ (w-60)   ├──────────────────────────────┤  Properties panel    │
│          │  Main content (flex-1)       │  (w-80, optional)    │
└──────────┴──────────────────────────────┴──────────────────────┘
```

- Sidebar: `w-60`, collapsible, contains CompanySwitcher + SidebarSections
- Properties panel: `w-80`, shown on detail views, hidden on lists
- Main content: scrollable, `flex-1`
