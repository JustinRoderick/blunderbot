# Neural Models That Imitate or Personalize to Individual Human Chess Players

Scope: research literature and open-source ecosystem for modeling human chess decisions, especially adapting a model to ONE specific player from their past games. Current as of 2026-10-09. Primary sources: arXiv papers (full text read via PDF extraction where noted), official CSSLab GitHub repos, maiachess.com.

Source key (used below):
- Maia (KDD 2020): https://arxiv.org/abs/2006.01855 (full PDF read: https://arxiv.org/pdf/2006.01855)
- Maia-Individual (KDD 2022, arXiv v3): https://arxiv.org/abs/2008.10086 (full PDF read: https://arxiv.org/pdf/2008.10086; DOI https://doi.org/10.1145/3534678.3539367)
- Maia-Individual 2021 preprint (arXiv v2): https://www.cs.toronto.edu/~ashton/pubs/maia-personalized2021.pdf
- Behavioral stylometry (NeurIPS 2021): https://arxiv.org/abs/2208.01366 (full PDF read: https://arxiv.org/pdf/2208.01366)
- Maia-2 (NeurIPS 2024): https://arxiv.org/html/2409.20553
- Allie (ICLR 2025): https://arxiv.org/html/2410.03893
- Maia4All (2025): https://arxiv.org/html/2507.21488v1
- Chessformer / Maia-3 (2026): https://arxiv.org/abs/2605.19091
- Repos: https://github.com/CSSLab/maia-chess, https://github.com/CSSLab/maia-individual, https://github.com/CSSLab/maia2, https://github.com/CSSLab/maia3

---

## 1. Maia (KDD 2020): architecture, data, rating bins, move-matching accuracy, blunder prediction

### Takeaway
Maia is a Leela/AlphaZero-style residual CNN trained by supervised learning on ~12M human Lichess games per 100-point rating bin (9 models, 1100–1900), used with NO tree search; it matches human moves ~51–53% of the time vs. 33–41% for depth-limited Stockfish and at most 46% for Leela. A separate residual CNN predicts "will the human blunder on the next move" at up to 71.7% accuracy on a balanced test set.

### Cited Findings
**Paper metadata**
- "Aligning Superhuman AI with Human Behavior: Chess as a Model System", Reid McIlroy-Young, Siddhartha Sen, Jon Kleinberg, Ashton Anderson; KDD '20 (Aug 23–27, 2020, virtual); arXiv v1 2 Jun 2020, v3 14 Jul 2020 — [arXiv 2006.01855](https://arxiv.org/abs/2006.01855)

**Architecture**
- Repurposes Leela's network architecture "but instead of learning from self-play games, we learn from real human games" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Move-prediction model: "6 blocks of 2 CNNs each with 64 channels" (residual blocks, 3×3 kernels), with a policy head and a value head; policy trained with cross-entropy, value head with MSE — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Authors chose 6 blocks × 64 filters "partially due to computational costs"; "a larger network, such as 24 blocks and 320 filters, yielded a small performance boost at a significant cost in compute" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Input includes game history: "we give Maia the previous 12 ply (6 moves for each player)"; including history gives ~3 percentage points boost — [Maia paper](https://arxiv.org/pdf/2006.01855)
- No tree search: "to make a move prediction, we do not conduct any tree search—the policy network is solely responsible for the prediction." Adding 10 rollouts of MCTS degraded accuracy; "performing no tree search give[s] large boosts in performance (... 5–10 percentage points)" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- The Maia-2 paper later reports ~10.3M parameters per Maia-1 model (92M total for the nine) — [Maia-2 paper](https://arxiv.org/html/2409.20553)

**Training data and rating bins**
- Data: Lichess (mean rating 1525; 5% below 1000, 10% above 2000) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- "9 training sets and 9 validation sets, one for each rating range between 1100 and 1900, with 12 million games and 120,000 games each" (100-point bins); games sampled from 2017–2019 in blocks of 200,000; test sets 200,000 games per bin — [Maia paper](https://arxiv.org/pdf/2006.01855)
- "12 million games is enough for Leela to go from random moving to a rating of 3000 (superhuman performance)" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Training config (Table 3): batch size 1024, 400,000 total steps, Adam, TensorFlow 2.0, LR burn-in 0.1 then 0.01 with drops by 10× — [Maia paper](https://arxiv.org/pdf/2006.01855)

**Move-matching accuracy (Table 1, highest accuracy per model and the test-bin where achieved)**
- Maia 1100: 50.8% (on 1200-rated test); Maia 1300: 51.8% (1400); Maia 1500: 52.2% (1700); Maia 1700: 52.7% (1800); Maia 1900: 52.9% (1900) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- "Maia's highest accuracy is over 52%"; lowest (Maia 1900 predicting 1100-rated players) is 46% "as high as the best performance achieved by any Stockfish or Leela model" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Stockfish depth 1–15 matches human moves 33–41% of the time; best depth-15 = 41.1% (on 1900 players); every Stockfish curve rises monotonically with human rating (depth 15 matches 1900s 5 pp more than 1100s) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Leela (8 versions of 5th-gen network): max 46.0% (Leela "3200" on 1900 players); Leela ordinal 2700 matches ~40% "no matter whether they are played by humans rated 1100, 1500, or 1900" — accuracy curves essentially flat across skill — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Over 1100–1900, "every Maia model outperforms every Leela and Stockfish model"; above 1900 Maia degrades but Maia 1900 still beats best Stockfish/Leela by 5 pp — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Maia is better than Leela "across the entire range" of move quality, "including large errors ('blunders')": "Maia still does the best job of predicting them" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Inter-Maia move agreement 65–79% — [Maia paper](https://arxiv.org/pdf/2006.01855)

**Blunder prediction (separate model)**
- Blunder = move that worsens the player's win probability by ≥10 percentage points (Stockfish eval mapped to empirical win prob) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Training data: Lichess games with Stockfish evals + clock info, Apr 2017–Sep 2019 (≈10% of games have evals); removed fast time controls and moves with <30s left; 182M blunders + 272M non-blunders for training; test 9M + 9M (balanced) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Test accuracy (Table 2): Random Forest 56.4% (board only) / 63.0% (board+metadata); fully connected 62.9% / 66.0%; residual CNN 67.7% / 71.7% — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Grouped task ("did >10% of the population blunder in this common position?"): ~1.07M training boards, 119k balanced test; FC 69.5%, residual CNN 75.3%, deep residual CNN (8 blocks, 256 filters) 76.9% — [Maia paper](https://arxiv.org/pdf/2006.01855)
- "Using the move prediction models for blunder prediction was attempted but the results were far exceeded by the specialized models" — [Maia paper](https://arxiv.org/pdf/2006.01855)

**Deployment**
- Lichess bots: maia1 (1100), maia5 (1500), maia9 (1900) — [Maia paper](https://arxiv.org/pdf/2006.01855); [maia-chess repo](https://github.com/CSSLab/maia-chess)

### Inferences
- A ~51–53% top-1 ceiling for population-level human move prediction means even the best model is wrong ~half the time on a specific move; for a lessons app it is better to use the full probability distribution (e.g., "this user's peers play the blunder X 30% of the time") than top-1 predictions.
- The blunder-prediction result (71.7% balanced accuracy) shows "difficulty of a position for a human" is learnable from board + metadata (rating, clock) — directly relevant to flagging positions likely to trip up a user.

### Gaps
- Exact GPU-hours to train Maia-1 and per-model file size were not reported in the text I read.
- Blunder model AUC numbers were not in the extracted text (only accuracy).

---

## 2. Maia-Individual: "Learning Models of Individual Behavior in Chess" (KDD 2022) / "Learning Personalized Models of Human Behavior in Chess" (2021 preprint)

### Takeaway
Fine-tuning the whole Maia 1900 network on one player's games (80% split, 30k steps, LR 1e-4, ~40 min on a V100) lifts move-matching accuracy by ~4.2 pp on average over the best population Maia, but ONLY for players with ≥5,000 games; with 1,000 games the personalized model is WORSE than base Maia (49.7% vs 52.7%). The 2021 preprint's "10 percentage points" claim was superseded by "4–5 percentage points" in the KDD 2022 version.

### Cited Findings
**Metadata and versions**
- arXiv 2008.10086: v1 23 Aug 2020, v2 13 Feb 2021 (titled "Learning Personalized Models of Human Behavior in Chess"), v3 15 Jun 2022 (titled "Learning Models of Individual Behavior in Chess", KDD '22, Aug 14–18, 2022); authors McIlroy-Young, Wang, Sen, Kleinberg, Anderson — [arXiv](https://arxiv.org/abs/2008.10086)
- SUPERSEDED CLAIM: the Feb 2021 preprint said personalization raised accuracy "by 10 percentage points on average above the highest-performing coarse model" — [2021 preprint](https://www.cs.toronto.edu/~ashton/pubs/maia-personalized2021.pdf); the KDD 2022 version says "on average by over 4 percentage points" / "4-5% higher accuracy", main effect "4.186%, with a confidence interval (standard error) of +/- 0.101%" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Data**
- Lichess, all rated games Jan 2013–Dec 2020; Blitz only (3–8 min); players with mean rating 1000–2000, low rating variance, account >1 year old, excluded titled players and bots — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Player counts by minimum games (Table 1, evaluation set): 1,000+: 30,821; 5,000+: 5,298; 10,000+: 1,866; 20,000+: 295; 30,000+: 86; 40,000+: 19 (40 random players taken from each starred bucket); exploration set 3,852 / 662 / 233 / 36 / 10 / 10 — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Final evaluation: 400 players; per-player game split 80% train / 10% val / 10% test + a "future" set (Dec 2020 games) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Fine-tuning method ("Transfer Maia")**
- Input: 112-channel 8×8 board representation; output: 1858-dim policy vector (Leela format) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Design decisions: initialize from Maia weights (random init "had a significant negative effect"); "freezing layers only hurts performance" — "the deeper the gradients flowed the better"; starting Maia choice "doesn't vary much", so always start from Maia 1900 — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Steps: most improvement in first 12,000 steps (3M board-move pairs); final = 30,000 steps with LR drops at 15k/20k/25k by 10×; optimal LR 0.0001 for 10k–40k-game regimes; SGD with momentum (Keras) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Accuracy measured after ply 10 to avoid memorized openings — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Accuracy vs. number of games (Table 2: Nearest Maia → Transfer Maia)**
- 1,000 games: 0.527 → 0.497 (WORSE)
- 5,000 games: 0.532 → 0.550
- 10,000 games: 0.528 → 0.556
- 20,000 games: 0.524 → 0.564
- 30,000 games: 0.520 → 0.567
- 40,000 games: 0.528 → 0.580
- — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- "Our architecture produces significant uplift over base Maia for players who have played at least 5,000 games. However, it is ineffective on players who have only played 1,000 games ... We speculate that the lower accuracy is due to over-fitting"; pooling data across players suggested as future work — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Personalized accuracy is largely independent of rating within 1100–1900 (R² = 0.089); "preliminary attempts to create personalized models for Grandmasters did not yield the same accuracy gains" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Perplexity on a player's moves: transfer model 1.95 vs Maia 1900 2.15 (opening positions: 0.92 vs 2.40) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Personalized models outperform base Maia "across the entire spectrum of move quality—everything from good moves to serious blunders"; gains persist in unseen positions (by ply 30 only 0.02% of positions were seen in training) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Stylometry with personalized models**
- Given one side of 100 games, correct player identified 98% of the time among 400 players; Table 3 top-1: all moves 10/30/100 games = 0.86/0.94/0.98; ply 10+ = 0.47/0.81/0.95; ply 30+ = 0.11/0.26/0.55; random = 0.25%; centipawn-loss Naive Bayes baseline peaks at 1.4% — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Blunders-only stylometry (moves losing >10% win prob, ~35% of moves): accuracy rises 0.983 → 0.989, "mistakes are the most discriminative moves ... a player's personalized model makes blunders like that individual player" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**"Towards Personalized Learning Tools" (Section 5.3)**
- Authors propose using personalized models to "surface the idiosyncratic mistakes that players make": moves where the personalized model's probability x exceeds base Maia's y for a blunder are situations "predictably difficult for this specific player" and "relatively fixable (since other players are less likely to err)" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Paper explicitly motivates: "a coach that understands how you could improve in particular would clearly be more helpful than one who only understands people at your level in general" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Compute**
- "trained ... on 6 core virtual machines with Tesla V100 GPUs. The training time for one model was 40 minutes when run alone ... (using ∼300MB of vRAM), our training procedure should be feasible on most GPUs" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

**Ethics noted by authors**
- Stylometry raises de-anonymization/privacy risk; personalized models could be used to evade cheat detection or to prepare against a specific opponent — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)

### Inferences
- The full fine-tune approach is a poor fit for a typical app user: most users won't have 5,000+ games on one platform/time control, and at 1,000 games it actively hurts. This is the key data-requirement finding for blunderbot.
- The "x > y" comparison (personalized vs. rating-matched population model on the user's actual blunders) is essentially the exact recipe for "mistakes this user makes that peers don't" lessons.

### Gaps
- No results reported between 1,000 and 5,000 games (e.g., 2,000–3,000), or for <1,000 games with this method.
- Per-player inference cost not reported (model is the same size as Maia-1, so similar).

---

## 3. Behavioral stylometry (NeurIPS 2021): transformer embeddings and few-game identification

### Takeaway
A residual-CNN + modified Vision Transformer trained with GE2E (speaker-verification) loss maps each game to a 512-dim embedding; averaging a player's game embeddings gives a "style vector" that identifies players among 2,844 candidates with 98% accuracy (k=0, 100 games) or 86% (ignoring first 15 moves), saturating around 50 games and "reasonably well" at 10 games. It is an identification model, not a move predictor, but it established that a compact per-player embedding learned from few games captures individual style — the idea later used for personalization in Maia4All.

### Cited Findings
- "Detecting Individual Decision-Making Style: Exploring Behavioral Stylometry in Chess", McIlroy-Young, Wang, Sen, Kleinberg, Anderson; NeurIPS 2021; arXiv 2208.01366 — [arXiv](https://arxiv.org/abs/2208.01366); [NeurIPS PDF](https://papers.nips.cc/paper_files/paper/2021/file/ccf8111910291ba472b385e9c5f59099-Paper.pdf)
- Architecture: 34-channel 8×8 move encoding (board before and after the move); residual blocks matching Maia (6 blocks, 64 channels, SE ratio 8) for move features → modified ViT over the move sequence → mean of encoder outputs (1024-dim) → MLP to a 512-dim game embedding; player vector = centroid of game vectors; GE2E loss; transformer beat LSTM; 512-dim beat 128-dim; ArcFace failed to converge — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Training: batches of N=40 players × M=20 games, random 32-move windows; 4 Tesla K80 GPUs; SGD LR 0.01, momentum 0.9 — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Training data: 63.7M games by 16,181 "seen" players (Table 1 buckets 1K–40K+ plus 1,813 high-ranked players) — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Main result (Table 2, 100 reference + 100 query games, 2,844 candidates with 10K+ games): P@1 = 0.860 on unseen players starting from move 15 (baseline 0.244, random 0.0004); with openings included (k=0) P@1 = 0.979 unseen / 0.982 all — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Generalizes to unseen players (0.860 unseen vs 0.853 seen) — i.e., new players can be embedded without retraining — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Number of games: "performance rises sharply for earlier values and starts to saturate around |xr| = |xq| = 50 games. Even when given a small set of labeled examples, i.e., |xr| = 10, our model performs reasonably well. In comparison, personalized move-prediction models require thousands of games to begin seeing improvements" — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Vs. personalized Maia stylometry on the same 400 players (k=15): personalized models P@1 0.552 vs embedding model 0.953 — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Scale: all 41,184 players with 1K+ games as candidates → P@1 0.540 (baseline 0.0849); 9,465 players with 5K+ games → 0.737 — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- High-ranked players (top 1,500 on Lichess and Chess.com leaderboards): P@1 0.308 among 2,157 (baseline 0.039) — much harder; narrow rating band near 1900 → 92.6% — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Embeddings cluster stylistically similar players (e.g., by preferred first move as White) — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Code at github.com/CSSLab/behavioral-stylometry, but "As there are ethical concerns regarding releasing the model and full training code it may not be publicly accessible" (access by request); incorporated code is GPL and MIT — [stylometry paper](https://arxiv.org/pdf/2208.01366)

### Inferences
- The embeddings themselves don't predict moves, so they can't directly answer "what would this user play here?" They could be used to find similar players (nearest neighbors in style space) and pool their data — which is essentially what Maia4All's prototype matching later did.
- Released model weights may be unavailable, so a developer would likely need to use Maia4All-style or Maia-2/3 embeddings instead.

### Gaps
- Exact P@1 at 5, 10, 20 games is only shown in a figure (Fig. 3a); numeric values not in text.
- No experiment used these embeddings for move prediction / personalization.

---

## 4. Maia-2 (NeurIPS 2024): skill-aware attention, rating conditioning, accuracy, per-player support, package/license

### Takeaway
Maia-2 replaces nine separate Maia models with one 23.3M-parameter model (ResNet + "skill-aware attention") conditioned on categorical embeddings of both the player's and opponent's rating; it reaches 53.25% top-1 vs 51.39% for the best Maia-1, is trained supervised on ~169M Lichess games, and ships as `pip install maia2` under MIT license. It does NOT itself personalize to individuals (only rating), but its rating embeddings are the hook that Maia4All later swapped for per-player embeddings.

### Cited Findings
- "Maia-2: A Unified Model for Human-AI Alignment in Chess", Zhenwei Tang, Difan Jiao, Reid McIlroy-Young, Jon Kleinberg, Siddhartha Sen, Ashton Anderson; NeurIPS 2024; arXiv 30 Sep 2024, v2 31 Oct 2024 — [arXiv](https://arxiv.org/abs/2409.20553)
- Architecture: ResNet backbone with 12 conv blocks + 2 skill-aware attention blocks; 16 heads, d_att = 1024, head dim 64; 18 input channels; 23.3M parameters vs 92M total for the nine Maia-1 models — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Rating conditioning: ratings as categorical embeddings (dim 128) in buckets "(0,1000]", "(1000,1100]", ... "(2000,+∞)"; active-player and opponent embeddings concatenated and added to attention queries — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Heads: policy (cross-entropy on human move), value (regression on game outcome 1/0/−1), auxiliary head (legal moves, piece moved/captured, squares, check) — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Data: Lichess Rapid games Jan 2013–Nov 2023 excluding Dec 2019 (test month); ~168.93M games / 9.15B positions after filtering/balancing (from 5.14B games consumed) — [Maia-2 paper](https://arxiv.org/html/2409.20553); repo says released training range 2018-05 to 2023-11 and offers both Rapid and Blitz models — [maia2 repo](https://github.com/CSSLab/maia2)
- Accuracy (Maia-2 test set, top-1): Maia-2 53.25% avg vs Maia 1500 51.39% and Maia 1100 48.82%; by group Skilled 51.72%, Advanced 54.15%, Master 53.87%; perplexity 4.07 vs 4.67 (Maia 1900) — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Coherence across skill: treats 27% of "monotonic" positions monotonically vs ~1% for Maia-1 — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Compute: "approximately 13 days" on 2× A100 80GB; batch 8192 positions; LR 1e-4 — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Personalization: the paper does not present per-player personalization (only mentions prior fine-tuning work) — [Maia-2 paper](https://arxiv.org/html/2409.20553); repo README does not mention per-player fine-tuning — [maia2 repo](https://github.com/CSSLab/maia2)
- Package: `pip install maia2` (PyPI v0.11.0 per badge; Python 3.10–3.12); `model.from_pretrained(type="rapid"|"blitz", device="auto")` picks CUDA → Apple MPS → CPU; `inference.inference_each(model, prepared, fen, elo_self, elo_oppo)` returns `(move_probs, white_expected_score)`; batch inference available — [maia2 repo](https://github.com/CSSLab/maia2)
- License: MIT ("Maia-2 is released under the MIT License"); README now recommends Maia-3 for new projects — [maia2 repo](https://github.com/CSSLab/maia2)
- Maia4All later reports, on 2023 Blitz unseen players, Maia-2 at 51.46% vs Maia at 51.32% top-1 — [Maia4All](https://arxiv.org/html/2507.21488v1)

### Inferences
- For blunderbot, Maia-2 is the most practical drop-in "what would a player of rating R play here (with probabilities)" model: MIT license, pip-installable, runs on CPU, and the rating conditioning allows comparing the user's actual move against peers of the same rating.

### Gaps
- No published CPU inference latency for Maia-2.
- Downloaded weight file size not stated in the sources read.

---

## 5. 2024–2026 follow-ups (Allie, Maia4All, Maia-3/Chessformer, other individual-modeling papers, Maia platform)

### Takeaway
The key development for blunderbot is Maia4All (Tang et al., 2025; listed TMLR 2026): per-player embeddings on a frozen Maia-2 backbone give gains from ~20 games (51.4% → 53.2%), 250× less data than Maia-Individual. Maia-3 (Chessformer, 2026) is the current state of the art for population human-move prediction (57.1%, AGPL, 5M/23M/79M models on HuggingFace, rating-conditioned UCI engine) but does not ship per-player personalization. Allie (ICLR 2025) adds human think-time/resignation modeling and adaptive search.

### Cited Findings
**Allie (ICLR 2025)**
- "Human-Aligned Chess With a Bit of Search", Yiming Zhang, Athul Paul Jacob, Vivian Lai, Daniel Fried, Daphne Ippolito; arXiv 4 Oct 2024; ICLR 2025 — [arXiv](https://arxiv.org/abs/2410.03893); [ICLR poster](https://iclr.cc/virtual/2025/poster/29104)
- Decoder-only transformer, 355M params, non-embedding weights initialized from GPT-2 medium; heads for policy, pondering time, value — [Allie paper](https://arxiv.org/html/2410.03893)
- Data: all Lichess blitz games from 2022 — 91M games, 6.6B tokens — [Allie paper](https://arxiv.org/html/2410.03893)
- Move-matching on 884,049 positions: Allie-Policy 55.7%, Allie-Adaptive-Search 55.9%, GPT-3.5 53.7%, rating-matched Maia 51.6% — [Allie paper](https://arxiv.org/html/2410.03893)
- Think time: Pearson r = 0.697 with human think time; resignation TPR 86.4%, FPR 0.1% — [Allie paper](https://arxiv.org/html/2410.03893)
- Adaptive MCTS: rollouts ∝ predicted human think time (avg 50 rollouts); human study 7,483 games vs 2,412 Lichess players (1000–2600 Elo); skill gap 49 Elo avg (policy-only 127 Elo) — [Allie paper](https://arxiv.org/html/2410.03893)
- Compute: ~2 weeks on 8× NVIDIA A6000; 2M steps; ablation: halving data −1.0%, 124M model −1.2%, 774M +0.3% — [Allie paper](https://arxiv.org/html/2410.03893)
- Code, data, weights: github.com/y0mingzhang/allie — [Allie paper](https://arxiv.org/html/2410.03893)
- No per-individual personalization in Allie (conditions on ratings/game logs) — [Allie paper](https://arxiv.org/html/2410.03893)

**Maia4All: "Learning to Imitate with Less: Efficient Individual Behavior Modeling in Chess"**
- Authors Zhenwei Tang, Difan Jiao, Eric Xue, Reid McIlroy-Young, Jon Kleinberg, Siddhartha Sen, Ashton Anderson; arXiv 29 Jul 2025 — [Maia4All](https://arxiv.org/html/2507.21488v1); listed as TMLR 2026 by mlanthology (secondary listing; not verified against TMLR/OpenReview directly) — [mlanthology](https://mlanthology.org/tmlr/2026/tang2026tmlr-learning)
- Method: built on Maia-2 policy head. (1) "Enrichment": replace Maia-2's 11 rating-bin embeddings with individual embeddings for 1,100 "prototype" players (100 most frequent per bin), fine-tune backbone + embeddings jointly. (2) "Democratization": for a new player, a Prototype Matching Network classifies their games over prototypes; top-k weighted average initializes their embedding (fallback: rating-bin embedding) — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Adaptation updates ONLY the new player's embedding; backbone frozen. Unfreezing helps slightly at 20,000 positions (0.5395 vs 0.5365) but hurts at 800 positions (0.5297 vs 0.5322) — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Data: Lichess Blitz, 2023; ~40 positions per game (800 positions ≈ 20 games); evaluation on last 2,048 positions per player (225,280 test positions) — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Top-1 on unseen players (Table 1): Maia 0.5132 and Maia-2 0.5146 (fixed) vs Maia4All-Prototype 0.5322 (~20 games), 0.5334 (~50), 0.5348 (~200), 0.5365 (~500); at ~2,500 games (100k positions) 0.5381 (Master group 0.5554) — [Maia4All](https://arxiv.org/html/2507.21488v1)
- "achieves individual human behavior modeling in chess with only 20 games, compared to the 5,000 games required previously" — "250 times more data-efficient" — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Perplexity: Maia 5.45, Maia-2 4.53, Maia4All 4.30 at 800 positions, 4.19 at 100k positions — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Prototype matching network identifies correct player with 89% one-shot accuracy among 1,100 candidates from 800 positions — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Maia-Individual excluded as baseline because it "does not apply to the sparser settings we consider (and which cover the vast majority of players on Lichess)" — [Maia4All](https://arxiv.org/html/2507.21488v1)

**Maia-3 / Chessformer (2026)**
- "Chessformer: A Unified Architecture for Chess Modeling", Daniel Monroe, George Eilender, Philip Chalmers, Zhenwei Tang, Ashton Anderson; arXiv 18 May 2026 — [arXiv](https://arxiv.org/abs/2605.19091)
- Encoder-only transformer with squares as tokens, Geometric Attention Bias, attention-based source-destination policy head; Maia-3 human models reach 57.1% move-matching "with fewer than a quarter of the parameters" of the prior state of the art; Chessformer also integrated into Leela Chess Zero (+100 Elo) — [arXiv](https://arxiv.org/abs/2605.19091)
- Venue CONFLICT: arXiv listing and maia3 repo / maiachess.com cite ICLR 2026 — [arXiv](https://arxiv.org/abs/2605.19091); [maia3 repo](https://github.com/CSSLab/maia3); the maia-chess README calls it "ICML '26" — [maia-chess repo](https://github.com/CSSLab/maia-chess)
- Models on HuggingFace: UofTCSSLab/Maia3-5M (recommended for CPU/GUIs), Maia3-23M, Maia3-79M, Maia3-ablate-3M; install from source (no PyPI); UCI engine with `SelfElo`/`OppoElo`/`Elo`, `Temperature`, `TopP`, `MultiPV` listing most likely human moves, WDL from value head predicting human-game outcomes — [maia3 repo](https://github.com/CSSLab/maia3)
- License AGPL-3.0; no per-player fine-tuning mentioned (only Elo conditioning) — [maia3 repo](https://github.com/CSSLab/maia3)

**Other 2026 individual-modeling preprints (not peer-reviewed)**
- "Toward Modeling Player-Specific Chess Behaviors", Sogliuzzo, Rautureau, Piette; arXiv 12 May 2026: Maia-2 with champion-specific embeddings for 16 historical world champions + limited MCTS; MCTS lowers move accuracy but "substantially" reduces Jensen-Shannon divergence of move distributions (style alignment); no numeric accuracy in abstract — [arXiv 2605.11893](https://arxiv.org/abs/2605.11893)
- arXiv 2606.25176 (single author Jason Carlson; v3 29 Jul 2026, titled "Matilda: Engine-Agnostic Search with Human Policy Guidance"; an earlier version appeared in search titled "Elo-Disentangled Player-Style Embeddings for Human Chess via a Rating-Conditioned Residual Move Model"): ~1.7M-param re-ranker over frozen Maia-3 23M top-16 candidates + 32-dim per-player style embedding (~0.2M params; only embedding+projection trained per player); reports newly fitted players "break even around 60 decisions" (~2 games); gains mostly at 2500+ Elo (e.g., 3000+ top-1 61.1% → 68.4%), ~+0.02% below 2500 — [arXiv 2606.25176](https://arxiv.org/html/2606.25176)

**Maia Chess platform (maiachess.com)**
- Features listed: Play Maia (Lichess-rating range 600–2600), Hand & Brain, Analysis ("human-style insights"), Puzzles, opening Drill, Bot or Not, Broadcasts; "Sign in with Lichess"; no per-user personalized model or Chess.com import described on the homepage; Lichess bots Maia 1100/1500/1900 — [maiachess.com](https://www.maiachess.com/)
- Web platform source is listed as open-source on the site — [maiachess.com](https://www.maiachess.com/)

### Inferences
- Maia4All's "embedding-only adaptation on a frozen backbone" is the most directly applicable recipe for blunderbot: a single shared model + a tiny per-user vector, trainable from tens of games, cheap to store per user.
- The absolute accuracy gain from personalization at low data is modest (~1.8–2.3 pp at 20–500 games); for lesson generation, the value is more in calibrated per-user probabilities on specific mistakes than in top-1 accuracy.
- Maia-3 is now the strongest population baseline but AGPL; Maia-2 is MIT. License choice matters for a closed-source app.

### Gaps
- No public code repo/license found for Maia4All in the sources read.
- No paper yet personalizes Maia-3 via the official CSSLab pipeline (only the single-author 2606.25176 preprint, which focuses on 2500+ players).
- Could not confirm which model (Maia-2 vs Maia-3) powers maiachess.com analysis, or whether "blunder meter"/"move map" features exist (not on homepage).

---

## 6. Supervised learning vs. reinforcement learning: what the papers say

### Takeaway
Every human-modeling model in this line (Maia, Maia-Individual, Maia-2, Maia4All, Allie, Maia-3) is trained by supervised learning / behavior cloning on human moves, not RL. The Maia paper explicitly tested AlphaZero-style RL networks (Leela) and found them unable to target a skill level, and found tree search degrades human-move prediction by 5–10 pp.

### Cited Findings
- Explicit statement: "Maia Chess, is a supervised learning adaptation of Leela that attempts to predict the next move the average human player at a particular rating level will make" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Maia's "fundamental change is to replace training on self-play games with training on human games" — [Maia paper](https://arxiv.org/pdf/2006.01855)
- RL/self-play evaluation: Leela (RL) "matches human moves with greater accuracy than depth-limited Stockfish, [but] its accuracy is relatively constant across the range of player skill levels ... Thus, using AlphaZero-style deep reinforcement learning off the shelf does not pass our test of accurately modeling granular human behavior either." Attenuating by early-stopping self-play training doesn't fix it — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Search hurts: "while tree search is clearly important to strong chess-playing performance, we found that it degrades our performance at predicting human moves" (5–10 pp drop with 10 rollouts) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Stockfish: "if your goal is to use Stockfish to predict the moves of even a relatively weak human player, you will achieve the best performance by choosing the strongest version of Stockfish you can" — weakening an engine does not make it play like weak humans — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Maia-2 policy trained with cross-entropy on human moves — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Allie: next-move log-likelihood on human games; MSE for think time and value; no RL for the network (search used only at play time) — [Allie paper](https://arxiv.org/html/2410.03893)
- The KDD 2022 paper frames personalization as transfer learning/fine-tuning (related to imitation learning, meta-learning) — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Commercial "personality" bots (Play Magnus, Chess.com personalized bots) "are thought to be simple attenuated versions of existing chess engines" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Search can trade accuracy for style/strength: Allie's adaptive MCTS improves skill calibration (49 Elo gap) and adds +0.2 pp move-matching — [Allie paper](https://arxiv.org/html/2410.03893); in champion modeling, MCTS lowers move accuracy but improves distributional style alignment — [arXiv 2605.11893](https://arxiv.org/abs/2605.11893)

### Inferences
- The developer's original assumption (train an RL network on the user's games) is not what the field does: modeling a human is a supervised prediction problem (predict the move the person played), and RL objectives optimize winning, which pushes the model away from the human. The right approach is supervised (behavior cloning), starting from a pretrained population human model and adapting cheaply to the user.
- Stockfish still has a role: labeling which of the user's moves were mistakes (eval/win-prob drop), which the human model alone cannot do.

### Gaps
- No paper found that trains a pure RL agent from scratch on one individual's games for comparison (none would be expected given the above).

---

## 7. What personalized models can practically produce for a learning app

### Takeaway
These models output a probability distribution over legal moves for a given position conditioned on rating (Maia-2/3) or on a player identity (Maia-Individual, Maia4All), plus a value/outcome estimate; combined with Stockfish-labeled mistakes, this supports "probability this user plays move X", "probability this user blunders here", and "where this user errs more than rating peers".

### Cited Findings
- Maia-2 returns per-move probabilities and a White-perspective expected score for a FEN + `elo_self` + `elo_oppo` — [maia2 repo](https://github.com/CSSLab/maia2)
- Maia-3 UCI engine gives most-likely human moves via MultiPV and WDL predicting human-game outcomes, with SelfElo/OppoElo conditioning — [maia3 repo](https://github.com/CSSLab/maia3)
- Personalized models predict "everything from good moves to serious blunders" better than base Maia, and assign higher probability to a player's actual blunders — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Authors' proposed lesson-selection rule: blunders where personalized probability x > base-Maia probability y — "predictably difficult for this specific player" and "relatively fixable (since other players are less likely to err)" — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Population blunder-likelihood per position: residual CNN 71.7% balanced accuracy (individual move) and 76.9% (whether >10% of population blunders in a common position) — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Maia (population) "does the best job of predicting" blunders vs. engines — [Maia paper](https://arxiv.org/pdf/2006.01855)
- Per-player embeddings from ~20 games can be learned and also identify a player (89% among 1,100 from 800 positions) — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Allie additionally predicts human think time (r = 0.697) and resignation, and "ponders" at critical positions — [Allie paper](https://arxiv.org/html/2410.03893)

### Inferences
- Concrete app pipeline implied by the literature: (1) Stockfish labels each user move by win-prob loss (Maia used ≥10 pp as "blunder"); (2) a rating-conditioned human model (Maia-2/3) gives P(peer plays the user's move) and P(peer finds the best move); (3) optionally a per-user model (Maia4All-style embedding) gives P(user plays move); (4) rank recurring motifs/positions where the user's error rate or P(user blunders) exceeds peers' — that ranking is the "personalized weakness" list for lessons.
- Even without per-user fine-tuning, "this user vs. rating-matched Maia" already separates idiosyncratic mistakes from level-typical ones; per-user modeling mainly adds better calibration and the ability to predict future mistakes in new positions.
- Predicted blunder probability for a new position can be approximated as the probability mass the (personalized) human model puts on moves Stockfish judges to be blunders — an inference, not a method evaluated in these papers.

### Gaps
- No paper evaluates an end-to-end personalized lesson/tutoring app or measures learning outcomes (rating improvement) from such lessons.
- No published per-user "blunder probability" model (the blunder classifier in the Maia paper is population-level, conditioned on rating/clock metadata, not personalized).

---

## 8. Compute and data requirements

### Takeaway
Pretraining is expensive (days–weeks on multi-GPU) but already done and released; per-user adaptation is cheap: full fine-tune ~40 min on one V100 (~300MB VRAM) needing ≥5,000 games, or embedding-only adaptation from ~20 games. Inference models are small (Maia-1 ~10M params; Maia-2 23.3M; Maia-3 5M–79M with 5M recommended for CPU).

### Cited Findings
- Maia-1: 12M games per rating bin, 400k steps × batch 1024; ~10.3M params per model (as reported by Maia-2 paper) — [Maia paper](https://arxiv.org/pdf/2006.01855); [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Maia-Individual fine-tune: 30k steps; most gain within 12k steps (3M board-move pairs); ~40 min per player on a V100, ~300MB VRAM; needs ≥5,000 games for gains — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- Stylometry model: 63.7M games, 16,181 players, 4× K80 GPUs — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- Maia-2: ~169M games / 9.15B positions, ~13 days on 2× A100 80GB; 23.3M params — [Maia-2 paper](https://arxiv.org/html/2409.20553)
- Allie: 91M games / 6.6B tokens, ~2 weeks on 8× A6000; 355M params — [Allie paper](https://arxiv.org/html/2410.03893)
- Maia4All: per-player gains from ~800 positions (~20 games); only the per-player embedding is optimized at adaptation time — [Maia4All](https://arxiv.org/html/2507.21488v1)
- Maia-3: 5M model recommended for CPU use and chess GUIs; 23M/79M for higher accuracy — [maia3 repo](https://github.com/CSSLab/maia3)
- Maia-2 runs on CUDA, Apple MPS, or CPU — [maia2 repo](https://github.com/CSSLab/maia2)
- Maia-1 runs in lc0 with search disabled ("a nodes limit of 1") — [maia-chess repo](https://github.com/CSSLab/maia-chess)

### Inferences
- Because Maia-style prediction uses a single forward pass (no search), CPU inference over a few hundred positions per user game should be feasible on commodity servers; this is an inference from model sizes, not a measured benchmark.
- Embedding-only personalization means per-user storage is a small vector (Maia-2 skill embeddings are 128-dim) rather than a full model copy — inferred from the Maia-2 embedding size and Maia4All design.

### Gaps
- No published CPU latency/throughput numbers for Maia-1, Maia-2, or Maia-3.
- Maia4All adaptation time per user not reported.

---

## 9. Code availability and licenses

### Takeaway
Population models are openly available: Maia-1 (GPL-3.0, lc0 weights), Maia-2 (MIT, pip), Maia-3 (AGPL-3.0, HuggingFace), Allie (GitHub with weights). Per-player code exists (maia-individual) but no personalized models are released, and the stylometry model release is restricted on ethical grounds; no Maia4All repo was found.

### Cited Findings
- CSSLab/maia-chess: GPL-3.0; nine weight files maia-1100…maia-1900 (.pb.gz); run with lc0, `go nodes 1`; Lichess bots maia1/maia5/maia9; README points to Maia-2 and Maia-3 — [maia-chess repo](https://github.com/CSSLab/maia-chess)
- CSSLab/maia-individual: README says GPL (includes Leela code) while repo sidebar shows AGPL-3.0 (CONFLICT — check LICENSE file); contains pipeline (player counting → data generation → `train_transfer.py`) and base maia-1900; tested with lc0 v23, Ubuntu 18.04, CUDA 10; "We do not have any public models of players publicly available at this time"; 5 commits; analysis code "hasn't been tested" — [maia-individual repo](https://github.com/CSSLab/maia-individual); code location also stated in paper — [KDD 2022 paper](https://arxiv.org/pdf/2008.10086)
- CSSLab/behavioral-stylometry: model and full training code "may not be publicly accessible" due to ethical concerns; access via request to corresponding author — [stylometry paper](https://arxiv.org/pdf/2208.01366)
- CSSLab/maia2: MIT; `pip install maia2`; Rapid and Blitz weights via `from_pretrained` — [maia2 repo](https://github.com/CSSLab/maia2)
- CSSLab/maia3: AGPL-3.0; install from source; HuggingFace UofTCSSLab/Maia3-5M/23M/79M — [maia3 repo](https://github.com/CSSLab/maia3)
- Allie: code, data, and weights at github.com/y0mingzhang/allie — [Allie paper](https://arxiv.org/html/2410.03893)
- Lichess game data used in stylometry work is CC0 — [stylometry paper](https://arxiv.org/pdf/2208.01366)

### Inferences
- For a commercial/closed-source app, Maia-2 (MIT) is the least-encumbered base; Maia-1/maia-individual (GPL) and Maia-3 (AGPL, which also covers network/server use) impose copyleft obligations that a developer should review.
- A developer implementing Maia4All-style per-user embeddings would likely need to re-implement it on top of Maia-2 from the paper description.

### Gaps
- Allie's license was not verified (repo not fetched).
- Licensing of model weights separately from code is not stated for any repo.
- Maia4All code release status unknown.
