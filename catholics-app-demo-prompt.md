# BUILD PROMPT — Catholics.app Trivia Demo (Local, Single Machine)

> **How to use this file:** Paste this entire document into Claude Code (or any capable coding agent) as the build instruction, or follow it manually as a build spec. It produces a fully local, no-auth, no-cloud demo of the adult trivia game so gameplay and pacing can be evaluated before any platform work begins.

---

## 1. Objective

Build a **runnable local demo** of a "You Don't Know Jack"-style countdown trivia game called **Catholics.app Trivia (Demo)**. The demo must:

- Run entirely on localhost with `dotnet run` — **no database, no cloud services, no authentication**.
- Load questions from a bundled JSON file (sample data provided in §7).
- Demonstrate the core loop: a 10-question round with a draining dollar timer, wrong-answer penalties, running bank total, and an end-of-round results screen.
- Be structured so the game engine is a separate, UI-free class library that survives into the production build unchanged.

This is a **feel prototype**. Visual polish matters only enough to judge pacing, tension, and readability. No leaderboards, no multiplayer, no persistence beyond the running session.

## 2. Tech Constraints

| Concern | Requirement |
|---|---|
| Framework | .NET 9, Blazor Web App, `InteractiveServer` render mode |
| UI library | MudBlazor (latest stable) |
| Data | `questions.json` embedded in `wwwroot/data/` — no DB |
| State | In-memory, per-circuit (scoped services) |
| Audio | Optional stub: play a short tick/buzz via `<audio>` if trivially easy; otherwise skip |
| Target OS | Must run on macOS (dev machine is Apple Silicon) via `dotnet run`; no Windows-only APIs |
| Tests | xUnit test project covering the scorer and round state machine |

## 3. Solution Layout

```
CatholicsApp.Demo/
├── CatholicsApp.Demo.sln
├── src/
│   ├── CatholicsApp.Engine/          # class library — NO UI references
│   │   ├── Models/
│   │   │   ├── Question.cs
│   │   │   ├── RoundState.cs
│   │   │   └── AnswerResult.cs
│   │   ├── CountdownScorer.cs
│   │   ├── RoundEngine.cs
│   │   └── QuestionShuffler.cs
│   └── CatholicsApp.Web/             # Blazor Server app
│       ├── Components/
│       │   ├── Pages/
│       │   │   ├── Home.razor        # title screen / start round
│       │   │   ├── Play.razor        # the game screen
│       │   │   └── Results.razor     # end-of-round summary
│       │   └── Game/
│       │       ├── CountdownMeter.razor
│       │       ├── QuestionCard.razor
│       │       └── BankDisplay.razor
│       ├── Services/
│       │   ├── QuestionRepository.cs # loads questions.json
│       │   └── GameSessionService.cs # scoped; owns RoundEngine instance
│       └── wwwroot/data/questions.json
└── tests/
    └── CatholicsApp.Engine.Tests/
```

**Hard rule:** `CatholicsApp.Engine` must have zero references to Blazor, MudBlazor, or ASP.NET. It is pure C#. All timing decisions come from an injected `TimeProvider` so tests can use `FakeTimeProvider` (from `Microsoft.Extensions.TimeProvider.Testing`).

## 4. Game Rules (authoritative spec)

### 4.1 Round structure
- A round = **10 questions** drawn from the loaded pack, shuffled, no repeats within a round.
- Question options are shuffled per serve; the engine tracks the correct option's post-shuffle index.
- Player has a **bank** (running total in cents) starting at $0. Bank may go negative in this demo.

### 4.2 Countdown value
- Each question starts at **$1,000** (`100_000` cents).
- Value drains **$100/second continuously** (compute from elapsed time, not tick counting).
- Value floors at **−$100** (`-10_000` cents). Once at floor, it stays there until answered or timed out.
- Total question lifetime: value hits the floor at 11.0s; question **auto-times-out at 15s** if unanswered.

### 4.3 Scoring
- **Correct answer:** bank += current value at the server-recorded answer instant.
- **Wrong answer:** bank −= current value at that instant (YDKJ rule — a fast wrong guess hurts more).
- **Timeout:** bank −= $100 flat.
- A **grace window of 350ms** is deducted from elapsed time before computing drain (simulates network fairness; keep it in the demo so the numbers match production).
- All money is `int` cents. Never floating point in scoring.

### 4.4 Flow
1. Home screen → "Start Round" button (this tap also unlocks any audio context).
2. Per question: 3-2-1 pre-roll (~1.5s total) → question + 4 options appear → drain starts.
3. On answer or timeout: freeze meter, reveal correct answer with a color flash (green/red), show delta (+$640 / −$430), display the **source citation** in small text ("CCC ¶1213"), auto-advance after 2.5s.
4. After Q10 → Results screen: final bank, per-question breakdown table (question, your answer, correct answer, time taken, delta), "Play Again" button.

### 4.5 Engine API shape (guidance, not gospel)

```csharp
public sealed class RoundEngine
{
    public RoundEngine(IReadOnlyList<Question> pack, TimeProvider clock, int questionsPerRound = 10);

    public RoundState State { get; }                 // NotStarted, AwaitingAnswer, Revealing, Complete
    public int BankCents { get; }
    public int QuestionNumber { get; }               // 1-based
    public ServedQuestion Current { get; }           // body, shuffled options, servedUtc

    public ServedQuestion ServeNext();               // throws if round complete
    public AnswerResult SubmitAnswer(int chosenIndex);  // uses clock.GetUtcNow()
    public AnswerResult Timeout();                   // -$100 flat
    public int CurrentValueCents();                  // for UI polling/render
}
```

`AnswerResult` carries: `WasCorrect`, `DeltaCents`, `CorrectIndex`, `ElapsedMs`, `Citation`.

## 5. UI Spec (MudBlazor)

- **Dark theme** (`MudTheme` with dark palette) — YDKJ energy: near-black background, one strong accent (suggest gold `#D4AF37` for value/bank, deep red for penalties).
- **CountdownMeter**: large centered dollar value (e.g., `MudText Typo="Typo.h2"`) that re-renders every **100ms** via a `PeriodicTimer` in the component, reading `RoundEngine.CurrentValueCents()`. Under it, a `MudProgressLinear` draining from 100→0 over the 11s drain window. Value text turns red when ≤ $0.
- **QuestionCard**: question body in `Typo.h5`; four full-width `MudButton` options in a 2×2 grid (stack 1×4 under 600px). Buttons disable instantly on click. On reveal: correct button gets `Color.Success`, a wrong pick gets `Color.Error`.
- **BankDisplay**: persistent top-right chip showing running bank; animate delta with a brief `+$640` / `−$430` float-up (CSS keyframe is fine).
- **Category chip** above each question (e.g., "Saints", "Scripture", "Gimmick: Saint or Metal Band?").
- Keyboard support: keys **1–4** answer the corresponding option (nice-to-have; skip if it fights the webview later).
- The 100ms UI tick is **cosmetic only** — the engine computes the authoritative value from timestamps at submit time. A janky render must never change a score.

## 6. Question JSON Schema

```json
{
  "packName": "Demo Pack v1",
  "questions": [
    {
      "id": 1,
      "category": "Saints",
      "format": "MC4",
      "body": "Question text?",
      "options": ["A", "B", "C", "D"],
      "correctIndex": 0,
      "citation": "Source reference",
      "audienceTier": "Adult",
      "reviewStatus": "Approved"
    }
  ]
}
```

`correctIndex` refers to the **authored** order; the engine shuffles at serve time. `citation` and `reviewStatus` are required fields even in the demo — the doctrinal-review discipline starts now.

## 7. Sample Question Data (`questions.json`)

Load exactly this content. 18 questions: enough for a full round plus shuffle variety across two plays.

```json
{
  "packName": "Demo Pack v1",
  "questions": [
    { "id": 1, "category": "Sacraments", "format": "MC4",
      "body": "Which sacrament is described as 'the source and summit of the Christian life'?",
      "options": ["Baptism", "The Eucharist", "Confirmation", "Holy Orders"],
      "correctIndex": 1, "citation": "CCC ¶1324; Lumen Gentium 11",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 2, "category": "Saints", "format": "MC4",
      "body": "Which saint is credited with driving the snakes out of Ireland (legendarily, at least — Ireland never had snakes)?",
      "options": ["St. Brigid", "St. Columba", "St. Patrick", "St. Brendan"],
      "correctIndex": 2, "citation": "Hagiographic tradition; noted as legend",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 3, "category": "Scripture", "format": "MC4",
      "body": "How many books are in the Catholic Old Testament?",
      "options": ["39", "42", "46", "50"],
      "correctIndex": 2, "citation": "Council of Trent, Session IV (1546)",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 4, "category": "Church History", "format": "MC4",
      "body": "The Council of Nicaea, which gave us the core of the Nicene Creed, was held in what year?",
      "options": ["A.D. 325", "A.D. 381", "A.D. 451", "A.D. 1054"],
      "correctIndex": 0, "citation": "First Council of Nicaea, A.D. 325",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 5, "category": "Gimmick: Saint or Metal Band?", "format": "MC4",
      "body": "'Sepultura' — canonized saint or metal band?",
      "options": ["Saint", "Metal band", "Both", "Neither"],
      "correctIndex": 1, "citation": "Sepultura: Brazilian metal band, f. 1984",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 6, "category": "Liturgy", "format": "MC4",
      "body": "What liturgical color is used during Advent and Lent?",
      "options": ["Green", "Red", "Violet", "White"],
      "correctIndex": 2, "citation": "GIRM ¶346",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 7, "category": "Scripture", "format": "MC4",
      "body": "Which Gospel begins with 'In the beginning was the Word'?",
      "options": ["Matthew", "Mark", "Luke", "John"],
      "correctIndex": 3, "citation": "John 1:1",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 8, "category": "Papacy", "format": "MC4",
      "body": "Who was the first pope?",
      "options": ["St. Paul", "St. Peter", "St. Linus", "St. Clement"],
      "correctIndex": 1, "citation": "Matthew 16:18; CCC ¶881",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 9, "category": "Saints", "format": "MC4",
      "body": "St. Thomas Aquinas's massive unfinished work of systematic theology is called the…",
      "options": ["Summa Theologiae", "City of God", "Confessions", "Imitation of Christ"],
      "correctIndex": 0, "citation": "St. Thomas Aquinas, Summa Theologiae (1265–1274)",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 10, "category": "Church History", "format": "MC4",
      "body": "The Great Schism of 1054 split the Church between Rome and which other see?",
      "options": ["Alexandria", "Antioch", "Constantinople", "Jerusalem"],
      "correctIndex": 2, "citation": "East–West Schism, 1054",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 11, "category": "Gimmick: Vatican or Fiction?", "format": "MC4",
      "body": "The Vatican has its own ATMs with instructions in Latin. Vatican fact or fiction?",
      "options": ["Vatican fact", "Fiction", "Only on feast days", "Only for cardinals"],
      "correctIndex": 0, "citation": "IOR/Vatican City ATMs historically offered Latin prompts",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 12, "category": "Sacraments", "format": "MC4",
      "body": "How many sacraments does the Catholic Church recognize?",
      "options": ["Five", "Six", "Seven", "Ten"],
      "correctIndex": 2, "citation": "CCC ¶1113",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 13, "category": "Liturgy", "format": "MC4",
      "body": "The word 'Eucharist' comes from a Greek word meaning what?",
      "options": ["Sacrifice", "Thanksgiving", "Bread", "Assembly"],
      "correctIndex": 1, "citation": "CCC ¶1328 (eucharistein)",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 14, "category": "Saints", "format": "MC4",
      "body": "Which saint, a former soldier, founded the Jesuits?",
      "options": ["St. Francis Xavier", "St. Ignatius of Loyola", "St. Dominic", "St. Benedict"],
      "correctIndex": 1, "citation": "Society of Jesus founded 1540; St. Ignatius of Loyola",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 15, "category": "Scripture", "format": "MC4",
      "body": "In the parable of the Prodigal Son, what does the father do when he sees his son returning?",
      "options": ["Waits at the door", "Sends a servant", "Runs to embrace him", "Demands repayment"],
      "correctIndex": 2, "citation": "Luke 15:20",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 16, "category": "Papacy", "format": "MC4",
      "body": "White smoke from the Sistine Chapel chimney signals what?",
      "options": ["A council has ended", "A new pope has been elected", "The conclave has begun", "A papal document was signed"],
      "correctIndex": 1, "citation": "Conclave custom, Universi Dominici Gregis",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 17, "category": "Church History", "format": "MC4",
      "body": "Which 20th-century council (1962–1965) permitted Mass in vernacular languages?",
      "options": ["Vatican I", "Trent", "Vatican II", "Lateran IV"],
      "correctIndex": 2, "citation": "Sacrosanctum Concilium ¶36 (Vatican II)",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 18, "category": "Gimmick: Saint or Metal Band?", "format": "MC4",
      "body": "'Perpetua' — early Christian martyr or metal band?",
      "options": ["Martyr", "Metal band", "Both exist", "Neither"],
      "correctIndex": 0, "citation": "Ss. Perpetua and Felicity, martyred A.D. 203",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 19, "category": "Sacraments", "format": "MC4",
      "body": "Which sacrament can, in an emergency, be validly administered by anyone — even a non-Christian — with the right intention?",
      "options": ["Confirmation", "Baptism", "Anointing of the Sick", "Eucharist"],
      "correctIndex": 1, "citation": "CCC ¶1256",
      "audienceTier": "Adult", "reviewStatus": "Approved" },

    { "id": 20, "category": "Liturgy", "format": "MC4",
      "body": "What is the name of the Sunday that marks the beginning of Holy Week?",
      "options": ["Laetare Sunday", "Gaudete Sunday", "Palm Sunday", "Divine Mercy Sunday"],
      "correctIndex": 2, "citation": "Roman Missal, Palm Sunday of the Passion of the Lord",
      "audienceTier": "Adult", "reviewStatus": "Approved" }
  ]
}
```

## 8. Required Tests (`CatholicsApp.Engine.Tests`)

Use `FakeTimeProvider`. Minimum cases:

1. **Value at t=0** (after grace) = $1,000.
2. **Value at t=5.35s** (5s post-grace) = $500.
3. **Value at t=11.35s+** = −$100 and never lower.
4. **Correct answer** at $700 → bank +70,000 cents.
5. **Wrong answer** at $700 → bank −70,000 cents.
6. **Timeout** → bank −10,000 cents flat, question advances.
7. **Round completes after 10 questions**; `ServeNext()` after completion throws.
8. **No duplicate questions** within a round across 100 seeded shuffles.
9. **Option shuffle** preserves correct-answer tracking (submit shuffled correct index → `WasCorrect == true`).

## 9. Run Instructions (must appear in the generated README)

```bash
dotnet test                          # all green before first run
cd src/CatholicsApp.Web
dotnet run                           # https://localhost:xxxx → Home → Start Round
```

## 10. Explicit Non-Goals (do not build)

- No login, accounts, or persistence between app restarts.
- No SQL Server / EF Core — JSON file only.
- No SignalR hubs beyond Blazor Server's own circuit.
- No multiplayer, leaderboards, daily challenge, or IAP.
- No kids/teen content — but keep `audienceTier` in the schema untouched.

## 11. Acceptance Checklist

- [ ] `dotnet run` from a clean clone starts the game on macOS.
- [ ] Full 10-question round is playable start → results in under 4 minutes.
- [ ] Dollar value drains smoothly and freezes at the exact submitted value.
- [ ] Wrong answers subtract the live value; timeout subtracts $100.
- [ ] Citation displays on every reveal.
- [ ] Results screen shows per-question breakdown and correct final bank.
- [ ] All engine tests pass; engine project has no UI package references.
