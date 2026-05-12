# Evaluar — Engineering

## Stack
- Single `index.html` at the repo root: inline HTML, CSS, and JS
- No framework, no bundler, no `package.json`, no dependencies
- State persisted in `localStorage`
- Design tokens are CSS variables in `:root`
- Deploys on Vercel as a static site (`vercel.json` → `outputDirectory: "."`, no build)

## Code architecture
The JS block in `index.html` is divided into banner-commented sections (`/* === NAME === */`). Order matches load order; everything is in one IIFE-less global scope.

| Section | Purpose |
|---|---|
| `CONFIG` | Storage keys, default seed data (areas, bimesters, students, indicators) |
| `UTILS` | `esc()` for XSS-safe HTML, `generateId()`, date helpers, `$()` shorthand |
| `DB` | `localStorage` abstraction — load/save data, evaluations, students |
| `STATE` | In-memory app state (`screen`, `setup`, `currentEval`, filters, etc.) |
| `ROUTER` | `showScreen` / `goBack` / `navigate` over a `SCREENS` map of render fns |
| `UI` | Per-screen render functions returning HTML strings (Home, Setup, Evaluation, Summary, History, Students, Config: Students/Indicators/Areas) |
| `HANDLERS` | Single event-delegation listener; routes `data-action` clicks to functions |
| `BUSINESS LOGIC` | Evaluation actions, bulk student import, area/bimester add forms |
| `CSV EXPORT` | Build and download CSV of evaluations |
| `INIT` | Boots the app on `DOMContentLoaded` |

## Conventions
- **Always `esc()` user-controlled values** before interpolating into HTML strings — there is no framework escaping for you.
- **Modular render functions**: each screen returns an HTML string; the router swaps it into `#app`.
- **Event delegation**: clicks are routed by `data-action="..."` attributes; do not add inline `onclick` handlers.
- Use existing design tokens (`var(--primary)`, `var(--surface-alt)`, etc.) instead of hard-coded colors.
- Keep everything in one file. Do not introduce a build step unless absolutely necessary.

## Quality Gates

This project uses the `code-review` skill for pre-push reviews. The following project-specific rules are loaded by the skill in addition to its general checklist:

- No `confirm()` nativo — usar siempre `showConfirm` modal
- `localStorage.setItem`/`removeItem` siempre dentro de try/catch
- No `innerHTML` sin `esc()` (XSS prevention)
- Comparaciones de nombres (áreas, bimestres, alumnos) case-insensitive con `.toLowerCase()`
- No duplicar lógica — extraer helpers si el mismo patrón aparece 2+ veces
- UI mutations deben fluir por state → render, no cambios directos al DOM
- Closures en event handlers capturan índices/IDs al montar, no al disparar
- Onboarding/modals que bloquean UI se ejecutan antes del render principal

_Rules extracted from code review 2026-05-12. Update as the project evolves._

## How to push
```bash
git add index.html
git commit -m "…"
git push origin main
```
Vercel picks up the push and redeploys within ~30s.

## Known issues / things to watch
- Single-file structure means risk of name collisions — keep new helpers prefixed or scoped.
- All state lives in `localStorage` per device; no sync, no backup. A user clearing site data loses everything.
- Default seed data (`DEFAULTS` in CONFIG) is teacher-specific (Spanish primary curriculum). Changing it affects first-run experience only.
