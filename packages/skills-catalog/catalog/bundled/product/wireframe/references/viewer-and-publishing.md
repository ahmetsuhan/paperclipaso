# Wireframe viewer page and publishing

Loaded from `SKILL.md` only when the user asks for a viewer page that bundles the wireframes, or asks to deploy, publish, or host them.

## Website / viewer page

Trigger on phrases like "make a page that shows the screens", "single page I can scroll", "build a viewer", "let me click through the wireframes", "show them all on one page", or any request to bundle multiple wireframes into a browsable artifact (not a production site).

Build **one static `index.html`** that loads the SVG wireframes directly. Do not turn this into a React app or component library — it's a review surface, not product UI.

**File layout** (default):

```
design/<task-slug>/
  index.html
  wireframes/    # the SVGs from this skill
  screenshots/   # any reference screenshots
```

**Page anatomy** (start from `assets/site-template.html` and adjust — do not re-derive the CSS):

- **Sticky sidebar TOC** (240px on desktop) listing every screen with anchor links. Group by Flow / Screens / Open questions.
- **Hero header** at the top: crumb (issue id), title, one-paragraph summary, pill row of meta tags (`12 screens`, `Lo-fi · monochrome`, `Click any wireframe to zoom`).
- **One section per screen** with a 2-column grid: wireframe on the left, reference image + numbered annotations + a "Why this changes" callout on the right. The wireframe `<img>` points to the SVG file directly — do not inline it.
- **Click-to-zoom lightbox** for any element marked `[data-zoom]`. Esc and backdrop click both close.
- **Flow diagram** section near the top that loads `wireframes/flow.svg` full-width.
- **Open questions** section at the bottom for unresolved decisions.

**House style for the viewer** (matches the wireframes themselves):

- Palette: `--bg: #fafaf8`, `--panel: #fff`, `--ink: #111`, `--muted: #666`, `--line: #e5e5e0`, `--accent: #d33` (red dashed callouts only).
- System font stack only: `-apple-system, system-ui, "Segoe UI", sans-serif`. No web fonts.
- 8px-based spacing, `border-radius: 8px` on cards, 1px `--line` borders, no shadows except the hover lift on `.wire`.
- The viewer chrome is allowed to be slightly more polished than the wireframes (subtle hover, rounded cards) — but never colourful. The wireframes themselves stay strictly monochrome.

**Responsive** (verify before reporting done):

- ≥980px: two-column grid, sidebar TOC visible.
- 900–980px: grid stacks to one column, TOC still sidebar.
- <900px (tablet/phone): TOC collapses to a sticky `<details>` disclosure at the top of the page, defaults closed; tapping a link auto-closes it. Sections get `scroll-margin-top: 80px` so anchor jumps clear the sticky bar.
- <560px (phone): tighter type/spacing scale, hero shrinks, lightbox switches from flex-centered to block layout at full viewport width with `touch-action: pinch-zoom` so users can pinch in further.

**Verification** before handing off: open the file in a browser and walk it at 1440×900, 768×1024, and 390×844. Confirm anchor jumps land cleanly, lightbox opens/closes, and SVGs render at the right aspect.

## Deploy / publish / host the wireframes

Defer to the **`here-now` skill** — it owns publishing, anonymous vs. permanent sites, claim tokens, and credentials.

Load the `here-now` skill and follow its `publish.sh` recipe. The shape is:

```bash
cd design/<task-slug>
{path-to-here-now}/scripts/publish.sh .
# → https://{adjective-noun-suffix}.here.now/
```

If `here-now` isn't installed for the current agent, install it (`npx skills add heredotnow/skill --skill here-now -g`) or escalate to whoever owns the agent's skill set. Do not roll your own hosting.

Things to remember when invoking it:

- Publish the **directory containing `index.html` at its root**, not the parent. `index.html` must be at the root of the published tree.
- Without saved credentials the site is **anonymous and expires in 24 hours**. With a saved API key, it's permanent. If the user wants a permanent URL, follow the `here-now` skill's sign-in-code flow — don't fake your way around it.
- For an update, pass `--slug {existing-slug}` so the URL stays stable across review rounds (the script auto-loads the claim token from `.herenow/state.json`).
- Read `publish_result.*` lines from script stderr to determine `auth_mode` and the claim URL — do not read `.herenow/state.json` and present its contents as the source of truth.
- Always share the `siteUrl` from the current run; if anonymous, also share the claim URL and the 24h expiry warning.
