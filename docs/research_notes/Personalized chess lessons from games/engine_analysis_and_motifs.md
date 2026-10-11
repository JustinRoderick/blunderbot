# Engine analysis, mistake detection, and motif classification for personalized chess lessons

Research date: 2026-10-09. Source code was read directly from shallow clones taken that day:
- `lichess-org/lila` @ `f5b261e` (2026-10-08), branch `master`
- `lichess-org/scalachess` @ `2ab0a40` (2026-10-05), branch `master`
- `lichess-org/fishnet` @ `ba114fd` (2026-10-07), branch `main`
- `ornicar/lichess-puzzler` @ `608cf7f` (2026-09-29), branch `master`. **Note:** `github.com/lichess-org/lichess-puzzler` returns "Repository not found". The puzzler is hosted at `github.com/ornicar/lichess-puzzler`.

Main conclusion: the developer's premise is mistaken. Nobody "trains the engine on the user". Every product and open-source pipeline found uses the engine (Stockfish) as a stateless oracle that labels each move, plus a deterministic rule-based tagger (python-chess geometry) that names the motif. Personalization is just aggregating those labels per user (counts and rates by motif, phase, opening, and clock state) and then picking training positions from the user's own games.

---

## Q1. Mistake thresholds and accuracy formulas (Lichess, Chess.com, ACPL)

### Takeaway
Lichess turns centipawns into "winning chances" on a [-1, +1] scale with a logistic curve (multiplier -0.00368208, cp clamped at ±1000). It labels a move Inaccuracy, Mistake, or Blunder when the mover's winning chances drop by ≥0.1, ≥0.2, or ≥0.3. That equals 5, 10, or 15 points on the 0–100 Win% scale. Lost or allowed forced mates are handled separately. Chess.com uses a proprietary, rating-dependent "Expected Points" model (0.00–1.00) with published loss bands: Best 0, Excellent ≤0.02, Good ≤0.05, Inaccuracy ≤0.10, Mistake ≤0.20, Blunder >0.20. Its Brilliant, Great, and Miss labels are rule-based on top of that.

### Cited Findings
**Lichess winning-chances function (scalachess, `chess.eval.WinPercent`)**
- Code (Scala): `winningChances(cp) = 2 / (1 + exp(-0.00368208 * cp)) - 1`, clamped to [-1, +1]. The comment cites lila PR #11148 for the multiplier. `WinPercent.fromCentiPawns(cp) = 50 + 50 * winningChances(cp.ceiled)`. — [scalachess eval.scala](https://github.com/lichess-org/scalachess/blob/master/core/src/main/scala/eval.scala)
- `Eval.Cp.CEILING = Cp(1000)`: centipawns are clamped to ±1000 before conversion. `WinPercent.fromMate(mate)` maps any mate to ±1000 cp, so mate scores become Win% of about 2.5 or 97.5. — [scalachess eval.scala](https://github.com/lichess-org/scalachess/blob/master/core/src/main/scala/eval.scala)
- The client-side copy (TypeScript) uses the same multiplier. cp is clamped to ±1000, and mates map to `cp = (21 - min(10, |mate|)) * 100`, so mate-in-1 counts as 2000 cp and mate-in-10+ as 1100 cp. `povDiff(color, e1, e2) = (povChances(e1) - povChances(e2)) / 2`, which means povDiff is already on a 0–1 "expected score" scale. — [lila ui/lib/src/ceval/winningChances.ts](https://github.com/lichess-org/lila/blob/master/ui/lib/src/ceval/winningChances.ts)
- The public Lichess page states: "Win% = 50 + 50 * (2 / (1 + exp(-0.00368208 * centipawns)) - 1)". — [lichess.org/page/accuracy](https://lichess.org/page/accuracy)

**Lichess move judgments (`Advice.scala`)**
- `CpAdvice` thresholds, verbatim: `List(.3 -> Blunder, .2 -> Mistake, .1 -> Inaccuracy)`. delta = (currentWinningChances − prevWinningChances), sign-flipped to the mover's point of view (`info.color.fold(-d, d)`), and the first threshold `d <= delta` wins. Both evals must be centipawn scores. — [lila modules/tree/src/main/Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala)
- `Info.color = prevPly.turn`, so the judgment applies to the player who just moved. — [lila modules/tree/src/main/Info.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Info.scala)
- `MateAdvice` covers transitions involving mate scores:
  - `MateCreated` (cp → being mated, "Checkmate is now unavoidable"): Inaccuracy if the previous POV cp was < −999, Mistake if < −700, otherwise Blunder.
  - `MateLost` (had forced mate → cp, or → being mated; "Lost forced checkmate sequence"): Inaccuracy if the new POV cp is > 999, Mistake if > 700, otherwise Blunder.
  - `MateDelayed` ("Not the best checkmate sequence") produces no judgment.
  - The generated comment format is `"(evalBefore → evalAfter) Blunder. Nf3 was best."` — [Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala)
- Unit conversion for comparison: a 0.3 drop in winningChances is 15 Win% points, 0.2 is 10, and 0.1 is 5, because Win% = 50 + 50·wc. Blogs that say "a 30% drop is a Lichess blunder" are mixing up the scales. (Arithmetic from the code above.)

**Lichess move accuracy and game accuracy (`AccuracyPercent.scala`)**
- Per-move accuracy: `if after >= before then 100 else 103.1668100711649 * exp(-0.04354415386753951 * (before - after)) - 3.166924740191411 + 1`. The `+1` is labelled "uncertainty bonus (due to imperfect analysis)". The result is clamped to [0, 100]. The coefficients come from a scipy `curve_fit` of `a*exp(-k*x)+b` to hand-picked points xs=[0,5,10,20,40,60,80,90,100], ys=[100,75,60,42,20,5,0,0,0]; the script is in a code comment. — [lila modules/analyse/src/main/AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala)
- The public page gives the formula without the +1: "Accuracy% = 103.1668 * exp(-0.04354 * (winPercentBefore - winPercentAfter)) - 3.1669". — [lichess.org/page/accuracy](https://lichess.org/page/accuracy)
- Game accuracy per color is computed as follows:
  - `windowSize = clamp(numPlies/10, 2, 8)`.
  - Each move's weight is the standard deviation of Win% over a sliding window, clamped to [0.5, 12].
  - Final accuracy = (volatility-weighted mean of move accuracies + harmonic mean of move accuracies) / 2.
  - Phase accuracies (opening/middlegame/endgame) re-run the same function on per-phase slices, using the scalachess `Division` boundaries.
  - — [AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala); [lichess.org/page/accuracy](https://lichess.org/page/accuracy)

**Lichess ACPL (`AccuracyCP.scala`)**
- Per-move centipawn loss = `max(0, (s2.ceiled − s1.ceiled) * (white ? -1 : 1))`, where both evals are forced to cp (mate becomes a huge cp value, then clamped to ±1000). ACPL is the rounded arithmetic mean over the player's moves. — [lila modules/analyse/src/main/AccuracyCP.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyCP.scala)

**Chess.com Game Review**
- Expected Points (EP) runs from 0.00 to 1.00 (1 = certain win, 0.5 = even) and "is based on the player's rating and the engine evaluation". A move's class comes from the EP it loses:

  | Class | EP lost |
  |---|---|
  | Best | 0.00 |
  | Excellent | 0.00–0.02 |
  | Good | 0.02–0.05 |
  | Inaccuracy | 0.05–0.10 |
  | Mistake | 0.10–0.20 |
  | Blunder | 0.20–1.00 |

  — [Chess.com Help: How are moves classified](https://support.chess.com/en/articles/8572705-how-are-moves-classified-what-is-a-blunder-or-brilliant-and-etc)
- Rule-based labels:
  - **Great** = "critical to the outcome", e.g. losing→equal, equal→winning, or the only good move.
  - **Brilliant** = "when you find a good piece sacrifice". It must be best or nearly best, must not leave you in a bad position, and does not count if you were already completely winning.
  - **Miss** = failing to capitalize on an opponent's mistake, often leaving the position equal or worse.
  - The cutoffs for winning, equal, and losing vary by rating. Brilliant and Great are "more generous" for newer players.
  - — [Chess.com Help](https://support.chess.com/en/articles/8572705-how-are-moves-classified-what-is-a-blunder-or-brilliant-and-etc)
- Chess.com's accuracy score is "CAPS2". No formula is public, and scores mostly fall between 50 and 95. — [Chess.com Help: accuracy](https://support.chess.com/en/articles/8708970-how-is-accuracy-in-analysis-determined) (via search summary)

### Inferences
- A good starting point is Lichess's wc-delta thresholds (0.1/0.2/0.3), computed from MultiPV-1 evals at about 1M nodes. They are open, well tested, and consistent with the Lichess puzzle DB. To make labels rating-aware like Chess.com, fit a rating-conditioned logistic `P(win | cp, rating)` on Lichess DB games that carry `%eval`. Chess.com does not publish its curve.
- Do not use raw centipawn loss for "mistake" detection. A 300 cp loss at +1500 is irrelevant, but at 0.00 it decides the game. Every primary source converts to win probability first. An independent write-up reached the same conclusion: "raw centipawn thresholds misjudged positions that were already won or lost" ([Sviridov, 2026-03-08](https://mrlokans.work/posts/building-self-hosted-chess-blunder-trainer/)).

### Gaps
- Chess.com's EP curve (how rating enters), the exact Brilliant/Great/Miss algorithms, and the CAPS2 formula are proprietary. Only the bands above are published.
- I did not open lila PR #11148 to confirm how the 0.00368208 multiplier was fitted (believed to be fitted to game outcomes, unverified).
- I did not confirm whether Chess.com's current help page still lists a "Book" class; the fetched page did not mention one.

---

## Q2. "Missed opportunity" vs "allowed tactic": detection from engine PVs

### Takeaway
Lichess already computes both concepts in Insights. **Awareness** asks: after the opponent's mistake or blunder, did you avoid a mistake or blunder of your own (i.e. did you punish it)? **Luck** asks: after your mistake or blunder, did the opponent fail to punish it? To detect a *missed* tactic, tag the engine's best line from the position *before* the user's move, from the user's point of view. To detect an *allowed* tactic, tag the engine's refutation line *after* the user's move, from the opponent's point of view.

### Cited Findings
- Lichess Insights `Awareness` metric ("How often you take advantage of your opponent mistakes") is computed per move:
  - If the advice at `ply-1` (the opponent's move) is Mistake or Blunder: awareness = `false` when the user's move at `ply` is also Mistake or Blunder, otherwise `true`.
  - If the opponent's move was not a mistake, the metric is undefined for that move.
  - — [lila modules/insight/src/main/PovToEntry.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/PovToEntry.scala); [InsightMetric.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightMetric.scala)
- Lichess Insights `Luck` ("How often your opponent fails to punish your mistakes"): if the user's move at `ply` is Mistake or Blunder, luck = `true` when the opponent's reply at `ply+1` is also Mistake or Blunder. — [PovToEntry.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/PovToEntry.scala)
- Lichess puzzle generation treats *the opponent's blunder* as the puzzle trigger. The puzzle starts after the opponent's bad move, and the solution is the punishing line. In other words, every Lichess puzzle is a "missed opportunity" exercise for whoever was to move. Trigger: `score >= Cp(200) and win_chances(score) > win_chances(prev_score) + 0.6`. — [lichess-puzzler generator/generator.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py)
- Chess.com's "Miss" label is defined as failing to capitalize on an opponent's mistake. — [Chess.com Help](https://support.chess.com/en/articles/8572705-how-are-moves-classified-what-is-a-blunder-or-brilliant-and-etc)
- An independent self-hosted blunder trainer (2026) uses the same split:
  - "Missed tactic – the best move exploited a pattern you didn't see (fork, pin, skewer, etc.)"
  - "Allowed tactic – your blunder let the opponent execute a tactic against you."
  - Motif detection uses python-chess geometry only, with no engine.
  - Mates in 1–5 are kept as blunders with training weight 2× (mate-in-1/2) or 1.5× (mate-in-3/4/5). Mate-in-6+ is downgraded to mistake.
  - Handling "dead-end" mate positions "cut false blunder counts by 30–40% in many games".
  - — [Sviridov, "Building a Self-Hosted Chess Blunder Trainer", 2026-03-08](https://mrlokans.work/posts/building-self-hosted-chess-blunder-trainer/)
- Another project reports pins and skewers that the user "executed, missed, or allowed" from a PGN, using python-chess and Stockfish. — [viveksarkar0/chess-assigment](https://github.com/viveksarkar0/chess-assigment) (search summary only)

### Inferences
Concrete algorithm (my synthesis of the code above). For user move `m` at position `P_before` (user to move), with engine output `E(P_before)` (MultiPV 2: best `b1`, second `b2`, PV of b1) and `E(P_after)` (PV = the opponent's best reply line):

1. `drop = wc_user(E(P_before).b1) − wc_user(E(P_after))`. Classify with 0.1/0.2/0.3 or rating-aware bands.
2. **Missed opportunity / missed tactic**: `drop ≥ T` and `wc_user(P_before) ≥ ~+0.35` (≈+200 cp; puzzler uses Cp(200)) and preferably `wc(b1) − wc(b2) ≥ 0.3–0.7` (a unique best move; puzzler uses 0.7). Build a pseudo-puzzle with root = position before the opponent's previous move, mainline = [opponent's previous move] + PV(b1), pov = user. Run the cook-style tagger (Q3) on it. "Missed because opponent blundered" = the opponent's previous move had judgment ≥ Mistake (Lichess Awareness logic).
3. **Allowed tactic / walked into**: `drop ≥ T` and the opponent's refutation PV in `E(P_after)` is forcing (`wc_opp(best) − wc_opp(second) ≥ 0.3` and/or the PV contains checks or captures). Build a pseudo-puzzle with mainline = [m] + PV(E(P_after)), pov = opponent, and run the tagger. The tags name what the user allowed: fork, pin, backRankMate, etc. "Hanging piece" is the common case: the opponent's first PV move captures an undefended piece.
4. A single move can be both (missed a win *and* allowed a counter-tactic). Store two labels with separate motif tags.
5. "Opponent didn't punish" (Luck) is useful for lessons too: the user got away with it, so the motif is still a weakness even though the eval recovered.
6. PV length matters. Truncate PVs to the forcing portion (stop when the best move is no longer unique, like puzzler's `cook_advantage` loop) before tagging, or the tagger fires on unrelated later moves.

### Gaps
- I did not find a published, evaluated precision/recall for "missed vs allowed" labelling in any product. Chess.com's internal Miss logic is not public.

---

## Q3. Lichess puzzle pipeline (`lichess-puzzler`): generator, uniqueness validation, tagger (cook.py), theme taxonomy

### Takeaway
The generator scans Lichess DB games that already carry `%eval`, finds the first large swing caused by an opponent's move, and extends a forced line with Stockfish (MultiPV 2, up to 25M nodes per position). It accepts only player moves where the best move is clearly better than the second (Δwc > 0.7) or is the only tablebase win. The tagger (`cook.py`, about 1,000 lines) applies about 45 hand-written python-chess predicates to the solution line. Most motif detectors check move geometry plus "hanging / takeable by a lower piece" heuristics. Zugzwang alone uses an engine null-move test.

### Cited Findings
**Generator (`generator/generator.py`, version = 50)** — [generator.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py), [util.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/util.py)
- Engine limits:
  - `pair_limit = Limit(depth=50, time=30, nodes=25_000_000)` for each solution step.
  - `mate_defense_limit = Limit(depth=15, time=10, nodes=8_000_000)` for the defender's replies in mate lines.
  - `get_next_move_pair` runs `engine.analyse(board, multipv=2, limit)` to obtain best and second-best.
- Input: monthly Lichess DB PGNs (`.pgn.zst`). Only games with `%eval` comments (games already analysed by fishnet) are used. Variants are skipped.
- **Game tiering** (lower tiers are skipped or held to stricter rules):
  - Rating: >1750 → 3, >1600 → 2, >1500 → 1, else 0.
  - Time control: `base + 40*inc`. ≥480 s → 3, ≥180 → 2, >60 → 1, else 0.
  - The game's tier is the minimum of the two.
  - +1 if a titled non-BOT player took part, +1 if the game has <38 plies, +1 more if <21 plies.
  - Tier 0 games are skipped entirely.
- **Position scan**:
  - Start with `prev_score = Cp(20)`.
  - Skip repeated positions until an irreversible move.
  - Skip positions whose castling rights are not "maximum" (to avoid FEN ambiguity).
  - Skip if `prev_score > Cp(300)` and the position is not a near mate ("too much of a winning position to start with").
  - Skip if the side to move is already up in material (P=1, N=B=3, R=5, Q=9).
- **Mate trigger**: `score > Mate(15)` (mate in under 15) → `cook_mate`. Every attacker move must pass the uniqueness test and keep a mate score. Defender moves come from `engine.play` at `mate_defense_limit`. Tier-1 games reject mate-in-2-length solutions.
- **Advantage trigger**: `score >= Cp(200) and win_chances(score) > win_chances(prev_score) + 0.6`.
  - Reject if `score < Cp(400)` and the side is not down material ("not clearly winning and not from being down in material").
  - `cook_advantage` keeps extending while best ≥ Cp(200) and the move is unique. It aborts on repetition.
  - Then it trims so the line ends on a player move that had an alternative.
  - Discards one-movers, and two-movers for tier < 3.
- **Uniqueness (`is_valid_attack`)**: second move is None, OR the tablebase says it is the only winning move, OR it is a valid mate-in-1, OR `win_chances(best) > win_chances(second) + 0.7`.
  - Mate-in-1 rule: if the second-best is not winning (`win_chances ≤ 0.6`), it is valid. If several mates-in-1 exist, re-run MultiPV = (#mates+1) and require the best non-mating move to have `win_chances ≤ 0.6`.
- **Tablebase**: positions with ≤7 pieces (non-mate puzzles only) query `http://tablebase.lichess.ovh/standard?fen=`, rate-limited to about 1 request per 550 ms. The first two returned moves decide whether only one move wins. WDL categories: `win, draw, loss, unknown, maybe-win, blessed-loss, maybe-loss, cursed-win`. — [generator/tb.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/tb.py)
- **Client-side faulty-puzzle check (lila)**: `hasMultipleSolutions = povChances(second) >= 0.3524 (≈200cp) || povDiff(best, second) < 0.14`. The code comment says this is "stricter than lichess-puzzler v49 check". — [winningChances.ts](https://github.com/lichess-org/lila/blob/master/ui/lib/src/ceval/winningChances.ts)

**Tagger (`tagger/cook.py`)** — [cook.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/cook.py), [tagger/util.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/util.py), [tagger/model.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/model.py)
- Data model:
  - `Puzzle.mainline[0]` is the opponent's move that creates the puzzle (the blunder).
  - `mainline[1::2]` are the solver's moves.
  - `pov = not game.turn()`, which is the solver.
  - `cp` is the final eval.
- Helper definitions (util.py):
  - `values = {P:1, N:3, B:3, R:5, Q:9}`; `king_values` adds K:99.
  - `is_defended(board, piece, sq)`: has a same-color attacker, OR (x-ray defense) removing an enemy ray attacker reveals a defender.
  - `is_hanging = not is_defended`.
  - `is_in_bad_spot(sq)`: attacked AND (hanging OR capturable by a lower-value non-king piece).
  - `is_trapped(sq)`: not in check, not pinned, not a pawn or king, in a bad spot, and every legal move of the piece either fails to capture something of ≥ value or lands in another bad spot.
  - `is_advanced_pawn_move`: promotion or pawn to rank ≥5 (relative). "Very advanced" means rank ≥6.
- **Goal tags**: `mateInN` (N = len(mainline)//2, capped at mateIn5) plus `mate` when the final position is checkmate. Otherwise `crushing` if cp > 600, `advantage` if cp > 200, else `equality`.
- **Mate-pattern tags** (checked on the final position in order; first match wins):
  - `smotheredMate`: a knight gives check and every king-adjacent square holds the king's own piece.
  - `backRankMate`: king on its back rank; the 2–3 squares in front are occupied by its own unattacked pieces; a checker sits on the back rank.
  - `anastasiaMate`
  - `hookMate`: rook adjacent to king, protected by a knight adjacent to king, which is protected by a pawn.
  - `arabianMate`: rook adjacent, king in a corner, knight a (2,2) offset away.
  - `bodenMate` / `doubleBishopMate`: all king-zone squares attacked only by bishops; opposite-side vs same-side bishops.
  - `dovetailMate`
- **Motif detectors**:
  - **fork**: for each solver move except the last, the moved piece is not a king and its destination is not a bad spot. Count attacked enemy *non-pawn* pieces that are either worth more than the forking piece (king = 99) or hanging and not attacking the forking square. `nb > 1` → fork.
  - **hangingPiece**: the *first* solution move captures a non-pawn piece that was undefended (pawn captures and check-escapes excluded). It is not counted if it is an equal recapture, i.e. the opponent's last move captured something of ≥ value on that square. If the line is longer, material after move 3 must be ≥ material after move 1.
  - **pin**, two variants:
    - `pin_prevents_attack`: after a solver move, an enemy pinned piece (`board.pin() != BB_ALL`) attacks a solver piece off its pin ray that is more valuable than it or hanging.
    - `pin_prevents_escape`: a solver piece attacks an enemy pinned piece along the pin ray, and the pinned piece is worth more than the attacker, or is hanging and can't step off the ray.
  - **skewer**: a solver ray piece captures, the opponent's previous move left from a square *between* the capturer's from and to squares, the piece that moved was more valuable (king_values) than the one captured behind it, and the capture square was a bad spot. Mate is excluded.
  - **discoveredAttack**: either discovered check (a checker that is not the moved piece) or a later solver capture whose line passes through the square the solver's previous move vacated. Castling is excluded.
  - **doubleCheck**: `len(checkers) > 1` after a solver move.
  - **trappedPiece**: a later solver capture of a non-pawn piece that `is_trapped` two plies earlier, following the piece if it just moved.
  - **sacrifice**: material diff relative to the start drops by ≥2 after a later solver move. Promotions are excluded.
  - **xRayAttack**: the solver recaptures on a square where the opponent just recaptured after the solver captured, and the opponent's recapturing piece came from between the solver's from and to squares.
  - **attraction**: the solver moves to a square, the opponent's K/Q/R captures there, then the solver attacks that square and either checks (if the king was attracted) or later captures there.
  - **deflection**: a later solver capture or promotion on a square that the opponent's last-moved piece used to guard, where the opponent's move was a capture of the solver's previous move or a check response.
  - **overloading**: stub, `return False` (not implemented).
  - **interference / self-interference**: a hanging enemy piece gets captured after a piece (own or enemy) moved onto the line between it and its ray-piece defender.
  - **intermezzo**: the solver plays an in-between move before recapturing.
  - **clearance**: a ray piece moves to or through a square the solver's previous (non-capturing, non-check) move vacated.
  - **quietMove**: a non-final solver move that is not a check, not a capture, does not threaten a piece, is not an advanced pawn push, and is not a king move.
  - **defensiveMove**: the last move is quiet with ≥3 legal alternatives, or a check escape.
  - **capturingDefender**
  - **exposedKing**: the enemy king is on its 6th–8th rank (relative) without pawn cover, and the solver gives check mid-line.
  - **kingsideAttack / queensideAttack**: scored on checks and captures near the corner; requires ≥20 or ≥18 pieces (not endgames); suppressed if backRankMate or fork was tagged.
  - **attackingF2F7**, **advancedPawn**, **promotion**, **underPromotion**, **enPassant**, **castling**, **collinearMove**.
  - **Endgames**: pawn/queen/rook/bishop/knight endgame (only K + P + that piece type in the first two positions), `queenRookEndgame`.
  - **Length**: `oneMove` (len 2), `short` (4), `long` (6), `veryLong` (≥8).
- **zugzwang** (`tagger/zugzwang.py`) is the only engine-based tag. For each position after a solver move (not in check, ≤15 legal moves), compare the engine score of the actual position with the same position after a null move (opponent "passes"). If `win_chances(actual) < win_chances(null) − 0.3`, it is zugzwang. Engine limit: `depth=30, time=10, nodes=12_000_000`. — [zugzwang.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/zugzwang.py)
- `TagKind` declares `coercion` and `simplification`, but cook.py never emits them. `overloading` is a stub. — [tagger/model.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/model.py)

**Full Lichess puzzle theme taxonomy (lila `PuzzleTheme.categorized`)** — [lila modules/puzzle/src/main/PuzzleTheme.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleTheme.scala)

| Category | Themes |
|---|---|
| recommended | mix |
| phases | opening, middlegame, endgame, rookEndgame, bishopEndgame, pawnEndgame, knightEndgame, queenEndgame, queenRookEndgame |
| motifs | advancedPawn, attackingF2F7, capturingDefender, discoveredAttack, doubleCheck, exposedKing, fork, hangingPiece, kingsideAttack, pin, queensideAttack, sacrifice, skewer, trappedPiece |
| advanced | attraction, clearance, collinearMove, discoveredCheck, defensiveMove, deflection, interference, intermezzo, quietMove, xRayAttack, zugzwang |
| mates | mate, mateIn1–mateIn5 |
| mateThemes | anastasiaMate, arabianMate, backRankMate, balestraMate, blindSwineMate, bodenMate, cornerMate, doubleBishopMate, dovetailMate, epauletteMate, hookMate, killBoxMate, pillsburysMate, morphysMate, operaMate, swallowstailMate, triangleMate, vukovicMate, smotheredMate |
| specialMoves | castling, enPassant, promotion, underPromotion |
| goals | equality, advantage, crushing, mate |
| lengths | oneMove, short, long, veryLong |
| origin | master, masterVsMaster, superGM |

`checkFirst` is a hidden theme.

- The Lichess puzzle CSV (with a `Themes` column) is published on database.lichess.org. A Kaggle mirror reports about 6 million puzzles, last updated 2026-06-11, and says themes are auto-tagged (and refined by player votes per one paper). — [Kaggle: lichess/chess-puzzles](https://www.kaggle.com/datasets/lichess/chess-puzzles) (search summary)

### Inferences
- cook.py can be reused almost as-is on a user's own blunders. Wrap each mistake as a `chess.pgn.Game` whose first move is the "trigger" move (opponent's blunder for missed tactics, the user's own move for allowed tactics) followed by the forcing PV. It is pure python-chess, but the repo is licensed **GNU AGPL v3** ([LICENSE](https://github.com/ornicar/lichess-puzzler/blob/master/LICENSE)). Copying cook.py into a closed-source network service would trigger AGPL obligations. Either reimplement the detectors clean-room from the descriptions above or open-source the tagging service.
- Known limitations of the rule-based detectors (from reading the code):
  - They describe the **solution line**, not the user's *thought error*. A "fork" tag means the punishing line contains a fork, not that the user failed to see the fork specifically.
  - Precedence and suppression rules hide some co-occurring motifs: one mate pattern only; `deflection` and `overloading` are exclusive; side attacks are suppressed by fork.
  - Many detectors only look at solver moves after the first (`[1::2][1:]`), so a one-move tactic gets fewer tags.
  - Value heuristics are static (P1/N3/B3/R5/Q9), with no SEE (static exchange evaluation).
  - `overloading`, `coercion`, and `simplification` are not implemented.
  - It needs a forcing, unique line. Quiet positional errors get only goal/phase tags.
- Lila's visible theme list includes mate patterns (balestra, blindSwine, corner, epaulette, killBox, morphys, opera, pillsburys, swallowstail, triangle, vukovic) and `discoveredCheck` that the public cook.py at HEAD does not emit. They are presumably tagged by other code or by player theme votes (see Gaps).

### Gaps
- I could not find where lila's extra mate themes and the `opening/middlegame/endgame` phase themes are computed. They are not in `ornicar/lichess-puzzler` @608cf7f. They may come from a separate or private tagger, the puzzle-import step, or community voting.
- I found no published precision/recall for cook.py tags against human labels.

---

## Q4. Other open-source tactic detectors and ML theme classifiers

### Takeaway
There is no canonical library beyond lichess-puzzler's cook.py. The community has many small python-chess heuristic detectors of varying quality, and I found no peer-reviewed multi-label "theme classifier" paper with per-theme F1. ML work on the Lichess puzzle DB focuses on difficulty prediction, move prediction, and generation. Themes appear there as auxiliary targets or metadata. In practice, use rule-based tagging (high precision on forcing lines) and validate it against the Lichess puzzle DB labels.

### Cited Findings
- **Naive detectors overfire.** In the chess-coach-ai PR #473, a naive fork detector fired a "confirmed fork" on 48% of puzzles *not* labelled fork, mostly checks that also hit a king-defended pawn. The fix was to require two pieces that can actually be won. They measured detectors against Lichess labels. — [AayanHetam/chess-coach-ai PR #473](https://github.com/AayanHetam/chess-coach-ai/pull/473) (search summary)
- Other small projects:
  - `CormacmacmCormac/chess_stats` claims to classify every mistake by theme into CSV. Not inspected.
  - "ChessGrammar" API: 10 motifs via python-chess heuristics, no Stockfish at runtime. — [dev.to write-up](https://dev.to/stevejvv/i-built-an-api-that-detects-chess-tactical-patterns-from-fen-and-pgn-5ef0)
  - `taylormccreary/ChessTacticClassifier`: forks, skewers, trapped pieces from Chess.com data. Apparently a student ML project with no accuracy reported. — [GitHub](https://github.com/taylormccreary/ChessTacticClassifier)
- **ML on Lichess puzzles**:
  - 2024 IEEE BigData Cup (puzzle difficulty): a 2nd-place CNN used auxiliary *theme* and move-prediction heads. No theme accuracy is in the snippet. — [Miłosz, arXiv 2410.11078](https://arxiv.org/abs/2410.11078) (search summary)
  - ChessMimic (2026) reports per-theme top-1 move accuracy on Lichess puzzles, e.g. xRayAttack 0.759 vs Maia-2's 0.427. — [arXiv 2606.04473](https://arxiv.org/html/2606.04473v1) (search summary)
  - Puzzle generation work: DeepMind "Generating Creative Chess Puzzles" (trained on 4.4M Lichess puzzles, cutoff 2024-12-01) — [arXiv 2510.23881](https://arxiv.org/html/2510.23881v1); diffusion-based theme-conditioned generation — [arXiv 2609.38577](https://arxiv.org/html/2609.38577) (search summaries)
  - An Open University of Israel thesis reverse-engineers Lichess puzzle attributes: difficulty classifier at 81.1%, plus per-theme models (accuracy not in snippet). — [Goren thesis PDF](https://www.openu.ac.il/Lists/MediaServer_Documents/Academic/CS/Mordechai%20Goren.pdf) (search summary)
- **Concept-based (non-motif) labelling**: the CCC paper scores "chess concepts" by the dot product of an expert model's position representation with extracted concept vectors, before and after a move. The most-changed concepts become the explanation. — [arXiv 2410.20811](https://arxiv.org/abs/2410.20811)

### Inferences
- A practical way to evaluate your own tagger: run it on Lichess DB puzzles (FEN + moves + Themes) and compute per-theme precision and recall against the DB tags, as PR #473 did.
- An ML classifier (multi-label head on a board encoder) could be trained cheaply on about 5–6M Lichess puzzles with their Themes column. But the labels are themselves mostly cook.py output, so it would mostly learn to imitate the rules. Its main benefit is softness, i.e. scoring "near-miss" motifs on non-forcing positions.

### Gaps
- I found no 2023–2026 peer-reviewed paper reporting multi-label tactical-theme classification metrics (per-theme F1 or Hamming loss) on the Lichess puzzle DB.
- `ChessGPT` (Feng et al., 2023, arXiv 2306.09200) is known background but was not re-verified in this session.

---

## Q5. Classifying non-tactical mistakes (opening, endgame, positional, time trouble, conversion)

### Takeaway
Products split non-tactical errors along metadata axes rather than motif geometry:
- **Game phase**: from a material/development divider.
- **Opening**: a deviation from the masters/book explorer, or from the user's own repertoire.
- **Endgame technique**: exact Syzygy WDL changes at ≤7 pieces.
- **Time trouble**: the `%clk` percentage when the mistake happened.
- **Conversion**: was the user winning (eval band) and then failed to win?

Truly "positional" mistakes have no reliable algorithmic label. Only concept-vector research (CCC) and DecodeChess's proprietary "Concepts" touch them.

### Cited Findings
- **Opening, as used by Lichess "Learn from your mistakes"**: candidate mistakes come from `evalSwings`. Moves with `|povDiff| > 0.1` qualify (equal to Δwc > 0.2, i.e. Mistake or Blunder level), as do moves that lost a mate-in-≤3. For a mistake before the middlegame boundary (`game.division.middle`), the client fetches the masters opening explorer. If the user's move was played in >1 master game, the "mistake" is cancelled (`explorerCancelPlies`). — [lila ui/analyse/src/nodeFinder.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/nodeFinder.ts); [ui/analyse/src/retrospect/retroCtrl.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/retrospect/retroCtrl.ts)
- **Retry success rule in "Learn from your mistakes"**: the user's retry move is accepted once the local engine is ready (bestmove, or ≥1M nodes, or >3 s) if `povDiff(color, evalAfterRetry, evalBeforeFault) > -0.04`, i.e. within 4 Win% points of the pre-mistake eval. — [retroCtrl.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/retrospect/retroCtrl.ts)
- **Phase**: Lichess computes accuracy per opening/middlegame/endgame using scalachess `Division` (middle/end plies). Insights expose a `Phase` dimension. — [AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala); [InsightDimension.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightDimension.scala)
- **Lichess Insights axes**:
  - Dimensions: Period, Perf, Phase, Result, Termination, Color, OpeningFamily, OpeningVariation, OpponentStrength, PieceRole, MovetimeRange, MyCastling, OpCastling, QueenTrade, MaterialRange, Blur, TimeVariance, CplRange, EvalRange, AccuracyPercentRange, WinPercentRange, ClockPercentRange, GameSource.
  - Metrics: MeanCpl, CplBucket, MeanAccuracy, Movetime, Result, Termination, Performance, RatingDiff, OpponentRating, NbMoves, PieceRole, Awareness, Luck, Material, ClockPercent, Blurs, TimeVariance.
  - Each move stores its phase, movetime (tenths), clockPercent, piece role, eval, cpl, winPercent, accuracyPercent, awareness, luck, and blur.
  - — [InsightDimension.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightDimension.scala); [InsightMetric.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightMetric.scala); [PovToEntry.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/PovToEntry.scala)
- **Endgame technique via tablebase**: lichess-puzzler queries `tablebase.lichess.ovh/standard?fen=` for ≤7-piece positions. The API returns moves ordered best-first with WDL categories (win/draw/loss/cursed-win/blessed-loss/maybe-*). — [generator/tb.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/tb.py)
- **Aimchess skill axes**: Tactics, Endgame, Advantage Capitalization, Resourcefulness, Time Management, Opening Performance. Trainers include Opening Improver, Advantage Capitalization Trainer, Endgames, Intuition Trainer, Retry Mistakes, Defender, and Time Trainer. — [Stork.ai Aimchess listing](https://www.stork.ai/en/aimchess) (third-party aggregator)
- **Chessbook** finds *opening* mistakes in your Lichess/Chess.com games relative to your repertoire and quizzes you on them. — [Chessbook Google Play](https://play.google.com/store/apps/details?id=com.chessbook.android&hl=en); [64squares review](https://64squares.substack.com/p/chessbook-review) (search summaries)
- **Positional**: DecodeChess offers "Threats" and "Concepts" ("important positional and tactical motives") tabs, using a proprietary "XAI" algorithm on top of Stockfish NNUE. — [decodechess.com](https://decodechess.com/)
- The Sviridov trainer notes that explanations are weak for "quiet positional blunders" and that trap (opening-trap) detection "still has errors". Phase is tagged by "piece count and move number". — [mrlokans.work](https://mrlokans.work/posts/building-self-hosted-chess-blunder-trainer/)

### Inferences
Suggested label schema per mistake (my synthesis):
- `phase` ∈ {opening, middlegame, endgame}, from python-chess plus a Divider port. Lila's Division works by material and development.
- `opening`: ECO/opening name from Lichess's `chess-openings` TSV. `left_book_at_ply` = the first ply where the move is not in the masters/lichess explorer (or the user's repertoire). An "opening error" is a Mistake/Blunder at ply < middlegame boundary *and* not an explorer move.
- `endgame_tb_error` (≤7 pieces): WDL before (user POV) ∈ {win} → after ∈ {draw, loss}, or draw → loss. This is exact and engine-independent. Use local Syzygy via python-chess `chess.syzygy` (`probe_wdl`, `probe_dtz`) or the Lichess TB API (respect rate limits; puzzler waits 550 ms). Treat cursed-win/blessed-loss as draws under the 50-move rule.
- `time_trouble`: from PGN `[%clk h:mm:ss]` (python-chess `GameNode.clock()`). Flag when the clock is < 10% of the initial time or < 30 s, and compare that mistake rate with the user's baseline (Lichess uses ClockPercentRange buckets). Also flag "too-fast" moves (movetime ≪ the user's median) on critical positions.
- `conversion_failure`: the game had `wc_user ≥ +0.5` (≈Win% 75, ≈+300 cp) for ≥N consecutive plies, and the result is not a win. Attribute it to the move(s) with the largest drop out of the winning band (Aimchess calls this "Advantage Capitalization").
- `resourcefulness` / defence: the user was losing and missed a drawing or swindle resource (the inverse of conversion).
- `positional` = residual: a Mistake/Blunder with no tactical tag from either PV, no TB change, and a non-forcing refutation (best-vs-second gap small). Present it to the user as "strategic", optionally with concept or LLM explanation.

### Gaps
- I could not confirm the exact Lichess Division (Divider) algorithm thresholds in this session (scalachess `Divider.scala` was not read).
- No primary source documents how Chess.com Insights computes "missed tactics" or its phase split.

---

## Q6. LLM explanations grounded in engine output

### Takeaway
Raw LLM commentary hallucinates frequently. A 2026 benchmark found GPT-5.4 without tools makes incorrect sub-claims 22% of the time, and smaller open models exceed 40%. State-of-the-art grounding means the LLM never computes chess itself. You feed it engine evals and PVs, the motif tags, and an enumerated list of legal attacks and threats, and you verify every claim against board and engine tools before display.

### Cited Findings
- **CCC (Concept-guided Chess Commentary, Oct 2024)**:
  - Ranks concepts by how much a move changes them (expert-model representation · concept vectors).
  - Passes the top concepts to an LLM with few-shot and chain-of-thought prompting.
  - Enumerates "all existing attacks toward opponent pieces to prevent mentioning non-existing pieces or illegal moves".
  - Proposes the GCC-Eval GPT-based evaluator.
  - — [arXiv 2410.20811](https://arxiv.org/abs/2410.20811)
- **ACT-Eval, "Hallucinations on the Board" (Hebbar, Sheng, Oh, Viswanath; submitted 2026-08-04)**:
  - Decomposes commentary into atomic claims and routes them to engine-supported tools.
  - 325-position benchmark (125 with expert gold atoms).
  - GPT-5.4 without tools produced "incorrect sub-claims 22.0% of the time"; smaller open-weight models exceeded 40%.
  - Tools improve factual correctness, but coverage of expert ideas stays limited.
  - — [arXiv 2608.04240](https://arxiv.org/abs/2608.04240)
- **"Grounded Chess Reasoning in Language Models via Master Distillation"**: a 4B model "C1" goes from near zero to 48.1% accuracy by distilling from a stronger expert system. — [arXiv 2603.20510](https://arxiv.org/pdf/2603.20510) (search summary)
- **"Language Models that Play Chess and Explain Their Moves"** (Oct 2026) reports low conceptual coherence, and that hallucinated motifs can propagate through self-improvement. — [arXiv 2610.03695](https://arxiv.org/html/2610.03695) (search summary, not fully read)
- **"Do Chess Explanations Reflect Model Decisions?"** tests the faithfulness of LLM chess explanations. — [arXiv 2609.22245](https://arxiv.org/pdf/2609.22245) (title only seen)
- **DecodeChess** explains moves "in rich, intuitive language" using "Explainable AI (XAI)" plus Stockfish NNUE, with no LLM mentioned. The site footer is dated 2022. — [decodechess.com](https://decodechess.com/)
- **Chess.com Game Review coach explanations** are tier-gated (per third-party reviewers, Diamond has per-move coach explanations). Reviewers say they are generic across ratings and are single-game only, with "no cross-game pattern recognition". — [chessvia.ai blog](https://www.chessvia.ai/blog/chess-com-game-review-free); [improvemychess.co comparison](https://www.improvemychess.co/learn/best-ai-chess-coaches-2026) (third-party, possibly competitor-biased)

### Inferences
Grounding recipe for the app (my synthesis of CCC and ACT-Eval):
1. Build a structured "fact sheet" per mistake. Include:
   - FEN, the move played (SAN), best move and PV (SAN, ≤6 plies), and the refutation PV.
   - wc before and after, judgment, and motif tags from cook.py for both missed and allowed lines.
   - Hanging pieces, pins (`board.pin`), checks, and attacked/defended maps.
   - Phase, opening name, clock, and TB WDL.
2. Prompt the LLM to explain **only** using the fact sheet. Forbid introducing moves not in the PVs.
3. Post-verify: parse every SAN in the output with python-chess, reject unknown or illegal moves, and check piece-on-square claims.
4. Keep explanation templates for the top motifs (fork, hanging piece, pin, back-rank) so the LLM is optional. Deterministic text is free of hallucinations.

### Gaps
- "Caissa" as an LLM chess tutor was not found in this session.
- I did not verify the internals of Chess.com's coach (LLM or templated).

---

## Q7. Survey of products doing personalized analysis and training

### Takeaway
Many tools detect mistakes per game, and several turn your blunders into puzzles: Lichess "Learn from your mistakes", Aimchess "Retry Mistakes", Noctie flashcards, ChessPuzzler, Puzzlik, Chessbook (openings only). Few aggregate *motif-level* weaknesses across hundreds of games into a curriculum. In that space only Aimchess and Chess.com Insights exist, and Aimchess appears stagnant since 2024. The open gap is: cross-game, motif-specific diagnosis (e.g. "you miss knight forks on the kingside 3× more than peers at your rating"), plus lessons generated from the user's own positions, plus spaced repetition, plus progress measured on new games.

### Cited Findings
- **Lichess "Learn from your mistakes"**: single game, own color. It walks through Mistakes/Blunders (Δwc > 0.2) and lost mates ≤3, skips master-book opening moves, and has the user find a move within 4 Win% points of the best. — [nodeFinder.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/nodeFinder.ts); [retroCtrl.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/retrospect/retroCtrl.ts)
- **Lichess Insights**: cross-game aggregation over the dimensions and metrics listed in Q5, including tactical "Awareness" and "Luck". There is no per-motif (fork/pin) dimension. — [InsightDimension.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightDimension.scala); [InsightMetric.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightMetric.scala)
- **Lichess puzzle dashboard** reports per-theme puzzle performance (e.g. `firstWinPercent` per theme), but for *puzzles solved*, not for the user's games. — [lila PuzzleDashboard.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleDashboard.scala)
- **Chess.com puzzles**: the engine looks for "major evaluation swings, usually caused by blunders" in games played on the site. An algorithm tags the best move with themes such as forks or skewers. "Any position from your own games could become a puzzle", but no personalized set is described. — [Chess.com Help: Where do puzzles come from](https://support.chess.com/en/articles/8709004-where-do-chess-com-puzzles-come-from)
- **Chess.com Insights**: forum users describe a basic "missed tactics and mates" section (pins, mates, hanging pieces, forks from your games). This is unverified against official docs. — [Chess.com forum](https://www.chess.com/forum/view/site-feedback/personalized-puzzles-from-your-own-game) (search summary)
- **Aimchess**:
  - Imports from Chess.com, Lichess, and Chess24. Diagnoses 6 skills and offers Retry Mistakes and other trainers.
  - Acquired by Play Magnus Group in 2021, which Chess.com acquired in December 2022.
  - The third-party Marlvel report says the last iOS release was July 2024.
  - Pricing sources conflict ($7.99 vs $9.99/mo).
  - — [Stork.ai](https://www.stork.ai/en/aimchess); [Marlvel report, 2026-05-20](https://marlvel.ai/apps/aimchess-learn-chess-online); [App Store](https://apps.apple.com/us/app/aimchess-learn-chess-online/id1524941307?uo=4)
- **Noctie.ai**:
  - Turns "your most instructive mistakes into flashcards" with spaced repetition (opening, tactic, and endgame mistakes).
  - Grades moves "from a human perspective" rather than by Stockfish eval, and focuses on "the most instructive mistakes rather than the 'biggest'".
  - Imports from Chess.com and Lichess. Around €14/month (unconfirmed).
  - — [noctie.ai](https://noctie.ai/); [Noctie FAQ](https://noctie.ai/faq/); [App Store](https://apps.apple.com/us/app/noctie-chess-trainer/id6444289077)
- **Chessbook**: repertoire trainer that checks recent online games against your repertoire and quizzes the deviations. Openings only. — [64squares review](https://64squares.substack.com/p/chessbook-review)
- **Listudy**: does *not* analyse the player's own games. It offers PGN-based spaced-repetition openings and fixed tactic and endgame banks. — [chesspivot Listudy review](https://www.chesspivot.com/en/reviews/listudy)
- **ChessPuzzler "Personal Training"**: reads Chess.com/Lichess games and finds "about 1 puzzle per 10 games". **Puzzlik** creates puzzles from missed tactics and blunders in your games. — [chesspuzzler.com blog](https://www.chesspuzzler.com/Blog/HowToSolveChessPuzzlesFromYourOwnGames); [puzzlik.com](https://puzzlik.com/)
- **DecodeChess**: natural-language explanation of engine moves (Threats/Concepts). No game import or per-user aggregation is described, and the site appears not updated since 2022. — [decodechess.com](https://decodechess.com/)
- **Maia-2 (NeurIPS 2024)**: a single skill-conditioned model of human moves across ratings. Personalization is by rating band, not by individual. — [arXiv 2409.20553](https://arxiv.org/abs/2409.20553)
- **Chessable**: per a forum mention, it can build puzzles from your games if you link Chess.com. Unverified. — [Chess.com forum](https://www.chess.com/forum/view/site-feedback/personalized-puzzles-from-your-own-game) (search summary)
- Reviewers' consensus in 2026 comparisons: Chess.com review is per game with "no cross-game pattern recognition". The suggested workflow is to use AI analysis to identify weaknesses and active-recall tools to fix them. — [improvemychess.co](https://www.improvemychess.co/learn/best-ai-chess-coaches-2026); [checkmatex.app](https://checkmatex.app/blog/best-ai-chess-coach-2026-tools-compared) (third-party blogs)

### Inferences
Differentiation opportunities for the app:
1. **Per-motif weakness rates normalized by opportunity.** For each motif M:
   - Opportunities = positions in the user's games where the best line is tagged M and is unique (puzzle-worthy).
   - Miss rate = missed / opportunities.
   - "Allowed" rate = user moves that let the opponent run motif M, per 100 moves.
   - Compare against a rating-band baseline computed from Lichess DB games (also tag-able).
2. Lessons built from the user's *own* positions, plus Lichess puzzle-DB positions with the same theme for volume. Schedule them with SRS and re-measure on new games.
3. Cross-dimensional insights that no one exposes per motif: motif × phase × clock × opening. For example: "back-rank blunders, 80% with <20 s".
4. Maia-style human-likelihood: use Maia-2 to rate how "findable" the missed move was at the user's rating. This prioritizes instructive misses, as Noctie claims to do.

### Gaps
- No official 2025–2026 Chess.com documentation of Insights' tactic section or a "personalized puzzles" feature was found.
- ChessMood was not researched (no results surfaced).
- Feature claims for Aimchess, Noctie, and Chessable rely on vendor pages and third-party aggregators, not hands-on testing.
- Pricing figures conflict across sources.

---

## Q8. Engine settings for batch analysis (fishnet, puzzler, MultiPV)

### Takeaway
Lichess server analysis is node-limited, not depth-limited:
- 1,000,000 nodes per position for user-requested game analysis (`manualRequest`)
- 5,000,000 for official broadcasts
- 300,000 for automatic "hunter" analysis
- 100,000 for "tutor" auto-analysis

It uses MultiPV 1. The puzzle generator spends far more (MultiPV 2, up to 25M nodes or 30 s per step) because it must prove uniqueness. For a personal app, ~1M nodes MultiPV 1 to find mistakes, then a deeper MultiPV 2–3 re-search only on flagged positions, is the standard split.

### Cited Findings
- `Work.Origin(nodesPerMove)`: `officialBroadcast 5_000_000`, `manualRequest 1_000_000` ("games & studies"), `autoHunter 300_000`, `autoTutor 100_000`. — [lila modules/fishnet/src/main/Work.scala](https://github.com/lichess-org/lila/blob/master/modules/fishnet/src/main/Work.scala)
- Lila sends `nodes: {sf18, sf17_1, sf16: nodes, classical: nodes*3}`. — [lila modules/fishnet/src/main/JsonApi.scala](https://github.com/lichess-org/lila/blob/master/modules/fishnet/src/main/JsonApi.scala)
- The fishnet client scales nodes by `MAX_POSITIONS/(MAX_POSITIONS+1)` (MAX_POSITIONS = 6) to offset chunk overlap. It issues `go nodes N` (plus `depth` if given) and `setoption MultiPV` only if the work specifies it. — [fishnet src/api.rs](https://github.com/lichess-org/fishnet/blob/main/src/api.rs); [src/ipc.rs](https://github.com/lichess-org/fishnet/blob/main/src/ipc.rs); [src/stockfish.rs](https://github.com/lichess-org/fishnet/blob/main/src/stockfish.rs)
- Lila times out an analysis job at `plies * 7 s + 3 s`. — [lila modules/fishnet/src/main/Cleaner.scala](https://github.com/lichess-org/lila/blob/master/modules/fishnet/src/main/Cleaner.scala)
- Fishnet's backlog heuristic assumes a batch of "60 positions at 1_450_000 nodes each", with a 7-minute deadline, and notes "top end clients take no longer than 35 seconds" per batch. — [fishnet src/stats.rs](https://github.com/lichess-org/fishnet/blob/main/src/stats.rs)
- Puzzler engine limits:
  - Generator: `Limit(depth=50, time=30, nodes=25M)` MultiPV 2 per solution step; mate-defence `Limit(depth=15, time=10, nodes=8M)`.
  - Zugzwang check: `Limit(depth=30, time=10, nodes=12M)`.
  - The tagger's `--bad_mate` mode re-checks mate puzzles at MultiPV 5, `nodes=30M`. A puzzle is flagged bad if any non-mate alternative at the second-to-last position scores > Cp(250), meaning a strong non-mating alternative exists.
  - — [generator.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py); [zugzwang.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/zugzwang.py); [tagger.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/tagger.py)
- In "Learn from your mistakes", the browser engine counts as ready at `bestmove || nodes >= 1_000_000 || millis > 3000`. — [retroCtrl.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/retrospect/retroCtrl.ts)
- Only-move / uniqueness thresholds:
  - Puzzler: `wc(best) > wc(second) + 0.7`.
  - Lila faulty-puzzle check: second ≥ 0.3524 wc (≈200 cp), or best−second povDiff < 0.14.
  - — [generator.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py); [winningChances.ts](https://github.com/lichess-org/lila/blob/master/ui/lib/src/ceval/winningChances.ts)

### Inferences
- Throughput math (inference from the fishnet comment): a top-end client does 60 × 1.45M nodes ≈ 87M nodes in ≤35 s, i.e. about 2.5M nodes/s. So a 40-move (80-ply) game at 1M nodes/ply ≈ 80M nodes takes about 30–40 s on a strong multi-core machine. On a single core at ~1M nps it takes about 80 s.
- Importing 500 games for one user at 1M nodes would cost roughly 4–11 CPU-hours. Mitigations:
  - Use lower nodes (300k, like `autoHunter`) for the first pass.
  - Use cached evals: Lichess PGN exports with `%eval` from existing server analysis, or the Lichess cloud-eval API.
  - Deep MultiPV re-search only on positions flagged with Δwc ≥ 0.1.
- Use python-chess `engine.analyse(board, Limit(nodes=...), multipv=k)` with a reused engine per worker, `Threads=1` per process, and parallelize across games. A fixed node count gives reproducible results; fixed time does not.

### Gaps
- No official Lichess statement of the average wall-clock time per analysed game was found. The numbers above are derived from code comments.
- The exact fishnet `multipv` usage (which work types set it) was not traced in lila.
