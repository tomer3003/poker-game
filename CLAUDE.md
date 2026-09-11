# Poker — Texas Hold'em / Omaha browser game

A single-file, zero-dependency browser poker game. You (the human) play against
bots at a table of up to 9 seats. Two bot systems exist: **difficulty bots**
(easy/medium/hard, equity-driven) and **people bots** (five fixed personalities
modelled on real players from a home game). There's also a headless
**simulation mode** that plays the people bots against each other for thousands
of hands and reports the notable ones with a step-by-step replay.

## Files

| File | What it is |
|---|---|
| `index.html` | The entire game. HTML + CSS + JS in one file, no build step, no dependencies. |
| `CLAUDE.md` | This file. |

Open `index.html` in a browser. That's the whole workflow — there is no bundler,
no package.json, no server required.

**Keep it single-file.** Every feature so far lives in `index.html` and that's
intentional: the user opens it directly from disk. Don't introduce a build step,
npm dependencies, or split into modules unless explicitly asked.

## Layout of `index.html`

One `<style>` block, one `<body>`, one `<script>`. The script is divided by
banner comments — search for `/* ===` to jump between them. In rough order:

- `CONSTANTS` — blinds, stack size, seat cap, bot name pool
- `CARD / DECK` — deck construction, Fisher-Yates shuffle
- `HAND READING` — draw detection, board texture, starting-hand classes (people bots only)
- `HOME-GAME BET SIZE LANDMARKS` — `HOME_OPEN_BB`, `HOME_SCARE_BB`
- `HAND EVALUATION` — `evaluate5`, `evaluate7`, `evaluateOmaha`, `compareHand`, `describeHand`
- `AI HEURISTICS` — `preflopStrength`, preflop percentile table
- `BOT DECISION ENGINE` — Monte Carlo equity, difficulty profiles
- `PERSONALITY BOT ENGINE` — traits, tilt/ego/beer, weighted action scoring, the five personalities
- `GAME STATE` — `game` object, `makePlayer`
- `SEAT / TABLE MANAGEMENT` — add/remove/queue bots, seat rotation helpers
- `HAND FLOW` — `startHand`, `handleAction`, `advancePhase`, `showdown`, `finishHand`
- `POT SETTLEMENT` — side pots
- `AI TURN TRIGGER` — `maybeTriggerAI`
- `RENDER` — all DOM output
- `HAND RECORDER` / `NOTABLE HAND ANALYSIS` — simulation capture and classification
- `SIMULATION MODE` / `SIMULATION UI` — runner, report, replay
- `SETTINGS`, `DRAGGABLE PANELS`, `EVENT WIRING`, `BOOT`

## Core invariants — break these and things get subtly wrong

**Players are identified by `id`, never by array index.** Bots can be added,
removed, or seated mid-hand, so `game.players` indices shift constantly. Use
`idxOf(id)` and the rotation helpers (`nextSeatIndexFrom`, `nextActingIndexFrom`,
`prevSeatIndexFrom`). Any new code holding an index across a mutation is a bug.

**Pot is derived, never stored.** `potAmount()` sums `player.totalContributed`.
This is what makes side pots work. Never keep a running `pot` variable.

**Side pots come from contribution levels.** `settlePot()` builds layers from the
distinct `totalContributed` values and awards each layer only to players eligible
for it. It returns a summary array that both the log and the table banner render.

**`render()` rebuilds seats from scratch** every call. It's cheap enough. Don't
try to do partial DOM updates.

**Blind rotation has special cases.** Dealer/SB/BB advance normally except:
- someone busts from the BB → dealer and SB stay put, only BB advances
- someone busts from the SB → dealer stays put
- the hand after an Omaha-blind hand → dealer, SB and BB all freeze

These are driven by `game.nextHandBlindMode` (`'normal' | 'freezeDealer' |
'freezeDealerAndSB' | 'freezeAll'`), set at hand end and consumed at the next
`startHand`. `'freezeAll'` (Omaha) outranks the bust rules. A people bot that
auto-rebuys keeps its seat, so it does **not** trigger these.

## The two bot systems

Both go through `aiDecide(player)`, which dispatches on whether the player has a
`personality`:

```
aiDecide(p)
  └─ getBotProfile(p)
       ├─ p.personality set → BOT_PERSONALITIES[p.personality]   (ignores difficulty)
       └─ otherwise        → DIFFICULTY_PROFILES[game.botDifficulty]
```

A profile is a plain object of tunable knobs. If it has a `decide` function, that
takes over completely; otherwise `defaultBotDecide` runs. The five people bots all
point their `decide` at the shared `personalityDecide`.

Difficulty is read **at decision time**, never copied onto the player, so changing
the setting affects bots already seated and mid-hand.

### Difficulty bots

Estimate equity by Monte Carlo (`estimateEquity`): deal random opponent hands and
board runouts, count wins, ties as half. Cached per street on `player._eq` and
invalidated at hand start.

**Aggression thresholds are RELATIVE, not absolute.** This is the single most
important thing in this file. Compare equity to a *fair share* of
`1 / (opponents + 1)`:

```js
const fairShare = 1 / (ctx.activeOpponents + 1);
let rel = eq / fairShare;   // 1.0 = average hand, 1.5 = 50% better than average
```

Profiles use `valueRatio` / `raiseRatio` / `trapRatio` against `rel`. Pot-odds
comparisons stay **absolute** (`eq >= toCall / (pot + toCall)`) — that's correct
either way.

> An earlier version used absolute equity thresholds (`raise if eq > 0.68`).
> 0.68 is a monster 9-handed but barely above average heads-up, so bots raised
> nearly every hand, every pot became an all-in war, measured swings hit ±100
> BB/hand, and **medium actually lost to easy**. If you ever add a new
> aggression threshold, express it as a ratio.

Verified win rates (heads-up, stacks reset each hand):

| matchup | winner's rate |
|---|---|
| hard vs easy | +1.59 BB/hand |
| hard vs medium | +1.06 BB/hand |
| medium vs easy | +0.31 BB/hand |

Holds 6-handed too: easy −2.48, medium +1.00, hard +1.49 BB/hand.

### People bots

Five personalities in `BOT_PERSONALITIES`, registered via
`registerBotPersonality(id, profile)`. They need things equity alone can't
express, so `readSpot()` assembles: draws (`readDraws`), board texture
(`boardIsDrawy`), starting-hand class (`readStartingHand`), plus tilt, ego
trigger, beer level and a read on how tight the human plays (`heroIsTight`).

| Bot | Archetype | Signature |
|---|---|---|
| Tomer | The believer | Limps everything, calls everything, chases every draw |
| Ori | The aggressive isolator | Raises over calling, opens far too many weak aces |
| Ben | The overbet merchant | Tight preflop, explosive after; tilts hardest |
| Guy | The scared-money nit | Folds most hands, check-folds misses, bets big to protect |
| Bar | The ego station | Can't check, can't fold, takes raises personally |

`personalityPostflop` builds fold/call/raise scores from traits and picks by
**weighted random**, not max score — that randomness is what makes them read as
human rather than algorithmic.

**Preflop is threshold-based, not weighted-scored.** VPIP targets can't be hit
with weighted scoring, so `personalityPreflop` ranks the hand with
`preflopPercentile()` (a cached table of all 1,326 two-card combos, verified
uniform to ~1%) and plays if `pct >= 1 - vpip`, adjusted by a bet-size ladder.
A bot with VPIP 0.20 plays roughly the top 20% of hands.

Measured VPIP with tilt disabled (260 hands, all five at one table):

| Bot | measured | spec target |
|---|---|---|
| Tomer | 76–80% | 75–90 ✓ |
| Ori | 50–54% | 35–50 ✓ (top edge) |
| Ben | 31–35% | 25–35 ✓ |
| Guy | 27–30% | 15–25 (~5 over) |
| Bar | 69–72% | 60–80 ✓ |

Guy runs looser than spec mostly because of blind defence: he checks free in the
big blind, then calls a later raise, which counts as VPIP. Tightening him further
made him disappear from hands entirely.

### Bet sizing is capped, deliberately

`personalitySize` and `botRaiseTo` both cap bet size (`maxPotFrac`, plus a soft
cap on committing a stack with a bluff). Without caps, five loose bots with
pot-plus sizing commit full stacks nearly every hand — in testing, three bots
busted on the very first hand. Don't remove these caps.

## Bet-size landmarks (`HOME_OPEN_BB`, `HOME_SCARE_BB`)

The personality spec was written for 0.1/0.2 blinds where "a raise to 7" is the
normal open and "20+" is the scare raise. Taken literally that's a 35 BB open,
unplayable at our 50 BB stacks. The *behaviour* is preserved by expressing those
landmarks in big blinds:

```js
const HOME_OPEN_BB  = 3.5;   // normal open — gets called far too widely
const HOME_SCARE_BB = 11;    // the raise that finally folds people
```

`personalityPreflop` uses a graduated ladder between them. Note the ladder must
**not** fire on the unraised big blind (1 BB) — that bug widened every bot's
range by ~5 points of VPIP.

## Simulation mode

`startSimulation(opts)` snapshots the live `game`, builds a fresh table of people
bots with no human, and runs hands in chunks via `setTimeout` so the tab stays
responsive. `finishSimulation` restores the live game exactly.

Three flags make hands run headless and instantly:

- `game.instant` — `schedule()` runs callbacks synchronously instead of on timers
- `game.silent` — `render()` and `log()` return early
- `game.record` — the hand recorder captures everything

Simulation conventions, all deliberate:

- **Stacks capped at `SIM_STACK_CAP` (3,000 ≈ 150BB)**, overflow banked as profit.
  Without a cap, unlimited re-entries inject chips until one bot holds everything
  and pots reach 3,300BB — at which point every BB-denominated threshold in the
  code stops meaning anything.
- **Hold'em only** (`game.simHoldemOnly`). The people bots are hold'em archetypes
  and mixing Omaha in makes equity stats incomparable between hands.
- **`SIM_EQUITY_SCALE = 0.5`** — half the Monte Carlo samples, ~14 hands/sec
  instead of ~8. There's a floor of 30 trials: below that, equity reads are
  garbage and the report claimed people called as "0% underdogs".

Throughput is ~14 hands/sec. 500 hands ≈ 36s, 10,000 ≈ 12 min.

### Hand recorder and classification

`recStartHand` / `recAction` / `recBoard` / `recResult` / `recFinishHand` capture
each hand: hole cards, board per street, and every action with the pot, the price
faced, and **the equity the acting bot actually saw**. Each hand is classified as
it finishes and only notable ones are kept (`NOTABLE_KEEP = 40` per category), so
long runs don't hoard memory.

Because the board runs out, folds can be scored against what *would* have
happened — that's what makes "hero folds" possible.

| Category | Criteria |
|---|---|
| Amazing calls | called ≥3BB with <22% equity and won |
| Hero calls | called ≥4BB on turn/river with a pair or worse, equity <55%, and won |
| Hero folds | folded two pair or better to a ≥5BB bet and *would have lost* |
| Interesting | bad beats (≥75% equity and lost), coolers, quads+, 3-way all-ins, pots ≥60BB |

If you loosen these, check the counts: an early version flagged 85 "amazing
calls" in 120 hands, which is meaningless.

### Replay

`buildReplaySteps` flattens a hand into street markers + actions; `buildReplayStates`
walks the recorded post-action values to reconstruct table state at each step.
Both are pure functions of the recorded hand — no game state involved.

## Testing

There is no test suite. Testing is done with **jsdom harnesses** driving the real
DOM, which works well and caught every significant bug in this project. Pattern:

```js
const { JSDOM } = require('jsdom');
const html = require('fs').readFileSync('index.html', 'utf8');
const dom = new JSDOM(html, { runScripts: 'dangerously', pretendToBeVisual: true });
const { window: w } = dom;
w.setTimeout = fn => fn();          // make the table resolve instantly
await new Promise(r => setTimeout(r, 1500));  // let the initial real timer fire

// `game` is script-scoped, so expose it:
w.eval('window.__g = () => game;');
const g = () => w.__g();
```

Useful overrides for measurement runs:

```js
w.eval('kickBustedPlayers = function(){};');     // no eliminations → chips balance exactly
w.eval('updateTiltAfterHand = function(){};');   // baseline stats without tilt drift
```

Always syntax-check after editing, since a typo in a 3,500-line HTML file is easy
to miss:

```bash
python3 -c "
import re; html=open('index.html').read()
open('/tmp/g.js','w').write(re.search(r'<script>(.*)</script>', html, re.S).group(1))"
node --check /tmp/g.js
```

**Verify chip conservation** in any measurement harness. With eliminations
disabled, total chips must stay constant. A non-zero balance means the harness is
losing track of players (usually because someone got kicked before being tallied),
not that the game is broken.

**Measure win rates with stacks reset each hand.** Letting bots bust makes the
result dominated by who happened to survive rather than who played better.

## Things that look like bugs but aren't

- `game.players` includes the human even when they're busted and sitting out
  (`sittingOut = true`). They occupy a seat but aren't dealt in.
- Chips are *not* conserved when re-entries are enabled — rebuys inject new
  money. That's correct for a rebuy game; judge results by profit and re-entry
  count together.
- `evaluateBest` reads `game.variant`. Anything that evaluates outside the
  current hand (the recorder, the analyser) must use `evaluateFor(variant, ...)`
  and pass the variant explicitly.
- The Omaha blind sits on the dealer at game start but deliberately does **not**
  trigger on hand 1.

## Conventions

- Vanilla JS, no frameworks. No `localStorage` (it's blocked in some embedding
  contexts and was never needed).
- CSS custom properties at `:root` for the felt/rail/card palette. Reuse them
  rather than hardcoding colours.
- Hand descriptions come from `describeHand()`, which has a specific agreed
  format: face cards and aces spelled out (`Jacks`, `ace high`), number ranks as
  numerals (`8's`, `6 high`), quads rendered as `Quad 7's`, and a royal flush
  with no parenthetical. Kicker annotations (`Two Pair (10's and 8's, Ace kicker)`)
  are added by `annotateShowdownLabels` **only** when two or more players share a
  label and actually differ.
- The human's name is highlighted in the log by `highlightHumanName`, which skips
  text inside HTML tags. Don't bypass it with direct `innerHTML` writes to the log.
