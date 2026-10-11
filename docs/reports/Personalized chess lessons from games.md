# Build blunderbot on engines and peer statistics

Yes, blunderbot can be built today from open components, and it does not need a reinforcement-learning network trained on the user's games. The premise is half right. Stockfish cannot learn from one person's moves, but no product or research pipeline asks it to. The engine acts as a stateless judge that labels every move by how much win probability it lost. A rule-based tagger then names the motif in the line that was missed or allowed. "Learning the user" happens in a handful of small per-user statistical models that update after every game and every drill: per-motif miss rates shrunk toward rating peers, per-theme puzzle ratings, spaced-repetition memory states, and bandit weights. RL is the wrong tool on two counts. Its objective is wrong: self-play networks optimize winning and match human moves at most **46%** of the time, against **~53%** for supervised Maia models. Its data scale is wrong too: even *supervised* fine-tuning of a whole network on one player makes it worse below about **5,000 games**. The supervised human-move model **Maia-2 (MIT license)** earns a central place for a different reason. It estimates how likely a player at the user's rating was to find the missed move, and puzzle-difficulty competitions show this is the single most valuable difficulty signal. Maia4All-style per-user embeddings, which show gains from about **20 games**, are a later refinement. Bandits are the only "RL" the evidence supports, and only for choosing what to teach next. Engine compute runs about **$0.001–0.002 per game**, so a 200-game onboarding costs roughly **$0.22–0.33**. The license traps are AGPL code (Lichess's motif tagger, Maia-3) and GPL board libraries shipped in the browser bundle. The biggest unknown is not technical: **no rigorous study shows that tactics training transfers to game results**. Blunderbot therefore has to measure opportunity-normalized miss rates in new games, with within-user controls, from the first release.

*Evidence labels used below: "proven" means established in a cited paper, production source code, or official documentation. "Inference" means the researchers' or this report's synthesis or arithmetic from cited facts. "Gap" means nobody has measured it. Section 6 collects all three.*

## Stockfish judges and statistics learn, so RL is the wrong tool

### The engine is a judge that never needs to know the user

The original assumption mixes two separate problems. Judging a move needs ground truth about the position, and Stockfish provides that without knowing who played. Modeling a person needs a record of that person's decisions, and that record is simply data you collect. Every production pipeline the research examined divides the work this way:

- **Lichess's puzzle generator** scans games for big evaluation swings. It proves the forced line with Stockfish at MultiPV 2 and up to 25 million nodes per step, then names the motif with about 45 hand-written python-chess predicates in `cook.py` ([lichess-puzzler generator](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py); [cook.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/cook.py)).
- **Lichess Insights** rolls per-move judgments up into "Awareness" (did you punish the opponent's mistake?) and "Luck" (did the opponent fail to punish yours?) ([PovToEntry.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/PovToEntry.scala)).
- **An independent 2026 self-hosted blunder trainer** does the same thing: engine labels, motif detection from board geometry with no engine involved, and aggregation per user ([Sviridov, 2026](https://mrlokans.work/posts/building-self-hosted-chess-blunder-trainer/)).

In blunderbot, "learning from the user" therefore means five small online estimators. Four of them carry the product:

1. per-motif miss-rate posteriors;
2. per-theme puzzle ratings;
3. per-card memory states;
4. bandit weights over themes.

The fifth is an optional per-user embedding added in a later phase. None of them needs a GPU.

### Reinforcement learning has the wrong objective and the wrong data scale

RL in the AlphaZero/Leela sense maximizes wins, which pulls a model away from any particular human. The Maia paper measured this directly on Lichess games:

- **Depth-limited Stockfish** matched human moves **33–41%** of the time.
- **Leela** matched at most **46.0%**, and its accuracy was nearly flat across the 1100–1900 rating range.
- **Supervised Maia models** reached **50.8–52.9%**, each peaking near the rating it was trained on.

([Maia, KDD 2020](https://arxiv.org/pdf/2006.01855))

The authors concluded that "AlphaZero-style deep reinforcement learning off the shelf does not pass our test." They also found that adding tree search cut human-move prediction by 5–10 points, and that a weakened Stockfish does not play like a weak human.

The data gap is just as wide. AlphaZero played about **44 million self-play games** in its first nine hours, according to a secondary source ([Chess.com on AlphaZero](https://chess.com/terms/alphazero-chess-engine)). A typical user has hundreds to low thousands of games.

Supervised fine-tuning of an entire network on one player also fails at realistic sizes:

| Games from the player | Accuracy before → after fine-tuning |
|---|---|
| 1,000 | 52.7% → **49.7%** (worse) |
| 5,000 or more | about **+4.2 points** on average |

([Maia-Individual, KDD 2022](https://arxiv.org/pdf/2008.10086))

The research found no work on chess personalization that uses inverse RL or RLHF.

### Human-move models answer the question Stockfish cannot

Stockfish can tell you a move was bad. It cannot tell you whether a player at the user's rating should have found the better one. That second question matters more, because how hard the position is predicts human error far better than who is playing. Three results show this:

- **Anderson, Kleinberg and Mullainathan (tablebase study):** difficulty features alone predicted blunders with **0.73–0.75** balanced accuracy. Rating alone reached 0.55 and clock alone 0.53. Raising a position's "blunder potential" by 0.2 increased blunder rates more than a 600-point rating gap did ([Anderson et al.](https://ar5iv.labs.arxiv.org/html/1606.04956)).
- **Biswas and Regan:** 2700-rated players chose the best move **91.0%** of the time in the easiest difficulty bin, but only **39.8%** in the hardest ([Biswas & Regan 2015](https://www.scitepress.org/papers/2015/52883/52883.pdf)).
- **2025 FedCSIS puzzle-difficulty competition:** the winner fed fine-tuned Maia-2 into gradient boosting and reached a final MSE of **52.3k (RMSE ≈ 229 rating points)**. In its ablation, removing the Maia-2 features raised MSE by **49%**. No top team used RL ([organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf); [winner's paper](https://annals-csis.org/Volume_43/drp/pdf/6497.pdf)).

Maia-2 is the practical tool for this job:

- one **23.3M-parameter** model conditioned on both players' ratings;
- **53.25%** move-matching;
- **MIT-licensed**, installed with `pip install maia2`;
- `inference_each(model, prepared, fen, elo_self, elo_oppo)` returns per-move probabilities.

([Maia-2 paper](https://arxiv.org/html/2409.20553); [maia2 repo](https://github.com/CSSLab/maia2))

The design that follows is "population difficulty model plus a per-user, per-motif offset", not a model trained from scratch on one person (inference).

Neural modeling of a single user is now feasible, but it is a refinement rather than the foundation. **Maia4All** works like this:

1. It replaces Maia-2's rating embeddings with per-player embeddings.
2. It initializes a new player's vector from 1,100 "prototype" players.
3. It trains only that vector, with the rest of the network frozen.

Top-1 accuracy rose from **51.46%** (Maia-2) to **53.22%** with about 20 games, and to **53.81%** with about 2,500 ([Maia4All](https://arxiv.org/html/2507.21488v1)).

The 2022 personalization paper also proposed the lesson rule that makes such a model worth having. Surface the blunders where the personalized model assigns more probability than the rating-matched base model, because those are "predictably difficult for this specific player" and "relatively fixable (since other players are less likely to err)" ([Maia-Individual](https://arxiv.org/pdf/2008.10086)). No public Maia4All code was found, so adopting it means reimplementing it.

### Bandits are the only reinforcement learning the evidence supports

Choosing which weakness to train next is a sequential decision, and here a light form of RL does help. **ZPDES** is a multi-armed bandit whose reward is learning progress: recent success rate minus earlier success rate. In a study of **400 children in 11 schools** it beat an expert-designed sequence. **95% vs 75%** of students succeeded at least once on the hardest exercise type. Its authors chose bandits precisely because fitting a full student model for a single learner is intractable ([Clement et al.](https://ar5iv.labs.arxiv.org/html/1310.3174)).

The broader evidence on RL for teaching is mixed. A review of RL for instructional sequencing found that **just over half** of studies showed RL policies significantly better than baselines. RL succeeded mostly when constrained by learning-science theory ([Doroudi et al. 2019](https://dx.doi.org/10.1007/s40593-019-00187-x)).

KL-regularized search over a human model (piKL, Allie) matters only if blunderbot later adds a sparring bot that plays like the user ([piKL](https://arxiv.org/abs/2112.07544); [Allie](https://arxiv.org/abs/2410.03893)).

| Tool | Job in blunderbot | Verdict | Phase |
|---|---|---|---|
| Stockfish 19 at a fixed node budget | Label every ply (eval, best move, principal variation); find the refutation of each user move | Required | 1 |
| Rule-based motif tagger (python-chess, clean-room reimplementation of `cook.py` logic) | Name the motif in the missed line or the refutation | Required | 1 |
| Beta-binomial per-motif miss rates with rating-peer priors | Decide which motifs are genuine weaknesses | Required: this is the personalization | 2 |
| Maia-2 (supervised, rating-conditioned, MIT) | P(peer finds best move), which gives expected misses, fixability, and item difficulty | Strongly recommended | 2 |
| Per-theme Glicko-2 plus FSRS-6 | Track drill skill and card memory | Required for lessons | 1–3 |
| Learning-progress bandit (ZPDES) | Choose the next weakness to train | Recommended | 3 |
| Maia4All-style per-user embedding | Predict this user's own errors; lessons where the user model beats the peer model | Optional enhancement | 4 |
| Whole-network per-user fine-tune (Maia-Individual) | Same as the row above | Avoid: hurts below about 5,000 games | — |
| Self-play or deep RL on the user's games | None | Reject | — |
| KL-regularized search over a human model (piKL, Allie) | Sparring bot that plays like the user | Outside diagnosis scope | Later |

## Three representations serve three different jobs

The user is right that games must be turned into numbers, but no single representation does everything:

- **Neural models** take tensors whose layout the pretrained model dictates.
- **Statistics** take one feature record per decision.
- **Similarity search** takes keys and vectors built for distance.

All three hang off one scalar transform.

### Every judgment starts from one win-probability transform

Raw centipawns mislead. A 300-centipawn loss when you are fifteen pawns up is irrelevant; the same loss at 0.00 decides the game. That is why Lichess, Chess.com, Maia and DeepMind all convert evaluations to win probability first ([Lichess accuracy page](https://lichess.org/page/accuracy); [scalachess eval.scala](https://github.com/lichess-org/scalachess/blob/master/core/src/main/scala/eval.scala)). The production Lichess formulas are:

```text
cp       engine score from the mover's point of view, clamped to [-1000, 1000]; any mate → ±1000
k        = 0.00368208
W(cp)    = 100 / (1 + exp(-k·cp))                 Win%, 0..100   (identical to 50 + 50·wc)
wc(cp)   = 2 / (1 + exp(-k·cp)) - 1               "winning chances", -1..+1   (Δwc = ΔW / 50)
ΔW       = W(best line before the move) - W(position after the move), mover POV
judgment ΔW ≥ 15 → Blunder;  ΔW ≥ 10 → Mistake;  ΔW ≥ 5 → Inaccuracy   (Δwc 0.3 / 0.2 / 0.1)
acc(ΔW)  = clamp(103.1668·exp(-0.04354·ΔW) - 3.1669 + 1, 0, 100);   acc = 100 if ΔW ≤ 0
slope    dW/dcp at 0.00 = 100·k/4 ≈ 0.092 Win% per cp   (a 100 cp drop from equality ≈ 9.2 points)
```

Sources for these formulas:

- **Thresholds** come from Lichess's `CpAdvice` list `(.3 → Blunder, .2 → Mistake, .1 → Inaccuracy)`. Transitions that create or lose a forced mate are judged separately ([Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala)).
- **The accuracy curve and its "+1 uncertainty bonus"** come from `AccuracyPercent.scala`. Game accuracy averages a volatility-weighted mean and a harmonic mean over sliding windows ([AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala)).
- **Chess.com** uses rating-dependent "expected points" with published bands (Inaccuracy 0.05–0.10, Mistake 0.10–0.20, Blunder above 0.20), but its curve is proprietary ([Chess.com Help](https://support.chess.com/en/articles/8572705-how-are-moves-classified-what-is-a-blunder-or-brilliant-and-etc)).
- **Maia's** blunder definition, a drop of at least 10 win-probability points, sits between the Lichess Mistake and Blunder thresholds ([Maia paper](https://arxiv.org/pdf/2006.01855)).

Use **ΔW** as the primary severity variable. It is bounded, and it saturates in positions that are already decided, so the app will not over-teach blunders made while +9.

### Neural networks dictate their own input tensors

In phases 1–3 blunderbot trains no network. The neural-representation question therefore becomes "what does the pretrained model expect?" The answer differs by generation:

| Model | Input tensor | Move output | Rating conditioning | License |
|---|---|---|---|---|
| AlphaZero | 8×8×119 (8 history steps × 14 planes + 7 constant planes) | 8×8×73 = 4,672 moves | none | paper only |
| Lc0 / Maia-1 | 112 planes of 8×8 (8 positions × 13 planes + 8 auxiliary) | 1,858 moves | a separate net per 100-Elo bin, 1100–1900; run in lc0 at `go nodes 1` | GPL-3.0 |
| **Maia-2** | **18×8×8**: 12 piece planes, 1 side-to-move, 4 castling, 1 en passant; black-to-move positions mirrored; no history | 1,880 moves (researchers' count of the code's vocabulary) | 128-d categorical Elo embeddings for self and opponent, added to attention queries | **MIT** |
| Maia-3 (Chessformer) | 64 square tokens × 352 dims = 12-d piece one-hot × (current + 7 past positions) + 2 × 128-d "soft" Elo embeddings | 64×64 from→to logits plus promotion biases | e_k = γ·e_weak + (1−γ)·e_strong, γ = (5000−k)/5000 | AGPL-3.0 code; weights carry no license |
| Stockfish NNUE | sparse inputs: HalfKAv2_hm 22,528 + Full_Threats 59,808 + PP_3Wide 4,560, into a 1,024-wide accumulator | eval only | — | GPL-3.0; do not reuse as an embedding |

Sources: [AlphaZero](https://arxiv.org/abs/1712.01815); [lc0 encoder.cc](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/encoder.cc); [maia2 utils.py](https://github.com/CSSLab/maia2/blob/main/maia2/utils.py); [Chessformer](https://arxiv.org/html/2605.19091); [nnue-pytorch docs](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md).

What this means for blunderbot:

- **Maia-2 needs only a FEN and two ratings.** The package builds the 18×8×8 tensor and masks illegal moves itself.
- **Maia-3 needs the previous seven positions.** It is the stronger model (57.1% vs 52.0% move-matching on its own test set), but to use it you must store each game's move list, not only FENs.
- **Every encoder puts the board in side-to-move perspective.** Store your own features the same way, so that mistakes with White and with Black are comparable.

If blunderbot ever trains its own model (phase 4 or later), the minimal input is the Maia-2-style 18×8×8 tensor. The 2026 best practice is 64 per-square tokens with a from-square × to-square attention policy (inference from the Chessformer and Lc0 results).

### Statistics run on one feature record per decision

The statistical layer needs one row per decision, holding interpretable, mover-relative features that python-chess computes in milliseconds. The record below is the researchers' synthesis of the Lichess, Lc0 and probing-research schemas (inference; no published standard exists). Probing studies support the feature choice: AlphaZero's activations linearly encode material, mobility, king safety and `has_mate_threat` ([McGrath et al.](https://arxiv.org/abs/2111.09259)).

| Feature group | Contents | Approx. dims | How computed |
|---|---|---|---|
| Outcome | W before, W after, ΔW, judgment one-hot, accuracy | 7 | Formulas above |
| Phase and material | PeSTO phase (N=B=1, R=2, Q=4, capped at 24); tapered material difference (MG/EG piece values P 82/94, N 337/281, B 365/297, R 477/512, Q 1025/936); opening/middlegame/endgame one-hot | 5 | [PeSTO](https://www.chessprogramming.org/PeSTO%27s_Evaluation_Function) |
| Tactical state | legal-move count; checks and captures available; hanging pieces (own, opponent); pinned pieces (own, opponent) | 7 | `board.attackers`, `board.pin` ([python-chess](https://python-chess.readthedocs.io/en/latest/core.html)) |
| Attack relations | 6×6 attacker-type × target-type counts, each direction (mirrors Stockfish's Full_Threats idea) | 72 | python-chess attack maps |
| King safety and pawns | king-zone attackers, pawn-shield count, and isolated, doubled, passed and backward pawns, for each side | 12 | Standard definitions |
| Motif labels | multi-hot over the Lichess theme vocabulary | ~30 | Tagger (Section 3) |
| Human difficulty | Maia-2 P(best), P(played), entropy of the move distribution; uniqueness gap W(best) − W(second) | 4 | Maia-2 at the user's rating; MultiPV 2 |
| Clock | seconds left, % of base time, ≤10 s flag, time spent ÷ the user's median | 4 | `%clk`, or Lichess `clocks` in centiseconds |
| Context | user rating, rating gap to opponent, speed one-hot | 6 | PGN headers |

Two rules apply to this record:

- **The working unit is counts per cell, not clustering.** A cell is motif × phase × clock state, and the record is what lets you slice into those cells. Standardize continuous features before any clustering.
- **Blunder potential can be approximated cheaply.** Anderson's measure, β(P) = losing moves ÷ legal moves, needs every legal move evaluated. A cheaper stand-in is β̂(P) = Σₘ P_Maia(m) · 𝟙[ΔW(m) ≥ 15], computed over Maia's top five or so moves. That is the probability a peer blunders here (inference; not evaluated in any paper).

### Similarity search layers exact keys, bitvectors, and embeddings

No standard "position2vec" exists. Lessons need positions that are similar *in the way the user failed*, so retrieval should go through three layers:

1. exact keys;
2. a lossless structural bitvector;
3. a learned embedding.

```text
epd4(P)      = first four FEN fields (placement, side, castling, en passant)   exact key; eval-cache key
x ∈ {0,1}^768 (96 bytes), one bit per (plane, square):
  bit(t, s, sq) = 64·(t + 6·s) + sq'     t ∈ {P,N,B,R,Q,K} = 0..5;  s = 0 mover's piece, 1 opponent's
  sq'          = sq if White to move, else sq XOR 56     (python-chess A1 = 0 … H8 = 63; mirror ranks)
Hamming(a,b) = popcount(a XOR b)         Jaccard(a,b) = popcount(a AND b) / popcount(a OR b)
pattern key  = motif + attacker→targets + phase, e.g. "fork:N>K,R:middlegame"
```

**What pgvector supports:**

- `bit(n)` columns can be indexed up to **64,000 bits**, with HNSW `bit_hamming_ops` / `bit_jaccard_ops`. The operators are `<~>` (Hamming) and `<%>` (Jaccard).
- `vector` columns can be indexed up to 2,000 dimensions, and `halfvec` up to 4,000.
- HNSW defaults are m = 16, ef_construction = 64, ef_search = 40.

([pgvector README](https://github.com/pgvector/pgvector))

**What learned embeddings can and cannot do:**

- **ChessLM** gives a 256-d self-supervised embedding built for similarity search. Its authors warn that embedding-similar positions "can have vastly different engine evaluations" ([ChessLM](https://huggingface.co/odestorm1/chesslm)).
- **Pooled Maia activations** are a reasonable choice: behavioral stylometry used 320-d global-average-pooled Maia-style features ([stylometry](https://arxiv.org/pdf/2208.01366)). However, no benchmark of them for position retrieval exists.

**The query plan** is therefore:

1. Hard-filter on motif tag, phase and material signature.
2. Run approximate nearest-neighbour search on the embedding.
3. Re-rank by Jaccard similarity of the 768-bit vector.
4. Re-verify the motif with the tagger before serving the position (inference).

### The user model is a handful of small posteriors

The "mathematical representation of the user" is a set of small, interpretable states rather than network weights:

| Component | State | Size | Updated |
|---|---|---|---|
| Weakness cell (motif × optional phase × speed) | Posterior Beta(α + x, β + n − x); expected misses E; variance V | 5 floats | After each analysed game |
| Theme skill | Glicko-2 (rating r, deviation RD, volatility σ), kept separately for blind and themed modes | 3 floats | After each attempt |
| Card memory | FSRS difficulty D, stability S, due date | 3 values | After each review |
| Curriculum | Bandit weight per (theme, level) | 1 float | After each session |
| Optional neural | Per-user embedding; 128-d if it follows Maia-2's skill-embedding size (inference) | 128 floats | Periodic refit, phase 4 |

## From imported game to ranked weakness in five stages

```text
import.user ─► analyze.pass1 (every ply) ─► detect.candidates ─► analyze.pass2 (flagged plies, MultiPV 2–3)
                                                                      │
                                       classify.motifs  +  human.difficulty (Maia-2 at user rating)
                                                                      │
     nightly: refit.peer_priors ─────────────────────────────► aggregate.weaknesses ─► plan.lessons
```

### Stage 1: import incrementally, and only from the two platforms that matter

**Lichess**

The export endpoint is `GET /api/games/user/{username}` with `Accept: application/x-ndjson`. Request these options:

- `clocks=true` (centisecond arrays)
- `evals=true`
- `opening=true`
- `division=true` (middlegame and endgame start plies)
- `accuracy=true`
- `since=<last import ms>`

Throttles are **20 games/s** anonymous, **30** with OAuth, and **60** for a user's own games. Already-analysed games arrive with per-ply evals, plus `best`, a SAN `variation`, and a `judgment` on each Inaccuracy, Mistake and Blunder ([Lichess games export spec](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-user-username.yaml)).

Operational rules:

- Make **one request at a time**, and wait at least 60 s after a 429 ([Lichess API spec](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)).
- Always send a descriptive User-Agent. A live test got 404s for existing users when it used curl's default User-Agent (undocumented anomaly).

Login with Lichess is OAuth2 with PKCE (S256 only) for public clients. There is no client secret, tokens last about a year, and there is no refresh token.

**Chess.com**

The PubAPI exposes monthly archives at `api.chess.com/pub/player/{user}/games/{YYYY}/{MM}`:

- Serial access is unlimited. Parallel requests can get a 429.
- Support for ETag and `If-None-Match` enables incremental sync.
- PGNs carry `%clk` to tenths of a second, and an optional per-game `accuracies` field.
- There are **no per-move evals** ([Chess.com PubAPI](https://www.chess.com/news/view/published-data-api)).

**Engine coverage**

Only about **6%** of Lichess database games carry Stockfish evals ([database.lichess.org](https://database.lichess.org/)). Blunderbot must therefore analyse most games itself, and every Chess.com game.

### Stage 2: run a cheap pass everywhere and a deep pass where it matters

Lichess spends a fixed node budget per position, not a fixed depth:

| Lichess use | Nodes per position |
|---|---|
| User-requested analysis | 1,000,000 |
| Broadcasts | 5,000,000 |
| Automatic "hunter" analysis | 300,000 |
| "Tutor" | 100,000 |

The puzzle generator spends up to 25M nodes at MultiPV 2, because it must prove the best move is unique ([Work.scala](https://github.com/lichess-org/lila/blob/master/modules/fishnet/src/main/Work.scala); [generator.py](https://github.com/ornicar/lichess-puzzler/blob/master/generator/generator.py)).

Copy that split:

- **Pass 1:** 300k–1M nodes, MultiPV 1, every ply.
- **Pass 2:** MultiPV 2–3 at several million nodes, only on flagged plies.

The flagged plies are:

- user moves with ΔW ≥ 5;
- positions right after an opponent move with ΔW ≥ 10 (a potential opportunity, which is Lichess's Awareness logic);
- positions where W(best) ≥ 67.5 jumped up from two plies earlier.

This flagging rule is an inference. It matters because it produces the denominator: opportunities the user *found* as well as the ones they missed.

Engine operations:

- **Process model.** Run one single-threaded Stockfish process per core. Stockfish 18 added shared NNUE weights across processes "for cloud analysis" ([SF18 release](https://github.com/official-stockfish/Stockfish/releases/tag/sf_18)).
- **Validate FENs first.** Stockfish 19 (2026-09-05) now terminates on invalid FEN or UCI input ([SF19 release](https://github.com/official-stockfish/Stockfish/releases/tag/sf_19)).
- **Use fixed node counts.** They give reproducible labels; fixed time does not.

Cache evals in a shared table keyed by `epd4`, and seed that cache before spending engine time:

- Lichess's in-game analysis.
- The no-auth `GET /api/cloud-eval` endpoint ([cloud-eval spec](https://github.com/lichess-org/api/blob/master/doc/specs/tags/analysis/api-cloud-eval.yaml)).
- Optionally, the CC0 eval dump of **416,442,401 positions** (22.4 GB compressed), best filtered to opening plies ([database.lichess.org](https://database.lichess.org/#evals)).

Endgames:

- Load the ≤5-piece Syzygy tables (about **0.98 GB**) on the analysis workers.
- Query `tablebase.lichess.org` sparingly for 6–7-piece positions. Responses are cacheable for 14 days ([tablebase spec](https://github.com/lichess-org/api/blob/master/doc/specs/tags/tablebase/standard.yaml); [Wikipedia: tablebase sizes](https://en.wikipedia.org/wiki/Endgame_tablebase)).

### Stage 3: detect mistakes, missed opportunities, and allowed tactics

A single bad move can mean two different weaknesses:

- **Missed opportunity:** the user didn't see their own tactic.
- **Allowed tactic:** the user walked into the opponent's tactic.

They need different lessons, so detect them separately. The thresholds below come from production code where possible.

| Quantity | Threshold | Origin |
|---|---|---|
| Inaccuracy / Mistake / Blunder | ΔW ≥ 5 / 10 / 15 | Lichess `Advice.scala` (proven) |
| "Found it" tolerance | within 4 Win% of the best line | Lichess "Learn from your mistakes" retry rule ([retroCtrl.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/retrospect/retroCtrl.ts)) |
| Opening grace | before the middlegame boundary, a move played in more than 1 master game is not a mistake | Lichess `explorerCancelPlies` ([nodeFinder.ts](https://github.com/lichess-org/lila/blob/master/ui/analyse/src/nodeFinder.ts)); the explorer now requires OAuth |
| Puzzle trigger | score ≥ +200 cp and wc jump > 0.6 | lichess-puzzler |
| Strict uniqueness | wc(best) − wc(second) > 0.7 (35 Win% points) | lichess-puzzler |
| Multiple solutions (reject) | second-best ≥ 0.3524 wc (≈ +200 cp), or gap < 0.14 wc (7 points) | lila [winningChances.ts](https://github.com/lichess-org/lila/blob/master/ui/lib/src/ceval/winningChances.ts) |
| Time pressure | ≤ 10 s left; blunder rate is nearly flat above this and rises sharply below | [Anderson et al.](https://ar5iv.labs.arxiv.org/html/1606.04956) |
| Opportunity gate | W(best) ≥ 67.5 (wc ≥ 0.35) and W(best) − W(second) ≥ 15 | Synthesis between the puzzler and lila rules (inference) |
| Conversion failure | wc ≥ +0.5 (≈ Win% 75) for ≥ N plies and the game is not won | Synthesis (inference) |

```text
for each user move m from position P (user to move); P' = P after m:
    best, second = pass2(P).lines[0], pass2(P).lines[1]
    ΔW  = W(best) − W_user(pass1(P'))
    gap = W(best) − W(second)

    # 1. tactic opportunity: emits BOTH found and missed rows, which gives the denominator
    if W(best) ≥ 67.5 and gap ≥ 15:
        line   = [opponent's previous move] + truncate_to_forcing(best.pv)   # Lichess puzzle convention
        motifs = tag(line, pov = user)
        emit tactic_opportunity(missed = ΔW > 4, motifs, root = P before opponent's move)

    # 2. allowed tactic: what the user walked into
    if ΔW ≥ 10:
        ref = pass2(P')                                      # opponent to move
        if W_opp(ref[0]) − W_opp(ref[1]) ≥ 15 or has_checks_or_captures(ref[0].pv):
            emit allowed_tactic(motifs = tag([m] + truncate_to_forcing(ref[0].pv), pov = opponent))

    # 3. one move can emit both rows; keep them separate
    # 4. "opponent failed to punish" (Lichess Luck) still counts as an allowed_tactic
```

`truncate_to_forcing` stops the principal variation once the best move is no longer unique, the way the puzzler's `cook_advantage` loop does. Without it, the tagger fires on unrelated later moves. The rows-and-denominator design is the researchers' synthesis of Lichess's code. No product publishes precision or recall for its missed-versus-allowed labels.

### Stage 4: classify the motif and the mistake type

`cook.py` covers most of what blunderbot needs:

- **Goal tags:** mate-in-N, crushing (> 600 cp), advantage (> 200 cp), equality.
- **Mate patterns:** back-rank, smothered, Arabian, hook, and others.
- **Motif detectors:** about 30 predicates over the solution line, including fork, hanging piece, two kinds of pin, skewer, discovered attack, double check, trapped piece, sacrifice, attraction, deflection, interference, intermezzo, clearance, x-ray, quiet move and defensive move.
- **The one engine-based tag:** zugzwang, found by a null-move test. A position is zugzwang if wc(actual) < wc(after a null move) − 0.3 ([cook.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/cook.py); [zugzwang.py](https://github.com/ornicar/lichess-puzzler/blob/master/tagger/zugzwang.py)).

For example, the fork predicate requires that the forking piece is not a king and does not land on a "bad spot", and that it attacks at least two enemy non-pawn pieces that are either worth more than the forker or hanging.

**Licensing.** The repository is **AGPL-3.0**, so copying `cook.py` into a closed network service would trigger source obligations ([LICENSE](https://github.com/ornicar/lichess-puzzler/blob/master/LICENSE)). Reimplement the detectors clean-room from their behavioural descriptions. Use Lichess theme names so that user mistakes map one-to-one onto the CC0 puzzle bank ([PuzzleTheme.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleTheme.scala)).

**Validation.** Measure per-theme precision and recall against the 6.16M Lichess puzzles that carry `Themes` labels. Naive detectors overfire badly: one project's first fork detector fired on **48%** of puzzles *not* labelled fork ([chess-coach-ai PR #473](https://github.com/AayanHetam/chess-coach-ai/pull/473)).

**Known limits of the rules:**

- Tags describe the *solution line*, not the user's thought error.
- There is no static exchange evaluation, and `overloading` is a stub.
- Quiet positional errors get only goal and phase tags.

Non-tactical mistakes are classified along metadata axes, not board geometry:

| Mistake type | Rule | Status |
|---|---|---|
| Opening error | Mistake or Blunder before the middlegame boundary, and the move is outside the masters explorer or the user's repertoire | Lichess logic plus inference |
| Endgame technique | ≤ 7 pieces, and the Syzygy WDL drops from win to draw/loss or from draw to loss (treat cursed-win and blessed-loss as draws) | Exact; inference on the schema |
| Conversion failure | Winning band held for N plies, game not won; blame the largest ΔW out of the band | Inference; Aimchess's "Advantage Capitalization" axis |
| Resourcefulness | Losing, and missed a drawing or swindling resource | Inference |
| Time trouble | Mistake made with clock ≤ 10 s or < 10% of base time | Inference from Anderson et al. |
| Positional (residual) | Mistake with no tactical tag in either line, no tablebase change, and a non-forcing refutation | Inference; no reliable algorithmic label exists |

### Stage 5: estimate weaknesses relative to rating peers, adjusted for difficulty

A raw miss rate mixes two things: "this motif is hard for everyone" and "this user is weak at it". Anderson's finding that position difficulty dominates, and that traps fool players of every rating, implies weakness must be measured **against peers at the same rating**. Small samples must also be **shrunk** toward that peer rate. The estimator, using standard empirical Bayes ([Beta-binomial reference](https://metricgate.com/docs/empirical-bayes/)) applied to chess by inference, is:

```text
user u, cell k (motif [× phase] [× speed]); opportunities i = 1..n_uk, excluding low-clock events
x_uk   = Σ missed_i                                 (low-clock misses go to a time-management cell instead)

peer prior for (k, 100-pt rating band r(u), speed), fitted on peer players j via the same pipeline:
  μ = mean_j(p_j);   s² = var_j(p_j) − μ(1−μ)·mean_j(1/n_j)     (remove binomial noise)
  κ = μ(1−μ)/s² − 1;   α = μκ;   β = (1−μ)κ

posterior          p_uk ~ Beta(α + x_uk, β + n_uk − x_uk)
shrunken rate      p̂_uk = (x_uk + α)/(n_uk + α + β)            shrink weight toward μ = κ/(κ + n_uk)
weakness flag      P(p_uk > μ | data) ≥ 0.9                      (Beta CDF)

difficulty-adjusted residual (Maia-2 at the user's rating; pFind_i = P(best move) for opportunity i):
  E_uk = Σ (1 − pFind_i);   V_uk = Σ pFind_i(1 − pFind_i);   z_uk = (x_uk − E_uk)/√V_uk   (flag z ≥ 2)

lesson priority    = P(worse than peers) × opportunities per 100 games × mean ΔW per miss × mean pFind over misses
```

The last factor in the priority formula is the KDD 2022 "fixability" idea: lessons should target the misses that peers usually find.

**How to build the peer priors.** Sample Lichess database games for each 100-point rating band and speed, run the identical pipeline over them, and refit monthly. The database is CC0, so this needs no proprietary data ([database.lichess.org](https://database.lichess.org/)). The 6% of games with embedded evals skip pass 1, but they still need pass 2 for motif tags. Running the same detectors on users and peers is not optional. Without it, the comparison is meaningless.

**Clock routing.** Anderson's data show that a motif missed with 5 seconds left is weak evidence of a knowledge gap. Maia and Maia-2 both drop positions with under 30 s from training ([Maia-2](https://arxiv.org/html/2409.20553)). So route low-clock misses to a time-management weakness instead of the motif cell.

**How existing products compare:**

| Product | What it does | Shrinkage |
|---|---|---|
| Lichess Puzzle Dashboard | Per-theme performance = average puzzle rating − 500 + 1000 × first-try win rate. Flags a theme as weak on as few as **3 failures**, compared only against the user's own overall performance ([PuzzleDashboard.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleDashboard.scala)) | None |
| Chess.com Insights | Found-versus-missed forks, pins and mates, with a "Similar Players" comparison ([Chess.com Insights](https://support.chess.com/en/articles/8708925-what-is-insights)) | Not described |
| Aimchess | Six skill categories against similar-rated players. A third-party report puts its last iOS release in July 2024 ([Marlvel](https://marlvel.ai/apps/aimchess-learn-chess-online)) | Not described |

None of the sources describes shrinkage or adjustment for difficulty. A peer-relative, difficulty-adjusted, shrunken per-motif diagnosis built from the user's own games is therefore the gap blunderbot can fill (market inference).

## Lessons must train detection, not memory of one position

### What learning science supports, and what it does not

**What predicts chess skill.** Serious solo study is the strongest single predictor of chess skill ([Charness et al. 2005](https://scinapse.io/papers/2134432271)). Deliberate practice explains only about **34%** of variance in chess performance ([Hambrick et al.](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4101876/)). A 2021 study of 178 players found the effect of highly individualized, high-quality practice was **more than 3×** the average effect ([Current Psychology 2021](https://link.springer.com/article/10.1007/s12144-021-02326-x)). That last result is the strongest evidence that personalization is worth building.

**How expertise is stored.** Chunking and template theory describe expertise as tens of thousands of perceptual patterns, with **~50,000** usually quoted for grandmasters. Templates form for *classes* of positions with variable slots ([Gobet 1998](https://bura.brunel.ac.uk/bitstream/2438/815/1/FullText.pdf)). Pattern memory is partly tied to board location: recall of mirror-reflected positions is impaired ([Gobet & Simon 1996](https://bura.brunel.ac.uk/handle/2438/1339)).

**Why players overlook tactics.** The Einstellung effect means a familiar first idea keeps steering attention even when players think they are searching for alternatives ([Sheridan & Reingold 2013](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3790829/)). Many "overlooked" tactics are this kind of fixation, not missing knowledge.

**What is missing.** No randomized or longitudinal study shows that puzzle training raises game results. The only evidence is correlational ([UNESA paper](https://proceeding.unesa.ac.id/index.php/iconbit/article/download/5316/1489/22655)).

General-learning effect sizes all point the same way:

| Technique | Effect size | Source |
|---|---|---|
| Retrieval practice | g ≈ **0.50–0.61** | [Rowland 2014, via Frontiers 2018](https://frontiersin.org/articles/10.3389/fpsyg.2018.02412/full) |
| Retrieval practice with feedback | g = 0.73 (vs 0.39 without) | same |
| Interleaving | g = **0.42** overall, 0.67 for visual material; largest when categories are hard to tell apart | [Brunmair & Richter 2019](https://www.psychologie.uni-wuerzburg.de/fileadmin/06020400/2019/Brunmair_Richter_in_press__2019_META-ANALYSIS_OF_INTERLEAVED_LEARNING.pdf) |
| Case comparison | d ≈ **0.50**; works best with the principle stated *after* comparing | [Alfieri et al. 2013](https://www.lrdc.pitt.edu/Schunn/papers/ContrastingCasesMeta-AlfieriEtAl2013.pdf) |
| Self-explanation | g = **0.55**; "explain" prompts 0.68 and "conceptualize" 0.87 beat "justify" 0.42 | [Bisra et al. 2018](https://www.Gwern.net/doc/psychology/spaced-repetition/2018-bisra.pdf) |
| Step-based tutoring systems | d = **0.76**, nearly matching human tutors at 0.79 | [VanLehn 2011](https://quality.mit.edu/files/2012/01/Quality-Symposium-Kurt-VanLehn.pdf) |

### Each weakness becomes a four-part item mix and a seven-step lesson

Retrieving one exact position trains recall of that position, not the pattern. So each weakness draws on four item types (inference from template theory and the interleaving and comparison results):

| Item type | What it is | Why |
|---|---|---|
| (a) Own-game position | The exact position from the user's game, motivating and relevant | Tests "would I find it now?" |
| (b) Near-transfer variants | Same motif and piece configuration, shifted, mirrored, or with different surrounding pieces | Mirror recall is impaired, so these must be trained explicitly. Retrieve them with the Section 2 similarity stack and re-verify each with Stockfish and the tagger |
| (c) Far-transfer database puzzles | Same theme tag from the **6,157,341**-puzzle CC0 bank | The CSV gives FEN, UCI moves, Glicko rating, RD and themes. The FEN is *before* the opponent's move: apply the first move, and the solution starts at move two ([database.lichess.org](https://database.lichess.org/#puzzles)) |
| (d) Foils | Look-alike positions where the tactic fails or no tactic exists | Train detection and disengagement against Einstellung |

The optimal ratio of these four types is unknown and should be A/B tested.

A lesson on one weakness, for example "knight forks on king and rook", runs in seven steps:

1. **Retrieval first.** Show the own-game position with no answer. This is a pretest and retrieval practice at once.
2. **Compare cases.** Put it beside two or three database positions sharing the motif.
3. **State the principle.** Give a short, engine-verified rule, *after* the comparison.
4. **Faded worked example.** Show a fully annotated line, then let the user play the key move, then solve fully.
5. **Self-explanation prompt.** Use "explain" or "conceptualize" style, for example "Which squares made the fork possible?"
6. **Drill.** 8–15 items at about 80–85% predicted success, mixing variants, puzzles and foils, with the refutation shown on errors.
7. **Unlabelled retests.** Schedule them with FSRS, plus an interleaved review one to three days later that mixes themes and does not announce them.

Hints should be graduated: "there is a tactic", then "look at the knight", then highlighted squares, then the move. Every hint downgrades the grade, following Chess.com's rule that a hint counts as a failure ([Chess.com puzzle rating 2025](https://www.chess.com/news/view/announcing-new-puzzles-rating-system)).

Rook endgames and other procedural skills use a different format. The user plays out key theoretical positions (Lucena, Philidor) against the engine, with move-level feedback, instead of single-move puzzles.

### Adaptive selection runs on three formulas

```text
Elo targeting     P(success) = 1 / (1 + 10^((b_i − θ_u,k)/400))
                  item rating for target P*:  b* = θ_u,k − 400·log10(P*/(1−P*))
                  P* = 0.85 → θ − 301;   P* = 0.75 → θ − 191;   P* = 0.65 → θ − 108
cold start        own positions/variants: b_i = θ_user + 400·log10((1 − pFind)/pFind),  initial RD ≈ 250–350
FSRS-6            R(t,S) = (1 + f·t/S)^(−w20),   f = 0.9^(−1/w20) − 1      (so R(S,S) = 0.90)
                  next interval for desired retention r:  I = (S/f)·(r^(−1/w20) − 1)
ZPDES bandit      reward r_a = success(last d/2 attempts) − success(previous d/2), on fresh items
                  w_a ← β·w_a + η·r_a;   p_a = (1 − γ)·w_a/Σw + γ/|A|;   γ ≈ 0.1–0.2
```

**Ability per theme.** Keep a Glicko-2 ability for each (user, theme, mode). Glicko-2 starts players at 1500 with RD 350 and σ 0.06, and its 95% interval is ±2·RD ([Glickman](http://www.glicko.net/glicko/glicko2.pdf)). Pelánek found that plain Elo is "simple, robust, and effective" for adaptive practice ([Pelánek 2016](https://www.fi.muni.cz/~xpelanek/publications/CAE-elo.pdf)). Count *themed*-session attempts for less: Lichess weights hinting themes at 0.2 on a win and 0.7 on a loss, because announcing the theme is a hint ([PuzzleFinisher.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleFinisher.scala)). Keep a separate "blind mixed" rating as the real detection measure.

**Difficulty targeting.** The **85% default** comes from Wilson et al., who found learning is fastest at about 15.87% error for gradient-descent learners. They warn against applying this directly to human grading ([Wilson et al. 2019](https://pmc.ncbi.nlm.nih.gov/articles/PMC6831579)). Chess.com's October 2025 redesign moved its default to "you should get most of them right".

**Cold-start difficulty.** The cold-start formula turns Maia-2's success probability into an item rating (inference). This is reasonable because difficulty predictors built this way reach roughly ±229 rating points.

**Memory scheduling.** FSRS-6 is the best interpretable scheduler in the 10k-collection Anki benchmark:

| Scheduler | Log loss | AUC |
|---|---|---|
| FSRS-6 | **0.3460** | 0.7034 |
| FSRS-5 | 0.3561 | — |
| HLR | 0.4694 | — |

Only neural sequence models do better ([srs-benchmark](https://github.com/open-spaced-repetition/srs-benchmark); [FSRS algorithm](https://github.com/open-spaced-repetition/awesome-fsrs/wiki/The-Algorithm)). Start from the default weights with desired retention 0.85–0.90, and refit per user once review logs accumulate ([py-fsrs](https://github.com/open-spaced-repetition/py-fsrs)).

Map behaviour to FSRS grades:

| Behaviour | Grade |
|---|---|
| Wrong first move | Again |
| Correct after a hint, or very slow | Hard |
| Correct at normal speed | Good |
| Correct and fast | Easy |

Give multi-move tactics a per-move timer, as Chessable's MoveTrainer does (its ladder is 4 h, 1 d, 3 d, 1 w, 2 w, 1 mo, 3 mo, 6 mo) ([Chess.com support](https://support.chess.com/en/articles/10319322-how-does-the-spaced-repetition-scheduling-work)).

**Choosing which weakness to train.** The bandit's arms are (weakness theme, difficulty level) pairs:

- Unlock arms through a prerequisite graph, for example "basic knight fork" before "fork after deflection".
- Seed each arm's weight from the Stage 5 lesson priority.
- Retire a theme to maintenance only when both its mastery estimate is high **and** its real-game miss rate has fallen.

The exploration rate, window size d, and the mastery threshold of about 0.95 are suggestions. ZPDES published no numbers, and the 0.95 figure is unverified.

### Explanations must verbalize engine facts, never compute them

Without tools, GPT-5.4 got **22.0%** of atomic chess sub-claims wrong, and smaller open-weight models got more than **40%** wrong. Engine tools substantially improved correctness ([ACT-Eval, arXiv 2608.04240](https://arxiv.org/abs/2608.04240)). Concept-guided commentary (CCC) shows the working pattern: the engine prioritizes the concepts and the LLM only verbalizes them ([CCC](https://arxiv.org/abs/2410.20811)).

For each mistake, build a JSON fact sheet containing:

- FEN, the played move, the best move with up to 6 plies of principal variation, and the refutation line;
- W before and after, and the judgment;
- motif tags for both the missed and the allowed line;
- hanging, pinned and attacked pieces;
- phase, opening, clock, and tablebase result.

Then:

1. Instruct the LLM to use only those facts.
2. Parse every SAN move it outputs with chess.js and reject anything illegal or unlisted.
3. Keep deterministic templates for the top motifs, so explanations still work without the LLM.

**Pedagogical guardrails matter as much as accuracy.** In a randomized trial with about 1,000 students, unguarded GPT-4 raised practice scores by **48%** but lowered exam scores by **17%**. A tutor version that withheld answers largely removed the harm ([Bastani et al., PNAS 2025](https://ideas.repec.org/a/nas/journl/v122y2025pe2422633122.html)). Blunderbot's tutor must never reveal a solution before an attempt, and an LLM-revealed answer counts as a failure.

**Cost.** For about 1,500 input and 200 output tokens per explanation ([Anthropic pricing](https://platform.claude.com/docs/en/about-claude/pricing)):

| Model | Cost per explanation |
|---|---|
| Claude Haiku 5.5 | ≈ **$0.00025** |
| Sonnet 5.5 | ≈ $0.005 |
| Opus 5.5 | ≈ $0.010 |

Nightly batch generation halves these costs.

### Measure improvement on new games, with each user as their own control

**Primary outcome:** the opportunity-normalized miss rate per theme in *subsequent rated games*: misses ÷ opportunities, using the same detector, over a rolling window, with shrunken estimates before and after.

**Secondary outcomes:**

- blind per-theme puzzle rating;
- FSRS retrievability;
- time to solve;
- game rating.

Game rating is the outcome users care about, but it is slow and noisy. Puzzle ratings are weak evidence. Chess.com re-rated its puzzles in 2025 because its old ratings were "significantly inflated", and practice performance can diverge from unassisted performance, as Bastani's 48%-up, 17%-down result shows.

**Regression to the mean.** Weaknesses are selected *because* they were unusually bad recently, so they will improve even with no training. The fix is **within-user randomization**:

1. Randomly assign some detected weaknesses to "train now" and others to "waitlist".
2. Compare how the miss rate changes in trained versus untrained themes over the same games.
3. Run between-user A/B tests for policy questions, such as an 85% vs 70% success target, or own-position-plus-variants vs variants only.
4. Take baselines from a window longer than the one used to detect the weakness.

**Sample size is the binding constraint (inference).** Detecting a drop in a motif's miss rate from 50% to 35% at α = 0.05 with 80% power needs about **167 opportunities per period**. ChessPuzzler reports finding about **1 puzzle-grade position per 10 games** ([ChessPuzzler](https://www.chesspuzzler.com/Blog/HowToSolveChessPuzzlesFromYourOwnGames)). So per-motif verdicts for one user will take hundreds of games. The consequences:

- Product-level claims must pool users.
- Per-user dashboards should show posterior intervals, not point changes.
- Delayed, unlabelled in-app probes at 1–4 weeks are the leading indicator.

## Map the pipeline onto the existing Bun, Hono, and Drizzle repo

The repo is a Better-T-Stack scaffold:

- `apps/server` runs Hono with tRPC v11 (`publicProcedure` / `protectedProcedure` in `src/lib/trpc.ts`).
- The database layer is Drizzle on `node-postgres` (`src/db/index.ts`).
- Auth is Better Auth with email and password only (`src/lib/auth.ts`).
- The only schema file is `src/db/schema/auth.ts`, where `user.id` is `text`.
- Schema changes are applied with `drizzle-kit push`.
- `apps/web` is TanStack Start with React 19 and the tRPC TanStack Query client.
- The package manager is Bun 1.2.10.

Nothing chess-specific exists yet, so the design below is additive.

### Service topology

```text
apps/web  (TanStack Start, React 19)            apps/server  (Bun + Hono + tRPC v11)             services/analysis (Python 3.11/3.12)
  react-chessboard or cm-chessboard (MIT)  ◄──►   routers: account, imports, games, weakness, ──►  FastAPI: POST /analyze  /tag  /maia
  chess.js (BSD-2) for SAN/legality                lessons, reviews, progress                       python-chess 1.11 (GPL, server-only)
  stockfish.js lite, single-thread WASM            pg-boss workers: import, orchestrate,            Stockfish 19 pool, 1 process per core,
  in a Web Worker (interactive only)               aggregate, plan, llm-batch                       fixed nodes; Syzygy ≤5-piece; maia2 (CPU)
                        └───────────────►  PostgreSQL (≥13) + pgvector 0.8.7  ◄───────────────────┘
```

**Job queue.** Use **pg-boss 12.37.1** (MIT). It officially supports "Node 22.12 or higher, or Bun" and PostgreSQL 13+, so it reuses the existing database with no Redis. It builds on `SKIP LOCKED`, has retries, priority and dead-letter queues, and offers a Drizzle adapter for enqueueing jobs inside the same transaction that inserts games ([pg-boss](https://github.com/timgit/pg-boss)). Its README advertises "SQL support for non-Node.js runtimes", but claiming jobs from Python through that interface is unverified. So keep queue logic in TypeScript, and have TS workers call a small Python HTTP service.

**Why Python.** The Python service exists because the strongest tooling is Python:

- python-chess for attack maps and pins;
- `maia2` (Python ≥ 3.10, < 3.13; runs on CPU, CUDA or Apple MPS) ([PyPI maia2](https://pypi.org/project/maia2/)).

**Database setup.** Enable `CREATE EXTENSION vector` and create the HNSW indexes through generated Drizzle migrations (`drizzle-kit generate` into the already-configured `src/db/migrations`), not through `push`.

### Drizzle schema for the analysis layer

```ts
// apps/server/src/db/schema/chess.ts  — sketch; enable pgvector in a migration first
import { pgTable, pgEnum, text, integer, smallint, real, boolean, timestamp, jsonb, uuid,
         primaryKey, index, uniqueIndex, vector, bit } from "drizzle-orm/pg-core";
import { user } from "./auth";

export const platform = pgEnum("platform", ["lichess", "chesscom"]);
export const eventKind = pgEnum("event_kind", [
  "tactic_opportunity",   // user to move, unique forcing win existed (found or missed)
  "allowed_tactic",       // user's move let the opponent execute a motif
  "opening_error", "endgame_tb_error", "conversion_failure", "positional_error",
]);

export const chessAccount = pgTable("chess_account", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
  platform: platform("platform").notNull(),
  username: text("username").notNull(),
  syncCursor: jsonb("sync_cursor"),          // Lichess {sinceMs}; Chess.com {"2026/09": etag}
}, (t) => [uniqueIndex("chess_account_uq").on(t.platform, t.username)]);

export const game = pgTable("game", {
  id: uuid("id").primaryKey().defaultRandom(),
  accountId: uuid("account_id").notNull().references(() => chessAccount.id, { onDelete: "cascade" }),
  externalId: text("external_id").notNull(),
  pgn: text("pgn").notNull(),
  userColor: text("user_color").notNull(),   // "w" | "b"
  userRating: integer("user_rating"),
  oppRating: integer("opp_rating"),
  speed: text("speed").notNull(),            // bullet | blitz | rapid | classical
  baseSeconds: integer("base_seconds"),
  incSeconds: integer("inc_seconds"),
  result: text("result"),
  middlePly: smallint("middle_ply"),         // Lichess `division`, else computed
  endPly: smallint("end_ply"),
  playedAt: timestamp("played_at").notNull(),
  analysisState: text("analysis_state").notNull().default("pending"),
}, (t) => [uniqueIndex("game_uq").on(t.accountId, t.externalId)]);

export const positionEval = pgTable("position_eval", {   // shared across users
  epd4: text("epd4").primaryKey(),           // first four FEN fields
  nodes: integer("nodes").notNull(),
  lines: jsonb("lines").notNull(),           // [{uci, cp, mate, pv: string[]}] in MultiPV order
  source: text("source").notNull(),          // sf19 | lichess_game | lichess_cloud
  tbWdl: smallint("tb_wdl"),                 // -2..2 when ≤ 7 pieces
});

export const plyAnalysis = pgTable("ply_analysis", {
  gameId: uuid("game_id").notNull().references(() => game.id, { onDelete: "cascade" }),
  ply: smallint("ply").notNull(),
  byUser: boolean("by_user").notNull(),
  fenBefore: text("fen_before").notNull(),
  playedUci: text("played_uci").notNull(),
  bestUci: text("best_uci").notNull(),
  winBefore: real("win_before").notNull(),   // W of best line, mover POV
  winAfter: real("win_after").notNull(),
  winDrop: real("win_drop").notNull(),
  accuracy: real("accuracy").notNull(),
  judgment: text("judgment"),                // inaccuracy | mistake | blunder | null
  phase: text("phase").notNull(),
  clockCs: integer("clock_cs"),
  spentCs: integer("spent_cs"),
  maiaPBest: real("maia_p_best"),            // Maia-2 at the mover's rating
  maiaPPlayed: real("maia_p_played"),
}, (t) => [primaryKey({ columns: [t.gameId, t.ply] })]);

export const chessEvent = pgTable("chess_event", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
  gameId: uuid("game_id").notNull().references(() => game.id, { onDelete: "cascade" }),
  ply: smallint("ply").notNull(),
  kind: eventKind("kind").notNull(),
  missed: boolean("missed").notNull(),       // false only for found tactic_opportunity rows
  motifs: text("motifs").array().notNull(),  // Lichess theme names
  rootFen: text("root_fen").notNull(),       // Lichess puzzle convention: before the trigger move
  lineUci: text("line_uci").array().notNull(),
  winDrop: real("win_drop"),
  uniqueGap: real("unique_gap"),             // W(best) − W(second)
  pFindPeer: real("p_find_peer"),
  lowClock: boolean("low_clock").notNull(),
  features: jsonb("features"),               // Section 2 feature record
  pieceBits: bit("piece_bits", { dimensions: 768 }),
  embedding: vector("embedding", { dimensions: 256 }),
}, (t) => [
  index("event_user_kind").on(t.userId, t.kind),
  index("event_motifs_gin").using("gin", t.motifs),
  index("event_bits_hnsw").using("hnsw", t.pieceBits.op("bit_jaccard_ops")),
]);
```

The learning layer needs eight more tables:

| Table | Key | Columns |
|---|---|---|
| `puzzle` | `id` (Lichess PuzzleId) | fen, moves `text[]` (UCI), rating, rating_deviation, popularity, nb_plays, themes `text[]` (GIN index), opening_tags `text[]`, piece_bits `bit(768)` |
| `peer_prior` | (motif, phase, speed, rating_band) | alpha, beta, mu, n_players, fitted_at |
| `weakness_estimate` | (user_id, motif, phase, speed) | n, x, expected_miss, var_miss, alpha_post, beta_post, p_worse, z, priority, updated_at |
| `theme_rating` | (user_id, theme, mode ∈ blind, themed) | rating, rd, volatility, attempts |
| `review_card` | `id` | user_id, source ∈ own_game, variant, lichess_puzzle, foil; event_id / puzzle_id; root_fen; line_uci; motifs; item_rating, item_rd; FSRS state, stability, difficulty, due, last_review, reps, lapses |
| `review_log` | `id` | card_id, user_id, mode, grade 1–4, first_move_correct, hints, elapsed_ms, reviewed_at |
| `bandit_arm` | (user_id, theme, level) | weight, unlocked, mastered, last_reward |
| `experiment_assignment` | (user_id, theme) | arm ∈ train_now, waitlist; assigned_at; baseline_n; baseline_x |

### tRPC surface, auth, and the web client

All procedures except the health check go through the existing `protectedProcedure`:

| Procedure | Purpose |
|---|---|
| `account.link` | Link a platform account |
| `imports.start` / `imports.status` | Start an import and poll its progress |
| `games.list` / `games.get` | List games; get one with per-ply analysis |
| `weakness.profile` | Shrunken rates, intervals and priority |
| `lessons.next` | Next lesson, chosen by the bandit |
| `reviews.due` / `reviews.submit` | Fetch due cards; write a review log and update FSRS and Glicko in one transaction |
| `progress.summary` | Trained-vs-waitlist miss-rate deltas |

**Auth.** Better Auth's existing `account` table already has `accessToken` columns, so Lichess OAuth tokens can live there. Whether Better Auth ships a built-in Lichess provider is unverified. A generic OAuth2 configuration with PKCE and no client secret matches what Lichess requires. Chess.com linking needs only a username, because its data is public.

**Web client.** Use react-chessboard (MIT, 5.12.1) or cm-chessboard (MIT) with chess.js (BSD-2, 1.4.0). **Keep chessground and chessops out of the browser bundle** unless the frontend will be GPL. Both are GPL-3.0-or-later, and shipping JavaScript to a browser counts as "conveying" ([GPL-3.0](https://www.gnu.org/licenses/gpl-3.0.txt); [npm chessops](https://www.npmjs.com/package/chessops)).

For interactive exploration, the single-threaded lite build of stockfish.js (about 1.6 MB, GPL-3.0) runs in a Web Worker without cross-origin isolation. The multi-threaded build needs COOP/COEP headers ([stockfish.js](https://github.com/nmrugg/stockfish.js)). Serving it to browsers counts as distribution, so ship the license and a source link.

### Build in four phases, each with an exit test

| Phase | Ships | Components | Exit test |
|---|---|---|---|
| 1. Mistake inbox | Import, analysis, "retry your mistakes" | Lichess + Chess.com import; pass 1 + pass 2; Lichess thresholds; missed/allowed split; clean-room tagger for about eight motifs (hanging piece, fork, pin, skewer, discovered attack, back-rank mate, mate-in-1/2, trapped piece); FSRS cards from own positions; puzzle CSV import | Per-theme tagger precision ≥ an agreed bar against Lichess labels; labels reproducible at fixed nodes |
| 2. Weakness profile | Peer-relative diagnosis | Opportunity rows; Lichess-sample peer priors; Beta-binomial with clock routing; Maia-2 P(best); opening, tablebase, conversion and time categories; dashboard with intervals | Flags reproduce on a held-out half of the user's games |
| 3. Adaptive curriculum | Lessons and measurement | Seven-step lessons; variants and foils via pgvector; per-theme Glicko; ZPDES bandit; grounded LLM explanations; within-user randomization | Primary metric and waitlist control live for all users |
| 4. Personal human model | Predictive personalization | Maia4All-style embedding reimplemented on Maia-2; "user model > peer model" lessons; own difficulty regressor (Maia features + gradient boosting); optional sparring bot | Beats Maia-2-only at ranking the user's held-out mistakes |

### Costs stay in fractions of a cent per game

| Item | Cost | Basis |
|---|---|---|
| Pass 1 at 1M nodes × 80 positions | ≈ **$0.0011 / game** | ~1.0–1.2M nodes/s per cloud vCPU ([SF benchmarks](https://github.com/amdw/enginecloud/blob/main/stockfish/benchmarks/SF_BENCHMARKS.md)); AWS c7a ≈ $0.051 per vCPU-hour ([Vantage](https://instances.vantage.sh/aws/ec2/c7a.xlarge)) |
| Pass 1 at Lichess's ~1.45M nodes | ≈ $0.0016 / game | Same |
| 200-game onboarding | ≈ **$0.22–0.33** on demand | Arithmetic (inference); spot capacity is cheaper |
| 16-vCPU analysis box | ≈ 600–850 games / hour | Arithmetic (inference) |
| Peer priors from 100,000 sampled games | ≈ $110–160, one-time per refit | Arithmetic (inference); 6% of games skip pass 1 |
| Maia-2 inference | Not benchmarked; a single forward pass on a 23.3M model, CPU is enough | Inference; no published latency |
| LLM explanation (1.5k in / 200 out) | Haiku 5.5 ≈ $0.00025; Sonnet 5.5 ≈ $0.005; Opus 5.5 ≈ $0.010 | [Anthropic pricing](https://platform.claude.com/docs/en/about-claude/pricing) |
| Storage | Puzzle CSV 307 MB compressed; ≤5-piece Syzygy ≈ 0.98 GB; optional eval dump 22.4 GB; 6-piece Syzygy 160 GB later | [database.lichess.org](https://database.lichess.org/); [Wikipedia](https://en.wikipedia.org/wiki/Endgame_tablebase) |

Pass 2 and the phase-4 embedding training are unmeasured. For comparison, a whole-network per-player fine-tune took **40 minutes on one V100 using about 300 MB of VRAM** ([Maia-Individual](https://arxiv.org/pdf/2008.10086)).

### License constraints decide where code runs

| Component | License | Constraint for a closed-source app |
|---|---|---|
| Stockfish 19 binary | GPL-3.0 | Server use is not distribution. Shipping it requires the license plus source ([Stockfish terms](https://github.com/official-stockfish/Stockfish#terms-of-use)) |
| stockfish.js (npm `stockfish` 19.0.0) | GPL-3.0 | Browser delivery is distribution: include license and source link. Treating a UCI Web Worker as an "aggregate" is an inference, not legal advice |
| python-chess 1.11.2 | GPL-3.0+ | Fine server-side; network interaction is not conveying |
| chess.js / react-chessboard / cm-chessboard | BSD-2 / MIT / MIT | Permissive |
| chessground / chessops | GPL-3.0-or-later | Forces GPL on the bundled client |
| lichess-puzzler (`cook.py`) | AGPL-3.0 | Reimplement clean-room, or open-source the tagging service ([AGPL §13](https://www.gnu.org/licenses/agpl-3.0.txt)) |
| Maia-2 (`maia2` 0.11.0) | MIT | Least encumbered human model |
| Maia-3 code / weights | AGPL-3.0 / no stated license | Contact CSSLab before commercial use ([maia3](https://github.com/CSSLab/maia3)) |
| Maia-1, maia-individual | GPL-3.0; maia-individual's README and sidebar conflict (GPL vs AGPL) | Server-side only; check the LICENSE file |
| Lichess games, puzzles, evals | CC0 (broadcasts are CC BY-SA) | No obligations ([database.lichess.org](https://database.lichess.org/)) |
| Chess.com PubAPI data | No explicit data license; respect Chess.com IP | Read-only use |
| pg-boss / pgvector / onnxruntime | MIT / PostgreSQL / MIT | Permissive |

## Separate what is proven from what is inferred

| Claim | Status | Basis |
|---|---|---|
| RL/self-play models predict human moves worse than supervised imitation (Leela ≤ 46% vs Maia ~52–53%) | Proven | [Maia paper](https://arxiv.org/pdf/2006.01855) |
| Whole-network per-user fine-tuning needs ≥ 5,000 games and hurts at 1,000 | Proven | [Maia-Individual](https://arxiv.org/pdf/2008.10086) |
| Embedding-only personalization gains about 1.8 points from ~20 games | Proven (move-matching only); value for lessons untested | [Maia4All](https://arxiv.org/html/2507.21488v1) |
| Position difficulty predicts blunders far better than rating or clock | Proven | [Anderson et al.](https://ar5iv.labs.arxiv.org/html/1606.04956) |
| Human-move models are the key ingredient in human-perceived difficulty | Proven (competition ablation, +49% MSE without) | [ousou paper](https://annals-csis.org/Volume_43/drp/pdf/6497.pdf) |
| Win%, judgment and accuracy formulas | Proven (production code) | [Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala) |
| Learning-progress bandits beat expert sequencing | Proven outside chess only | [ZPDES](https://ar5iv.labs.arxiv.org/html/1310.3174) |
| FSRS-6 is the best interpretable scheduler | Proven on Anki data, not chess | [srs-benchmark](https://github.com/open-spaced-repetition/srs-benchmark) |
| Testing, interleaving, comparison, self-explanation and ITS effects | Proven in general domains; untested in chess | Section 4 sources |
| Missed-vs-allowed algorithm and the opportunity gate thresholds | Inference (synthesis of Lichess code) | Section 3 |
| Peer-prior Beta-binomial plus Maia-difficulty residual as the weakness score | Inference; no chess paper | Section 3 |
| Item mix ratios and the 75–85% success targets for chess | Inference | Section 4 |
| Pooled Maia activations as similarity embeddings | Inference; no retrieval benchmark | Section 2 |
| Maia-2 CPU latency of "seconds per game" | Inference; no benchmark | Section 5 |
| Tactics training transfers to game results | **Gap**: no rigorous study | [UNESA](https://proceeding.unesa.ac.id/index.php/iconbit/article/download/5316/1489/22655) |
| Precision and recall of rule-based motif tags | **Gap**: none published; naive detectors overfire | [PR #473](https://github.com/AayanHetam/chess-coach-ai/pull/473) |
| Per-individual blunder predictor | **Gap**: none published; Maia's blunder model is population-level (71.7%) | [Maia paper](https://arxiv.org/pdf/2006.01855) |

Six risks fall outside the evidence table.

1. **Rating scales.** Maia models are conditioned on Lichess ratings, while Chess.com users bring Chess.com ratings. The research did not examine how to map one onto the other, and a miscalibrated Elo input would bias every difficulty estimate.
2. **Positional mistakes.** They have no reliable algorithmic label. Only concept-vector research and proprietary tools touch them, so the "positional" bucket will stay a residual explained cautiously.
3. **Per-user identifiability.** A personalized model identified a player from one side of 100 games **98%** of the time among 400 candidates ([Maia-Individual](https://arxiv.org/pdf/2008.10086)). Per-user embeddings and event logs are therefore personal data, and should be deletable through the `onDelete: "cascade"` chain from `user`.
4. **Unpublished operating details.** Lichess publishes no numeric rate limits, and the User-Agent 404 anomaly is undocumented.
5. **Unverified integrations.** The pg-boss Python path and a Better Auth Lichess provider still need verification.
6. **Statistical power.** As computed in Section 4, most per-user, per-motif verdicts will stay uncertain for months of play. Interface honesty about intervals is a product requirement, not a nicety.

## Conclusion

The user's two assumptions point at the right concerns but the wrong mechanisms. Personalization is real, but it lives in *measurement*, not in a trained engine:

- the opportunity denominator;
- the rating-peer prior;
- the Maia-derived expectation of what a peer would have found.

What makes blunderbot better than per-game review is that denominator. "You missed 6 of 9 knight-fork chances that 1500s usually find" is a statement no current product publishes with shrinkage and difficulty adjustment. Building it needs CPU, CC0 data and careful statistics, not GPUs and reinforcement learning.

That reframing has a consequence. The scarce resource is not compute but *evidence*. Opportunities for any single motif accumulate slowly: by ChessPuzzler's figure, puzzle-grade positions surface about once per ten games. And puzzle-to-game transfer has never been demonstrated. So the most valuable early investment is the experiment harness: waitlisted themes, blind retests, and pooled analysis across users. Running it would let blunderbot become the first product to show whether personalized lessons from a player's own mistakes actually reduce those mistakes. That finding would be worth more than any model architecture choice.
