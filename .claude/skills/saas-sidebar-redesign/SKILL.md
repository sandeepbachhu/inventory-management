---
name: saas-sidebar-redesign
description: Redesign a Vue 3 + Vite application's shell from a top navigation bar to a left vertical sidebar with a modern SaaS look, consistent spacing via design tokens, card-based layouts, and a collapsible drawer on narrow screens. Use when asked to restyle or restructure a Vue app's navigation, layout, or visual system, or to make a dashboard look more like a SaaS product.
---

# SaaS Sidebar Redesign (Vue 3)

This skill describes how to move a Vue 3 app from a top nav bar to a left sidebar, and bring its styling onto a shared token system. It was written from the inventory management app (`client/`), which is the worked example. Adapt file names to the target app.

Work in the order below. Each step leaves the app running.

## 1. Audit first

Before changing anything, record the following. The audit is what makes the rest of the work predictable.

| Check | How to find it | Why it matters |
|---|---|---|
| Nav markup and active logic | Search for the top-level `<header>`, `<nav>`, and `router-link` | You will move these into a sidebar |
| Global vs. scoped styles | Look for an unscoped `<style>` in the root component | Global rules leak into every view. Move shared primitives to a base file |
| Fixed-width layouts | Grep for `px` widths, `minmax(NNNpx`, `min-width`, and `table-layout: fixed` | These break first in a narrower content column |
| Anchored dropdowns | Grep for `position: absolute` with `right: 0` | A dropdown that opened down-right from the top nav will open off-screen when moved to a sidebar |
| Sticky offsets | Grep for `sticky` and hardcoded `top:` values | A `top: 70px` offset assumes the old header height |
| Locale label width | Compare nav label strings across locales | CJK labels and long English labels both need room |
| Responsive rules | Grep for `@media` | If none exist, the drawer behavior is new code |
| Hardcoded text | Look for nav or titles written as literals instead of translation keys | Hardcoded labels won't translate |

Record the findings in a short list before editing. Note any component that is not routed or not in the nav, and leave it alone unless asked.

## 2. Create design tokens

Put all shared values in one file, loaded before the app mounts. Use CSS custom properties on `:root`.

- **Neutrals:** ink (darkest text), text, muted text, border, soft border, surface (cards), canvas (page background), hover.
- **Accent:** one accent for the active nav item, links, and primary actions.
- **Status:** success, warning, danger, info, each with a soft background variant.
- **Spacing:** a 4px-based scale (`--space-1` to `--space-8`). Components use these instead of raw rem values.
- **Radius:** small, medium, large.
- **Elevation:** a subtle shadow and a medium shadow.
- **Layout:** sidebar width, max content width, the breakpoint for the drawer, and a z-index scale (sidebar, sticky bar, dropdown, modal).

Keep the existing palette when the brief says so. Changing only tokens is the lowest-risk way to restyle, because views that already reference the old colors keep working until they are migrated.

## 3. Move shared primitives out of the root component

Move `.page-header`, `.card`, `.card-header`, stat tiles, table styles, badges, loading, and error states into a base stylesheet. Import it in the entry file after the tokens.

- Delete the duplicated rules from the root component so there is one definition.
- Keep the global `*` reset in the base file.
- Make grids collapse with `repeat(auto-fit, minmax(min(100%, Npx), 1fr))`. The `min(100%, …)` part stops a fixed minimum from forcing horizontal overflow on phones.

## 4. Restructure the shell

Replace the top header with a sidebar and a content column.

```
.app                      display: flex (row), min-height: 100vh
├── aside.sidebar         width: var(--sidebar-w); sticky; top: 0; height: 100vh
│   ├── logo
│   ├── nav (vertical)    router-links with an icon, label, and active state
│   └── footer            language switcher and profile menu, at the bottom
└── .content-area         flex: 1; min-width: 0
    ├── topbar            shows only the drawer button on narrow screens
    ├── filter bar        sticky top: 0 inside this column
    └── main              max-width: var(--content-max); centered; padded
```

Rules:
- Add `min-width: 0` to the content column. Without it, a wide table forces the whole row to grow.
- Each nav item needs an icon. Use inline SVG with `stroke="currentColor"` so it follows the text color. Do not use emojis (the inventory app's CLAUDE.md forbids them in the UI).
- Keep the active-route check exact (`$route.path === '/x'`) unless the app has nested routes. If it does, use `route.path.startsWith` on the section root.
- Keep logic and data fetching unchanged. This step is layout and markup only.

## 5. Mobile drawer

Below the breakpoint, the sidebar becomes an off-canvas drawer.

- Hold open state in one `ref` in the root component.
- The sidebar is `position: fixed; inset: 0 auto 0 0; transform: translateX(-100%)` when closed and `translateX(0)` when open. Use a transition on `transform`.
- Show a dim overlay behind the open drawer. Clicking it closes the drawer.
- Close the drawer on route change (watch the route) and on Escape (keydown listener, removed on unmount).
- Show the drawer button only below the breakpoint.
- Comment the breakpoint and close logic. These are the non-obvious parts that a future change could break.

## 6. Anchored dropdowns move with the chrome

A dropdown that opened downward from the top bar must now open upward when its trigger sits at the bottom of a sidebar.

- Change `top: calc(100% + 0.5rem); right: 0` to `bottom: calc(100% + 0.5rem); left: 0`.
- Do not put `overflow: hidden` on the sidebar container itself. It clips the dropdown. If the drawer needs overflow control, put it on an inner wrapper.
- Keep the z-index in the dropdown tier so the menu sits above the sticky filter bar.

## 7. Sticky offsets

- Any element that used to stick below the old header needs `top` set to `0` inside the new content column, not a fixed pixel value.
- Remove hardcoded header heights. Use a token if the value must be shared.

## 8. Per-view pass

For each view, check the following. Edit only what breaks.

- Tables wider than their column: wrap them in a container with `overflow-x: auto`. Only the table scrolls.
- Fixed-width grid tracks such as `minmax(450px, …)`: change the minimum to `min(100%, …)`.
- Two-column blocks that need a stacked layout on narrow screens: add a media query at the breakpoint that fits the content column (about 1100px here), not the full viewport.
- Fixed-width inputs: allow them to shrink with `min-width: min(Npx, 100%)`.
- Do not change script logic, data loading, or the markup of shared primitives while doing this pass.

## 9. Localization check

- Replace hardcoded nav and title strings with translation keys, then add the key to every locale file.
- In CJK locales, labels are shorter in character count but wider in glyph width. Let logo and label text wrap rather than truncate.

## 10. Verification

Run these checks. Do not skip the browser checks, because layout bugs appear only when rendered.

1. The production build passes (for example `npm run build` in the client directory).
2. The dev server runs and every route renders.
3. Browser checks, at a desktop width (about 1440px) and a phone width (about 390px), in light mode:
   - Screenshot each route.
   - The page never scrolls horizontally. Only tables scroll, inside their container.
   - The console shows no new warnings or errors.
   - Dropdowns open upward and are not clipped.
   - The filter bar stays pinned while the content scrolls.
   - The drawer opens, closes on overlay click, on Escape, and on route change.
   - Switch to the other locale and check that labels and the logo wrap cleanly.
4. Keyboard: Tab reaches each nav link, and the focus style is visible.

## Checklist for a new project

- [ ] Audit table filled in
- [ ] Tokens file created and loaded first
- [ ] Base stylesheet created, duplicates removed from the root component
- [ ] Sidebar replaces the top header, with icons and active state
- [ ] Content column has `min-width: 0`; main is centered with max width
- [ ] Drawer below the breakpoint, with overlay, route close, and Escape close
- [ ] Dropdowns open upward and left-aligned, with no clipping ancestor
- [ ] Sticky offsets use `top: 0` inside the content column
- [ ] Per-view fixes applied only where something breaks
- [ ] Nav and titles use translation keys in all locales
- [ ] Build passes; browser checks at 1440px and 390px; console clean
