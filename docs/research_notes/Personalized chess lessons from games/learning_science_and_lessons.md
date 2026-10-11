# Learning science and adaptive-learning algorithms for turning a player's detected weaknesses into personalized chess lessons and practice

Scope: assumes another component has already found weaknesses such as "misses knight forks" or "mishandles rook endgames." These notes cover how to generate, sequence, schedule and evaluate lessons and drills so the player actually improves. Research done 2026-10-09. Dates are given wherever a fact can go out of date.

---

## 1. Chess expertise research: deliberate practice, chunking/templates, and whether tactics training transfers

### Takeaway
Two things predict chess skill best: accumulated *serious study alone* (strongest single predictor in Charness et al. 2005) and a very large store of perceptual patterns (chunks/templates). But practice explains only about a third of the variance, and no rigorous study shows that puzzle solving transfers to game results. A lesson engine should therefore aim at pattern acquisition across many varied positions, and it should measure transfer to real games directly rather than assume it.

### Cited Findings
- **Charness, Tuffiash, Krampe, Reingold & Vasyukova (2005), *Applied Cognitive Psychology* 19:151–165.** Two large samples of rated players. Serious study alone, tournament play and formal instruction were all significant bivariate correlates of rating. In multivariate regression, "serious study alone was the strongest predictor of chess skill in both samples," and the combined chess activities explained about 40% of rating variance. The roles of tournament play and instruction varied with age (above vs. below 40) and with peak vs. current rating. Grandmasters logged about 5,000 hours of serious solo study in their first decade, roughly 5x the figure for intermediate players. — [Scinapse abstract](https://scinapse.io/papers/2134432271); [MSU istina record](https://istina.msu.ru/publications/article/2590471/)
- **Burgoyne et al. (2019), *American Journal of Psychology*, reanalysis (Ns = 239 and 180).** Total hours of serious study predicted current and peak rating better than formal instruction or tournament play. Latent "serious chess activity," formal instruction and serious starting age together explained 63% of the variance in peak rating. Starting age had a negative effect (β = −.15). The claim that formal instruction makes serious study more effective was *not* supported. — [Burgoyne et al. 2019 PDF](https://englelab.gatech.edu/articles/2019/Burgoyne_et_al_2019_AJP.pdf)
- **Hambrick et al. (2014) and Macnamara et al. (2014).** Assuming reliability of 0.80, deliberate practice explained about 34% of the variance in chess performance (30% in music), so most variance comes from other factors. Across domains, deliberate practice explained 26% of the variance for games, 21% for music, 18% for sports, 4% for education and under 1% for professions. — [Hambrick et al., "Facing facts about deliberate practice" (PMC)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4101876/); [Macnamara 2014 summary](https://stafforini.com/works/macnamara-2014-deliberate-practice-and/)
- **Counterpoint, a 2021 study of 178 chess players.** Earlier meta-analyses counted practice that was not individualized. When practice was highly individualized and of high quality, the effect size of structured practice was more than 3x the effect at the average level. This bears directly on a personalized-lesson product. — [Current Psychology 2021, s12144-021-02326-x](https://link.springer.com/article/10.1007/s12144-021-02326-x)
- **Campitelli & Gobet (2007/2008), Argentine sample.** Masters started serious practice earlier than experts and were rated higher after only 3 years of serious play, with no difference in hours practised. Cumulative hours varied widely: non-rated players about 8,303, rated untitled about 11,715, FMs about 19,618, IMs about 27,929. — [Murdoch research portal](https://researchportal.murdoch.edu.au/esploro/outputs/journalArticle/The-role-of-practice-in-chess/991005542731507891); [ChessBase summary](https://en.chessbase.com/post/handedne-practice-and-talent-in-che)
- **Chunking theory (Chase & Simon 1973), with parameters as summarized by Gobet (1998, *Cognition* 66:115–152).** Expertise is a large database of chunks indexed by a discrimination net. Model parameters: STM capacity about 7 chunks, about 10 ms per discrimination test, about 8 s to learn a new chunk (about 8 s to create a node, about 1 s to add information to an existing node). Grandmasters were estimated to hold 10,000–100,000 configurations (50,000 is the figure usually quoted), taking at least 10 years to acquire. Chunks act as the conditions of productions: a recognized pattern triggers candidate moves, which explains expert "intuition." — [Gobet 1998, "Expert memory: a comparison of four theories" (Brunel PDF; note: this repository URL is labelled in some indexes as Gobet & Jansen, but the file is Gobet 1998)](https://bura.brunel.ac.uk/bitstream/2438/815/1/FullText.pdf)
- **Template theory (Gobet & Simon 1996).** In positions a player has studied or played extensively, chunks grow into *templates*: a large core chunk plus slots for variable features (piece locations, candidate moves, plans, tactical/strategic features). Templates are specific to *classes of positions*. A slot is created when the same kind of information is tested in several branches of the net. Chunks and templates can also be reached through verbal descriptions of tactical/strategic features, not only visually. In one experiment a master recalled 9 boards at >70% accuracy (up to 160 pieces). — [Gobet 1998 PDF](https://bura.brunel.ac.uk/bitstream/2438/815/1/FullText.pdf); [Gobet & Simon 1996 PDF](https://www.Gwern.net/doc/psychology/chess/1996-gobet.pdf); [Brunel record](https://bura.brunel.ac.uk/handle/2438/1339)
- **Gobet & Clarkson (2004).** Masters replaced chunks of up to 15 pieces. The original chunking theory overestimated STM capacity and underestimated chunk size. — [Brunel record 2438/1345](https://bura.brunel.ac.uk/handle/2438/1345)
- **Einstellung effect (Bilalić, McLeod & Gobet 2008; Sheridan & Reingold 2013).** A familiar first idea pulls attention toward information consistent with it and keeps biasing attention even when players believe they are looking for alternatives. Experts who found the better move gradually disengaged from the familiar move. When the familiar move was a blunder, all experts and most novices avoided it. — [Sheridan & Reingold 2013, PLoS ONE (PMC)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3790829/); [Bilalić et al. 2010 PDF](https://cognition.aau.at/download/Publikationen/Bilalic/Bilalic_etal_2010b.pdf)
- **Puzzle-to-game transfer.** I found no randomized or longitudinal study showing that tactics-puzzle training raises over-the-board or online game rating. The available evidence is correlational: an Indonesian conference paper on Lichess users linked usage frequency and duration with tactical-skill scores. The "transfer problem" (puzzle rating far above game rating, because a puzzle announces that a tactic exists) is widely discussed but has not been measured rigorously. — [UNESA conference paper](https://proceeding.unesa.ac.id/index.php/iconbit/article/download/5316/1489/22655); [Chessiro blog, July 2026 (commercial)](https://chessiro.com/blog/puzzles-from-your-own-games)

### Inferences
- **"Misses knight forks" is a missing or weakly indexed chunk, or a missing production (pattern → candidate move).** The cure is many exposures to varied instances of the pattern in realistic contexts, so the discrimination net learns which perceptual features cue it. Template theory says the instances should share a position class (for example, "knight fork on king + rook after a pawn exchange in the center") but vary in the slots.
- **Einstellung explains many "overlooked" tactics.** The player fixates on a first plan and never looks at the fork. Drills should include positions where a plausible "natural" move competes with the tactic, and positions with no tactic at all, so the player practises detection and disengagement, not just execution.
- **Give serious-study time first priority.** The product is essentially structured serious study. Personalization is the lever the 2021 individualization study suggests can multiply effect size, but that is a hypothesis to test, not an established result.
- **Treat transfer as an empirical question.** Section 6 metrics should track the miss rate in real games, not only puzzle accuracy.

### Gaps
- I found no controlled study of tactics-puzzle training measured against later game performance. The Campitelli & Gobet range of hours-to-master (variously reported as 728–16,120 or a 3,000-hour minimum) could not be checked against the primary paper.
- I could not retrieve Gobet & Jansen (2005), "Training in chess: a scientific approach," which gives template-theory-based teaching recommendations. The Brunel URL indexed for it returned Gobet (1998).

---

## 2. Spaced repetition algorithms (SM-2, FSRS) and how chess tools apply them

### Takeaway
FSRS-6 (21 parameters, trainable forgetting-curve decay) is the current open-source state of the art among interpretable schedulers. In the 10k-user Anki benchmark it beats FSRS-5, FSRS-4.5, HLR and Ebisu. Only neural sequence models (GRU/LSTM) predict better. Chess tools mostly use simple Leitner-style ladders (Chessable MoveTrainer: 4h → 1d → 3d → 1w → 2w → 1m → 3m → 6m). Lichess's "Learn from your mistakes" does not schedule at all. A personalized-mistakes product can use FSRS per *card* (position) and add a per-*theme* skill model on top (section 4).

### Cited Findings
- **FSRS versions (from the open-spaced-repetition wiki, last edited 2026).**
  - FSRS v4: R = (1 + t/(9S))^−1.
  - FSRS-4.5: DECAY = −0.5, FACTOR = 19/81, interval I(r,S) = (S/FACTOR)·(r^(1/DECAY) − 1), so I = S when r = 0.9.
  - FSRS-5 (19 params): adds same-day review stability S′ = S·e^(w17·(G−3+w18)); initial difficulty D0(G) = w4 − e^(w5·(G−1)) + 1; linear damping ΔD = −w6·(G−3), D′ = D + ΔD·(10−D)/9; mean reversion D″ = w7·D0(4) + (1−w7)·D′ (target D0(4), earlier versions used D0(3)).
  - FSRS-6 (21 params): same-day stability gains an S^(−w19) factor, so small S grows faster and large S slower. The forgetting curve becomes R(t,S) = (1 + factor·t/S)^(−w20) with factor = 0.9^(−1/w20) − 1, so R(S,S) = 90% and the decay is trainable via w20.
  - FSRS-6 default weights: [0.212, 1.2931, 2.3065, 8.2956, 6.4133, 0.8334, 3.0194, 0.001, 1.8722, 0.1666, 0.796, 1.4835, 0.0614, 0.2629, 1.6483, 0.6014, 1.8729, 0.5425, 0.0912, 0.0658, 0.1542]. w0–w3 are initial stabilities in days for Again/Hard/Good/Easy (S0(G) = w_(G−1), stated in the v4 section).
  - Post-lapse stability (v4 section): S′_f = w11·D^(−w12)·((S+1)^(w13) − 1)·e^(w14·(1−R)). Recall stability includes a hard-penalty multiplier w15 (G = 2) and an easy-bonus multiplier w16 (G = 4).
  — [awesome-fsrs wiki, "The Algorithm"](https://github.com/open-spaced-repetition/awesome-fsrs/wiki/The-Algorithm) (moved from [fsrs4anki wiki](https://github.com/open-spaced-repetition/fsrs4anki/wiki/The-Algorithm))
- **SRS benchmark (anki-revlogs-10k: about 727M reviews, 9,999 collections, about 350M reviews evaluated without same-day reviews; mean ± 99% CI).**

  | Model | Params | Log loss | RMSE(bins) | AUC |
  |---|---|---|---|---|
  | FSRS-6 | 21 | 0.3460 | 0.0653 | 0.7034 |
  | FSRS-rs (FSRS-6) | 21 | 0.3443 | 0.0635 | 0.7074 |
  | FSRS-5 | 19 | 0.3561 | 0.0742 | 0.7010 |
  | FSRS-4.5 | 17 | 0.3625 | 0.0764 | 0.6891 |
  | DASH | 9 | 0.3682 | 0.0838 | 0.6311 |
  | HLR | 3 | 0.4694 | 0.1275 | 0.6369 |
  | Ebisu v2 | — | 0.4989 | 0.1627 | 0.6051 |
  | GRU | 503 | 0.3328 | 0.0549 | 0.7324 |
  | LSTM | 8,869 | 0.3332 | 0.0538 | 0.7329 |
  | Constant-average baseline | — | 0.3945 | 0.1034 | 0.4997 |

  SM-2 does not appear in the text tables (the comparison is in images). — [srs-benchmark README](https://github.com/open-spaced-repetition/srs-benchmark)
- **py-fsrs (official Python package).** 21 weights; `desired_retention` default 0.9; learning steps (1 min, 10 min); relearning step 10 min; `maximum_interval` 36,500 days; fuzzing on by default; ratings Again = 1, Hard = 2, Good = 3, Easy = 4; card states Learning/Review/Relearning. `scheduler.get_card_retrievability(card)` returns P(recall). An optimizer (`fsrs[optimizer]`) fits weights from ReviewLogs (`compute_optimal_parameters`, `compute_optimal_retention`). fsrs-rs is the Rust implementation used by Anki. — [py-fsrs GitHub](https://github.com/open-spaced-repetition/py-fsrs)
- **Spacing-gap evidence (Cepeda, Vul, Rohrer, Wixted & Pashler 2008, *Psychological Science*, n > 1,350).** The optimal gap between study sessions, as a share of the test delay, fell from about 20–40% for a 1-week delay to about 5–10% for a 1-year delay. The absolute optimal gap still grows with the delay. — [Cepeda et al. 2008 PDF](https://www.evullab.org/pdf/CepedaVulRohrerWixtedPashler-PS-2008.pdf); [USF record](https://digitalcommons.usf.edu/psy_facpub/1766/)
- **Chessable MoveTrainer (now documented on Chess.com support).** Approximate levels: L1 4 h, L2 1 d, L3 3 d, L4 1 w, L5 2 w, L6 1 mo, L7 3 mo, L8 6 mo. Each *move* in a variation has its own timer, and a mistake resets that move to Level 1. — [Chess.com support: spaced repetition scheduling](https://support.chess.com/en/articles/10319322-how-does-the-spaced-repetition-scheduling-work); [Andy Matuschak notes on MoveTrainer](https://notes.andymatuschak.org/zDr94hP6bG3jJYrdYy8B5hx); [FIDE article "From flashcards to Chessable"](https://www.fide.com/from-flashcards-to-chessable/)
- **Listudy.** Each repertoire move is a card. A correct move raises the card's value (seen less often) and a wrong move resets it to zero. The approach is Leitner-like and aimed at openings. Listudy itself says spaced repetition helps with memorizing openings, while theory is better learned in other ways. — [Listudy blog](https://listudy.org/en/blog/spaced-repetition-for-chess)
- **Chessbook.** An opening-repertoire app with spaced repetition. It checks the user's online games for deviations from prep (Lichess and Chess.com), and the free tier is capped at 400 moves. A Lichess-blog review says review and scoring use Lichess data for the user's level. "Chess Lines" (a browser extension) uses SM-2. — [App Store listing](https://apps.apple.com/us/app/-/id6466343415); [Lichess blog review](https://en.lichess.org/ublog/B3xp08nE/redirect); [Chess Lines](https://chesslines.carrd.co/)
- **Lichess "Learn from your mistakes" (2017).** After computer analysis, it walks the user through each mistake and asks for a better move before revealing the engine's best. It is a one-shot review tool with no scheduling. Users have asked for spaced repetition on top of it. Lichess's puzzle history lets users re-attempt failed puzzles, and the puzzle dashboard shows accuracy and puzzles solved per theme. — [Lichess blog](https://lichess.org/blog/WFvLpiQAACMA8e9D/learn-from-your-mistakes); [Lichess forum request](https://lichess.org/forum/lichess-feedback/love-learn-from-your-mistakes-can-it-be-implemented-to-use-all-my-games); [lila puzzles feature docs (third-party generated docs)](https://mintlify.wiki/lichess-org/lila/features/puzzles)
- **Woodpecker Method (Smith & Tikkanen, Quality Chess, 2018).** Solve the same set of about 1,100+ puzzles (1,128 per one source) in repeated cycles, faster each time. Tikkanen's original schedule ended with the whole set in a single day. Commentators suggest redoing the set about every 6 weeks or before tournaments. I found no peer-reviewed evaluation. — [New in Chess listing](https://www.newinchess.com/the-woodpecker-method-tikkanen); [ChessTempo book page](https://chesstempo.com/chess-books/the-woodpecker-method/book/183); [Chess.com blog](https://chess.com/blog/AdityaPikach/improve-your-chess-with-the-woodpecker-method); [Lichess blog "Woodpecker revisited"](https://lichess.org/@/CheckRaiseMate/blog/the-woodpecker-method-revisited/ClV6CvGE)

### Inferences
- **Card design.**
  - Card = (FEN, side to move, correct move or line, theme tags, source = own game / database / generated variation).
  - Grade from behaviour rather than self-rating:
    - wrong first move → Again
    - correct after a hint, or very slow → Hard
    - correct at normal speed → Good
    - correct and fast (for example, under a theme- and rating-specific time threshold) → Easy
  - Map these onto FSRS ratings 1–4.
- **Use FSRS-6 via py-fsrs or ts-fsrs (or fsrs-rs).** Start with default weights. Refit per user once they have enough review logs (the py-fsrs README gives no minimum count). Until then, consider fitting one population-level weight set on all users' logs.
- **Desired retention.** For tactical-pattern cards, 0.85–0.90 is a reasonable start (the py-fsrs default is 0.9). `compute_optimal_retention` can tune it against workload. This is an inference; no chess-specific evidence was found.
- **Use FSRS only for the exact-card memory layer.** Exact-position recall is not the goal (see section 3). FSRS should schedule *re-exposure*. The *selection* of new variant positions per theme belongs to the skill model in section 4.
- **Woodpecker-style massed cycling is the opposite of expanding spacing** and risks memorizing answers rather than patterns. Better: spaced plus varied instances, with occasional speed cycles on a theme set as a fluency (automaticity) drill.
- **Per-move timers (as in Chessable) map naturally onto multi-move tactics.** Track the key move (for example, the forking move) separately from the forced follow-up captures.

### Gaps
- No published, peer-reviewed evaluation was found of spaced repetition specifically for chess tactics or for a player's own mistakes. The evidence comes from vocabulary/fact learning (the Anki datasets).
- I could not retrieve FSRS-6's exact full recall-stability formula from the wiki in this session. Use the open-source implementations as the reference.
- No release dates for FSRS-6 or the current py-fsrs/ts-fsrs versions were obtained. The benchmark's last-update date was not shown. Whether an "FSRS-7" exists as of October 2026 was not confirmed: the wiki lists FSRS-6 as latest.

---

## 3. The user's own mistake positions vs. similar positions: interleaving, desirable difficulties, testing effect, generalization

### Takeaway
Use **both**. The user's own position is a highly motivating, retrieval-practice anchor. But retrieving one exact position trains recall of that position, not the pattern. The research on testing (g ≈ 0.50–0.61, larger with feedback), case comparison (d ≈ 0.50) and interleaving (g = 0.42, strongest for visually similar categories) all point to the same design:
1. Retrieval practice on the own-game position.
2. Then side-by-side comparison with structurally similar variants.
3. Then interleaved, mixed-theme retrieval later, including look-alike "no-tactic" foils.

### Cited Findings
- **Testing effect meta-analyses.** Phelps (2012), Rowland (2014) and Adesope et al. (2017) report average testing effects of g ≈ 0.50–0.61. Rowland: g = 0.50 vs. restudy (159 effect sizes, 61 studies). Adesope: g = 0.61 vs. non-testing (272 effects, 118 articles, about 15k participants). With feedback, the testing effect roughly doubles (Rowland: g = 0.73 with feedback vs. 0.39 without). The meta-analyses disagree on multiple-choice vs. short-answer formats. — [Frontiers 2018 summary](https://frontiersin.org/articles/10.3389/fpsyg.2018.02412/full); [Rowland 2014 PDF](https://courseware.epfl.ch/assets/courseware/v1/fdde2f0aa590bf3b1324077a6bf1540c/asset-v1%3AEPFL%2BDEMO%2B2020%2Btype%40asset%2Bblock/Rowland2014-meta-analysis.pdf)
- **Interleaving meta-analysis (Brunmair & Richter 2019, *Psychological Bulletin*, "Similarity matters").** 59 studies, 238 effect sizes, overall g = 0.42. Largest for paintings and visual materials (g = 0.67). Smaller and mixed for mathematics. Interleaving helps most when categories are similar and hard to discriminate. — [Brunmair & Richter 2019 PDF](https://www.psychologie.uni-wuerzburg.de/fileadmin/06020400/2019/Brunmair_Richter_in_press__2019_META-ANALYSIS_OF_INTERLEAVED_LEARNING.pdf); [Frontiers corrigendum note](https://frontiersin.org/articles/10.3389/fpsyg.2019.02296/full)
- **Kornell & Bjork (2008).** Interleaving paintings by 12 artists improved classification of *new* paintings compared with blocked study. Learners nonetheless believed blocking had worked better, a gap between perceived and actual learning that is central to "desirable difficulties." The proposed mechanism is discriminative contrast. Kang & Pashler (2012) found interleaving helped induction while pure temporal spacing did not. — [Memory & Cognition 2013 "Why interleaving enhances inductive learning"](https://www.springermedizin.de/article/10.3758/s13421-012-0272-7); [Yan, Bjork & Bjork 2016 PDF](https://sites.lifesci.ucla.edu/psych-bjorklab/wp-content/uploads/sites/13/2016/11/YanBjorkBjork2016.pdf)
- **Case comparison (Alfieri, Nokes-Malach & Schunn 2013, *Educational Psychologist* 48(2):87–113).** 57 experiments, 336 tests. Comparing cases beat sequential or single-case study, with a moderate effect (d ≈ 0.50; one summary says 0.60, so the value is unresolved). Reported moderators: stating the principle *after* the comparison works better than before or not at all; emphasizing similarities beats emphasizing both similarities and differences; perceptual content benefits most. The moderator details come from secondary summaries. — [Alfieri et al. 2013 PDF](https://www.lrdc.pitt.edu/Schunn/papers/ContrastingCasesMeta-AlfieriEtAl2013.pdf)
- **Template theory implies generalization through variation.** Templates form for *classes of positions*, and slots are created when the same kind of information varies across branches. — [Gobet 1998 PDF](https://bura.brunel.ac.uk/bitstream/2438/815/1/FullText.pdf)
- **Gobet & Simon (1996, *Memory & Cognition*).** Recall of mirror-reflected positions was impaired, so chunks encode specific pieces on specific *locations*. Pattern knowledge is partly location-bound, and transfer to shifted or mirrored versions is not automatic. — [Brunel record](https://bura.brunel.ac.uk/handle/2438/1339); [core.ac.uk](https://core.ac.uk/works/344255)

### Inferences
- **Recommended per-weakness item mix.**
  - (a) The **exact own-game position**: high relevance, tests "would I find it now?"
  - (b) **Near-transfer variants**: same motif and piece configuration, shifted, mirrored, or with different surrounding pieces. The mirror-recall finding suggests these must be trained explicitly.
  - (c) **Far-transfer database puzzles** with the same theme tag from different openings and phases.
  - (d) **Foils**: look-alike positions where the fork fails (defended square, intermezzo) or no tactic exists. These train discrimination and resist Einstellung/"puzzle-mode" cueing.
- **Order: block briefly at acquisition, then interleave.** Show (a) and 2–3 variants side by side (case comparison), state the principle after the comparison, then switch to interleaved practice mixing the weakness theme with other themes and foils. Most later reviews should be variants, not the original. Repeated exact retrieval mostly measures memory of that card.
- **Don't tell the user the theme during interleaved review.** Hiding the label makes it a detection test; themed sessions act as a hint. Lichess, for instance, awards fewer rating points for themed puzzles for exactly this reason (see the section 4 source).
- **Expect users to prefer blocked practice.** Kornell & Bjork's metacognitive illusion means satisfaction surveys will favour the less effective design. Measure outcomes, not preferences.

### Gaps
- I found no chess-specific study comparing the user's own positions with generic puzzles. Personal-relevance or self-generated-material benefits in chess are unmeasured. The design above extrapolates from general cognitive-psychology results.
- The optimal ratio of exact/variant/foil items is unknown. It should be A/B tested.

---

## 4. Adaptive item selection: difficulty targeting, Elo/Glicko/IRT, knowledge tracing, bandits

### Takeaway
Use a two-level model:
1. **Item level.** An Elo/Glicko-style (Rasch-like) model estimates user ability per theme and puzzle difficulty. Lichess and Chess.com both rate puzzles with Glicko, and Pelánek shows simple Elo is robust for adaptive practice.
2. **Curriculum level.** A bandit (ZPDES-style learning-progress reward) or a mastery model (BKT, or extended BKT, which matches DKT) decides *which* weakness to practise next.

Target success rates well above 50%. The 85% rule (Wilson et al. 2019) is a principled default for detection-style tasks, and Chess.com's 2025 redesign moved its default toward "most puzzles solved."

### Cited Findings
- **Lichess.** Puzzles use Glicko-2, the same system as games, with each attempt scored as a game between user and puzzle. Puzzle ratings stabilize after about 20–30 attempts. The initial puzzle rating is estimated from the source game. New users start around 1500. Themed puzzles pay fewer points because the theme is a hint. Angle training offers theme/opening/difficulty ("easier or harder than your current rating"; the numeric offset was not found). In a 2026 forum thread Lichess said large rating swings followed a reset of puzzle rating deviations to a high starting value. — [Lichess forum: puzzle rating question](https://lichess.org/forum/general-chess-discussion/puzzle-rating-question); [lila puzzles docs (third-party)](https://mintlify.wiki/lichess-org/lila/features/puzzles)
- **Chess.com (Oct 9, 2025).** Puzzles were re-rated with "the same Glicko-style rating logic" as games, by replaying about 17 billion attempts. Inflationary features were removed (the minimum +5 per solve, partial credit), because inflated ratings "lost a lot of their meaning." Example: a mate-in-one once rated as high as 1800 is now 963. The new default difficulty "Standard" means "you should get most of them right," citing research that a higher success rate is optimal for learning. Hard and Extra Hard are also offered. No numeric target is published. Using a hint counts as a failed attempt. — [Chess.com announcement](https://www.chess.com/news/view/announcing-new-puzzles-rating-system); [Chess.com support (Oct 2025)](https://support.chess.com/en/articles/12488563-why-did-my-puzzle-rating-change)
- **Wilson, Shenhav, Straccia & Cohen (2019), *Nature Communications* 10:4646.** For gradient-descent-style learners on binary classification, learning is fastest at an error rate of about 15.87%, i.e. about 85% accuracy. The evidence is from ML models and animal-learning models, and the authors warn against direct extrapolation to grades. — [PMC6831579](https://pmc.ncbi.nlm.nih.gov/articles/PMC6831579)
- **Pelánek (2016, *Computers & Education*).** Treats a student answer as a match between student and item and updates both. Elo is "simple, robust, and effective" for adaptive systems (geography case study). The follow-up (UMUAI 2017) compared basic Elo, a Bayesian extension, hierarchical and networked (multivariate) models, and concluded basic Elo gives good prediction accuracy. A multivariate Elo variant handles items tagged with several concepts. BKT/IRT generally need pre-calibration on large samples. — [Pelánek 2016 PDF](https://www.fi.muni.cz/~xpelanek/publications/CAE-elo.pdf); [Pelánek et al. 2017 UMUAI PDF](https://www.fi.muni.cz/~xpelanek/publications/umuai-adaptive-practice.pdf); [Abdi et al., multivariate Elo (arXiv 1910.12581)](https://arxiv.org/pdf/1910.12581)
- **BKT and DKT.** BKT is a per-skill HMM with P(L0) (prior knowledge), P(T) (learning per step), P(G) (guess) and P(S) (slip). Khajah, Lindsey & Mozer (2016) found that BKT extended to capture recency, contextualized trial sequence, inter-skill similarity and individual ability performs "indistinguishable from" DKT. — [Khajah et al. 2016, arXiv 1604.02416](https://arxiv.org/abs/1604.02416); [Baker lab BKT paper](https://learninganalytics.upenn.edu/ryanbaker/behaviormetrika_vfinal.pdf); [IRT–BKT relation (arXiv 1803.05926)](https://arxiv.org/pdf/1803.05926)
- **ZPDES / RiARiT (Clement, Roy, Oudeyer & Lopes, *Journal of Educational Data Mining* 7(2):20–48, 2015).**
  - *Reward* = learning progress: the success rate over the most recent d/2 attempts minus the success rate over the previous d/2.
  - *Selection* is a variant of EXP4: weights w_a ← βw_a + ηr, and p_a = w̃_a(1−γ) + γ·uniform.
  - An expert-defined prerequisite graph bounds the zone of proximal development (ZPD). A harder level unlocks when w_(a_i) < θ·w_(a_(i+1)) and success > ω. Activities that stop progressing leave the ZPD. No numeric values for d, γ, β, η, θ or ω are given.
  - User study: 400 children aged 7–8 in 11 schools, 40-minute sessions, groups Control / Expert sequence / ZPDES / RiARiT. ZPDES reached higher levels on the harder exercise types (R 2.01 vs. 1.66, MM 1.83 vs. 1.41, RM 1.05 vs. 0.14) but lower on the simple type M (2.79 vs. 5.42). 95% of ZPDES students succeeded on at least one R exercise vs. 75% with the expert sequence. RiARiT, which uses more domain knowledge, did best in simulation, but ZPDES did best with real students. Gains were largest for heterogeneous, weaker populations.
  — [arXiv 1310.3174](https://arxiv.org/abs/1310.3174); [ar5iv full text](https://ar5iv.labs.arxiv.org/html/1310.3174)
- **Follow-on work.** ZPDES keeps a mastered/unmastered belief per activity and proposes unmastered activities whose prerequisites are mastered. Mu et al. (2018) combined ZPDES with a forgetting model, and a 2024 paper extends it to hierarchical bandits over concepts × difficulty levels. — [arXiv 2408.07208](https://arxiv.org/pdf/2408.07208)
- **Predicting difficulty for unrated positions.** GlickFormer (Miłosz & Kapusta, arXiv Dec 2024) was trained on 4.2M Lichess puzzles to predict Glicko-2 difficulty from the position and solution sequence. It reached MAE ≈ 218 rating points, with 50.45% of predictions within 2 RD (baseline ChessFormer: 227 MAE). It placed 11th in the IEEE BigData 2024 Cup puzzle-difficulty competition. — [arXiv 2410.11078](https://arxiv.org/html/2410.11078v2)

### Inferences
- **Item-difficulty model.** Keep a per-user, per-theme Glicko (or Elo) rating θ_u,k plus a global user rating. Database puzzles already have Lichess Glicko ratings b_i. The user's own positions and generated variants do not, so seed b_i with a difficulty predictor (GlickFormer-like, or engine features such as solution depth and move "only-ness") and update online.
- **Selection rule.** Choose items where P(success) = 1/(1+10^((b_i − θ_u,k)/400)) ≈ 0.75–0.85. That is roughly 190–300 rating points *below* the user's theme rating (at 85%, b ≈ θ − 301; at 75%, b ≈ θ − 191; this is derived from the Elo logistic formula, not from a source).
  - For pure *detection* drills of a known weakness, lean toward about 85% to keep the motif salient and build fluency.
  - For "stretch" items, use about 60–70%.
  - Never let theme-rated success drift near 50%. The puzzle sites' traditional default produces about 50%, and Chess.com moved away from it in 2025.
- **Choosing which weakness to practise (skill-level bandit).** Arms are themes from the weakness profile (forks, back-rank, rook endgames: Lucena/Philidor/active rook, and so on), each with difficulty sub-levels. Reward = ZPDES-style learning progress (recent-window minus earlier-window success rate on fresh, never-seen items, corrected for difficulty). Explore with γ ≈ 0.1–0.2. Use a prerequisite graph (for example, "basic knight fork" before "fork after a deflection sacrifice"). The priority prior is weakness severity from game analysis (frequency × eval loss). The numeric values are suggestions, since the paper provides none.
- **Mastery gate.** Retire a theme to maintenance mode when the BKT-style P(mastery) is above a threshold (0.95 is commonly used in Cognitive Tutors, though this was not verified in this research) **and** the real-game miss rate has dropped (section 6). Keep FSRS scheduling running for retention.
- **Don't count themed-mode success as evidence of detection skill.** Lichess discounts themed puzzles for this reason. Have a separate "blind mixed" rating per theme.

### Gaps
- Lichess's exact puzzle-selection offsets and Glicko-2 constants for puzzles were not obtained. They would need the lila source code on GitHub.
- No published target-success numbers from Chess.com.
- No study applying bandits or knowledge tracing to chess training was found.
- The 0.95 BKT mastery threshold could not be verified from a primary source in this session.

---

## 5. Lesson structure: worked examples vs. problem solving, feedback timing, hints, explanations, ITS evidence

### Takeaway
Step-based ITSs reach about d = 0.76, nearly matching human tutors (d = 0.79) (VanLehn 2011), so an interactive, move-by-move tutoring loop is worth building. For novices on a motif, start with worked examples and fade into problem solving (expertise reversal: examples stop helping as skill grows). Prompt self-explanation (g = 0.55). Give feedback on retrieval attempts, which roughly doubles the testing effect. State the principle after the learner has compared cases.

### Cited Findings
- **VanLehn (2011), *Educational Psychologist* 46(4):197–221.** Studies from 1975–2010. Effect sizes vs. no tutoring: human tutoring d = 0.79 (10 effects), step-based tutoring systems d = 0.76 (28 effects), substep-based d = 0.40 (26 effects). VanLehn's summary hypothesis is that human tutors and ITSs both sit at about 0.75 SD, not the 2.0 SD Bloom claimed. — [VanLehn MIT symposium slides](https://quality.mit.edu/files/2012/01/Quality-Symposium-Kurt-VanLehn.pdf); [ICCE 2011 keynote abstract](https://www.nectec.or.th/icce2011/speakers/keynoteSpeakers.php); [Marginal Revolution summary](https://marginalrevolution.com/marginalrevolution/2013/10/how-good-are-computers-as-tutors.html)
- **Worked examples and expertise reversal.** Learning from examples beat problem solving for inexperienced learners, but the advantage faded as expertise grew. Faded worked examples (Renkl) structure the move from studying examples to solving problems. Effectiveness depends on prior knowledge. — [2006 engineering-education study on fading and expertise reversal (ASU-hosted PDF; authorship not verified here)](https://faculty.engineering.asu.edu/mre/wp-content/uploads/sites/31/2020/02/Exp_Rev_LI06.pdf); [Wikipedia: Expertise reversal effect (secondary)](https://en.wikipedia.org/wiki/Expertise_reversal_effect)
- **Self-explanation (Bisra, Liu, Nesbit, Salimi & Winne 2018, *Educational Psychology Review*).** 69 effect sizes from 64 reports, g = 0.55. By task type: studying text g = 0.79, solving problems g = 0.48, studying worked examples g = 0.36. By prompt type: "explain" g = 0.68 and "conceptualize" g = 0.87 beat "justify" g = 0.42; metacognitive prompts were not significant. The authors recommend research into *computer-generated* self-explanation prompts. — [Bisra et al. 2018 PDF](https://www.Gwern.net/doc/psychology/spaced-repetition/2018-bisra.pdf); [TUM Clearing House summary](https://www.clearinghouse.edu.tum.de/wp-content/uploads/2023/09/CHU-KR-25_ENG_Bisra_2018_Selbsterklaerungen.pdf)
- **Feedback with retrieval.** Rowland (2014): testing effect g = 0.73 with feedback vs. 0.39 without. — [Frontiers 2018](https://frontiersin.org/articles/10.3389/fpsyg.2018.02412/full)
- **Feedback timing evidence is mixed.** Some studies favour immediate feedback for conceptual knowledge. Others find that delayed feedback strengthens initially correct responses through spaced re-encoding. — [Smith & Kimball 2010 manuscript](https://www.ou.edu/memorylab/pdfs/SmithKimball_2010_LearningFromFeedback_ms.pdf); [Brandeis record: "Immediate versus delayed feedback"](https://scholarworks.brandeis.edu/esploro/outputs/journalArticle/Immediate-Versus-Delayed-Feedback-on-Learning/9924143224801921)
- **Case comparison then principle.** Giving the principle *after* the case comparison helps more than giving it before or not at all (moderator per secondary summary of Alfieri et al. 2013). — [Alfieri et al. 2013 PDF](https://www.lrdc.pitt.edu/Schunn/papers/ContrastingCasesMeta-AlfieriEtAl2013.pdf)
- **Hints.** Chess.com treats using a hint as a failed attempt for rating purposes. — [Chess.com announcement](https://www.chess.com/news/view/announcing-new-puzzles-rating-system)

### Inferences
- **Proposed lesson template per weakness.** Example: "knight forks on king + rook."
  1. **Diagnose / hook (retrieval first).** Show the user's own game position *without* the answer and ask "find the best move" (a pre-test that is also retrieval practice). Record the result as a baseline.
  2. **Compare cases.** Put the user's position next to 2–3 database positions with the same motif and ask "what do these have in common?"
  3. **State the principle after the comparison.** Short, engine-verified explanation (section 7) of the cues: the unprotected king/rook pair a knight's move apart, forcing checks, removing the defender.
  4. **Worked example, then faded steps.** One fully annotated line, then a version where the user plays the key move and the system plays the rest, then full problem solving.
  5. **Self-explanation prompt.** Use "explain"/"conceptualize" style ("Which squares made the fork possible?"), not "justify." Keep it short, perhaps as multiple choice to limit friction.
  6. **Drill.** 8–15 items at about 80–85% predicted success: own-game variants, database puzzles and foils, with immediate correctness feedback and a refutation line shown on errors.
  7. **Delayed retest.** Schedule with FSRS, plus an *interleaved* review session 1–3 days later that mixes this theme with others, unlabelled (Cepeda: for a goal several weeks out, gaps of a few days are in the efficient range).
- **Hints should be graduated.** For example: (i) "there is a tactic", (ii) "look at the knight", (iii) highlight the target squares, (iv) show the move. Each hint downgrades the FSRS grade (Hard or Again) and counts as a partial failure in the skill model, following Chess.com's hint=fail convention.
- **For rook endgames (procedural knowledge), use a different interaction.** Use play-it-out-against-the-engine drills from key theoretical positions (Lucena, Philidor, Vancura), not single-move puzzles. Success = holding the draw or converting within N moves, with move-level feedback (the step-based tutoring that VanLehn found effective).
- **Detection vs. execution.** For "overlooked" tactics, much of each session should be *unannounced*: positions shown as "play a move" in a game-like context, where some contain the motif and some do not.

### Gaps
- No chess-specific experiments on worked examples vs. puzzles, hint policies, or feedback timing were found.
- No primary meta-analytic effect size was retrieved for worked examples or expertise reversal. Shute (2008) and Kulik & Kulik (1988) were not retrieved.

---

## 6. Measuring improvement

### Takeaway
Primary outcome: the **per-theme miss rate in subsequent real games**, normalized by *opportunities* (the number of positions where the motif was available, found by the same detector). Puzzle-rating gains are a secondary measure because of inflation and "puzzle mode" artifacts. Practice-time performance can diverge sharply from unassisted performance (Bastani et al.: +48% practice, −17% exam). Use randomized within-user designs, because users are selected for treatment at their worst moment, which guarantees regression to the mean.

### Cited Findings
- **Practice performance ≠ learning (Bastani et al., PNAS 2025, about 1,000 high-school students in Turkey).** Unguarded GPT-4 access raised practice-problem performance by 48% (GPT Base) and by 127% for the guard-railed GPT Tutor. When access was removed, the GPT Base group scored 17% *worse* on the exam than controls. Guardrails largely mitigated the harm. — [PNAS via RePEc](https://ideas.repec.org/a/nas/journl/v122y2025pe2422633122.html); [SSRN preprint](https://papers.ssrn.com/abstract=4895486)
- **Learners misjudge what works.** In Kornell & Bjork (2008) learners judged blocking more effective when interleaving actually was. — [Memory & Cognition 2013 summary](https://www.springermedizin.de/article/10.3758/s13421-012-0272-7)
- **The optimal schedule depends on the retention interval (Cepeda et al. 2008).** Measurement windows must match the goal, for example retention at 1 week vs. 1 month. — [Cepeda et al. 2008 PDF](https://www.evullab.org/pdf/CepedaVulRohrerWixtedPashler-PS-2008.pdf)
- **Puzzle ratings can be invalid measures.** Chess.com's old puzzle ratings were "significantly inflated" by generous features and lost meaning as indicators of playing strength (re-rated Oct 2025). Players commonly have puzzle ratings far above their game ratings (the "transfer problem," documented only informally). — [Chess.com announcement](https://www.chess.com/news/view/announcing-new-puzzles-rating-system); [Chessiro blog (commercial)](https://chessiro.com/blog/puzzles-from-your-own-games)
- **ZPDES evaluation design as a template.** Pre- and post-tests were given a few days before and after use, with an untreated control group in the same classrooms (n = 400). — [ar5iv 1310.3174](https://ar5iv.labs.arxiv.org/html/1310.3174)

### Inferences
- **Primary metric: opportunity-normalized miss rate per theme in rated games.** misses_k / opportunities_k, where an opportunity is a position in the user's own games where the engine-verified best move involves theme k and wins at least X centipawns over alternatives. Track it on a rolling window (for example, the last 30 games or 100 opportunities). Report it alongside eval-loss per opportunity. Raw counts confound improvement with changes in opponents and openings.
- **Secondary metrics.**
  - Theme-specific *unlabelled* puzzle Glicko rating.
  - FSRS retrievability on cards.
  - Time-to-solve.
  - Game rating, which is noisy and slow but is the outcome users care about.
- **Regression to the mean.** Weaknesses are chosen *because* they were unusually bad recently, so they will tend to improve even with no treatment. Mitigations:
  1. **Within-user randomization.** Of the detected weaknesses, randomly assign some to "train now" and others to "train later" (a waitlist), then compare the change in miss rate between trained and untrained themes over the same games. Each user is their own control.
  2. **Between-user A/B tests** for engine policies (for example, own-position-plus-variants vs. variants only, 70% vs. 85% target success, FSRS vs. fixed ladder), with pre-registered primary metrics.
  3. **Baseline from a longer window** than the one used for detection.
  4. **Account for selection on noise.** Use a hierarchical/Bayesian (shrinkage) estimate of the per-theme miss rate before and after.
- **Delayed retention tests.** Unlabelled retests of trained themes at 1–4 weeks are a better learning measure than in-session accuracy, given Bastani's practice-vs-exam divergence.
- **Don't optimize for in-app accuracy or satisfaction alone.** Both favour blocked, easy, hinted practice (Kornell & Bjork, Bastani).

### Gaps
- No published study was found that measured chess-training effects on real-game miss rates, and no chess-specific guidance on regression to the mean exists. The recommendations above are standard experimental-design inferences, not sourced chess findings.
- The statistical power needed (games per user to detect a change in miss rate) was not researched.

---

## 7. LLM-generated explanations in tutoring: effectiveness and the need for engine grounding

### Takeaway
Well-designed LLM tutors can produce large gains:
- Harvard physics RCT: students learned more in less time than with in-class active learning.
- Nigeria RCT: about 0.3 SD.

But unguarded "answer-giving" chatbots can harm unassisted performance (−17% on the exam in Bastani et al.). For chess specifically, LLMs hallucinate heavily: GPT-5.4 without tools made incorrect sub-claims 22% of the time, and smaller open models more than 40%. Engine-tool grounding substantially improves factual correctness. Explanations must be generated from engine-verified facts and the tutor must not give answers away.

### Cited Findings
- **Kestin, Miller et al. (2025), *Scientific Reports* 15:17458.** Randomized crossover with 194 Harvard intro-physics students. An AI tutor built on the same pedagogical principles and the instructor's feedback notes produced more learning in less time than in-class active learning (the preprint headline says "more than twice as much"), plus higher self-reported engagement. Caveats: the authors built the tutor, and the study covered one course and two lessons. — [PMC12179260](https://pmc.ncbi.nlm.nih.gov/articles/PMC12179260/); [Harvard Gazette](https://news.harvard.edu/gazette/story/2024/09/professor-tailored-ai-tutor-to-physics-course-engagement-doubled/)
- **Bastani et al. (PNAS 2025).** GPT Base (ChatGPT-like) +48% practice, −17% exam vs. control. GPT Tutor (prompts designed to protect learning, such as not giving answers directly) +127% practice, with exam harm largely mitigated. Mechanism: students used the unguarded tool as a crutch. — [PNAS via RePEc](https://ideas.repec.org/a/nas/journl/v122y2025pe2422633122.html); [SSRN](https://papers.ssrn.com/abstract=4895486)
- **Nigeria RCT (World Bank).** Supervised GPT-4 tutoring in pairs with reasoning-promoting prompts gave about 0.3 SD overall (about 0.24 SD in English) and better end-of-year exams. Attributed to structured routines, guardrails and monitoring against over-reliance. — [VoxDev summary](https://www.voxdev.org/topic/education/how-ai-tutors-improved-learning-nigeria)
- **Concept-guided Chess Commentary (CCC; Kim et al., NAACL 2025, arXiv Oct 2024).** Expert engines are accurate but hard to interpret, and LLMs are fluent but hallucinate. CCC feeds *prioritized, engine-derived concepts* to the LLM, which only verbalizes. Human judges and the proposed GCC-Eval metric (informativeness plus linguistic quality) rated CCC's commentary accurate, informative and fluent. — [arXiv 2410.20811](https://export.arxiv.org/abs/2410.20811); [NAACL 2025 listing](https://papers.cool/venue/2025.naacl-long.481@ACL)
- **"Hallucinations on the Board" (Hebbar, Sheng, Oh & Viswanath, arXiv 2608.04240, Aug 4, 2026).** The ACT-Eval framework breaks commentary into atomic claims and checks each against chess-engine tools and expert reference annotations. Benchmark: 325 position–move pairs, 125 with expert references, plus a five-class error taxonomy. GPT-5.4 without tools: 22.0% of sub-claims incorrect. Smaller open-weight models: over 40%. Engine-tool augmentation substantially improves factual correctness and move-quality judgments. All models, with or without tools, still cover expert strategic and tactical ideas poorly. ACT-Eval's factual judgments fall within human inter-expert agreement. — [arXiv 2608.04240](https://arxiv.org/abs/2608.04240)
- **Self-explanation prompts by computer.** Bisra et al. (2018) explicitly recommend researching computer-generated self-explanation prompts. This is a natural LLM role that does not require the LLM to assert chess facts. — [Bisra et al. 2018 PDF](https://www.Gwern.net/doc/psychology/spaced-repetition/2018-bisra.pdf)

### Inferences
- **Grounding architecture.** The engine and the tactic detector produce a structured "fact sheet" per position: best move, principal variation (PV), evaluation before and after, theme tags, the attacked pieces and squares, why the natural alternative fails (refutation PV), and the user's actual move with its refutation. The LLM may only verbalize facts on that sheet (the CCC pattern). Post-generation, an ACT-Eval-style verifier splits the explanation into atomic claims and checks each move/square/eval claim against the engine and a legal-move generator, then regenerates or drops failed claims. This matters because even frontier models are wrong on about 1 in 5 sub-claims without tools.
- **Pedagogical guardrails, following Bastani's GPT Tutor.**
  - The tutor must not reveal the solution before an attempt.
  - Use graduated hints (section 5).
  - Prefer Socratic prompts and self-explanation questions.
  - Count any LLM-revealed answer as a failure for scheduling purposes.
- **Where LLMs add most value at low risk.**
  - Natural-language framing of the user's *own* mistake ("In your game vs X on move 23 you played Rd1; Nf6+ forked king and rook").
  - Generating self-explanation and comparison prompts.
  - Summarizing progress.
  - Use LLMs least for unconstrained strategic claims, where coverage remains poor even with tools.
- **Evaluate explanation quality offline before showing it to users.** Use ACT-Eval-like factual accuracy plus coverage. Online, A/B test explanation variants against the delayed-retest and real-game miss-rate metrics from section 6, not against user ratings.

### Gaps
- No RCT of LLM explanations specifically for chess learning outcomes was found.
- The Nigeria result comes from a VoxDev summary; the full paper was not retrieved.
- The Kestin "more than twice" figure is from the preprint/press and was not checked against the published effect sizes.
- ACT-Eval numbers for models other than GPT-5.4 were not individually reported in the abstract.
