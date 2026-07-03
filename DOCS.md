# InfiniToyota — Code Map

Everything lives in a single file: [index.html](index.html). This doc explains what controls what, so you can find your way around before adding new features.

## 1. Game concept

Players drag/click "insight" cards onto a board and combine pairs of them to discover new, higher-tier insights. There are 4 tiers:

- **Tier 1 (Source)** — the 5 starting data tables, always known.
- **Tier 2/3/4** — derived insights, unlocked by combining two known insights that match a recipe.

## 2. Data model (script, top section)

| Name | Line | What it is |
|---|---|---|
| `STARTING_ITEMS` | ~663 | The 5 Tier-1 items every player starts with. |
| `RECIPES` | ~671 | The master recipe table: `"A+B": "Result"`. This is the single source of truth for what combines into what. |
| `CLEANED_RECIPES` | ~715 | `RECIPES` with each key's two ingredients alphabetically sorted, so `"A+B"` and `"B+A"` both resolve. All lookups during play use this, not `RECIPES` directly. |
| `CONCEPT_TIERS` | ~722 | Computed map of `itemName → tier (1-4)`. Built once by `calculateAllTiers()`, which walks the recipe graph until tiers stop changing (a fixed-point loop, capped at 100 iterations). |
| `RECIPE_PARENTS` | ~752 | Reverse index: `resultName → [ingredientA, ingredientB]`. Used to render the Discovery Tree. |
| `ALL_UNIQUE` | ~759 | Every item name that exists in the game (starting + all recipe results), deduplicated. |
| `TIER_META` | ~765 | Display info (label, color, light color) per tier, used by the sidebar, tree modal, and Time's Up summary. |

**To add/change recipes:** edit `RECIPES` only. Tiers, totals, and the Discovery Tree all recompute automatically from it.

## 3. Game state (mutable, during play)

| Variable | Line | Meaning |
|---|---|---|
| `discoveredItems` | ~713 | `Set` of item names the current player has unlocked. Reset to `STARTING_ITEMS` on `resetGame()`. |
| `draggedElement` | ~712 | Tracks the DOM element currently being drag-and-dropped. |
| `treeOpenedFromTimeUp` | ~948 | `true` if the Discovery Tree modal was opened *from* the Time's Up popup — controls whether closing the tree returns you to that popup (see §6). |
| `timerInterval` / `countdownEndTime` | ~1024 | The running countdown timer's interval handle and target end timestamp. |

## 4. UI sections (HTML) and their controllers

| Section | HTML id | Controlling JS |
|---|---|---|
| Brand header (logo + wordmark + timer) | `#app-header`, `#brand-mark`, `#game-timer` / `#timer-display` | `startTimer()`, `stopTimer()`, `formatTime()` — toggles `.urgent` on `#game-timer` in the final `TIMER_URGENT_MS` (10s) |
| Start Game gate (hero card with steps + name input) | `#start-overlay`, `#start-game-btn`, `#player-name-input` | `startGameBtn` click handler |
| Sidebar (inventory + stats + progress bar) | `#inventory`, `#base-items-container`, `#progress-track`/`#progress-fill` | `initInventory()`, `updateProgress()` |
| Board (drag/drop canvas) | `#board`, `#main-row` wraps sidebar + board | `board` drop/dragover listeners, `createBoardItem()` |
| Discovery toast (new-insight popup) | `#discovery-toast`, `#toast-name` | `showDiscoveryToast()`, called from the `drop` handler's match branch |
| Confetti burst | `.confetti-piece` (spawned/removed on the fly) | `spawnConfetti()`, called on new discoveries and when the Time's Up modal opens |
| Failed-combo shake | `.board-item.shake` | applied in the `drop` handler's no-match branch, removed on `animationend` |
| Clue chat bubble (bottom-left of board) | `#clue-bubble`, `#clue-text` | `showClue()`, `hideClue()`, `pickClueIngredient()` |
| Instructions ("?" button) | `#instructions-btn`, `#instructions-overlay` | inline listeners |
| Discovery Tree modal | `#modal-overlay`, `#tree-container` | `renderTree()`, `getTierBadgeStyle()` |
| Time's Up modal (with score) | `#timeup-overlay`, `#timeup-summary`, `#timeup-score` | `showTimeUpModal()`, `computeGameResult()`, `submitScore()` |
| Leaderboard modal (medal styling for top 3) | `#leaderboard-overlay`, `#leaderboard-container` | `renderLeaderboard()`, `closeLeaderboardModal()` |

## 5. Core gameplay loop

1. `startGameBtn` click → clears `localStorage`/`sessionStorage`, calls `resetGame()`, `startTimer()`, hides the start overlay. (~1102)
2. Clicking an inventory `.item` → `document` click listener (~868) drops a copy onto the board at a randomized position via `createBoardItem()`.
3. Dragging a board item onto another board item → `board`'s `drop` listener (~887) builds a sorted key from both item names and looks it up in `CLEANED_RECIPES`:
   - **Match** → new result item is created, added to `discoveredItems`, `updateProgress()` refreshes the sidebar counts.
   - **No match** → nothing is added; the two items collapse into whichever name sorts first (a "no-op" merge, not a removal). **This is the "failed attempt" case** — see §7.
4. Double-clicking a board item removes it (~943).
5. When `countdownEndTime` is reached, `startTimer()`'s tick calls `showTimeUpModal()` (~1051), which locks the board behind `#timeup-overlay` and renders a found/total row per tier.
6. `Reset Workspace` (sidebar) and `Close` (Time's Up modal) both call the shared `returnToStartScreen()` (~1114): reset game state, stop the timer, and bring back the Start Game screen.

## 6. Modal stacking rule (tree ↔ time's up)

If a player opens the Discovery Tree **from the Time's Up popup**, closing the tree should return them to that popup instead of just revealing the board. This is done with the `treeOpenedFromTimeUp` flag:

- `timeupTreeBtn` click sets the flag `true` before opening the tree (~1090).
- The sidebar's `treeBtn` click sets it `false` (~950) — normal mid-game tree views don't reopen anything.
- `closeTreeModal()` (~956) is the single exit path for the tree (bound to both the ✕ button and clicking the backdrop) — it checks the flag and re-shows `#timeup-overlay` if needed.

**Gotcha:** every interactive element must have a unique `id`. `document.getElementById()` silently returns only the *first* match in the DOM if two elements share an id — this caused the "Close button does nothing" bug earlier, when the Time's Up close button was accidentally given the same id as the sidebar's Reset button.

## 7. Clue feature

After **3 failed combine attempts** (the no-match branch of the board's `drop` handler), a chat bubble pops up from the bottom-left of the board with a hint in the form `"A + ? = New Insight"`.

- `failedAttempts` increments on each no-recipe merge; a successful match, `resetGame()`, or the clue firing resets it to 0 (so it fires again after 3 more fails).
- `pickClueIngredient()` builds the hint pool: recipes whose result is undiscovered but whose two ingredients are **both** already unlocked ("things you could make right now"). It picks a random such recipe, then reveals a random one of its two ingredients as `A`.
- The bubble (`#clue-bubble`) auto-hides after 8s (`CLUE_VISIBLE_MS`) or immediately on a successful combine. `FAILS_BEFORE_CLUE` controls the threshold.
- If nothing is craftable right now, no clue is shown (the fail counter still resets).

## 8. Leaderboard (Firebase)

Ranks players by **score = (insights found ÷ total insights) × 100 × duration**, where *duration* is the seconds from game start to the **last insight discovered** (`gameStartTime` / `lastDiscoveryTime`). Computed in `computeGameResult()` when time runs out.

- **Name capture:** `#player-name-input` on the Start card is required (red border if empty); stored in `playerName`.
- **Storage:** Firestore collection `leaderboard`, one doc per finished round: `{ name, insightsFound, time, score, createdAt (serverTimestamp) }`. Written by `submitScore()` from `showTimeUpModal()`.
- **Display:** top-10 by `score` desc, via the sidebar "Leaderboard" button or "View Leaderboard" on the Time's Up modal (the latter returns to the Time's Up popup on close — same stacking pattern as §6). Player names are inserted with `textContent`, never `innerHTML`.
- **Backend seam:** a separate `<script type="module">` at the bottom of the file loads the Firebase SDK from the CDN and exposes `window.leaderboardDB = { isConfigured, addScore, getTopScores }` to the main (non-module) game script. All game code checks `window.leaderboardDB?.isConfigured` and degrades gracefully (offline notices) when Firebase isn't set up.

**To activate:** replace the placeholder `firebaseConfig` in that module script with your project's config (Firebase Console → Project settings → Your apps), create a Firestore database, and publish [`firestore.rules`](firestore.rules) (see below).

## 9. Security model (public deploy)

The `firebaseConfig` (including `apiKey`) is committed to the repo **on purpose**. A Firebase *web* API key is **not a secret** — it only identifies the project to Google and grants no data access. Google documents that embedding it in client code is expected. Hiding or rotating it does essentially nothing for security. The real protection is three layers, none of which involve keeping the key private:

1. **Firestore Security Rules** — [`firestore.rules`](firestore.rules) is the source of truth. Public `read` (the leaderboard is meant to be seen), validated **create-only** (payload must be exactly `{ name, insightsFound, time, score, createdAt }` with sane types/bounds and `createdAt == request.time`), and `update`/`delete` **denied** so scores are immutable. Publish via Firebase Console → Firestore → Rules (paste), or `firebase deploy --only firestore:rules`. **Without these rules, anyone can read/overwrite/wipe the collection via a plain REST call — that, not the key, is the real risk.**

2. **App Check (reCAPTCHA v3)** — the module script calls `initializeAppCheck()` so Firestore requests must carry a valid App Check token; bots/curl are rejected. Setup: Firebase Console → App Check → register the web app with reCAPTCHA v3, put the **site key** in `RECAPTCHA_V3_SITE_KEY` (public, safe), keep the **secret** in the console, then **enforce** App Check for Cloud Firestore *after* verifying real plays still work. For local dev, uncomment `self.FIREBASE_APPCHECK_DEBUG_TOKEN = true;` to get a debug token to register for `localhost` (keep it commented out in production).

3. **API key HTTP-referrer restriction** — Google Cloud Console → APIs & Services → Credentials → the Browser key → Application restrictions → HTTP referrers: add `https://<project>.vercel.app/*`, any custom domain, and `http://localhost:*/*`. This makes the exposed key unusable from any other origin.

Deploy hygiene: [`.gitignore`](.gitignore) blocks real secrets (service-account JSON, `.env`) from ever landing in git; [`vercel.json`](vercel.json) sets basic security headers. Note the current key already exists in git history — that's fine given the three layers above; rotating/scrubbing is optional, not required.
