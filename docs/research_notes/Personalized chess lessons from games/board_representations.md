# Board, Move, and Game Representations for Chess ML (as of Oct 2026)

Scope: how chess positions, moves, games, and mistake events are encoded for (a) neural nets such as human-move predictors (Maia family), (b) feature vectors for statistics and clustering of a user's mistakes, and (c) similarity search over positions. Numbers come from primary sources and source code wherever possible. Notation: "cp" means centipawns, and "STM" means side to move.

---

## 1. Text formats: FEN, EPD, PGN, SAN/UCI, and Lichess PGN metadata (%clk, %eval)

### Takeaway
FEN (6 fields) is the canonical single-position key. PGN (with SAN movetext) is the canonical game container. UCI long algebraic (`e2e4`, `e7e8q`) is the canonical move token for ML vocabularies. Lichess PGNs embed per-move `[%clk h:mm:ss]` and, for about 6% of games, `[%eval x.xx]` / `[%eval #n]` in brace comments. That is enough to rebuild the (position, move, eval, clock) sequence for most of a user's games, but evals must usually be computed by your own engine.

### Cited Findings
- **PGN Seven Tag Roster** (required, in order): Event, Site, Date, Round, White, Black, Result. Result is one of `1-0`, `0-1`, `1/2-1/2`, `*` and must equal the movetext termination marker — [PGN standard](https://www.saremba.de/chessgml/standards/pgn/pgn-complete.htm)
- **FEN six fields:**
  1. Piece placement, rank 8→1. Uppercase is White and lowercase is Black. Digits are runs of empty squares.
  2. Active color `w`/`b`.
  3. Castling availability `KQkq` or `-`.
  4. En-passant target square or `-`. The spec says it is given "if and only if the last move was a pawn advance of two squares", but many tools only emit it when a capture is legal.
  5. Halfmove clock (plies since last pawn move or capture; 50-move rule).
  6. Fullmove number (starts at 1 and increments after Black moves).

  — [PGN standard §16.1](https://www.saremba.de/chessgml/standards/pgn/pgn-complete.htm)
- **EPD** = the first four FEN fields (no clocks) followed by zero or more `opcode operand;` operations. Examples: `ce` (centipawn evaluation, from the side to move's viewpoint), `v0`/`v1`/`v2` (opening/variation), `eco`, `nic`. `bm`/`am`/`id`/`acd`/`pv` are also standard opcodes, but my fetch truncated before their definitions — [PGN standard §16.2](https://www.saremba.de/chessgml/standards/pgn/pgn-complete.htm)
- **SAN:**
  - Piece letters are K Q R B N, and pawns have no letter.
  - `x` marks a capture.
  - Disambiguation is by origin file, then rank, then full square.
  - `+` marks check and `#` marks mate.
  - `=Q` marks promotion.
  - `O-O` / `O-O-O` mark castling (letter O).
  - Comments use `{...}` (no nesting) or `;` to end of line.
  - NAGs: `$1`=!, `$2`=?, `$3`=!!, `$4`=??, `$5`=!?, `$6`=?!

  — [PGN standard](https://www.saremba.de/chessgml/standards/pgn/pgn-complete.htm)
- **UCI move strings in python-chess:** `Move.uci()` / `Move.from_uci()`. The null move is `0000`. `Board.fen()`, `Board.epd()`, `Board.san(move)` convert between forms — [python-chess core docs](https://python-chess.readthedocs.io/en/latest/core.html)
- **Lichess standard-game PGN example** (verbatim): `1. e4 { [%eval 0.17] [%clk 0:00:30] } 1... c5 { [%eval 0.19] [%clk 0:00:30] }` — [database.lichess.org](https://database.lichess.org/)
- **Lichess headers** in the sample: Event, Site, Date, Round, White, Black, Result, UTCDate, UTCTime, WhiteElo, BlackElo, WhiteRatingDiff, BlackRatingDiff, WhiteTitle, ECO, Opening, TimeControl, Termination. Ratings are Glicko-2. Bots carry `[WhiteTitle "BOT"]` — [database.lichess.org](https://database.lichess.org/)
- **Lichess eval conventions:**
  - About 6% of games include Stockfish evals.
  - Evals are always from White's point of view.
  - `[%eval 2.35]` = +235 cp.
  - `[%eval #-4]` = getting mated in 4.

  — [database.lichess.org](https://database.lichess.org/)
- **Lichess clocks:** `%clk` comments to the second since April 2017. A separate 2013–2021 export carries centisecond `%clkc` — [database.lichess.org](https://database.lichess.org/)
- **Lichess database scale and quirks:**
  - 8,220,312,882 standard rated games in 165 monthly `.pgn.zst` files (Jan 2013–Sep 2026), totalling 2.6 TB compressed, about 7.1× larger uncompressed.
  - Older exports (to Dec 2020) may lack Round/Date and omit black move numbers after comments.
  - Many traditional PGN tools (SCID, ChessBase) can't open the files. The page suggests python-chess or Scoutfish instead.

  — [database.lichess.org](https://database.lichess.org/)
- **Lichess evaluation dump** `lichess_db_eval.jsonl.zst` (416,442,401 positions):
  - One JSON object per line. `fen` holds only pieces, STM, castling, and en passant (no clocks, so effectively EPD-like).
  - `evals[]` is ordered by number of PVs. Each entry has `knodes`, `depth`, and `pvs[]` with `cp` or `mate` plus `line` (UCI_Chess960 notation).
  - Lichess recommends taking the highest-depth eval and its first PV.

  — [database.lichess.org](https://database.lichess.org/)

### Inferences
- Use FEN as the display and interchange key. For dedup and caching use a normalized "EPD-4" key (the first four FEN fields), which is what the Lichess eval dump already uses. You can then join a user's positions against the 416M-position cloud-eval dump before spending your own engine time.
- Store moves as UCI in all ML tensors and indexes, and keep SAN only for UI and lesson text. UCI is context-free, while SAN needs the position to resolve.
- Clock deltas between consecutive `%clk` values of the same player, plus the increment from the `TimeControl` tag, give time-spent-per-move. That is a key "was this a time-pressure blunder?" feature.
- Because only about 6% of Lichess games carry `%eval`, a personalized-lesson app must run its own engine (or use the cloud-eval dump or API) for most user games.

### Gaps
- I didn't retrieve the exact EPD definitions for `bm`, `am`, `id`, `acd`, and `pv` (truncated fetch). They are standard in the same spec.
- I didn't verify the original "PGN Specification Supplement" that defines `[%clk]`/`[%emt]` command syntax. Lichess's usage is confirmed above.

---

## 2. Bitboards: 64-bit integer boards, library usage, and magic-bitboard attack generation

### Takeaway
A bitboard is a 64-bit integer with one bit per square (python-chess: A1=0 … H8=63). A full position is about 12 piece bitboards (6 types × 2 colors), or equivalently 6 type masks + 2 color masks, plus STM, castling, en passant, and clocks. Sliding-piece attacks are computed with magic-bitboard perfect hashing (mask → multiply → shift → table lookup). Bitboards are the engine-level representation. The "12×8×8 binary planes" used by every neural net are simply these 12 bitboards unpacked to tensors.

### Cited Findings
- python-chess `SquareSet`: "Square sets are internally represented by 64-bit integer masks of the included squares". Bitwise ops give union, intersection, and shifts. Example: `SquareSet([A8, A1])` = `0x0100_0000_0000_0001`. Squares are numbered `chess.A1=0 … chess.H8=63` — [python-chess core](https://python-chess.readthedocs.io/en/latest/core.html)
- python-chess exposes bitboard-derived tactical primitives that are directly usable as features:
  - `attacks(square)`: pinned pieces still attack.
  - `attackers(color, square)`
  - `is_attacked_by`
  - `is_pinned(color, square)`
  - `pin(color, square)`: returns the pin ray mask, or the full board if not pinned.

  — [python-chess core](https://python-chess.readthedocs.io/en/latest/core.html)
- **chessops** (TypeScript, by the python-chess author and used in the Lichess front end) implements `SquareSet` "as a bitboard" stored as two 32-bit halves (`lo`, `hi`), because JS numbers are not 64-bit integers. Its slider attacks use Hyperbola Quintessence. The source comments that "Magic Bitboards would deliver slightly faster lookups, but also require" more memory — [chessops squareSet.ts](https://github.com/niklasf/chessops/blob/master/src/squareSet.ts), [chessops attacks.ts](https://github.com/niklasf/chessops/blob/master/src/attacks.ts)
- **chess.js does NOT use bitboards.** Its source documents "the 0x88 Move Generation Algorithm", where `square & 0x88 != 0` is a cheap off-board test — [chess.js source](https://github.com/jhlywa/chess.js/blob/master/src/chess.ts)
- **Magic bitboards** are perfect hashing in four steps:
  1. Mask the relevant occupancy.
  2. Multiply by a per-square magic number.
  3. Right-shift by (64 − n) bits.
  4. Index a precomputed attack table.

  The core code is `occ &= mask; occ *= magic; occ >>= shift; return aptr[occ];`. "Fancy" magics need about 800 KiB (rooks) + 38 KiB (bishops). Magics are the "de facto standard" in Stockfish, Crafty, Arasan, and Houdini. PEXT (BMI2) is a hardware alternative — [chessprogramming.org Magic Bitboards](https://www.chessprogramming.org/Magic_Bitboards)
- Lc0's own training-data format stores the board as 104 `uint64_t` bitboards (8 history positions × 13 planes) rather than floats — [lc0 trainingdata_v6.h](https://github.com/LeelaChessZero/lc0/blob/master/src/trainingdata/trainingdata_v6.h)

### Inferences
- An app developer doesn't need to implement magic bitboards: python-chess (Python), chessops (TS), or shakmaty (Rust) already provide them. What matters is that **12 bitboards = a lossless 768-bit vector** (12×64) for the piece layout. That vector is simultaneously:
  - the input to every neural net (unpacked to 12×8×8);
  - a compact storage format (96 bytes);
  - an exact-match or Hamming/Jaccard similarity key (see §6).
- In a browser, use chessops rather than chess.js if you want bitboard-based features (attack maps, pins, x-rays) computed client-side.

### Gaps
- I didn't fetch shakmaty, Stockfish `bitboard.cpp`, or PEXT-specific docs. PEXT table sizes are not recorded here.

---

## 3. Policy/value-network input planes: AlphaZero (8×8×119 / 4672), Lc0 (112 planes / 1858), Maia, Maia-2, Maia-3

### Takeaway
- **AlphaZero:** 8×8×119 input (8-step history × 14 planes + 7 constant planes) and an 8×8×73 = 4,672-way policy.
- **Lc0:** keeps the idea but uses 112 planes (8 × 13 + 8 aux) and a compacted 1,858-move policy.
- **Original Maia (2020):** literally Lc0 weights (6 blocks × 64 filters, SE), so it uses Lc0's 112-plane format with history and the 1,858 policy, run in lc0 at `nodes=1`.
- **Maia-2 (NeurIPS 2024):** drops history and uses an 18-channel current-position-only tensor, mirrored to White's perspective, with categorical Elo embeddings and an 1,880-move vocabulary.
- **Maia-3 (ICLR 2026, "Chessformer"):** a 64-square-token transformer with history and soft Elo embeddings. It is the current state of the art for human-move prediction at 57.1% move-matching.

### Cited Findings

**AlphaZero (Silver et al., arXiv 1712.01815 / Science 2018)**

Table S1 (chess), with features per history step and T=8 steps:

| Feature | Planes |
|---|---|
| P1 piece | 6 |
| P2 piece | 6 |
| Repetitions | 2 |
| Colour | 1 |
| Total move count | 1 |
| P1 castling | 2 |
| P2 castling | 2 |
| No-progress count | 1 |
| **Total** | **119** |

— [AlphaZero arXiv 1712.01815](https://arxiv.org/abs/1712.01815)

- Input is an N×N×(MT+L) stack with M=14, T=8, L=7 → 14·8+7 = 119. History steps before move 1 are zeroed. "The board is oriented to the perspective of the current player". "Counts are represented by a single real-valued input; other input features are... one-hot" — [AlphaZero arXiv 1712.01815](https://arxiv.org/abs/1712.01815)
- **Policy:** "8 × 8 × 73 stack of planes encoding a probability distribution over 4,672 possible moves". Each square is the "pick up" square.
  - 56 planes for "queen moves" (distance 1..7 × 8 compass directions).
  - 8 knight-move planes.
  - 9 underpromotion planes (3 directions × N/B/R). Other 7th-rank pawn moves promote to queen.

  — [AlphaZero arXiv 1712.01815](https://arxiv.org/abs/1712.01815)
- McGrath et al. restate the input as z0 ∈ R^{8×8×(14h+7)}, giving 8×8×21 for h=1 and 8×8×119 for h=8. Black-to-move boards are "flipped horizontally and vertically". History gives only "a marginal empirical increase in performance" — [McGrath et al. arXiv 2111.09259](https://arxiv.org/abs/2111.09259)

**Leela Chess Zero (Lc0)**

- "Input to the neural network is 112 planes 8×8 each." The convolutional policy head outputs 80×8×8, of which "only 73×8×8 is actually used", and these are gathered into a length-1858 vector. The value head is WDL (3-way softmax), and there is an optional moves-left head. Typical CNN sizes are 10×128, 20×256, 24×320 (blocks×filters, SE-ResNet) — [lczero.org NN topology](https://lczero.org/dev/backend/nn/)
- Constants: `kMoveHistory = 8`, `kPlanesPerBoard = 13`, `kAuxPlaneBase = 13·8 = 104` — [lc0 encoder.h](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/encoder.h)
- **Per history slot i (base = 13i):**
  - Planes 0–5 are our P, N, B, R, Q, K.
  - Planes 6–11 are their pieces.
  - Plane 12 is all-ones if the position has repeated (`repetitions >= 1`).

  — [lc0 encoder.cc](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/encoder.cc)
- **Aux planes 104–111 in `INPUT_CLASSICAL_112_PLANE`:**

  | Plane | Contents |
  |---|---|
  | +0 | us O-O-O |
  | +1 | us O-O |
  | +2 | them O-O-O |
  | +3 | them O-O |
  | +4 | all-ones if we are black |
  | +5 | rule-50 counter |
  | +6 | zeros ("used to be movecount plane") |
  | +7 | all-ones (lets convolutions detect the board edge) |

  — [lc0 encoder.cc](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/encoder.cc)
- **Newer Lc0 input formats:**
  - Castling is encoded as rook-square masks (Chess960-safe).
  - `HECTOPLIES`: rule-50 / 100.
  - `CANONICALIZATION`: picks one of 8 board symmetries when no castling rights remain (only horizontal flip if pawns exist), puts the en-passant mask in plane +4, and stops history at irreversible moves.
  - `ARMAGEDDON`: plane +6 marks black-to-move.
  - Missing history is either left at zero or padded by repeating the earliest position, depending on `FillEmptyHistory`.

  — [lc0 encoder.cc](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/encoder.cc)
- **Lc0 V6 training record (8,356 bytes):**
  - `probabilities[1858]` and `planes[104]` (uint64 bitboards).
  - Four castling bytes, `side_to_move_or_enpassant`, `rule50_count`, `invariance_info`.
  - root/best/played/orig Q, D, M values, `result_q`/`result_d`, `plies_left`, `visits`.
  - `played_idx`, `best_idx`, `policy_kld`, `q_st`.

  — [lc0 trainingdata_v6.h](https://github.com/LeelaChessZero/lc0/blob/master/src/trainingdata/trainingdata_v6.h)

**Original Maia (McIlroy-Young et al., KDD 2020)**

- Maia models are "saved as Leela Chess neural networks" (`.pb.gz`). They require lc0 as the "body", and "a nodes limit of 1 is what we use". There are 9 models for ratings 1100–1900. Training used Lichess games from Jan 2017–Nov 2019 — [CSSLab/maia-chess](https://github.com/CSSLab/maia-chess)
- The network is "6 blocks and 64 filters". 24×320 gave only "a small performance boost at a significant cost in compute". The policy output is described as 8×8×73. Giving Maia "the previous 12 ply (6 moves for each player)" of history "significantly improves our move-matching accuracy". Positions where the player had under 30 s are excluded — [Maia paper arXiv 2006.01855](https://arxiv.org/abs/2006.01855)

**Maia-2 (Tang et al., NeurIPS 2024, arXiv 2409.20553)**

- **Input:** 18 channels of 8×8.
  - 12 piece planes.
  - 1 side-to-move plane.
  - 4 castling planes.
  - 1 en-passant plane.

  Black-to-move positions are mirrored. It uses "only... the current board position as input (as opposed to six)" — [Maia-2 arXiv 2409.20553](https://arxiv.org/html/2409.20553)
- **Code confirmation:** `board_to_tensor` builds `(12 + 6, 8, 8)` (turn, 4 castling, en passant). `get_all_possible_moves()` = every queen and knight move from every square on an empty board + White promotion moves × {q, r, b, n}. That is 1,792 + 88 = **1,880 move classes** (my count of the code's enumeration) — [maia2 utils.py](https://github.com/CSSLab/maia2/blob/main/maia2/utils.py)
- **Architecture:**
  - ResNet (12 blocks, 256 channels) → 8 channels of 8×8, each flattened (64) and projected to d=1024 (sequence length 8).
  - "Skill-aware attention": 128-d categorical Elo embeddings for the active and opponent player are concatenated and added to the queries, Q* = Q + (e_a ⊕ e_o)W*. There are 2 blocks with 16 heads.
  - Elo bins are 100 points wide, from <1100 to ≥2000.
  - Heads: policy (human move), auxiliary multi-hot (legal moves, moved and captured piece, from/to squares, check), and value (game outcome).
  - 23.3M parameters.
  - Trained on 9.15B positions / 168.93M Lichess rapid games.
  - The first 10 plies and positions with under 30 s on the clock are dropped.

  — [Maia-2 arXiv 2409.20553](https://arxiv.org/html/2409.20553)
- **pip API:** `pip install maia2`, then `inference.inference_each(model, prepared, fen, elo_self, elo_oppo)` returns `move_probs` (dict) and `white_expected_score`. The latter is "not a calibrated win probability". The README now recommends Maia-3 for new projects — [CSSLab/maia2](https://github.com/CSSLab/maia2)

**Maia-3 / Chessformer (Monroe, Eilender, …, Anderson; ICLR 2026; arXiv 2605.19091)**

- **Input:** 64 square tokens, each a 12-d piece one-hot for the current position plus n=7 past positions (earliest repeated if unavailable), flipped to STM.
  - **Maia-3 Elo conditioning:** two 128-d "soft embeddings" are appended to every token, e_k = γ·e_weak + (1−γ)·e_strong with γ = (5000−k)/5000. Input depth is 12×(1+n) + 2×128 = 352.
  - **Leela-CF variant:** adds repetition flags, 4 castling flags, black-to-move, rule50/100, and edge planes, for 112 per token.

  — [Chessformer arXiv 2605.19091](https://arxiv.org/html/2605.19091)
- **Policy head:** query per from-square · key per to-square gives a 64×64 logit matrix. Promotions get additive per-piece biases. Castling is encoded as king-moves-two, and en passant as a diagonal move — [Chessformer arXiv 2605.19091](https://arxiv.org/html/2605.19091)
- **Maia-3 results and data:**
  - Sizes 3M/5M/23M/79M, all 8 layers.
  - Trained on Lichess blitz Jan 2023–Jul 2025 with 22 rating bins equalized.
  - Move-matching: Maia-3-79M **57.1%**, 23M 56.6%, 5M 55.4%, Allie-Policy (355M) 55.7%, Maia-2 52.0%.

  — [Chessformer arXiv 2605.19091](https://arxiv.org/html/2605.19091); [ICLR 2026 poster](https://iclr.cc/virtual/2026/poster/10011702)
- The maia-chess repo README now points to Maia-3 (Chessformer) as the newest architecture — [CSSLab/maia-chess](https://github.com/CSSLab/maia-chess)

**Move-vocabulary arithmetic** (verified by my own enumeration)

- Queen-line + knight from→to pairs on 8×8 = 1,792.
- Promotion from→to pairs for one side = 22 (8 straight + 14 diagonal).
- So:
  - **Lc0 1858** = 1792 + 22×3 underpromotions (N/B/R). Queen promotion is folded into the plain move.
  - **Maia-2 1880** = 1792 + 22×4.
  - **DeepMind 1968** = 1792 + 2 sides × 22×4. There is no board flip in its FEN-token setup.

  Lc0's policy map source is [lc0 policy_map.h](https://github.com/LeelaChessZero/lc0/blob/master/src/neural/tables/policy_map.h); the DeepMind count is in [arXiv 2402.04494](https://arxiv.org/html/2402.04494).

### Inferences
- **For (a), feeding a user's positions into a human-move model:**
  - Use Maia-2 via pip for the easiest Python path. It takes FEN + both Elos, has no history, and returns per-move probabilities.
  - Use Maia-3 if you want state of the art (57% vs 52%). It needs the previous 7 positions, so keep the move list, not just the FEN.
  - Use original Maia only through lc0 with `nodes=1` and the 112-plane format.
- Every one of these encoders is "12 piece planes (STM-relative) + small metadata". If you write your own model, the standard minimal tensor is 8×8×18 (Maia-2 style) or 8×8×112 (Lc0 style, with history). History matters more for human-move prediction than for engine strength (the Maia paper found it significant).
- Perspective flipping is universal. Store features from the mover's perspective (STM-relative) so a user's mistakes with White and with Black are comparable.

### Gaps
- The Maia paper says "previous 12 ply (6 moves for each player)", but the Lc0 112-plane format holds 8 positions (current + 7). The Maia-2 paper says Maia used "six" positions. The exact history length used by released Maia-1 weights isn't reconciled in the sources I read.
- I didn't confirm the exact Maia-2 Elo bin edges beyond the paper's "<1100 … ≥2000, 100-point bins" description.

---

## 4. Stockfish NNUE features: HalfKP, HalfKA(v2), HalfKAv2_hm, threat and pawn-pair inputs, and the accumulator

### Takeaway
NNUE turns a position into a very sparse binary vector. Each active feature is a (king-square bucket, piece, square) triple, so at most 32 of tens of thousands of inputs are 1. The first layer (the "accumulator") is then updated incrementally by adding or subtracting weight rows for the few features a move changes. As of Stockfish master (Sept 30 2026, SFNNv17), the input has three parts:
- **HalfKAv2_hm:** 22,528 features.
- **Full_Threats:** 59,808 features (piece-attacks-piece pairs).
- **PP_3Wide:** 4,560 features (pawn pairs).

These feed a 1024-wide accumulator, then 32 → 32 → 1 through one of 8 piece-count "layer stacks". This is an engine-internal representation. It is great for evaluation speed but not designed as a semantic embedding.

### Cited Findings
- **HalfKP:** feature = (our_king_square, piece_square, piece_type, piece_color) for non-king pieces. `p_idx = piece_type*2 + piece_color`; `halfkp_idx = piece_square + (p_idx + king_square*10)*64`. Size 64·64·5·2 = 40,960, plus "64 additional features that are unused" (so 41,024 in legacy files). Example net: `HalfKP[40960]->256x2->32->32->1`. The first layer has about 10M parameters — [nnue-pytorch docs/nnue.md](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md)
- **Sparsity:** "Current best architectures have input sparsity in the order of 0.1%". With king-inclusive (A) sets there are "only 32 non-zero inputs" at most. The accumulator adds the weight column for each feature that turns on and subtracts it for each that turns off. A king move triggers a full refresh — [nnue-pytorch docs/nnue.md](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md)
- **Quantization:** int16 accumulators (feature-transformer weights scaled by 127). Linear layers use int8 weights with int32 accumulation and a weight scale of 64 (max float weight 127/64 = 1.984375). ClippedReLU output is 0..127 — [nnue-pytorch docs/nnue.md](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md)
- **HalfKAv2 and HalfKAv2_hm:**
  - HalfKA includes kings.
  - HalfKAv2 merges "our king" and "their king" into one piece bucket, because with the king square fixed they are disjoint. That gives 11 piece buckets instead of 12.
  - HalfKAv2_hm ("horizontally mirrored") mirrors each perspective so its king is on files e–h, halving the king squares used. Stockfish has used it "since early August 2021".
  - LayerStacks: 8 buckets selected by `(piece_count - 1) / 4`.

  — [nnue-pytorch docs/nnue.md](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md)
- **Stockfish source (master):**
  - `HalfKAv2_hm::Dimensions = SQUARE_NB * PS_NB / 2` with `PS_NB = 11 * SQUARE_NB`, giving 64·704/2 = **22,528**.
  - 32 king buckets via a mirrored `KingBuckets[64]` table, and `MaxActiveDimensions = 32`.
  - `FullThreats::Dimensions = 59808`. Its comment notes that pawn diagonal threats only target knights and rooks because pawn-pawn relations are handled by PP_3Wide.
  - `PP_3Wide`: `PawnIds = COLOR_NB*48` = 96, `Dimensions = 96*95/2` = **4,560**, with indices concatenated after threats (`IndexBase = 59808`).

  — [half_ka_v2_hm.h](https://github.com/official-stockfish/Stockfish/blob/master/src/nnue/features/half_ka_v2_hm.h), [full_threats.h](https://github.com/official-stockfish/Stockfish/blob/master/src/nnue/features/full_threats.h), [pp_3wide.h](https://github.com/official-stockfish/Stockfish/blob/master/src/nnue/features/pp_3wide.h)
- **Stockfish network body (master):**
  - `L1 = 1024`, `L2 = 32`, `L3 = 32`, `LayerStacks = 8`.
  - fc_0: sparse-input affine 1024 → 32, followed by SqrClippedReLU ‖ ClippedReLU (64).
  - fc_1: 64 → 32, followed by SqrClippedReLU ‖ ClippedReLU.
  - fc_2: 128 → 1, plus a skip term from fc_0.
  - Output is scaled so 1.0 = 600·OutputScale.

  — [nnue_architecture.h](https://github.com/official-stockfish/Stockfish/blob/master/src/nnue/nnue_architecture.h)
- **Latest architecture change:** commit a17c0ce (2026-09-30) "SFNNv17: Remove psqt accumulator and update default net". The default net is `nn-252f33942263.nnue` — [Stockfish commits for nnue_architecture.h](https://github.com/official-stockfish/Stockfish/commits/master/src/nnue/nnue_architecture.h); [evaluate.h](https://github.com/official-stockfish/Stockfish/blob/master/src/evaluate.h)
- **Full_Threats in the docs:** models "AA" pairs where the first piece attacks the second, mirrored to files a–d. Redundant pairs are deduplicated: attacks where the attacker's set is a subset of the target's (pb, pq, pk, bq, rq, kq), and king-king. Weights are stored as int8 and accumulated as int16. PP_3Wide models unordered pawn pairs on the same or adjacent files — [nnue-pytorch docs/nnue.md](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md)
- **Stockfish 18 (2026-01-31):** SFNNv10, adding "Threat Inputs" to the input layer. Up to +46 Elo vs SF17 — [Stockfish 18 blog](https://stockfishchess.org/blog/2026/stockfish-18/)
- **Stockfish 19 (2026-09-05):** SFNNv16. It removes "redundant threat features" and adds "new pawn-pair features". The secondary (small) net introduced in 16.1 "has been retired". Training adds Quantization-Aware Training, and positions are rescored with "a strong Leela net". Up to +44 Elo vs SF18 — [Stockfish 19 blog](https://stockfishchess.org/blog/2026/stockfish-19/)

### Inferences
- **Outdated:**
  - HalfKP (2020) and `256x2-32-32` nets.
  - Big/small dual nets (SF 16.1–18; retired in SF19).
  - The PSQT accumulator (removed in SFNNv17).
- For a lesson app, NNUE is a **black-box evaluator**: run Stockfish to get cp/mate/PV. Don't reuse NNUE features as position embeddings. They are king-bucketed one-hots designed for incremental updates, not semantic similarity, and the accumulator isn't exposed by UCI.
- The HalfKAv2 idea is still useful conceptually for hand-crafted features: "piece-square relative to own king" is a cheap, interpretable king-safety and structure feature.
- The new Full_Threats set ("which piece attacks which") is effectively an engine-validated statement that attack relations matter. Reproducing a small "attacker-type × target-type" count matrix (6×6 per side) is a good interpretable feature for clustering a user's tactical blind spots.

### Gaps
- I could not verify from primary sources the SF16/17 big-net L1 width (commonly cited as 3072) or the SFNNv9 label. The chessprogramming snippet I saw gives "12×64×64 = 45056" for HalfKA, which is arithmetically inconsistent (12·64·64 = 49,152; 11·64·64 = 45,056 = HalfKAv2), so treat secondary NNUE size claims cautiously.
- I didn't read `nnue_feature_transformer.h`, so the pairwise-multiplication details of the L1 output are not recorded here.

---

## 5. Transformer tokenizations: DeepMind searchless chess, Lc0 BT4/smolgen, Chessformer, ChessGPT/LLM text

### Takeaway
There are two families:
1. **FEN-as-string tokenization**, e.g. DeepMind's fixed 77-character tokenization with 128-bin win% targets. It is simple and LLM-like but needs huge data.
2. **One token per square**, as in Lc0 BT4 and Chessformer/Maia-3. Each token carries the piece one-hot (plus history and metadata), with learned geometry-aware attention biases (smolgen, GAB) and a from-square × to-square attention policy.

Per-square tokens are the current state of the art for both strength (Lc0) and human-move prediction (Maia-3). They also make activations attributable to squares, which helps interpretability and lesson explanations.

### Cited Findings

**DeepMind, "Amortized Planning with Large-Scale Transformers: A Case Study on Chess"**

This is arXiv 2402.04494, first titled "Grandmaster-Level Chess Without Search".

- **State tokenization:** FEN flattened to a fixed **77 tokens**:
  - 64 board characters ('.' for empty).
  - 1 side-to-move token.
  - 4 castling tokens (padded with '.').
  - 2 en-passant tokens.
  - 3 halfmove-clock tokens.
  - 3 fullmove tokens.

  The code asserts `SEQUENCE_LENGTH = 77`. The paper's prose summary I extracted mis-sums to 76; the code resolves it as 3+3 clock digits — [searchless_chess tokenizer.py](https://github.com/google-deepmind/searchless_chess/blob/main/src/tokenizer.py); [arXiv 2402.04494](https://arxiv.org/html/2402.04494)
- **Actions:** the index of a UCI move in a sorted list of **1,968** possible UCI actions.
- **Targets:**
  - Action-value: bin of Stockfish Q(s,a).
  - State-value: bin of V(s).
  - Behavioral cloning: best-move index.
- **Value bins:** K=128 uniform bins over win% with HL-Gauss label smoothing (σ = 0.75/K).
- **Win%:** 100/(1+exp(−0.00368208·cp)). Mate maps to 100%.

— [arXiv 2402.04494](https://arxiv.org/html/2402.04494)

- **ChessBench:**
  - 10M Lichess games (Feb 2023) annotated by Stockfish 16 at 50 ms per eval.
  - About 530M state values and 15.3B action values.
  - Test set from March 2023.
- **Models:** 9M / 136M / 270M parameters. The 270M model has 16 layers, 8 heads, and d=1024.
- **Results:** Lichess blitz Elo 2895 vs humans, 2299 vs bots, 95.4% puzzle accuracy.

— [arXiv 2402.04494](https://arxiv.org/html/2402.04494)

**Lc0 transformers (BT1–BT4)**

- "Our models use 64 tokens, one for each square". Token order flips with STM. Inputs are 8 one-hot 12-d piece vectors (current + 7 past plies), en passant/castling, rule-50, and repetition flags, linearly projected plus a learnable positional embedding.
- BT3/BT4 add an embedding layer that projects the flattened 12×64 board to C channels per square, giving about 15% effective size for 5% latency.

— [Lc0 blog "Transformer Progress" (2024-02)](https://lczero.org/blog/2024/02/transformer-progress/)

- **Smolgen:** compresses the position to a small vector (64×32 → flatten → 256). Per head, 256 values pass through a shared 256×4096 projection, producing 64×64 supplemental attention logits added before softmax. It acts "as if 50% larger" for about a 10% throughput cost.
- **Earlier step:** a static trainable 64×64×h bias per layer, which also gave about 50% effective size.

— [Lc0 blog](https://lczero.org/blog/2024/02/transformer-progress/)

- **BT sizes:**

  | Model | Layers | Embedding | FFN | Heads | Params | GFLOPS | Policy Elo vs T78 CNN (194.5M) |
  |---|---|---|---|---|---|---|---|
  | BT1 | 12 | 1024 | 1536 | 8 | 92M | 5.637 | +13 |
  | BT2 | 15 | 768 | 1024 | 12 | 82M | 3.964 | +123 (smolgen) |
  | BT3 | 15 | 768 | 1024 | 24 | 105.5M | 4.158 | +179 |
  | BT4 | 15 | 1024 | 1536 | 32 | 191.3M | 7.613 | +270 |

  T78 (the CNN baseline) runs at 12.45 GFLOPS. The intro says BT4 is "nearly 300 elo" stronger, which is slightly inconsistent with +270 — [Lc0 blog](https://lczero.org/blog/2024/02/transformer-progress/)
- The blog notes the architecture is superseded by and published as Chessformer (ICLR 2026). "Leela-CF" there is the same network as BT4 — [Lc0 blog](https://lczero.org/blog/2024/02/transformer-progress/); [Chessformer arXiv 2605.19091](https://arxiv.org/html/2605.19091)

**Chessformer (ICLR 2026)**

- **Geometric Attention Bias (GAB):** tokens are projected (d1=32) → flattened → d2 → per-head d3, then a shared projection to h×64×64 biases is added to the logits before softmax. This is smolgen's successor.
- **Leela-CF:** 191M parameters, 15 layers, hidden 1024. It beat Leela-CNN by about +105 to +112 Elo. Searchless policy is about 2374 Elo (93.5% puzzles). Configurations won TCEC Cup 11 (beating Stockfish in the final) and TCEC Swiss 6 and 7.

— [Chessformer arXiv 2605.19091](https://arxiv.org/html/2605.19091)

**Earlier Chessformer preprint**

Monroe & Chalmers, "Mastering Chess with a Transformer Model" (arXiv 2409.12272, 2024). It claims to outperform AlphaZero "with 8x less computation" and to match prior GM-level transformers "with 30x less computation". Position representation is central to the claim — [arXiv 2409.12272](https://arxiv.org/abs/2409.12272)

**ChessGPT (Feng et al., NeurIPS 2023 Datasets & Benchmarks)**

A GPT trained on mixed game-record + natural-language chess data. It also released ChessCLIP (aligning move sequences with text) and an evaluation framework — [arXiv 2306.09200](https://arxiv.org/abs/2306.09200)

### Inferences
- If the app ever trains its own model, per-square tokens (64 × [12-d piece one-hot × history + metadata]) with a from×to attention policy is the 2026 best practice. It also gives a 64×64 move-logit matrix that is easy to visualize for lessons ("the model looked at d5→f7").
- FEN-string tokenization (DeepMind) is attractive for LLM integration but is data-hungry (15B labels). It is better treated as a benchmark than a template for a small app.
- For lesson text generation, LLMs should receive FEN + UCI/SAN + engine facts (eval, PV, themes) rather than being asked to read the board themselves.

### Gaps
- I didn't retrieve ChessGPT dataset sizes or ChessCLIP embedding details.
- The Lc0 blog didn't describe the attention-policy-to-1858 mapping (see `attention_policy_map.h` in the lc0 repo).

---

## 6. Learned embeddings and similarity search over positions (plus exact/structural search)

### Takeaway
There's no single standard "position2vec". Practical options, in increasing semantic depth:
1. **Exact or structural keys.** EPD/Zobrist, or the 768-bit piece-square bitvector with Hamming/Jaccard distance, which pgvector indexes natively up to 64,000 bits. Add pattern queries (CQL).
2. **Self-supervised position embeddings**, e.g. ChessLM: 256-d, masked-piece + move-distance objectives. These capture structure but explicitly NOT tactics.
3. **Contrastive embeddings** with positives defined by eval similarity.
4. **Pooled activations of a strong or human-like network** (Maia/Lc0 penultimate layer).
5. **Behavior-level embeddings of games and players** (behavioral stylometry; Maia4All individual embeddings).

For "find positions like the ones this user got wrong", combine a structural filter with a tactic-theme or feature filter. Embedding nearness alone does not imply the same tactical motif.

### Cited Findings
- **ChessLM** (Hugging Face): ViT-style encoder, 6 layers × 8 heads, about 4.5M/4.76M parameters, **256-d** position embedding.
  - Self-supervised objectives: masked piece prediction (10% masked) and "moves difference prediction" (plies between two positions of a game).
  - Intended for "Position Similarity Search".
  - Warns that embeddings are "not highly sensitive" to tactical nuances and "positions deemed similar by the embeddings can have vastly different engine evaluations".

  — [odestorm1/chesslm](https://huggingface.co/odestorm1/chesslm)
- **chesspos** (GitHub): "Embedding based chess position search". It uses triplet / metric-learning embeddings of positions and a Milvus vector DB, and the README says the repo is being restructured — [patrickfrank1/chesspos](https://github.com/patrickfrank1/chesspos)
- **"Retrieval of similar chess positions"** (SIGIR 2014): an IR approach. Each position is encoded as text terms for piece positions, reachability, and connectivity (attack and defence relations), with BM25 candidate retrieval followed by re-ranking — [SIGIR 2014 anthology entry](https://ir.webis.de/anthology/2014.sigirconf_conference-2014.72/)
- **Contrastive eval-similarity embeddings** (2025): about 5M ChessBench positions. Positives are positions whose Stockfish win probabilities differ by less than a margin δ, and a planning variant does beam search in embedding space (reported ~2593 Elo). Detail is from search-result summaries, not a full read — [arXiv 2506.04892](https://arxiv.org/html/2506.04892v1); [arXiv 2511.09477](https://arxiv.org/pdf/2511.09477)
- **Behavioral stylometry** (McIlroy-Young, Wang, Sen, Kleinberg, Anderson, NeurIPS 2021):
  - Each move is a **34-channel 8×8** tensor: the board before and after the player's move (24 piece channels) plus 10 metadata channels (repetition, castling, side, 50-move count, border).
  - A Maia-sized SE-ResNet (6 blocks, 64 channels, SE ratio 8) with global average pooling gives a **320-d move vector**.
  - The move vectors are projected to 1024-d and fed to a ViT-style encoder (12 blocks, 8 heads × 64, FFN 2048, up to 500 moves) to give a **512-d game embedding**.
  - The **player vector is the centroid** of game vectors, trained with the GE2E loss.
  - It identifies a player "from among thousands of candidate players with 98% accuracy given only 100 labeled games". In one setting it reaches Precision@1 = 0.86 among 2,844 Lichess players, using 100 query games and ignoring the first 15 moves.

  — [NeurIPS 2021 paper PDF](https://www.cs.toronto.edu/~ashton/pubs/chessembed-neurips2021.pdf)
- **Maia4All** ("Learning to Imitate with Less", arXiv 2507.21488; TMLR 2026): learns an individual player embedding from about 20 games (vs ~5,000 previously). Move-matching rises from 51.4% to 53.2% for a 20-game player. Initialization uses population prototypes — [arXiv 2507.21488](https://arxiv.org/abs/2507.21488)
- **pgvector types:**

  | Type | Index limit | Storage limit |
  |---|---|---|
  | `vector` | 2,000 dims | 16,000 dims |
  | `halfvec` | 4,000 dims | 16,000 dims |
  | `bit` | 64,000 dims | — |
  | `sparsevec` | 1,000 non-zeros | 16,000 non-zeros |

  — [pgvector README](https://github.com/pgvector/pgvector)
- **pgvector operators:** `<->` L2, `<#>` negative inner product, `<=>` cosine, `<+>` L1, `<~>` Hamming (bit), `<%>` Jaccard (bit). HNSW supports `bit_hamming_ops` and `bit_jaccard_ops`, while IVFFlat supports Hamming only — [pgvector README](https://github.com/pgvector/pgvector)
- **pgvector index tuning:**
  - HNSW defaults are m=16, ef_construction=64, ef_search=40.
  - IVFFlat `lists` should be about rows/1000 (up to 1M rows) or √rows above that, with `probes` about √lists.
  - For a binary-quantize-then-rerank pattern, use `binary_quantize(embedding)::bit(n)` with `<~>`, then re-rank by `<=>`.

  — [pgvector README](https://github.com/pgvector/pgvector)
- **CQL (Chess Query Language) 6.2:** "searches for user-defined patterns in chess games and studies" in PGN files and outputs the matching games. Patterns can be move sequences: e.g., the Greek-gift `⊢ ♗×♟h7 check ♚×h7 ♘――g5 check`, with color-reversal and flank-mirroring modifiers and regex-like repetition. Version 6.2 adds a `similarposition` filter — [CQL home](https://www.gadycosteff.com/cql/)
- **Lichess puzzle theme vocabulary** (a ready-made tactical label set):
  - **Tactics:** fork, pin, skewer, discoveredAttack, doubleCheck, deflection, attraction, clearance, interference, intermezzo, xRayAttack, hangingPiece, trappedPiece, capturingDefender, sacrifice, quietMove, defensiveMove, zugzwang, exposedKing, kingsideAttack, queensideAttack, advancedPawn, promotion, underPromotion, enPassant, castling.
  - **Mates:** backRankMate, smotheredMate, anastasiaMate, arabianMate, bodenMate, hookMate, …, and mateIn1–mateIn5.
  - **Phases:** opening, middlegame, endgame.
  - **Endgame types:** rook, pawn, queen, knight, bishop, queenRook.
  - **Lengths:** oneMove, short, long, veryLong.
  - **Outcome labels:** advantage, crushing, equality, mate.

  — [lila PuzzleTheme.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleTheme.scala)

### Inferences
- **Recommended layered retrieval for "positions similar to ones the user got wrong":**
  1. **Hard filters** on interpretable columns (phase, material signature, tactic theme label, pawn-structure hash).
  2. **ANN** over an embedding column, with pgvector HNSW on a 256–1024-d `vector`/`halfvec`. The embedding can be pooled Maia-2/Maia-3 or Lc0 activations (mean-pool the 64 square tokens or the ResNet's global-average-pooled channels), or a ChessLM-style model.
  3. **Optional Hamming/Jaccard** on a 768-bit (12×64) piece-placement `bit` vector for "same structure" matches. 768 ≪ the 64,000-bit index limit, so this can be indexed directly.
- Using a network's penultimate-layer activations as an embedding is a reasonable engineering choice. Stylometry did exactly this with global-average-pooled Maia-like ResNet features (320-d). But no paper I found benchmarks Maia or Lc0 activations specifically for position-similarity retrieval. Validate with your own "same theme" recall metric.
- Since structural embeddings can map tactically different positions together (the ChessLM caveat), similarity for lessons should be defined by the **mistake pattern**, not just the board. A practical key is: theme label + the attacker/target relation of the missed tactic + the user's move vs the best move. CQL-style pattern queries or python-chess attack and pin predicates can implement this exactly.
- Player-level vectors (stylometry centroids, Maia4All embeddings) are the natural way to represent "this user" for personalization. Individual embeddings are now feasible from about 20 games.

### Gaps
- I found no canonical paper titled "position2vec" for chess; the term seems informal.
- I found no published benchmark comparing Maia/Lc0 activation embeddings against ChessLM or bitboard Hamming distance for similar-position retrieval.
- I didn't read the full text of the CQL `similarposition` filter docs or the 2014 SIGIR paper (authors/metrics not recorded).

---

## 7. Interpretable hand-crafted features and concept-probing research

### Takeaway
Classical evaluation terms (material, piece-square tables, mobility, king safety, threats, passed pawns, space, game phase) remain the best interpretable features for statistics and clustering of a user's mistakes. They are also exactly the human concepts that research finds linearly decodable inside AlphaZero/Lc0. Concept probing (McGrath 2022), novel-concept discovery (Schut 2023), look-ahead evidence (Jenner 2024), and sparse autoencoders/transcoders on Lc0/Maia-3 (2024–2026) show that network activations contain tactic-level features (mate threats, forks, pins). That supports either computing features directly or reading them off activations.

### Cited Findings
- **PeSTO tapered evaluation:**
  - **Material:**

    | Piece | MG | EG |
    |---|---|---|
    | P | 82 | 94 |
    | N | 337 | 281 |
    | B | 365 | 297 |
    | R | 477 | 512 |
    | Q | 1025 | 936 |
    | K | 0 | 0 |

  - **Phase increments:** P=0, N=1, B=1, R=2, Q=4, giving 24 at the start (capped at 24).
  - **Score:** `(mgScore*mgPhase + egScore*(24-mgPhase))/24`.
  - **Piece-square tables:** 6 piece types × 64 squares × {MG, EG} = 12 tables from White's view. Black uses `sq ^ 56`. Values were tuned by Texel tuning.

  — [chessprogramming.org PeSTO](https://www.chessprogramming.org/PeSTO%27s_Evaluation_Function)
- **McGrath et al., "Acquisition of Chess Knowledge in AlphaZero"** (PNAS 2022, doi 10.1073/pnas.2206625119; arXiv 2111.09259):
  - **Concept sets:** Stockfish 8 eval sub-functions (material, imbalance, pawns, knights, bishops, rooks, queens, mobility, king_safety, threats, passed_pawns, space, total; by side and phase), plus **116 custom concepts** (forks, pins, contested files, pawn-structure features, `has_mate_threat`, `can_capture_queen`, …).
  - **Probes:** sparse linear (continuous concepts) or logistic (binary concepts) regression from each block's 8×8×256 = 16,384 activations, across 20 blocks and many training checkpoints.
  - **Probe data:** 10^5 games from ChessBase, deduplicated by FEN.

  — [arXiv 2111.09259](https://arxiv.org/abs/2111.09259)
- **McGrath findings:**
  - Stockfish 8 total score is decodable with r² > 0.75 after 10 blocks at ≥64k training steps.
  - `has_mate_threat` accuracy rises through the depth of the network, which suggests it models the opponent's replies.
  - Material is learned early. King safety and mobility emerge later.
  - Simple piece-count material is trivially decodable at all layers.

  — [arXiv 2111.09259](https://arxiv.org/abs/2111.09259)
- **Schut et al., "Bridging the Human-AI Knowledge Gap"** (arXiv 2310.16410, 2023):
  - Concepts are sparse vectors in latent space, found by convex optimization so that AZ's chosen MCTS line scores higher than subpar lines (dynamic concepts).
  - Concepts are filtered by teachability (a student network learns from prototypes; 97.6% filtered out) and novelty (better reconstructed by an AZ-games basis than a human-games basis; a further 27.1% removed).
  - Four GMs improved on concept-prototype puzzles: +42, +25, +16, +6 percentage points.

  — [arXiv 2310.16410](https://arxiv.org/abs/2310.16410)
- **Jenner et al.,** "Evidence of Learned Look-Ahead in a Chess-Playing Neural Network" (arXiv 2406.00877). In the Lc0 policy network (square tokens), activations on squares of future best moves are causally important, and attention heads move information between future-move squares. A simple probe predicts the best move "2 turns ahead with 92% accuracy" in single-best-line positions — [arXiv 2406.00877](https://arxiv.org/abs/2406.00877)
- **Contrastive Sparse Autoencoders on Lc0** (arXiv 2406.04028) find concepts without labels, contrasting with labeled linear probing — [arXiv 2406.04028](https://arxiv.org/html/2406.04028v1)
- **Lin et al. (2026),** "Tracing the Thought of a Grandmaster-level Chess-Playing Transformer" (arXiv 2604.10158). Sparse replacement layers decompose both the MLP and attention modules of Lc0 into interpretable tactical pathways, and the paper reports "parallel reasoning" consistent with the policy head's inductive bias. Code is at github.com/JacklE0niden/Leela-SAEs — [arXiv 2604.10158](https://arxiv.org/abs/2604.10158)
- **Chessformer/Maia-3 transcoder:** a cross-layer transcoder on Maia-3 layers 3–4 (8,192 features) finds features matching tactical motifs such as forks and pins, attributable to squares. The authors call this preliminary — [arXiv 2605.19091](https://arxiv.org/html/2605.19091)

### Inferences
- **Suggested interpretable feature vector per mistake position (b).** All of these are computable with python-chess in milliseconds.
  - **Material:** PeSTO material diff, imbalance flags (bishop pair, minor-vs-rook).
  - **Phase:** PeSTO phase 0–24.
  - **Mobility:** legal-move counts per piece type.
  - **King safety:** attackers on the king zone, open files near the king, pawn-shield count.
  - **Pawn structure:** counts of isolated, doubled, passed, and backward pawns, and pawn islands.
    - Isolated means no friendly pawn on adjacent files.
    - Doubled means two or more friendly pawns on one file.
    - Passed means no enemy pawn ahead on the same or adjacent files.
    - These are standard definitions, not taken from a fetched source.
  - **Hanging pieces:** attacked and under-defended pieces.
  - **Pins and attack relations:** pinned-piece count, and an attacker-type × target-type matrix (mirrors Stockfish's Full_Threats idea).
  - **Checks available, mate threats:** e.g. McGrath's `has_mate_threat`.
  - **Puzzle-theme labels** of the missed tactic.
- These same features are what probing research shows strong nets internalize. So a clustering built on them should align with what Maia/Lc0 "see", and it keeps lesson explanations human-readable.
- SAE/transcoder features on Maia-3 (square-attributable fork/pin features) are a promising 2026 route to automatic motif labeling, but they are research-grade, not production-ready.

### Gaps
- I didn't fetch Stockfish 8's classical eval source or chessprogramming pages for exact king-safety and mobility formulas. The pawn-structure definitions above are standard but not cited from a fetched page.
- I couldn't confirm a PNAS venue or year for Schut et al. from the arXiv page; it's listed only as a 2023 arXiv preprint here.

---

## 8. Representing an entire game or a mistake event; published schemas and datasets

### Takeaway
A game should be stored as an ordered sequence of ply records: (position before, move played, engine best move/PV, eval before/after, win% before/after, accuracy, judgement, clock). Three published schemas cover most of this:
- **Lc0's V6 training record**, which already stores `played_idx`, `best_idx`, played/best Q, visits, and policy KLD.
- **The Lichess eval JSONL** (fen → multi-PV cp/mate/line/depth).
- **The Lichess puzzle CSV**: PuzzleId, FEN, Moves, Rating, RatingDeviation, Popularity, NbPlays, Themes, GameUrl, OpeningTags, DailyDate.

Lichess also defines move judgements (inaccuracy, mistake, blunder) as thresholds on win-chance drop.

### Cited Findings
- **Lichess puzzle CSV header:** `PuzzleId,FEN,Moves,Rating,RatingDeviation,Popularity,NbPlays,Themes,GameUrl,OpeningTags,DailyDate`.
  - Moves are UCI.
  - The FEN is the position before the opponent's move (the first UCI move). "The position to present to the player is after applying the first move to that FEN", and the second move starts the solution.
  - Sample: `00sHx,q3k1nr/1pp1nQpp/3p4/1P2p3/4P3/B1PP1b2/B5PP/5K2 b k - 0 17,e8d7 a2e6 d7d8 f7f8,1760,80,83,72,mate mateIn2 middlegame short,https://lichess.org/yyznGmXs/black#34,Italian_Game Italian_Game_Classical_Variation,`
  - 6,157,341 puzzles.

  — [database.lichess.org](https://database.lichess.org/)
- **Lichess eval JSONL:** `{fen, evals:[{knodes, depth, pvs:[{cp|mate, line}]}]}`, 416,442,401 positions — [database.lichess.org](https://database.lichess.org/)
- **Lichess move judgement** (`CpAdvice`): `winningChanceJudgements = List(.3 -> Blunder, .2 -> Mistake, .1 -> Inaccuracy)`. The delta is the drop in `WinPercent.winningChances(cp)` (range −1..+1) from the mover's perspective, and the first threshold ≤ delta wins.
  - Mate transitions are judged separately (`MateCreated` "Checkmate is now unavoidable", `MateDelayed`, `MateLost` "Lost forced checkmate sequence").
  - Comments read e.g. "(+0.3 → −1.2) Mistake. Nf3 was best."

  — [lila Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala)
- **Lc0 V6 training record fields:** probabilities[1858], planes[104], castling bytes, `side_to_move_or_enpassant`, `rule50_count`, `invariance_info`. Eval fields are root_q/best_q/root_d/best_d/root_m/best_m, plies_left, result_q/result_d, played_q/played_d/played_m, and orig_q/orig_d/orig_m. Search fields are visits, played_idx, best_idx, policy_kld, and q_st — [lc0 trainingdata_v6.h](https://github.com/LeelaChessZero/lc0/blob/master/src/trainingdata/trainingdata_v6.h)
- **Human-move dataset filtering conventions:** Maia excludes moves made with under 30 s on the clock ([Maia paper](https://arxiv.org/abs/2006.01855)). Maia-2 drops the first 10 plies and positions where either player has under 30 s ([Maia-2](https://arxiv.org/html/2409.20553)). Stylometry ignores the first 15 moves to remove memorized openings ([stylometry PDF](https://www.cs.toronto.edu/~ashton/pubs/chessembed-neurips2021.pdf)).
- **ChessBench** (DeepMind): (state, action) → Stockfish 16 win% for 15.3B action-values. It is a published "every legal move scored" dataset format — [arXiv 2402.04494](https://arxiv.org/html/2402.04494)

### Inferences
- **Proposed "mistake event" record.** This is a synthesis of the above schemas, not a published standard.

  | Group | Fields |
  |---|---|
  | Identity | `game_id`, `ply`, `color`, `user_elo`, `opp_elo`, `time_control` |
  | Position | `fen_before` (+ `epd4` key), `move_history_uci[-7:]` (for Maia-3/Lc0 history) |
  | Moves | `move_played_uci/san`, `best_move_uci`, `pv_uci[]`, `multipv_top3 [{uci, cp/mate}]` |
  | Evals | `cp_before`, `cp_after` (mover-relative, with mate encoded separately), `win_before`, `win_after` (Lichess formula), `win_drop`, `accuracy` (Lichess formula) |
  | Labels | `judgement` (Lichess 0.1/0.2/0.3 winning-chance thresholds), `themes[]` (run the tactic tagger on the best line from `fen_before`, or match the Lichess puzzle themes) |
  | Human model | `maia_p(played)`, `maia_p(best)` at the user's Elo (measures "how human-natural was the error / how findable was the best move") |
  | Time | `clock_before`, `time_spent` |
  | Features | `features{}` (§7) |
  | Embeddings | `embedding vector(256–1024)`, `bitvector bit(768)` |

- A whole game is then the list of these per-ply records plus the PGN headers. A user profile is aggregates over mistake events (counts by theme × phase × time-pressure), optionally plus a stylometry/Maia4All-style player embedding.
- To turn a user's mistake into a training item, reuse Lichess's puzzle convention: FEN before the opponent's last move, then the solution moves in UCI.

### Gaps
- Lichess's puzzle-generation pipeline (lichess-puzzler: how themes are auto-tagged, and the criteria for a "unique best move") was not fetched in this pass.
- No published schema for "personalized mistake events" was found. The record above is my synthesis.

---

## 9. Win-probability transform of centipawn evals and Lichess accuracy formulas

### Takeaway
- **Win%:** Lichess (via scalachess) uses Win% = 50 + 50·(2/(1+e^(−0.00368208·cp)) − 1), with cp clamped to ±1000 and mate mapped to ±1000. This is algebraically identical to 100/(1+e^(−0.00368208·cp)), the formula DeepMind also used.
- **Move accuracy:** 103.1668·e^(−0.04354·ΔWin%) − 3.1669, plus a +1 "uncertainty bonus", clamped to [0,100]. It is 100 if Win% didn't drop.
- **Game accuracy:** the mean of a volatility-weighted mean and a harmonic mean of move accuracies.
- **Lichess's "Inaccuracy/Mistake/Blunder" thresholds:** winning-chance drops of 0.1/0.2/0.3 on the −1..1 scale, i.e. 5/10/15 Win% points.

### Cited Findings
- **Lichess accuracy page:**
  - "Win% = 50 + 50 * (2 / (1 + exp(-0.00368208 * centipawns)) - 1)"
  - "Accuracy% = 103.1668 * exp(-0.04354 * (winPercentBefore - winPercentAfter)) - 3.1669"
  - Game accuracy uses sliding windows sized by game length. Each window's volatility is the std-dev of its Win%, giving a volatility-weighted mean; the final value averages that with the harmonic mean.

  — [lichess.org/page/accuracy](https://lichess.org/page/accuracy)
- **scalachess source:**
  - `winningChances(cp) = 2/(1+exp(-0.00368208*cp)) - 1`, clamped to [−1, 1]. The code comment links lila PR #11148.
  - `fromCentiPawns(cp) = 50 + 50*winningChances(cp.ceiled)` with `Cp.CEILING = 1000`.
  - Mate → ±CEILING.
  - The initial position score is `cp(15)`.

  — [scalachess eval.scala](https://github.com/lichess-org/scalachess/blob/master/core/src/main/scala/eval.scala); [lila PR 11148](https://github.com/lichess-org/lila/pull/11148)
- **lila `AccuracyPercent.fromWinPercents`:** if `after >= before` the result is 100. Otherwise it is `103.1668100711649*exp(-0.04354415386753951*winDiff) - 3.166924740191411`, then `+1` ("uncertainty bonus (due to imperfect analysis)"), clamped to [0, 100].
  - The curve was fit (scipy `curve_fit`) to anchor points (0→100, 5→75, 10→60, 20→42, 40→20, 60→5, 80→0).

  — [lila AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala)
- **lila `gameAccuracy`:**
  - `windowSize = (cps.size / 10)` clamped to [2, 8].
  - Window weights are `standardDeviation(winPercents)` clamped to [0.5, 12].
  - Per color, the result is `(weightedMean + harmonicMean)/2`.
  - A `phaseAccuracies` variant computes opening, middlegame, and endgame accuracy separately.

  — [lila AccuracyPercent.scala](https://github.com/lichess-org/lila/blob/master/modules/analyse/src/main/AccuracyPercent.scala)
- **Judgement thresholds** on `winningChances` drop: ≥0.3 Blunder, ≥0.2 Mistake, ≥0.1 Inaccuracy — [lila Advice.scala](https://github.com/lichess-org/lila/blob/master/modules/tree/src/main/Advice.scala)
- **DeepMind's conversion:** win% = 100/(1+exp(−0.00368208·cp)), with mate → 100% — [arXiv 2402.04494](https://arxiv.org/html/2402.04494)
- **Lc0's native value** is an expected score Q ∈ [−1, 1] (plus a WDL head), not centipawns — [Lc0 technical explanation](https://lczero.org/dev/wiki/technical-explanation-of-leela-chess-zero/); [lczero.org NN topology](https://lczero.org/dev/backend/nn/)

### Inferences
- **The two Win% formulas are the same function.** 50 + 50(2σ(kx) − 1) = 100σ(kx), with k = 0.00368208. One cp unit ≈ 0.092 Win% near equality (derivative 100·k/4), so a 100-cp drop from equal ≈ 9.2 Win% points.
- **On the 0–100 Win% scale,** Lichess thresholds are 5 (inaccuracy), 10 (mistake), and 15 (blunder) percentage points, since winningChances = (Win% − 50)/50.
- **Recommended mistake-severity variable:** use `win_drop` (Win% before − after, mover-relative, cp clamped ±1000) as the primary severity score. It is bounded, saturates in already-decided positions (so you don't over-teach blunders made while +9), and is directly comparable to Lichess's UI.
- **Cross-cutting synthesis** for the developer's three needs:
  - **(a) NN input.** Don't invent a representation. Feed FEN (+ last 7 moves) to Maia-2 (pip, FEN-only) or Maia-3 (64 square tokens, history, soft Elo). Use Stockfish for ground-truth best move and eval.
  - **(b) Stats and clustering.** Use the per-ply mistake record (§8) with interpretable features (§7), Lichess Win%/accuracy/judgement (§9), puzzle-theme labels, phase, and clock.
  - **(c) Similarity search.** Use pgvector with hard filters on theme/phase/material plus HNSW over a learned embedding (pooled Maia/Lc0 activations or ChessLM-style). Add an optional 768-bit piece-placement `bit` vector with Hamming/Jaccard for structural matches, and CQL or python-chess predicates for exact motif matching.

### Gaps
- I didn't fetch lila PR #11148 itself, so the dataset and regression behind k = 0.00368208 is not documented here. A forum post cites an older 0.004 coefficient.
- Lc0's internal Q→cp conversion formula (used for UCI `score cp`) was not retrieved.
