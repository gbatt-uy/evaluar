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

> These rules were extracted from a full code review done on **2026-05-12** and should be updated as the project evolves. Every code change must satisfy this checklist; if a change has to break a rule, call it out explicitly in the commit message so the user can decide.

- **No native `confirm()`** — always use the `showConfirm` modal helper. Native dialogs break the mobile UI style.
- **`localStorage.setItem` / `removeItem` always inside `try/catch`** — surface failures via `toast()` so quota errors don't silently drop user data.
- **No dead code** — no unused variables, functions, CSS classes, or event handlers. If it has no caller/reader, delete it.
- **No `innerHTML` without `esc()`** for user-controlled values. XSS prevention is a manual responsibility here.
- **Name comparisons are case-insensitive** — areas, bimesters, students compared with `.toLowerCase()` for duplicate detection. Still allow case-only renames of the same entry.
- **Don't duplicate logic** — if the same pattern appears 2+ times, extract a helper (e.g. `countValoraciones`, `matchesEval`, `EMPTY_FILTER`).
- **UI mutations go through state → persist → render** — any function that changes app state must update the in-memory model, persist via `DB`, and trigger `render()`. Direct DOM edits are only acceptable when they explicitly mirror state to preserve an animation (e.g. `toggleObs`, the inline label sync in `onObsInput`).
- **Event-handler closures capture at mount, not at fire time** — handlers that depend on `studentIdx`, `currentEval.id`, or other mutable state must capture those values in `attachHandlers` so the listener still saves to the correct record after a re-render swaps the DOM (see `onObsInput` / `onObsBlur`).
- **Blocking UI (onboarding, modals) runs before the first paint** — invoke before the initial `render()` so the overlay doesn't flash on top of an already-painted screen.

## Pre-Push Self-Review

Before every `git push`, Claude Code must:

1. **Diff scan** — run `git diff HEAD~1` (or the appropriate range) and look for:
   - Dead code (variables, functions, CSS classes without callers/users)
   - Duplicated logic that should be a helper
   - `innerHTML` interpolations missing `esc()`
   - Native `confirm()` calls
   - `localStorage` writes outside `try/catch`
   - UI inconsistencies (mixing patterns for the same concept — e.g. some delete flows using `showConfirm` and others using `confirm()`)
   - Race conditions in event handlers (closures reading mutable state at fire time instead of capture time)
   - Variables declared but never read
2. **Triage**:
   - **CRITICAL or HIGH** → fix before pushing, in the same commit if scoped, otherwise as a follow-up commit on the same push.
   - **MEDIUM** → report to the user and ask whether to fix before pushing.
   - **Only LOW** → push and include them in the post-push report.
3. **Report** — summarize what was checked, what was found, and what was fixed in the final message to the user.

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
