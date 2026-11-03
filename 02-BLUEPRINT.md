# Pixel-Commit: Blueprint

## 1. Product blueprint

### Screens
1. **Landing** (`/`): one-line pitch, live demo grid animating a letter, "Start designing" button.
2. **Editor** (`/editor`): the main workspace.
   - Left: tools (Text, Draw, Image, Templates).
   - Centre: calendar grid preview.
   - Right: settings (start date, weeks, colours, intensity).
   - Bottom: plan table and export bar.
3. **Templates** (`/templates`): gallery, click to open in the editor.
4. **About / honesty page** (`/about`): what the tool does and does not do.

### User journey
Open → pick tool → design appears live → adjust dates and colours → review plan → export.

### User stories (P0)
- As a visitor, I want to type my initials and see them as squares so that I can judge the look instantly.
- As a designer, I want to click squares on and off so that I can fix details.
- As a planner, I want a date for every selected square so that I know which days are work days.
- As a user, I want to download PNG/SVG/CSV/ICS so that I can keep my design.
- As a visitor with no account, I want everything to work without login.

### Acceptance criteria (examples)
- Given the text "AB", when it is entered, then 12 columns are filled using the 5×7 font with a 1-column gap.
- Given text wider than the grid, when entered, then the overflow is shown as a warning and nothing crashes.
- Given a start date that is not a Sunday, when chosen, then the grid aligns back to the previous Sunday and says so.
- Given an exported JSON, when imported, then the grid is identical cell-for-cell.
- Never: any action that sends data to GitHub or creates a commit.

## 2. Data blueprint

```ts
type Level = 0 | 1 | 2 | 3 | 4;          // 0 = empty, 4 = darkest

interface Pattern {
  version: 1;
  name: string;
  rows: 7;                                // fixed: Sun..Sat
  cols: number;                           // weeks, 1..53
  cells: Level[][];                       // cells[col][row]
  theme: "github" | "ocean" | "sunset" | "mono";
}

interface PlanSettings {
  startDate: string;                      // ISO, aligned to Sunday
  weeks: number;
}

interface PlanRow {
  date: string;                           // ISO
  weekday: "Sun" | "Mon" | "Tue" | "Wed" | "Thu" | "Fri" | "Sat";
  selected: boolean;
  level: Level;
  activity: "Work on something real" | "Rest day";
}
```
Rule: `cells[col][row]` maps to `startDate + col*7 + row` days.

## 3. Feature blueprint

| Feature | Input | Output | Core module | Phase |
|---|---|---|---|---|
| Text-to-grid | string | Pattern | `text-to-pixel.ts` + `font5x7.ts` | 1 |
| Pixel editor | pointer events | Pattern | `PixelEditor.tsx` + store | 1 |
| Templates | template id | Pattern | `templates/*.json` | 2 |
| Image-to-grid | File | Pattern | `image-to-pixel.ts` | 2 |
| Date mapping | Pattern + settings | PlanRow[] | `calendar-generator.ts` | 2 |
| Exports | Pattern + plan | file | `export/*.ts` | 3 |
| Share link | Pattern | URL string | `share.ts` (compressed, base64url) | 3 |
| GitHub compare | token | actual counts | `github.ts` (server only) | 4 |

## 4. Algorithm blueprint

**Text-to-pixel:** for each character, copy its 5×7 glyph, add one blank column, move cursor 6 columns right. Capacity at 53 weeks is 8 characters. Space is 3 blank columns.

**Image-to-pixel:** draw to an offscreen canvas scaled to height 7 (keep aspect ratio, cap width at 53) → grayscale → invert if the "dark = filled" toggle is on → map brightness to 5 levels with an adjustable threshold → optional Floyd–Steinberg dithering.

**Calendar mapping:** align start to previous Sunday → loop columns and rows → compute date → build `PlanRow`.

**ICS export:** one all-day `VEVENT` per selected cell, fixed `UID`, summary "Pixel-Commit: work on something real".

## 5. UI blueprint

- **Look:** dark editor surface, GitHub-green accents, rounded squares, mono font for dates.
- **Layout:** three-panel on desktop; on phones, stacked with the grid scrolling sideways inside its own container.
- **Components:** shadcn/ui `Button`, `Tabs`, `Slider`, `Popover`, `Calendar`, `Table`, `Toast`.
- **Accessibility:** every cell is keyboard reachable (arrow keys + Space), visible focus ring, colour never the only signal (levels also have labels), respects reduced-motion.

## 6. Quality blueprint

| Layer | Tool | What it proves |
|---|---|---|
| Pure logic | Vitest | font, mapping, exports, import/export round trip |
| Components | Testing Library | editor toggles, input validation |
| Whole flows | Playwright | type text → export; draw → undo; mobile viewport |
| Accessibility | axe via Playwright | no serious violations |
| Visual | Playwright snapshots | grid and PNG output stay stable |
