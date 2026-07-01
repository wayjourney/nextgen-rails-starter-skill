# Tailwind CSS 4 + Vite Ruby + Rails ERB

## The Problem

Tailwind v4 with `@tailwindcss/vite` only scans files under the Vite source tree (`app/frontend` by default). Utility classes written in `app/views/**/*.erb` are invisible unless registered with `@source`.

A broken `@source` path produces CSS with only preflight/base (~4KB) and no utilities like `bg-amber-600`.

## Correct Configuration

`app/frontend/stylesheets/index.css`:

```css
@import "tailwindcss";

@source "../../views/**/*.{erb,html}";
@source "../../helpers/**/*.rb";

@import "./base.css";
```

Paths are **relative to the CSS file**, not the project root.

From `app/frontend/stylesheets/index.css`:
- `../../views` → `app/views`
- `../../helpers` → `app/helpers`

## Common Mistakes

```css
/* BAD — invalid path from a bad generator or copy-paste */
@source "^./^./views/**/*.{erb,html}";

/* BAD — project-root relative; Tailwind resolves from the CSS file */
@source "app/views/**/*.{erb,html}";

/* BAD — missing @source entirely; ERB classes never generated */
@import "tailwindcss";
```

## Verification

After `yarn vite build`, grep the output CSS:

```bash
grep -o 'bg-amber-600\|my-8' public/vite/assets/*.css
```

If utilities are missing, fix `@source` paths first before debugging Vite or layout tags.

## Checklist

- [ ] `vite.config.ts` includes `tailwindcss()` from `@tailwindcss/vite`
- [ ] `application.js` imports `~/stylesheets/index.css`
- [ ] Layout has `vite_client_tag` + `vite_javascript_tag "application"`
- [ ] `@source` points at `app/views` and `app/helpers` with correct relative paths
- [ ] Dev: restart `bin/dev` or `bin/vite dev` after CSS changes; hard-refresh browser
