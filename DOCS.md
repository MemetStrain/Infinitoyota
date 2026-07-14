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
| `gameMode` | near top of state | `'sandbox'` or `'challenge'` (default). Set by the mode picker on the Start card. Sandbox = no timer, no Time's Up, no score submission, no name/division needed. |
| `playerName` / `playerDivision` | near top of state | Captured from the Start card in Challenge mode only; both go into the leaderboard doc. |
| `treeOpenedFromTimeUp` | ~948 | `true` if the Discovery Tree modal was opened *from* the Time's Up popup — controls whether closing the tree returns you to that popup (see §6). |
| `timerInterval` / `countdownEndTime` | ~1024 | The running countdown timer's interval handle and target end timestamp. |

## 4. UI sections (HTML) and their controllers

| Section | HTML id | Controlling JS |
|---|---|---|
| Brand header (logo + wordmark + timer) | `#app-header`, `#brand-mark`, `#game-timer` / `#timer-display` | `startTimer()`, `stopTimer()`, `formatTime()` — toggles `.urgent` on `#game-timer` in the final `TIMER_URGENT_MS` (10s) |
| Start Game gate (hero card: mode picker, steps, name + division) | `#start-overlay`, `#mode-select`/`.mode-btn`, `#challenge-fields`, `#player-name-input`, `#division-select`, `#start-game-btn` | mode-btn click handlers (toggle `gameMode`, show/hide `#challenge-fields`), `startGameBtn` click handler |
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

0. On the Start card the player picks a mode: **Sandbox** (free play — no timer, no score, no name/division) or **Challenge** (default — timed, score tracked, requires name + division from the LoV). The picker toggles `#challenge-fields` visibility and the Start button label.
1. `startGameBtn` click → in Challenge, validates name + division (red ring on whichever is missing); clears `localStorage`/`sessionStorage`, calls `resetGame()`, hides the start overlay. Challenge also calls `startTimer()`; Sandbox instead hides `#game-timer` (`.hidden`) and never starts a countdown, so the Time's Up modal can't fire.
2. Clicking an inventory `.item` → `document` click listener (~868) drops a copy onto the board at a randomized position via `createBoardItem()`.
3. Dragging a board item onto another board item → `board`'s `drop` listener (~887) builds a sorted key from both item names and looks it up in `CLEANED_RECIPES`:
   - **Match** → new result item is created, added to `discoveredItems`, `updateProgress()` refreshes the sidebar counts.
   - **No match** → nothing is added; the two items collapse into whichever name sorts first (a "no-op" merge, not a removal). **This is the "failed attempt" case** — see §7.
4. Double-clicking a board item removes it (~943).
5. When `countdownEndTime` is reached, `startTimer()`'s tick calls `showTimeUpModal()` (~1051), which locks the board behind `#timeup-overlay` and renders a found/total row per tier.
6. `Reset Workspace` (sidebar) and `Close` (Time's Up modal) both call the shared `returnToStartScreen()` (~1114): reset game state, stop the timer, un-hide `#game-timer` (in case a Sandbox round hid it), and bring back the Start Game screen. This is also the only way a Sandbox round ends.

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

Ranks players by **score = completionScore + efficiencyBonus**, computed in `computeGameResult()` when time runs out. `completionScore = (insights found ÷ total insights) × 5000` is the dominant term. `efficiencyBonus` is a small, capped pace tiebreaker — `0` unless at least `MIN_INSIGHTS_FOR_PACE_BONUS` (3) insights were found, otherwise scaled toward `BONUS_CAP` (80% of one insight's marginal value, so pace alone can never let fewer insights outscore more) by how close the average pace (`durationSeconds ÷ insights found`, where *duration* runs from game start to the **last insight discovered**) is to the `IDEAL_SECONDS_PER_INSIGHT` (6s) benchmark.

- **Challenge mode only:** Sandbox rounds never reach `showTimeUpModal()` (no timer), so nothing is ever submitted from them.
- **Name + division capture:** `#player-name-input` and `#division-select` on the Start card are both required in Challenge (red ring on whichever is missing); stored in `playerName` / `playerDivision`. The division LoV lives in the `#division-select` markup and must stay in sync with the allowlist in `firestore.rules`.
- **Storage:** Firestore collection `leaderboard`, one doc per finished round: `{ name, division, insightsFound, time, score, createdAt (serverTimestamp) }`. Written by `submitScore()` from `showTimeUpModal()`.
- **Display:** top-10 by `score` desc with a Division column, via the sidebar "Leaderboard" button or "View Leaderboard" on the Time's Up modal (the latter returns to the Time's Up popup on close — same stacking pattern as §6). Player names are inserted with `textContent`, never `innerHTML`. Older docs without `division` render as "—".
- **Backend seam:** a separate `<script type="module">` at the bottom of the file loads the Firebase SDK from the CDN and exposes `window.leaderboardDB = { isConfigured, addScore, getTopScores }` to the main (non-module) game script. All game code checks `window.leaderboardDB?.isConfigured` and degrades gracefully (offline notices) when Firebase isn't set up.

**To activate:** replace the placeholder `firebaseConfig` in that module script with your project's config (Firebase Console → Project settings → Your apps), create a Firestore database, and publish [`firestore.rules`](firestore.rules) (see below).

## 9. Security model (public deploy)

The `firebaseConfig` (including `apiKey`) is committed to the repo **on purpose**. A Firebase *web* API key is **not a secret** — it only identifies the project to Google and grants no data access. Google documents that embedding it in client code is expected. Hiding or rotating it does essentially nothing for security. The real protection is three layers, none of which involve keeping the key private:

1. **Firestore Security Rules** — [`firestore.rules`](firestore.rules) is the source of truth. Public `read` (the leaderboard is meant to be seen), validated **create-only** (payload must be exactly `{ name, division, insightsFound, time, score, createdAt }` with sane types/bounds, `division` in the LoV allowlist, and `createdAt == request.time`), and `update`/`delete` **denied** so scores are immutable. Publish via Firebase Console → Firestore → Rules (paste), or `firebase deploy --only firestore:rules`. **Without these rules, anyone can read/overwrite/wipe the collection via a plain REST call — that, not the key, is the real risk.**

2. **App Check (reCAPTCHA v3)** — the module script calls `initializeAppCheck()` so Firestore requests must carry a valid App Check token; bots/curl are rejected. Setup: Firebase Console → App Check → register the web app with reCAPTCHA v3, put the **site key** in `RECAPTCHA_V3_SITE_KEY` (public, safe), keep the **secret** in the console, then **enforce** App Check for Cloud Firestore *after* verifying real plays still work. For local dev, uncomment `self.FIREBASE_APPCHECK_DEBUG_TOKEN = true;` to get a debug token to register for `localhost` (keep it commented out in production).

3. **API key HTTP-referrer restriction** — Google Cloud Console → APIs & Services → Credentials → the Browser key → Application restrictions → HTTP referrers: add `https://<project>.vercel.app/*`, any custom domain, and `http://localhost:*/*`. This makes the exposed key unusable from any other origin.

Deploy hygiene: [`.gitignore`](.gitignore) blocks real secrets (service-account JSON, `.env`) from ever landing in git; [`vercel.json`](vercel.json) sets basic security headers. Note the current key already exists in git history — that's fine given the three layers above; rotating/scrubbing is optional, not required.
