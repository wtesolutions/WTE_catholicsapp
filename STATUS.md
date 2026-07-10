# Catholics.app Trivia — Project Status

_Last updated: 2026-07-09_

## What this is
A **local, client-side HTML/CSS/JS mockup** ("feel prototype") of a YDKJ-style countdown
trivia game for Catholics.app. Built to evaluate gameplay/pacing before any production build.
The original build spec (a full .NET 9 / Blazor stack) lives in `catholics-app-demo-prompt.md`;
we deliberately built the quick mockup in plain HTML/JS instead, per request.

## Files
- **`index.html`** — the entire app (one self-contained file: engine + UI + data + audio + self-tests). Open it in a browser to run.
- **`README.md`** — run instructions + current rules/structure.
- **`catholics-app-demo-prompt.md`** — the original production spec (source of truth for eventual .NET build).
- **`STATUS.md`** — this file.

## How to run
```bash
open index.html      # macOS — just open in a browser
```
Engine self-tests run automatically on load — open DevTools console to see
`[CatholicsApp.Engine self-tests] N/N passed`.

## Current state — DONE / working
Everything below is implemented and tested (engine assertions pass in Node + in-browser):

### Intro splash
- On load, a full-bleed **video splash** (`intro.mp4`, 720×1280) plays for **~3s** (tunable via
  `SPLASH_MS`), then fades to Home. Autoplays **muted + playsinline** (browser autoplay rule);
  **tap to skip**, and it also dismisses if the clip ends first. A timeout always dismisses it,
  so a blocked/failed video never traps the user. The clip itself is ~8s; the splash caps at 3s.
- `intro.mp4` is precached by the service worker (offline-ready).

### Question bank: **115 questions** across all packs
Six core categories are represented per request — **Bible, Saints, Doctrine, History, Liturgy,
Popes** (plus gag categories, Bible Stories, The Chosen, American Catholics, The Blessed Mother).
Sources cited include **USCCB, New Advent Catholic Encyclopedia, Vatican.va, EWTN, the Catechism
(CCC), and Scripture**. Counts: PACK 68 · PACK2 10 · PACK3 8 · PACK4 14 · PACK5 15 = **115**.
Each round serves fewer than its pool (e.g. Round 4 serves 10 of 14, Round 5 serves 10 of 15),
so testers get **fresh questions on replay**. A full playthrough is **46 questions** (10+8+8+10+10).

### Structure: 5-round campaign, one running bank across all rounds
1. **Round 1 — Catholic Trivia** (10 of 60 Q, six categories) → mid-round **Bonus Wheel** (after Q5)
2. **Round 2 — Bible Stories** (8 Q: Noah, Jonah, Moses, David, Daniel…) → mid-round **Manna reaction game** (after Q4)
3. **Round 3 — The Chosen, Season 1** (8 Q) → mid-round **Bonus Wheel** (after Q4)
4. **Round 4 — American Catholics** (10 Q: Fr. Mike Schmitz, Jonathan Morris, Seton, JFK, Fulton Sheen…) → mid-round **Match-3 game** (after Q5)
5. **Round 5 — The Blessed Mother** (10 of 12 Marian Q) → mid-round **Mother Mary Match-3, 3 tiers** (after Q5)
- Each round opens with an iOS-style **round intro card**.
- Between rounds: **"Next Round"** screen asks Easier/Same/Harder — always "goes easier" (gag).
- **Play Again** only appears after Round 5 (final results).
- ⚠️ Content review: all `citation` fields are present, but Rounds 4 & 5 (and The Chosen) facts should get a **human accuracy pass** before any public demo.
- Note: 5 rounds makes for a long full playthrough with two match-3 breaks — round order/count is trivially adjustable in the `ROUNDS` array if you want a shorter demo.

### Scoring rules (current)
- All money in **integer cents**. Options shuffled per serve; no repeats within a round.
- **Countdown per question:** hold **$1,000 for 2 seconds** → drain **$100/sec** to **$0** →
  sit at **$0** for 1s (value **flickers**) → drain to **−$100** floor → auto-timeout at **16s**.
  (350ms grace before drain begins.)
- Timing landmarks: $500 ≈ 7.35s, $0 ≈ 12.35s, floor −$100 ≈ 14.35s.
- **Correct** → banks live value (never below $0).
- **Wrong BEFORE clock hits $0** → **no penalty ($0)**.
- **Wrong AT/AFTER $0** (flicker zone) → flat **−$100**.
- **Timeout** → flat **−$100**. (No more big negative swings — this was a deliberate change.)
- **Bonus Wheel:** weighted spin, 9 slots incl. loose **$200**, **BINGO $2,000** (rare), and the special **☕ Holy Coffee** ($150).
- **Manna game:** tap falling manna 🍞 to fill a 10-piece basket. **<5s = $200**, **20s+ = $50**, linear between.
- **Match-3 game (Round 4):** gentle Candy-Crush-style board (6×7) of religious symbols (🐟 🍞 ✝️ 🍇 🕊️ 🔥) that "fall from heaven." Tap-select → tap-adjacent-to-swap; **15 moves**; only a matching swap costs a move; no-match swaps revert free; deadlocks reshuffle. Reward maps cleared tiles to **$50–$200** (rounded to $5); a big single cascade (≥8 tiles) → **BINGO $200**. Pure board logic (`makeBoard/findMatches/collapse/…`) is DOM-free and self-tested.
- **Mother Mary Match-3 (Round 5):** a longer, reverent **3-tier** variant on the same board engine, with Marian symbols (🌹 👑 ⭐ 🕊️ 💙 ⚜️) and a Marian-blue board. Tiers — **The Annunciation** (12 moves, clear 15), **Queen of Heaven** (14 / 20), **Our Lady of Victory** (16 / 25) — each open with a themed banner; reach the goal to advance. Reward $100–$400 by total cleared, +$50 per tier goal met, **$500 BINGO** for completing all three. Reuses the shared pure helpers. Future enhancements (Rainbow Rosary Chain, apparition themes, "Hail Mary" combos, "Intercession" booster, reverent art) are captured in **MOTHER_MARY_GAME_ROADMAP.md**.

### Easter eggs (WebAudio + emoji, no assets)
- **🤧 Saintly Sneeze:** rare (~50%, tunable via `SNEEZE_CHANCE`) after a non-jackpot wheel spin — a saint pops in and sneezes, glowing sparkles + floating "Bless you!", ACHOO + gentle choir; small **+$75** manna bonus ("Even saints need blessings!").
- **☕ Holy Coffee:** special wheel segment — steaming mug with a glowing cross on the foam overflowing golden manna, a winking "Fuel for your faith journey!", pour + angelic "ahhh"; **+$150** ("have a blessed cup!").

### UI / look
- Styled as a **native iOS 27 app**: iPhone device frame, Dynamic Island, status bar,
  SF Pro Rounded numerals (`ui-rounded`), iOS dark cards/buttons/pills. Goes full-bleed on narrow windows.
- **Applause** (WebAudio-synthesized) on correct answers; tick/chime/buzz/jackpot sounds. Audio unlocks on first tap.
- **Results screen** redesigned: hero summary (emoji + headline + final bank), 3 stat tiles
  (Correct / Accuracy / Best), and a card list grouped by round (no more overflowing table).

### Design polish pass (demo-readiness, 2026-07-10)
Professional "cozy casual game" pass targeting the core demographic (warm, reverent, early-Candy-Crush
gloss). All CSS/emoji/inline-SVG — still a single offline file:
- **Palette**: deep-plum "chapel at dusk" background (was pure black), plum-tinted cards, creamy
  honey gold, rose & Marian-blue accents, warm white text.
- **Candy-gloss components**: glossy pill buttons (shine + pressed state), soft-lit cards/stat tiles,
  blush hover on answers, candy-pad match-3 tiles with per-tile gloss (blue-tinted on the Mary board).
- **Ambient delight**: drifting warm glow + candlelight specks behind the home hero, soft screen
  vignette, celebratory shine sweep on results (reduced-motion respected; question screen untouched).
- **Demo-readiness fixes from review** (multi-agent audit): home rules now match actual scoring
  (wrong-before-$0 = no penalty); difficulty-gag quips acknowledge the player's pick; SNEEZE_CHANCE
  0.5 → **0.12** + softer sound + halo-face saint; **"Skip this break →"** escape hatch on all three
  mini-games (awards $50, never stuck); $0 flicker slowed from ~7Hz strobe to a gentle 1.2s breathe
  (photosensitivity); softened Manna/coffee copy; "Bring it on" → "Onward!"; Enter restarts from
  results; manifest screenshot now portrait 1080×1920.
- Known/accepted: match-3 clears ✝️/🕊️ symbols (user's chosen symbol set — flagged for a
  target-audience gut-check); intro.mp4 (3 MB) is precached by the SW.

### Recent fixes
- **Round 3 wheel bug FIXED:** spin button had a duplicate `addEventListener` + `onclick` that
  double-fired (phantom spins, double awards, broke Round 3). Now uses `onclick` only, reset on each open.
- Bumped pre-countdown hold from 1s → **2s**.

## Engine architecture note
`RoundEngine` + `valueAtElapsed()` are **UI-free / DOM-free** (mirrors the spec's rule that the
engine ports to production C# unchanged). Everything timing-related derives from timestamps, not
tick-counting. The 100ms UI meter is cosmetic only — score is computed authoritatively at submit time.

## PWA groundwork (installable / PWABuilder-ready)
- `manifest.webmanifest` with a stable `id`, name/short_name, `theme_color`, `background_color`,
  and a full **PNG icon set**: `icon-192.png`, `icon-512.png` (also referenced as `maskable`),
  plus `icon.svg` and `apple-touch-icon.png` (180). One promo `screenshot-1.png` (1080×1080).
  Icons/screenshot are generated from `icon-src.svg` / `screenshot-src.svg` (rasterized with
  macOS `qlmanage` + `sips`).
- Minimal cache-first `sw.js` (v4) precaches the app shell + icons + screenshot + `intro.mp4`;
  network-first for the HTML page so edits show on reload.
- **Must be served over http(s)** for the service worker to register: `python3 -m http.server 8000`,
  then open `http://localhost:8000` (not `file://`). GitHub Pages serves HTTPS, so it just works.
- The fake iOS **status bar (time / signal / battery) was removed**; content uses
  `env(safe-area-inset-*)` so it clears the real device notch in standalone mode.
- Store-listing screenshots: `screenshot-1.png` is a branded placeholder that clears
  PWABuilder's warning — replace with **real gameplay screenshots** in the store-submission step
  (or drop PNGs in the folder and add them to the manifest `screenshots` array).

## Native path (documented future work)
The eventual plan toward the stores:
1. **PWA** (this groundwork) → installable web app.
2. **PWABuilder** — package the PWA for the stores.
3. **Capacitor** (Ionic) — wrap the PWA into a native iOS/Android shell for native hardware/device access.
4. Eventual full rebuild in **.NET 9 / Blazor + MudBlazor** (spec in `catholics-app-demo-prompt.md`),
   deployed to the **Apple App Store**.

## Possible next steps (not started)
- Tune-ables if desired: `HOLD_FULL_MS`, `HOLD_ZERO_MS`, `TIMEOUT_MS`, wheel odds/values,
  Manna spawn rate/fall speed/basket size, Match-3 `M3_MOVES`/reward band, `SNEEZE_CHANCE`,
  auto-advance delay (2500ms).
- Open question left with user: should a late-but-correct answer ever pay the negative live value?
  (Currently correct never goes negative.)
- Wheel currently shows dollar values, not player names like the reference image — easy to swap.
- Content review: `citation` fields are present on every question (doctrinal-review discipline);
  The Chosen S1 **and** the new American Catholics (Round 4) facts should get a human accuracy
  pass before any public demo.
