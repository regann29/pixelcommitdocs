# Pixel-Commit: Architecture

## 1. Chosen tech stack

| Layer | Choice | Why this one |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript strict** | Routing, great Vercel fit, one language end to end |
| Styling | **Tailwind CSS + shadcn/ui** | Fast, consistent, accessible parts you own |
| Grid rendering | **SVG** (not Canvas) | Only ~371 cells; SVG gives free per-cell events, focus and crisp export |
| State | **Zustand** | Tiny, no boilerplate, easy undo/redo history |
| Image work | **Browser Canvas API** | No Python or server needed; images never leave the device |
| Dates | **date-fns** | Small, tree-shakeable, no timezone surprises if kept in UTC |
| Validation | **Zod** | Safely validates imported JSON and share links |
| Tests | **Vitest, Testing Library, Playwright** | Covers logic, components, and real flows |
| Hosting | **Vercel** | Zero-config deploys, preview URLs per PR |
| Later (Ph. 4) | **Auth.js (GitHub provider), Supabase (Postgres), GitHub GraphQL** | Added only when needed |

Deliberately left out of v1: Python, a backend, a database, auth. The engineering rule applied: boring, proven tools, smallest number of moving parts.

## 2. Layered design

```
┌───────────────────────────────────────────────┐
│  UI (components/, app/)                       │  React, Tailwind, shadcn
│  ContributionGrid · PixelEditor · ExportPanel │  no business logic
└───────────────┬───────────────────────────────┘
                │ reads / dispatches
┌───────────────▼───────────────────────────────┐
│  State (store/)                               │  Zustand: pattern, settings,
│  pattern · settings · history (undo/redo)     │  history, ui
└───────────────┬───────────────────────────────┘
                │ calls pure functions
┌───────────────▼───────────────────────────────┐
│  Core (lib/core/)  ← pure, no DOM, no React   │  100% unit-tested
│  text-to-pixel · calendar · levels · schema   │
└───────────────┬───────────────────────────────┘
                │ used by
┌───────────────▼───────────────────────────────┐
│  Adapters (lib/adapters/)                     │  touch browser APIs
│  image-to-pixel (Canvas) · export · share     │
└───────────────────────────────────────────────┘
        Phase 4 only:  lib/server/github.ts, Supabase client (server side)
```
Rule: arrows only point downward. Core never imports UI or the browser.

## 3. Data flow

```
 user input ──► tool (Text / Draw / Image / Template)
                      │
                      ▼
                 Pattern (store) ──► ContributionGrid (SVG preview)
                      │
                      ├──► calendar-generator ──► PlanRow[] ──► Plan table
                      │
                      └──► exporters ──► PNG · SVG · JSON · CSV · ICS
```

## 4. Folder structure

```
pixel-commit/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                 # landing
│   ├── editor/page.tsx
│   ├── templates/page.tsx
│   ├── about/page.tsx
│   └── api/                     # empty in v1; Phase 4 GitHub routes
├── components/
│   ├── grid/ContributionGrid.tsx
│   ├── editor/{PixelEditor,TextTool,ImageTool,TemplatePicker}.tsx
│   ├── panels/{SettingsPanel,PlanTable,ExportPanel}.tsx
│   └── ui/                      # shadcn components
├── lib/
│   ├── core/
│   │   ├── font5x7.ts
│   │   ├── text-to-pixel.ts
│   │   ├── calendar-generator.ts
│   │   ├── levels.ts
│   │   └── schema.ts            # Zod + types
│   ├── adapters/
│   │   ├── image-to-pixel.ts
│   │   ├── share.ts
│   │   └── export/{png,svg,json,csv,ics}.ts
│   └── server/                  # Phase 4 only
├── store/usePatternStore.ts
├── templates/*.json
├── tests/{unit,e2e}/
├── docs/
└── .github/workflows/ci.yml
```

## 5. Key design decisions

| Decision | Choice | Reason |
|---|---|---|
| SVG vs Canvas | SVG | Small cell count; accessibility and export are easier |
| Client-only v1 | Yes | No hosting cost, instant, privacy by default |
| Grid orientation | `cells[col][row]`, rows = Sun..Sat | Matches GitHub's own week/day structure |
| Undo/redo | Snapshot stack (max 50) in store | Patterns are tiny; snapshots are simpler than diffs |
| Share link | Compressed pattern in URL hash | No database needed |
| GitHub data | Server route only, read-only scope | Token never reaches the browser |

## 6. Failure modes

| Path | Realistic failure | Handled by | User sees |
|---|---|---|---|
| Text-to-grid | Text longer than 53 weeks | Truncate + warning | Clear banner |
| Text-to-grid | Unsupported character | Replace with blank, list skipped chars | Notice |
| Image upload | Huge or corrupt file | Size cap (5 MB), try/catch on decode | Error toast |
| Image upload | Fully white image | Detect empty result | "Nothing detected, adjust threshold" |
| Import JSON | Malformed or old version | Zod validation + version check | Error with reason |
| Share link | Tampered or truncated hash | Zod validation | Falls back to empty editor |
| Date mapping | Start date not Sunday, DST shifts | Align to Sunday, compute in UTC | Info line |
| ICS/CSV | Special characters break format | Escape per RFC 5545 / RFC 4180, tested | Opens correctly |
| PNG export | Canvas size limit on phones | Cap scale factor | Slightly smaller file |
| GitHub API (Ph. 4) | Rate limit or expired token | Retry + sign-in prompt | Message, app still usable |

No silent failures: every row above has either a test or visible error handling.

## 7. Security and privacy
- No data leaves the browser in v1; images are processed locally.
- No GitHub write scope, ever. Phase 4 uses `read:user` only, via server routes.
- Content Security Policy headers set in `next.config`.
- Dependencies pinned and checked with `npm audit` plus Dependabot.

## 8. Performance
- Grid is ~371 `<rect>` elements; memoize cells so painting re-renders only changed ones.
- Image processing runs on a small downscaled canvas (max 53×7), effectively instant.
- Code-split the image tool and export modules (load on first use).
- Targets: LCP under 2 s, painting at 60 fps.

## 9. Parallelisation
Independent tracks once Phase 0 is done:
- **A:** `core/` (font, calendar, schema) with tests
- **B:** `ContributionGrid` + store
- **C:** exporters (need only the `Pattern` type)
- **D:** landing page and docs
Merge point: the editor page.

## 10. Implementation checklist
1. Scaffold repo and CI, define `Pattern` schema.
2. Build `core/` with tests.
3. Build `ContributionGrid` and store.
4. Text tool, pixel editor, undo/redo.
5. Settings, themes, templates.
6. Calendar mapping and plan table.
7. Exporters and share link.
8. Image tool.
9. Accessibility pass, e2e tests, deploy to Vercel.
10. (Optional) GitHub login, compare view, Supabase.
