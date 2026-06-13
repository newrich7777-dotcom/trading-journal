# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

This project is a single-file web app — no build step, no package manager, no framework.

- **Run locally**: Open `index.html` directly in a browser, or serve it with any static file server:
  ```
  npx serve .
  python3 -m http.server 8080
  ```
- **Deploy**: Hosted on Vercel. Push to `main` to trigger a deploy.

There are no tests, no linting config, and no CI pipeline.

## Architecture

The entire application lives in a single `index.html` file (~1850 lines) with inline CSS and JavaScript. There is no bundler, no module system, and no external JS dependencies beyond the Firebase SDK loaded via CDN.

### Firebase backend
- **Auth**: Google OAuth via `firebase.auth()`. Desktop uses `signInWithPopup`, mobile uses `signInWithRedirect`. In-app browsers (KakaoTalk, Instagram, etc.) are blocked with a custom alert.
- **Firestore data path**: `users/{uid}/journal/data` — a single document per user containing `trades[]`, `weekplan`, and `config`.
- **Chart images** are intentionally excluded from Firestore to keep the document small. They are stored as base64 strings in `localStorage` under the key `ljh-charts` (a `{id: base64}` map) and merged with trade objects at load time via `mergeLocalCharts()`.
- Firebase config (`FB_CONFIG`) is hardcoded in the file with the project's public API key.

### Page structure
The app has 5 tab-based pages rendered via CSS `display:none/block` toggling (`goPage(name)`):

| Tab | Page ID | Purpose |
|-----|---------|---------|
| 주간 플랜 | `page-weekly` | Weekly strategy, trend direction, weekly PnL progress |
| 체크리스트 | `page-checklist` | Pre-entry 5-condition checklist; shows GO/NO-GO result |
| 진입 기록 | `page-entry` | Trade entry form including discipline check chips |
| 히스토리 | `page-history` | Trade history with stats and filterable trade cards |
| 90일 | `page-disc` | 90-day discipline challenge tracker |

### Key state variables (module-level globals)
- `trades[]` — in-memory array of trade objects; source of truth for all rendered data
- `challengeStart`, `riskPct`, `dailyLossLimit` — 90-day challenge config, persisted to Firestore under `config`
- `discState = { check:{}, hard:{} }` — discipline chip state for the trade form currently being filled
- `cState`, `sState`, `waveChoice`, `formDir`, `formPnl`, `weekTrend` — UI state for checklist and entry form

### Data flow
1. `auth.onAuthStateChanged` fires → if signed in, calls `fbLoad()` then `mergeLocalCharts()` then renders all pages
2. All mutations (`saveTrade`, `saveWeekPlan`, `setChallengeStart`, `setRiskCfg`) call `fbSave()` which writes the full `trades` array (minus chart fields) back to Firestore in one `set({ merge: true })` call
3. Chart images are written to `localStorage` separately via `saveChartLocally(id, base64)`

### Discipline scoring
`tradeDiscScore(trade)` returns `0` if any hard rule is violated, otherwise the percentage of soft check rules satisfied. Both `HARD_RULES` and `CHECK_RULES` are plain objects mapping key → Korean label. The score is stored on each trade as `discScore` at save time.

### Image handling
Chart images are compressed before storage: max 900px wide, JPEG at 72% quality (`compressImage()`). Paste via `Ctrl+V`, drag-and-drop, and file picker are all supported on the entry page.
