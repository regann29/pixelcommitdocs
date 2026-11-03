# Pixel-Commit: Project Plan

## Primary goal
A person opens the site with no account, draws or types something, sees it on a realistic GitHub-style calendar, and downloads a picture and a day-by-day plan in under two minutes.

## Positioning
A **contribution design and planning tool**. It never writes to GitHub, never creates commits, never backdates anything. Exports are pictures and reminders only. This is a hard rule (see Non-goals).

## Decisions made (from the original questions)
| Question | Decision | Why |
|---|---|---|
| Main purpose | **Both: designer + planner** | Designer is the hook, planner is the useful part |
| First priority | **Letters/words, then custom editor, then image** | Text and editor share one grid model; image is a later, riskier add-on |
| Where it runs | **Web app (responsive)** | Zero install, easiest to share, mobile browsers work |

## Goals (measurable)
1. First design visible in under 30 seconds from landing.
2. Text-to-grid works for A–Z, 0–9 and spaces, up to the visible width.
3. Every export (PNG, SVG, JSON, CSV, ICS) opens correctly in its standard app.
4. App works fully offline-capable in the browser with no login.
5. Lighthouse score 90+ on performance and accessibility.

## Non-goals
- Auto-committing, scheduling commits, or pushing to any repository (misuse risk, against GitHub's intent).
- Backdating or faking activity.
- Native desktop or mobile apps (the web app is responsive instead).
- Accounts and cloud saves in v1.

## Phases

### Phase 0: Foundation (day 1–2)
**Serves the goal:** gives every later phase the same ground to stand on.
- Create repo, Next.js + TypeScript (strict), Tailwind, shadcn/ui, ESLint, Prettier, Vitest, Playwright.
- Define the `Pattern` type (see Blueprint) before writing any UI.
- CI: lint, type-check, unit tests on every push.
- **Verify:** `npm run dev`, `npm test` and CI all green on an empty page.

### Phase 1: Core MVP (week 1–2)
**Serves the goal:** this is the "draw and see it" moment.
- `ContributionGrid` SVG component (7 rows by 53 columns, Sunday-first).
- 5×7 bitmap font data for A–Z, 0–9, and `text-to-pixel`.
- `PixelEditor`: click and drag to paint or erase, undo/redo.
- Live preview sharing one state store.
- **Verify:** typing "REGAN" renders the exact pixels in unit tests; painting 100 cells stays above 55 fps.

### Phase 2: Smart Designer (week 3–4)
**Serves the goal:** more ways to reach a good design quickly.
- Intensity levels 0–4 and four colour themes (GitHub green, ocean, sunset, mono).
- Template library (heart, star, wave, stairs, checkerboard, arrow).
- `calendar-generator`: start date aligned to Sunday, week count, date for each cell.
- `image-to-pixel` using Canvas: resize to 7 rows, grayscale, map to 5 levels.
- **Verify:** every template fits the grid; a known test image produces a stored expected result.

### Phase 3: Plan and Export (week 5)
**Serves the goal:** turns the picture into something the person keeps.
- Plan table (date, weekday, selected, level, suggested activity).
- Export PNG, SVG, JSON, CSV, ICS (all-day reminder events).
- Share-by-link: design encoded in the URL, no server needed.
- **Verify:** CSV opens in Excel and Sheets; ICS imports into Google Calendar and Apple Calendar; JSON re-imports to an identical design.

### Phase 4: GitHub Integration, optional (week 6–7)
**Serves the goal:** compares the plan with real life.
- Sign in with GitHub (read-only `read:user` scope), fetch the contribution calendar via GraphQL.
- Overlay "planned vs actual".
- Supabase for saved designs.
- **Verify:** app still fully works when signed out; token never reaches the browser.

### Phase 5: AI features, later
- Text prompt to pattern, suggestions, auto-fit to the available weeks. Behind a feature flag.

### Final verification (every release)
Type-check, unit tests, Playwright end-to-end flows, export file checks, accessibility scan, manual test on a phone.

## Risks
| Risk | Answer |
|---|---|
| Used to fake activity | Honest copy, no GitHub write access, no commit tooling, documented non-goal |
| GitHub calendar layout changes | Keep layout in one config file; GraphQL data is the source of truth in Phase 4 |
| Image conversion looks bad | Offer threshold slider and manual cleanup in the editor |
| Scope creep | Phases are gated; new ideas go to a parking-lot issue label |

## Open questions
- Public domain name (blocking for launch only).
- Should ICS reminders say "Work on something real"? Recommended yes (non-blocking).
