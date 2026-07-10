# Catholics.app Trivia — Local Demo (HTML/CSS/JS mockup)

A **feel prototype** of the "You Don't Know Jack"-style countdown trivia game.
This is the fast, client-side mockup requested for evaluating gameplay pacing
and tension *before* the production .NET 9 / Blazor build (spec in
`catholics-app-demo-prompt.md`).

No framework, no build step, no server, no dependencies. One file.

## Run it

Just open the file in a browser:

```bash
open index.html          # macOS
```

Or serve it (optional — not required, but avoids any file:// quirks):

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Intro splash

On launch a short **video splash** (`intro.mp4`) plays full-bleed for ~3 seconds (autoplay,
muted, tap-to-skip) and then fades into the Home screen. Adjust the duration with the
`SPLASH_MS` constant in `index.html`.

## Game structure (5-round campaign)

The demo plays as a **5-round campaign** with one running bank across all rounds, drawing from a
bank of **~115 questions** (categories: Bible, Saints, Doctrine, History, Liturgy, Popes, and more;
sources: USCCB, New Advent, Vatican.va, EWTN, CCC, Scripture). Each round serves fewer than its
pool, so testers see fresh questions on replay; one full playthrough is **46 questions**:

1. **Round 1 — Catholic Trivia** (10 of 60 Q, six categories) · mid-round **Bonus Wheel** (loose $200 … BINGO $2,000)
2. **Round 2 — Bible Stories** (8 Q · Noah, Jonah, Moses…) · mid-round **Manna reaction game**
3. **Round 3 — The Chosen, Season 1** (8 Q) · mid-round **Bonus Wheel**
4. **Round 4 — American Catholics** (10 Q · Fr. Mike Schmitz, Jonathan Morris, Seton, JFK, Fulton Sheen…) · mid-round **Match-3 game**
5. **Round 5 — The Blessed Mother** (10 of 12 Marian Q) · mid-round **Mother Mary Match-3 (3 tiers)**

Between rounds a **"Next Round"** screen asks *Easier / Same / Harder* — and, as a running
gag, always "goes easier" no matter what you pick. **Play Again** only appears after Round 5.

### Mini-games / easter eggs
- **Match-3 (Round 4):** a gentle Candy-Crush-style board (6×7) of religious symbols
  (🐟 🍞 ✝️ 🍇 🕊️ 🔥) that "fall from heaven." Tap a symbol then a neighbor to swap; **15 moves**;
  match 3+ to clear and cascade. Reward $50–$200 (big cascade → BINGO $200).
- **Mother Mary Match-3 (Round 5):** a longer, reverent **3-tier** version (🌹 👑 ⭐ 🕊️ 💙 ⚜️)
  with themed tiers — Annunciation → Queen of Heaven → Our Lady of Victory. Reach each tier's goal
  to advance; complete all three for a **$500 BINGO**. Future ideas: **MOTHER_MARY_GAME_ROADMAP.md**.
- **☕ Holy Coffee:** a special wheel segment — an overflowing blessed cup (+$150).
- **🤧 Saintly Sneeze:** a rare surprise after a wheel spin — a saint sneezes, "Bless you!" (+$75).

## Install as an app (PWA)

The prototype now ships a web manifest + service worker, so it's installable and PWABuilder-ready.
The service worker only registers over http(s), so serve it (below) — then use your browser's
**Install / Add to Home Screen**. (`file://` still runs the game; it just won't install offline.)

## Scoring rules

- Options shuffled per serve; the engine tracks the correct post-shuffle index; **no repeats** in a round.
- **Countdown per question:** hold **$1,000** for 1s → drain **$100/second** to **$0** →
  sit at **$0** for 1s (the value flickers) → drain to the **−$100** floor → auto-timeout at 16s.
- **350ms grace window** before the drain begins; **all money is integer cents** (no floats).
- **Correct** → bank the live value (never below $0).
- **Wrong before the clock hits $0** → **no penalty** ($0). **Wrong at/after $0** → flat **−$100**.
- **Timeout** → flat **−$100**. (No more big negative swings from fast wrong guesses.)
- **Bonus Wheel:** weighted spin; BINGO ($2,000) is rare.
- **Manna game:** tap falling manna to fill a basket — **under 5s = $200**, **20s+ = $50**, linear between.

## UI (from spec §5)

- Dark YDKJ theme, gold accent for value/bank, red for penalties.
- Countdown meter re-renders every **100ms** (cosmetic only — score is computed
  from timestamps at submit, so a janky render can never change a score).
- Draining progress bar, 2×2 option grid (stacks under 600px), buttons disable
  on click, correct=green / wrong=red on reveal.
- Category chip, running bank chip with floating +$/−$ delta, citation on every
  reveal, 3-2-1 pre-roll, auto-advance, results screen with per-question breakdown.
- Keyboard: keys **1–4** answer; Enter/Space starts a round.

## Engine tests

The pure engine (`RoundEngine`, `valueAtElapsed`) has **zero DOM dependencies**,
mirroring the spec's rule that the engine survives into production untouched.
Self-tests (spec §8) run automatically on page load — **open DevTools console**
to see `[CatholicsApp.Engine self-tests] 10/10 passed`.

## Notes / divergences from the production spec

- Built in plain HTML/CSS/JS instead of .NET 9 / Blazor / MudBlazor, per the
  explicit request for a quick local mockup. The `RoundEngine` class is written
  UI-free and cent-based so its logic ports directly to the C# `RoundEngine`.
- Audio is a small WebAudio-generated tick/buzz (no external assets).
