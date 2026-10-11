# Predicting when and why a chess player errs, modeling difficulty, and where RL fits in a personalized training app

Research date: 2026-10-09. Scope: (a) blunder/mistake prediction and human-perceived difficulty modeling; (b) a comparison of RL, supervised learning and statistical models for a personalized chess-training app. Another researcher covers the Maia family in depth, so Maia appears here only where it bears on blunder or difficulty prediction, or on how much data personalization needs.

## Q1. Research on predicting blunders and errors: what predicts that a human will make a mistake?

### Takeaway
How hard the position is predicts errors much better than player skill or remaining clock time. In Anderson, Kleinberg and Mullainathan's tablebase study, difficulty features alone reach about 73–75% balanced accuracy, versus 55% for skill and 53% for time. Deep models that read the board directly (Maia's blunder head) predict whether a single move will be a blunder with 67.7% accuracy from the board alone, and 71.7% when rating and clock are added. Predicting whether a position is blunder-prone for a population of players is easier, at about 77%. Regan's "swing", "complexity" and "difficulty" measures show the same pattern: positions where the best move only reveals itself at deeper engine search cut the best-move rate roughly in half at every level, from 2200 through 2700.

### Cited Findings
**Anderson, Kleinberg & Mullainathan, "Assessing Human Error Against a Benchmark of Perfection" (KDD 2016; ACM TKDD 11(4), Article 45, 2017)**
- The study uses positions with 6 or fewer pieces, where chess tablebases give ground truth. Data: FICS, about 200M games, giving 24.6M non-losing instances (mostly 3-minute games); a GM dataset of about 1M games, giving 880K non-losing instances. A blunder is a move that turns a theoretically non-losing position into a losing one. — [ar5iv 1606.04956](https://ar5iv.labs.arxiv.org/html/1606.04956); [TKDD PDF](https://www.cs.toronto.edu/~ashton/pubs/tbase-tkdd.pdf)
- Features fall into three groups:
  - Skill: the Elo of the player and of the opponent.
  - Time: time remaining.
  - Difficulty: n(P), the number of legal moves; b(P), the number of blundering moves; "blunder potential" β(P) = b/n, the chance that a random move is a blunder; plus depth-2 features. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Balanced binary task "will this player blunder in this position?" (600K balanced instances; random = 0.50):

  | Features | Accuracy |
  |---|---|
  | β alone | 0.73 |
  | Depth-1 difficulty | 0.73 |
  | Depth-1 + depth-2 difficulty | 0.75 |
  | Skill alone | 0.55 |
  | Time alone | 0.53 |
  | All features | 0.75 |

  On GM data, difficulty features alone reach 0.77. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- With the position held fixed (25 positions with more than 500 blunders each), rating alone gives 0.62, both players' clocks give 0.54, and the two together give 0.63. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Raising blunder potential by 0.2 increases the blunder rate more than a 600-point rating gap does (FICS, 1200–1800). GM players rated 2700 blunder more often at β = 0.9 than 1200-rated players do at β = 0.3. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Some positions are "skill-anomalous": higher-rated players blunder more often in them. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Time effects:
  - The blunder rate rises sharply as remaining time approaches 0 and is nearly flat above about 10 seconds.
  - Extra time helps higher-rated players more.
  - Players who spend longer on a move blunder more often in aggregate. This is a selection effect: people think longer in hard positions.

  — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Humans asked to guess which of two positions is more blunder-prone scored about 0.69 (about 6,000 responses from 720 participants). The model scored 0.79 on the same pairs. — [ar5iv](https://ar5iv.labs.arxiv.org/html/1606.04956)

**Maia paper, McIlroy-Young, Sen, Kleinberg & Anderson, "Aligning Superhuman AI with Human Behavior: Chess as a Model System" (KDD 2020), Section 5 on blunder prediction**
- Blunder definition: a move that lowers the mover's win probability by at least 10 percentage points. Stockfish centipawns are converted to win probability with an empirical lookup table. — [ar5iv 2006.01855](https://ar5iv.labs.arxiv.org/html/2006.01855)
- Training data: Lichess games from April 2017 to September 2019 that carry Stockfish evaluations, which is only about 10% of games. Fast time controls and moves made with under 30 seconds left were excluded. 182M blunders and 272M non-blunders. Test set: October 2019, balanced, 9M + 9M. — [ar5iv](https://ar5iv.labs.arxiv.org/html/2006.01855)
- Accuracy predicting whether a single move will be a blunder (board only / board + metadata, where metadata is both ratings, both players' % of clock left, and current eval):

  | Model | Board only | Board + metadata |
  |---|---|---|
  | Random forest | 56.4% | 63.0% |
  | Fully connected | 62.9% | 66.0% |
  | Residual CNN | 67.7% | 71.7% |

  — [ar5iv](https://ar5iv.labs.arxiv.org/html/2006.01855)
- Collective task: does more than 10% of the population blunder in this position? Only positions seen more than 10 times; 1.06M training boards, 119K balanced test boards. Fully connected 69.5%, residual CNN 75.3%, deep residual CNN 76.9%. The authors attribute the easier task to noise averaging. — [ar5iv](https://ar5iv.labs.arxiv.org/html/2006.01855)

**Regan / Biswas: intrinsic ratings and complexity**
- Intrinsic Performance Ratings (Regan & Haworth, AAAI 2011) rate players by the engine-judged quality of their moves, not by game results.
  - The model maps deep engine evaluations of all legal moves to the probability that an agent with parameters sensitivity s and consistency c picks each move.
  - The parameters are fitted by regression to tournament games across Elo levels and map smoothly to the Elo scale.
  - Because it counts moves rather than games, the sample is larger: for example, 50 games × about 30 relevant moves ≈ 1,500 decisions.

  — [Regan IPR paper](https://cse.buffalo.edu/~regan/papers/pdf/Reg12IPRs.pdf); [Regan & Haworth, Intrinsic chess ratings](https://centaur.reading.ac.uk/19778/)
- Biswas & Regan, "Quantifying Depth and Complexity of Thinking and Knowledge" (ICAART 2015, pp. 602–607) define:
  - **Swing:** how much a move's value relative to the alternatives changes between shallow and deep Stockfish search (up to depth 19). A move that "swings down" falls into a trap.
  - **Complexity:** disagreement between move rankings across search depths.
  - **Difficulty:** the relative depth at which half the swing has happened, multiplied by complexity, expressed in depth units.
  - Data: round-robin games from 2006–2009 by players within 10 Elo of each milepost from 2200 to 2700, plus all 900 games of the 2013 World Blitz (average rating 2611).

  — [Biswas & Regan 2015 PDF](https://www.scitepress.org/papers/2015/52883/52883.pdf)
- Results, with rates computed from the counts in the paper's Tables 1 and 4:

  | | Easiest bin | Hardest bin |
  |---|---|---|
  | Difficulty < 1 vs 4–5, best-move rate, 2700 players | 91.0% | 39.8% |
  | Same, 2200 players | 84.8% | 31.8% |
  | Same, World Blitz | 80.1% | 31.0% |
  | Swing < 1 vs 4–5, best-move rate, World Blitz | 59.7% | 18.1% |
  | Same, 2700 players | 71.9% | 29.4% |

  The best-move probability falls monotonically as complexity rises. World Blitz play "seemed to function as if" it were at an 1800–2100 level. — [Biswas & Regan 2015 PDF](https://www.scitepress.org/papers/2015/52883/52883.pdf)
- Positions with a "swing-down" or trap structure catch players of every rating, and higher-rated players are only slightly less fooled. — [Biswas & Regan 2015 PDF](https://www.scitepress.org/papers/2015/52883/52883.pdf)

**Human-perceived difficulty via "meaningful" search trees**
- Stoiljkovikj, Bratko & Guid, "A computational model for estimating the difficulty of chess problems" (Advances in Cognitive Systems 2015) predicts how hard a tactical problem is for humans. They analyze properties of search trees restricted to moves a human would consider "meaningful", trained on difficulty ratings from human solvers. The earlier work is Guid & Bratko, AIED 2013. — [chessprogramming wiki: Matej Guid](https://chessprogramming.org/Matej_Guid); [UL repository](https://repozitorij.uni-lj.si/IzpisGradiva.php?id=70145)

### Inferences
- For "predict which positions this user will miss", the strongest signal is **position difficulty measured for humans at the user's level**, not the user's identity. In practice that means features such as:
  - a Maia/Maia-2 probability that a player at the user's rating finds the correct move;
  - blunder potential: the share of legal or plausible moves that lose;
  - Regan-style swing: the best move only appears at deeper search.

  The user-specific part is a smaller residual on top. A per-user model should therefore be built as "population difficulty model + per-user, per-motif offset", not as a model trained from scratch on one user.
- "Skill-anomalous" positions, plus Regan's finding that traps fool every level, imply that motif-level weakness should be measured *relative to peers at the same rating* (see Q6). Raw miss rates mix together "this motif is hard for everyone" and "this user is weak at it".
- A per-user blunder classifier can reach only about 70% accuracy even with population-scale data, so per-move predictions should be treated as probabilities for ranking and prioritizing, not as reliable yes/no calls.

### Gaps
- I found no published model trained to predict blunders for one *specific individual*. Maia's personalization work (KDD 2022; Maia4All) measures move-matching, not blunder prediction specifically (see Q5).
- I did not retrieve the full text of Regan & Haworth (AAAI 2011), so the exact (s, c) formula is not reproduced here.
- I did not get the Stoiljkovikj et al. 2015 accuracy numbers (full text not retrieved).

## Q2. Predicting chess puzzle difficulty: IEEE BigData Cup 2024 and FedCSIS 2025

### Takeaway
Both competitions asked teams to predict a Lichess puzzle's Glicko-2 rating from its FEN and solution moves. Pretrained *human-imitation* networks (Maia-1, Maia-2), sometimes combined with Leela embeddings, won both times:
- **2024:** won by the "bread emoji" ensemble of Maia + Leela embedders.
- **2025:** won with fine-tuned Maia-2 feeding LightGBM/CatBoost. Final MSE was 52.3k (RMSE ≈ 229 rating points).
- **Ablation:** removing the fine-tuned Maia-2 predictions raised MSE by 49%. Human-move-probability models are therefore the key ingredient for estimating difficulty as humans experience it.

### Cited Findings
**IEEE BigData Cup 2024 (knowledgepit.ai, run with Lichess data)**
- Task: predict a puzzle's difficulty (Glicko-2 rating) from its configuration and solution. The organizers' report is Świechowski et al., IEEE BigData 2024, pp. 8423–8429. — [IEEE report](https://ieeexplore.ieee.org/document/10825289/)
- Participation and winners (per a Lichess blog post dated 24 Dec 2024): "almost a hundred teams". 1st "bread emoji" (Woodruff, Filatov, Cognetta); 2nd "anansch" (Schütt, Huber, André); 3rd "ousou" (Björkqvist). — [Lichess blog, frugofruit90](https://lichess.org/@/frugofruit90/blog/ai-research-for-better-puzzles-competition-results-and-summary/ww03rPco)
- The winner used "ensembled, pretrained, neural chessboard embedders (truncated Maia and Leela models)" plus distribution-rescaling postprocessing. Its MSE was more than 16.4% lower than second place in the preliminary evaluation and more than 13.3% lower in the final evaluation. — [IEEE 10826037](https://ieeexplore.ieee.org/document/10826037/)
- The winning 2024 solution scored 49.1k MSE on the 2024 preliminary set (RMSE ≈ 222). Run unchanged on the 2025 set, it scored 64.1k. — [bread emoji FedCSIS 2025 paper](https://annals-csis.org/Volume_43/drp/pdf/6771.pdf)
- Problems with the 2024 test labels:
  - Many test puzzles stayed near the 1500 Glicko initialization because they had too few solving attempts.
  - The 2025 organizers fixed this with more solvers and a "fairness algorithm" in a Lichess fork that served puzzles with the fewest attempts first.

  — [FedCSIS 2025 organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)
- Other 2024 approaches:
  - **GlickFormer (transformer, Miłosz & Kapusta):** trained on 4.2M Lichess puzzles. Inputs: 16-channel 8×8 boards for up to 5 positions along the solution line. Results: MAE 217.7 rating points; 27.0% of predictions within 1 RD of the true rating, 50.5% within 2 RD. Placed 11th.
  - **Pairwise learning-to-rank** (Rafaralahy).
  - A network "inspired by human problem-solving" (Schütt et al.).

  — [arXiv 2410.11078](https://arxiv.org/html/2410.11078v2); [IEEE 10825356](https://ieeexplore.ieee.org/document/10825356/); [IEEE/CSDL Schütt et al.](https://www.computer.org/csdl/proceedings-article/bigdata/2024/10826087/23yk3ZeoJRS)

**FedCSIS 2025 "Predicting Chess Puzzle Difficulty Part 2" (Kraków, 2025)**
- Participation:
  - The organizers' report counts 42 teams and 1,185 submissions, with 8 of the top 10 teams writing papers.
  - The winner's paper says "top 5 teams out of 31", probably counting teams on the final leaderboard. The two sources differ on this.

  — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf); [ousou paper](https://annals-csis.org/Volume_43/drp/pdf/6497.pdf)
- Data:
  - Test set: 2,283 puzzles after dropping those with fewer than 10 solving attempts; ratings range from about 800 to 3000.
  - Training data: Lichess puzzles with RatingDeviation, Popularity, NbPlays, Themes, GameUrl and OpeningTags. The runner-up describes this as about 4.5M Lichess puzzles.
  - Metric: MSE.

  — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf); [bread emoji 2025](https://annals-csis.org/Volume_43/drp/pdf/6771.pdf)
- **New in 2025:** the organizers supplied 22 Maia-2 "success probability" features for each puzzle. Each is the probability that a player of a given rating band and game type (rapid or blitz) solves the puzzle. — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)
- Final standings (MSE):

  | Rank | Team | Final MSE | Approx. RMSE | Method |
  |---|---|---|---|---|
  | 1 | ousou | 52.3k | ≈ 229 | Fine-tuned Maia-2 → LightGBM → CatBoost, linear rescaling |
  | 2 | bread emoji | 54.4k | ≈ 233 | Ensembled Maia/Leela embedders, pre-RNN pooling, log rescaling |
  | 3 | transformer_enjoyer | 55.9k | ≈ 236 | Spatial transformer with masked-square pretraining |
  | 4 | — | 57.5k | | |
  | 5 | Cyan | 61.0k | | |
  | 6–10 | | 62.6k–68.1k | | Mostly Elo-banded models + gradient boosting, or CNNs on rendered board images |

  — [organizers' report, Table I](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)
- The common winning pattern was pretrained neural board embeddings (Maia-1, Maia-2 or Leela), then gradient boosting (LightGBM, XGBoost or CatBoost), then calibration or rescaling for distribution shift. — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)
- Winner's ablation (Björkqvist; test set of 50K Lichess puzzles):

  | Configuration | MSE |
  |---|---|
  | Final CatBoost ensemble | 35.4k |
  | LightGBM, all features | 36.4k |
  | LightGBM without hand-crafted features | 36.8k |
  | LightGBM without engine features | 37.3k |
  | Fine-tuned Maia-2 alone, with extra features | 38.9k |
  | Fine-tuned Maia-2 alone, without extra features | 40.8k |
  | LightGBM without fine-tuned Maia-2 | **52.6k (+49%)** |

  — [ousou paper](https://annals-csis.org/Volume_43/drp/pdf/6497.pdf)
- The winner's engine features include the policy probability of the correct puzzle move under:
  - all Maia-1 nets (1100–1900);
  - several Leela nets;
  - Stockfish 17 evals, and whether Stockfish finds the correct move at node limits from 1 to 100,000 (powers of 10).

  It also used hand-crafted features: number of legal moves, checks and captures available, whether the correct move is a check or captures an undefended piece, mate threats, and similar. — [ousou paper](https://annals-csis.org/Volume_43/drp/pdf/6497.pdf)
- Runner-up results on its own Lichess test set:
  - Best single ensemble (Maia-{1100, 1500, 1700} + Leela-small): 38.3k MSE.
  - Ensemble of all its models: 36.1k.
  - Its 2024 architecture with the same embedder scored about 16k (27.7%) worse. The authors credit the new architecture plus the Maia-2 metadata, but ran no separate Maia-2 ablation.

  — [bread emoji 2025](https://annals-csis.org/Volume_43/drp/pdf/6771.pdf)
- Error analysis:
  - Errors are largest on very easy (< 1000) and very hard (> 2400) puzzles.
  - Errors grow monotonically with a puzzle's Glicko rating deviation.
  - Once RD has stabilized, more attempts make little difference (no significant trend from 10–15 attempts upward).

  — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)
- Uncertainty subtask: teams flagged the 10% of puzzles most likely to be mispredicted. "Uncertainty ratios" ranged from 1.43 to 1.71 (1.0 is perfect). The best was transformer_enjoyer at 1.43, a different team from the MSE winner. — [organizers' report](https://annals-csis.org/Volume_43/drp/pdf/5937.pdf)

### Inferences
- The 2025 competition supplied, as a feature, almost exactly what a personalized app needs: **P(a player at rating R in time control T finds this move)**, computed with Maia-2. Running Maia-2 at the user's current rating on every critical position in their games, and on candidate puzzles, gives a principled "expected miss probability". The user's actual results can then be compared against it.
- The best achievable error for difficulty prediction is about ±190–230 Glicko points RMSE: 35–38k MSE on in-distribution Lichess data, 52k on the competition test set. That is good enough to place puzzles in bands, not to pinpoint them. An app should keep its own online rating per puzzle and per user (Glicko-style) and use model predictions only to cold-start new positions taken from the user's games.
- No winning team used RL. All used supervised regression on top of supervised imitation models.

### Gaps
- I could not find the exact 2024 final-leaderboard MSE for the winner (the IEEE organizers' report is paywalled). Only the relative margins and the 49.1k preliminary-set number are confirmed.
- I found no 2026 successor competition in the sources retrieved.

## Q3. How Lichess and Chess.com rate puzzles, and using puzzle ratings as per-user, per-theme skill estimates

### Takeaway
Lichess treats each puzzle attempt as a Glicko-2 "game" between the user and the puzzle. Both ratings update, with weights reduced for themed or "obvious" puzzles. The per-theme Puzzle Dashboard uses a simple performance formula: average puzzle rating − 500 + 1000 × first-try success rate. It flags a theme as "weak" with as few as 3 failures and applies no statistical shrinkage. That makes it a usable heuristic and a reasonable baseline, but it is noisy for small samples.

### Cited Findings
- **Glicko-2 (Glickman):**
  - Each player has a rating, a rating deviation (RD) and a volatility σ.
  - New players start at rating 1500, RD 350, σ 0.06.
  - The 95% interval is rating ± 2·RD.
  - The system constant τ should be between 0.3 and 1.2.
  - The system "works best when … at least 10–15 games per player in a rating period".
  - The paper's example: rating 1850 with RD 50 gives an interval of 1750–1950.

  — [Glickman, Example of the Glicko-2 system (PDF)](http://www.glicko.net/glicko/glicko2.pdf)
- **Lichess puzzle rating, from the lila source code** (`modules/puzzle/src/main/PuzzleFinisher.scala`):
  - Each attempt runs as a Glicko game between the player and the puzzle.
  - Puzzle rating updates are capped at ±700 (`maxRatingDelta`) and rate-limited to 300 puzzle-rating updates per user per day.
  - The player's update is blended by theme:
    - "obvious" themes (all mates, mate-in-1, en passant, double check, castling, attacking f2/f7): weight 0.1 on a win, 0.4 on a loss;
    - other "hinting" themes: 0.2 / 0.7;
    - non-hinting themes (opening, middlegame, endgame types, master games): 0.7 / 0.8;
    - mixed puzzles: 1.0.
  - A provisional puzzle reduces the weight further, with a floor of 0.1.

  — [lila PuzzleFinisher.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleFinisher.scala)
- Lichess Glicko constants (`modules/rating/src/main/Glicko.scala`): default rating 1500, maximum deviation 500, default volatility 0.09, minimum deviation 45, minimum rating 400. — [lila Glicko.scala](https://github.com/lichess-org/lila/blob/master/modules/rating/src/main/Glicko.scala)
- Lichess staff and blog posts describe the system the same way: puzzles use the same Glicko-2 as games, every attempt is a game, new puzzles have volatile ratings, and themed puzzles move the user's rating less. — [Lichess forum quoting the blog](https://lichess.org/forum/general-chess-discussion/puzzle-rating-question)
- **Lichess Puzzle Dashboard per-theme logic** (`PuzzleDashboard.scala`):
  - For each theme over a selectable window (1, 2, 3, 7, 10, 14, 21, 30, 60 or 90 days), it computes the count, wins, wins after a retry ("fixed"), first-try wins and failures.
  - performance = average puzzle rating − 500 + round(1000 × firstWins / count).
  - A theme counts as **weak** if it is more than 1/40 of all puzzles played, has at least 3 failures, and its performance is below the user's global performance. A theme counts as **strong** if it has at least 3 first-try wins and performance above global.
  - At most 8 themes are shown in each list.
  - Result-only themes are excluded: mate-in-N, short/long, equality/advantage/crushing, master.

  — [lila PuzzleDashboard.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleDashboard.scala)
- Chess.com puzzles: I did not retrieve a primary source on Chess.com's puzzle rating algorithm (see Gaps).

### Inferences
- The Lichess dashboard is effectively a crude per-theme, Rasch/Elo-style performance rating. At a 50% first-try success rate, performance equals the average rating of the puzzles faced, and each 10 percentage points of success adds 100 points. It compares the user only against their *own* global performance, not against rating peers, and it uses no shrinkage. A theme with 3 failures out of 4 attempts can be flagged as weak.
- An app can do better by keeping a separate Glicko or Elo rating per (user, theme), or a multidimensional IRT model (Q4). It should show the uncertainty (RD) and only call something a weakness when the interval is clearly below the user's overall level. Lichess's weighting of hinting and obvious themes is a useful detail to copy: telling the solver the theme makes the puzzle easier and should count for less.
- The public Lichess puzzle database exposes Rating, RatingDeviation, NbPlays and Themes per puzzle (these fields appear in the competition data). That gives the app a calibrated, theme-tagged item bank for targeted practice.

### Gaps
- Chess.com's puzzle rating algorithm (it reportedly factors in solve time) was not verified from a primary source in this session.
- Lichess's puzzle tagger logic (how themes are assigned) was not inspected here.

## Q4. Knowledge tracing and skill modeling per theme: Elo-based item response, BKT, DKT, IRT

### Takeaway
The education-modeling literature maps directly onto chess puzzles: student = player, item = puzzle, knowledge component = tactical theme. Elo-style updates are recommended as simple, robust and effective (Pelánek 2016). Tuned shallow models such as Bayesian Knowledge Tracing roughly match Deep Knowledge Tracing in accuracy while being far more interpretable (Khajah et al., EDM 2016 best paper). I found no published application of knowledge tracing specifically to chess puzzle themes.

### Cited Findings
- Pelánek, "Applications of the Elo rating system in adaptive educational systems" (Computers & Education, 2016, DOI 10.1016/j.compedu.2016.03.017) treats a student's answer to an item as a match, estimating student skill and item difficulty together and dynamically. It surveys Elo variants and compares them with alternatives, and concludes that Elo is "simple, robust, and effective" for adaptive systems. — [muni.cz record](https://is.muni.cz/publication/1376317); [author PDF](https://www.fi.muni.cz/~xpelanek/publications/CAE-elo.pdf)
- Follow-up: Pelánek et al., "Elo-based learner modeling for the adaptive practice of facts" (UMUAI 2017) covers prior versus current knowledge estimation with Elo. — [muni.cz record](https://www.muni.cz/en/research/publications/1359654)
- Khajah, Lindsey & Mozer, "How Deep is Knowledge Tracing?" (EDM 2016, best paper):
  - Enhanced BKT "appears to perform as well on average as DKT across the four data sets". By AUC it is better on SYNTHETIC (+0.05) and Spanish (+0.01), and worse on ASSISTments (−0.03) and Statics (−0.01).
  - Replicating the BKT baseline gave AUC 0.73 on ASSISTments, versus 0.67 in the original DKT paper.
  - The authors conclude that knowledge tracing "may be a domain that does not require 'depth'".

  — [arXiv 1604.02416](https://arxiv.org/abs/1604.02416v2); [CU news on best paper](https://www.colorado.edu/ics/2016/08/25/mike-mozers-group-receives-best-paper-edm)
- Khajah et al. 2014, "Integrating Knowledge Tracing and Item Response Theory": IRT captures differences between students and between items but not learning over time, while BKT models learning but assumes all students and problems are identical. Hybrid models combine the two. — [Pitt d-scholarship](https://d-scholarship.pitt.edu/26044/); [PDF](https://home.cs.colorado.edu/~mozer/Research/Selected%20Publications/reprints/KhajahHuangGonzalesBrenesMozerBursilovsky2014.pdf)
- Two papers formally link the families:
  - Deonovic et al. (arXiv 1803.05926): the stationary distribution of BKT relates to an IRT model.
  - arXiv 2309.12334: DKT is "an implicit dynamic multidimensional IRT model".

  — [arXiv 1803.05926](https://arxiv.org/pdf/1803.05926); [arXiv 2309.12334](https://arxiv.org/pdf/2309.12334)
- Deep-IRT (arXiv 1904.11738) keeps DKT-level performance while producing interpretable estimates of knowledge-component difficulty and student ability over time. — [arXiv 1904.11738](https://arxiv.org/pdf/1904.11738)
- Regan and Biswas explicitly connect Elo with Rasch/IRT and treat chess positions as test items with measurable difficulty and discrimination. — [Biswas & Regan 2015](https://www.scitepress.org/papers/2015/52883/52883.pdf)

### Inferences
- A practical skill model for the app:
  - Each attempt (a puzzle, or a critical position from a real game) is an item with known difficulty: the Lichess puzzle rating, or a Maia-2-derived difficulty for game positions.
  - Keep one Elo/Glicko ability per (user, theme), possibly with a shared global ability plus theme offsets, which is a 2-level or multidimensional IRT.
  - Update online after each attempt.

  This is essentially Lichess's puzzle rating, split by theme.
- BKT fits a "mastered / not mastered" view of discrete skills such as "sees back-rank mates". Elo/IRT fits graded difficulty better. Given Khajah et al., DKT is unlikely to justify its data needs and opacity at the scale of a single user.

### Gaps
- I found no paper applying BKT, DKT or IRT to chess puzzle themes or to chess learners' game errors.
- I did not retrieve Pelánek's quantitative comparisons (only the abstract).

## Q5. Where RL fits: self-play RL vs behavior cloning vs KL-regularized search, data scale, and where bandits/RL help

### Takeaway
RL in the AlphaZero/Leela sense optimizes winning, not resemblance to a particular person. It needs tens of millions of self-play games and gives *worse* human-move prediction than supervised imitation: Leela peaks at about 46% move-matching against Maia's about 53%. Human-likeness comes from supervised behavior cloning (Maia), plus fine-tuning for individuals (Maia4All needs about 20 games of the target user, on top of a base pre-trained at population scale). RL or search is layered on top only when a strong *and* human-like opponent is wanted (piKL, Allie). For the three product tasks:
- (1) weakness identification and (2) miss prediction are supervised or statistical problems;
- (3) choosing what to teach next is a sequential decision problem where **multi-armed or contextual bandits**, not full deep RL, are the evidence-backed choice.

### Cited Findings
**Self-play RL vs imitation for human-likeness**
- Move-matching accuracy on Lichess human moves (Maia paper):
  - Stockfish, depths 1–15: 33–41% (best 41.1% at depth 15 on 1900 players), rising monotonically with player rating.
  - Leela: at most 46.0% (Leela "3200" on 1900 players); Leela 2700 is roughly flat at about 40%.
  - Maia: 50.8–52.9% at peak, with a curve that peaks near the rating band it was trained on.

  — [ar5iv 2006.01855](https://ar5iv.labs.arxiv.org/html/2006.01855)
- Jacob et al., "Modeling Strong and Human-Like Gameplay with KL-Regularized Search" (piKL; ICML 2022, PMLR 162:9695–9728):
  - Self-play and search methods such as AlphaZero "may produce policies that differ markedly from human behavior".
  - Imitation learning predicts humans well but plays weaker than experts.
  - The paper's fix is to regularize search with KL divergence from an imitation-learned policy.

  — [PMLR](https://proceedings.mlr.press/v162/jacob22a.html); [arXiv 2112.07544](https://arxiv.org/abs/2112.07544)
- piKL chess results (Table 1, top-1 human-move accuracy, Lichess; about 10M games per rating bucket; 50-playout MCTS):
  - Maia1900 raw: 53.2%. With KL-regularized MCTS: 54.3% at cpuct = 2.
  - Maia1100: 51.1% raw, 51.3% at best.
  - The MCTS versions won more than 85% of games against the imitation-only model.

  — [ar5iv 2112.07544](https://ar5iv.labs.arxiv.org/html/2112.07544)
- Allie, Zhang, Jacob, Lai, Fried & Ippolito, "Human-Aligned Chess With a Bit of Search" (ICLR 2025):
  - Trained on human game logs to model moves, pondering time and resignations.
  - Uses time-adaptive MCTS, searching more where humans would think longer.
  - Against players rated 1000–2600 it kept the average skill gap to 49 Elo, while learning only from human games.

  — [arXiv 2410.03893](https://arxiv.org/abs/2410.03893); [ICLR 2025 poster](https://iclr.cc/virtual/2025/poster/29104)

**Data scale**
- AlphaZero played about 44M self-play chess games in its first 9 hours of training. These are secondary sources; the primary is Silver et al., Science 362(6419), 2018. — [chess.com AlphaZero](https://chess.com/terms/alphazero-chess-engine); [chess24 via Wayback](https://chess24.com/en/read/news/alphazero-really-is-that-good)
- Personalization data needs:
  - Individual Maia fine-tuning (McIlroy-Young et al., "Learning Models of Individual Behavior in Chess", KDD 2022) improved the target player's move-matching by about 4 percentage points in the earlier arXiv version and by "10 percentage points on average" in the later version (the versions differ). The models can identify a player from their moves alone. A follow-up describes this approach as needing about 5,000 games per player.
  - Maia4All (Tang et al., "Learning to Imitate with Less", arXiv 2507.21488; TMLR 2026) personalizes from **20 games**: move-matching rises from 51.4% to 53.2%. It first "enriches" the model with prototype players, using about 2,500 games per prototype. Its baselines did poorly: Maia-2 shows "virtually no progress" with 20 games, and Maia fine-tuned on 1,000 games got worse than the base model.

  — [arXiv 2008.10086](https://arxiv.org/pdf/2008.10086); [arXiv 2507.21488](https://arxiv.org/html/2507.21488v1); [mlanthology TMLR 2026](https://mlanthology.org/tmlr/2026/tang2026tmlr-learning)
- A 2026 follow-up, "Toward Modeling Player-Specific Chess Behaviors" (arXiv 2605.11893), argues that move-accuracy metrics miss the natural variability in human decisions. — [arXiv 2605.11893](https://arxiv.org/pdf/2605.11893)

**Where bandits and RL do help: choosing what to teach next**
- Clement, Roy, Oudeyer & Lopes, "Multi-Armed Bandits for Intelligent Tutoring Systems" (ZPDES and RiARiT; arXiv 1310.3174; 2015):
  - The bandit reward is *learning progress* (recent success versus earlier success), and only activities whose prerequisites are mastered can be chosen.
  - User study: 400 children aged 7–8 in 11 schools. Compared with an expert-designed sequence, ZPDES students reached higher maximum levels on 3 of 4 exercise types; for example, 95% of ZPDES students succeeded at least once on R exercises, versus 75% with the expert sequence.
  - Simulations: 1,000 virtual students per population. The bandits beat the expert sequence when some activities were unsolvable for some students.
  - Why bandits rather than POMDP/RL planning: identifying a student model's many parameters for *one* student is hard (limited data, intractability, identifiability), and planning usually optimizes for a population rather than the individual.

  — [ar5iv 1310.3174](https://ar5iv.labs.arxiv.org/html/1310.3174); [arXiv 2408.07208 (hierarchical MAB follow-up)](https://arxiv.org/pdf/2408.07208)
- Doroudi, Aleven & Brunskill, "Where's the Reward? A Review of Reinforcement Learning for Instructional Sequencing" (IJAIED 29(4):568–620, 2019):
  - Of the empirical studies comparing RL-induced teaching policies with baselines, *just over half* found RL significantly better.
  - RL worked best when constrained by cognitive psychology and learning-science theory, and when backed by robust offline analyses that do not depend on one particular student model.

  — [Springer DOI 10.1007/s40593-019-00187-x](https://dx.doi.org/10.1007/s40593-019-00187-x)

### Inferences
- **The developer does not need to train an RL network on the user's games.** Three reasons:
  1. RL's objective (maximize wins) is the wrong objective for modeling one person.
  2. RL needs millions of games, while one user has hundreds to low thousands.
  3. Every evidence-backed way to capture a person's tendencies is supervised (behavior cloning, then fine-tuning) or statistical.

  Inverse RL is a theoretical option, but I found no chess-personalization work that uses it.
- The recommended stack:
  - **Analysis layer:** an engine (Stockfish) labels errors as a ≥10-point drop in win probability, plus motif tagging.
  - **Difficulty layer:** pretrained Maia-2 or Maia-1 gives the probability that a player of the user's rating finds the move. No training is needed; optional per-user fine-tuning à la Maia4All is a later enhancement.
  - **Skill layer:** per-theme Elo/Glicko or Beta-Binomial with peer shrinkage (Q4, Q6).
  - **Curriculum layer:** a bandit (Thompson sampling, or a ZPDES-style learning-progress bandit) chooses among themes where the user is weak relative to peers, with a reward such as improvement in theme rating or puzzle success near the user's current level.

  This is "RL" only in the narrow contextual-bandit sense.
- piKL- or Allie-style search on top of human models is relevant only if the product also wants a sparring bot that plays like the user or like a target rating. It is not needed to diagnose weaknesses.

### Gaps
- I found no published randomized study of bandit-driven puzzle selection in a chess app.
- I found no chess-specific inverse RL or RLHF personalization work.
- The exact AlphaZero game count comes from secondary sources; the Science paper was not fetched.

## Q6. Simple statistical baselines (per-motif miss rates with Beta-Binomial shrinkage against rating peers) and product examples

### Takeaway
Statistically, a "personalized weakness" is a motif where the user's miss rate is credibly worse than that of *peers at the same rating*, after shrinking small-sample rates toward the peer average. Commercial products come close to this: Chess.com Insights tracks found versus missed forks, pins, mates and hanging pieces and has a "Similar Players" rating-peer comparison, and Aimchess reportedly benchmarks six skill categories against similar-rated players. Lichess compares a theme only against the user's own overall performance, with no shrinkage.

### Cited Findings
- **Beta-Binomial empirical Bayes:**
  - Each unit's successes x out of n trials are binomial, with the rate drawn from a Beta(α, β) prior fitted to the population by method of moments or maximum likelihood.
  - Posterior mean: (x + α̂)/(n + α̂ + β̂).
  - The shrinkage weight toward the population mean is κ/(κ + n), with κ = α + β, so small samples are pulled harder toward the mean.

  This is standard theory; the source is a tutorial-grade documentation page, not a primary paper. — [metricgate empirical Bayes docs](https://metricgate.com/docs/empirical-bayes/)
- Historical anchor: in Efron & Morris (1975), the shrinkage estimates of early-season baseball batting averages had pooled error typically 50–80% smaller than the raw averages when predicting full-season values. This is reported secondhand by the same documentation source; the 1975 paper was not fetched. — [metricgate hierarchical beta-binomial](https://metricgate.com/docs/hierarchical-beta-binomial-baseball/)
- **Chess.com Insights:**
  - A Tactics section shows found versus missed forks, pins and mates, plus pieces the user left hanging and pieces the opponent left hanging, scanned from games where the tactic was the engine's top move.
  - The user can open and retry the positions.
  - A "Similar Players" option compares stats with players of similar rating, and "Past Performance" compares with the user's own history. The rating comparison is explicitly documented for accuracy by move number. I could not confirm whether it covers the tactics counts.
  - Insights was described as a Diamond-membership feature.

  — [Chess.com support: What is Insights](https://support.chess.com/en/articles/8708925-what-is-insights); [Chess.com news: New Insights](https://www.chess.com/news/view/new-insights?page=3)
- **Aimchess:**
  - Six categories: tactics, endgame, advantage capitalization, resourcefulness, time management, opening.
  - Imports games from Chess.com, Lichess and Chess24, and generates training from the user's mistakes.
  - Reportedly compares the user with players of similar rating. One user describes the comparison as against their *opponents*, so the benchmark group is unclear.
  - Pricing per a directory listing: free tier with one 40-game report per month; Premium $7.99/month.

  All Aimchess claims come from third-party listings and blogs, not official documentation. — [stork.ai listing](https://www.stork.ai/en/aimchess); [eliteai.tools listing](https://eliteai.tools/tool/aimchess); [user blog](https://edusites.uregina.ca/kaelinkrukoff/?p=94); [Chess.com blog example](https://chess.com/blog/PixelatedParcel/free-insight-into-your-games)
- **Lichess Insights** (lila source):
  - Users can cross-tabulate metrics against dimensions.
  - Metrics: average centipawn loss, accuracy, move time, "Tactical awareness" (how often you take advantage of your opponent's mistakes), "Luck" (how often your opponent fails to punish your mistakes), material imbalance, "Time pressure" (clock percent remaining), time variance, blurs, and others.
  - Dimensions: game phase, piece moved, opening family/variation, opponent strength, move-time range, clock-percent range, evaluation, winning chances, and others.
  - The source files show no rating-peer comparison.

  — [lila InsightMetric.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightMetric.scala); [lila InsightDimension.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightDimension.scala)
- Lichess Puzzle Dashboard weak-theme rule: at least 3 failures and performance below the user's own global performance. — [lila PuzzleDashboard.scala](https://github.com/lichess-org/lila/blob/master/modules/puzzle/src/main/PuzzleDashboard.scala)

### Inferences
- A concrete baseline recipe:
  1. Run Stockfish over the user's games.
  2. At each "opportunity" (a position where a motif-tagged winning or defensive move existed, or where the user faced a threat), record found or missed.
  3. Build the peer prior per (motif × rating band × time control) from a large Lichess sample, using the same pipeline, and fit Beta(α, β).
  4. The user's shrunken miss rate is (misses + α)/(opportunities + α + β). Report the posterior probability that it exceeds the peer mean, for example P > 0.9.

  The Lichess open database makes step 3 possible without proprietary data.
- Adjust for difficulty. Peers at the same rating may have met harder versions of the same motif, so a stronger version replaces the raw miss count with "observed misses minus expected misses". Expected misses come from Maia-2's success probability at the user's rating, giving a per-motif residual (observed − expected) / √variance. This merges Q1 (difficulty dominates) with Q6 (personal deviation).
- The products differ in exactly this respect: Chess.com shows peer-relative stats, Lichess shows self-relative ones, and none of the sources describe statistical shrinkage. A shrunken, difficulty-adjusted, peer-relative weakness score is therefore a plausible differentiator, though that is an inference about the market.

### Gaps
- None of the products publish their weakness algorithms. Aimchess details are unverified (no official documentation retrieved).
- I found no peer-reviewed paper applying Beta-Binomial shrinkage to chess motif miss rates.

## Q7. Time pressure and clock usage as error predictors

### Takeaway
Clock time matters mostly at the extremes. In 3-minute FICS endgames the blunder rate is roughly flat above about 10 seconds remaining and rises sharply below it. Adding clock and rating metadata to a board-only blunder model raises accuracy by about 4 points (67.7% to 71.7%). A causal regression-discontinuity study on Lichess, using the low-time alarm, also finds that less time raises blunder probability, but no published numbers were found. Lichess already exposes clock percent as a "Time pressure" insight dimension.

### Cited Findings
- **Anderson et al. (FICS, mostly 3-minute games):**
  - The blunder rate rises sharply as remaining time approaches 0 and is nearly flat above about 10 seconds.
  - Extra time helps higher-rated players more.
  - 1200-rated players with 5–8 seconds left at β = 0.4 blunder less than 1800-rated players with 42–58 seconds at β = 0.8.
  - Clock features alone give 0.53 balanced accuracy, versus 0.73 for difficulty.

  — [ar5iv 1606.04956](https://ar5iv.labs.arxiv.org/html/1606.04956)
- Maia's blunder model uses each player's percentage of clock remaining as metadata. Board + metadata (ratings, clock %, eval) reaches 71.7%, against 67.7% for board only, with a residual CNN. Training excluded moves made with under 30 seconds left. — [ar5iv 2006.01855](https://ar5iv.labs.arxiv.org/html/2006.01855)
- Jiang Ming (SJTU), "Decision Quality Under Time Pressure: Evidence from a Large Chess Database" (seminar abstract, 18 Nov 2021):
  - Data: millions of time-stamped Lichess games, with move quality graded by an engine.
  - Method: a regression discontinuity at Lichess's audio-visual low-time alarm.
  - Findings: blunder probability rises as available time falls, even when players spend more time per move. Higher-rated and titled players err less but are not immune.
  - The abstract gives no effect sizes.

  — [SWUFE seminar page](https://ccbef.swufe.edu.cn/info/1251/1451.htm)
- Allie models pondering time and allocates more search where humans think longer. This shows that time-use patterns can be learned per position from game logs. — [arXiv 2410.03893](https://arxiv.org/abs/2410.03893)
- Lichess Insights stores a per-move `clockPercent` and exposes "Time pressure" and "Move time" both as metrics and as dimensions. — [lila InsightMetric.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/InsightMetric.scala); [lila model.scala](https://github.com/lichess-org/lila/blob/master/modules/insight/src/main/model.scala)
- Lower-quality sources:
  - A Chess Engine Lab blog analysis of over-the-board games reports that blunders rise when players are low on time while "inaccuracies" fall, possibly because evaluations become more extreme in endgames.
  - A Lichess forum analysis of 68M games claimed a blunder spike at the low-time alarm. Commenters noted it was observational.

  — [Chess Engine Lab substack](https://chessenginelab.substack.com/p/mistakes-clock); [Lichess forum](https://lichess.org/forum/redirect/post/1SL2qHTI)
- A CNN-LSTM paper estimates player rating from moves *and clock times* (arXiv 2409.11506). It was not reviewed in detail. — [arXiv 2409.11506](https://www.arxiv.org/pdf/2409.11506)

### Inferences
- For the app, clock features should enter mainly as a **conditioning variable that separates "didn't know the motif" from "ran out of time"**. Examples: clock % remaining, seconds remaining (with a ≤10 s flag), time spent on the move relative to the user's median, and increment. A motif missed with 5 seconds left is weak evidence of a knowledge gap. A motif missed with plenty of time after a long think is strong evidence. Time-management weakness is its own lesson category, as Aimchess and Lichess Insights already treat it.
- Because people think longer in hard positions, raw "time spent" correlates *positively* with blunders. It should not be read causally without controlling for difficulty.

### Gaps
- I found no peer-reviewed effect sizes for time-pressure blunder rates on Lichess beyond Anderson et al.'s FICS endgame data. The Jiang Ming study's published version, if any, was not located.
- I did not verify the Lichess database documentation on `%clk` PGN annotations in this session. The Maia paper confirms only that about 10% of Lichess games carry Stockfish evaluations, so an app must run its own engine analysis on most of a user's games.
