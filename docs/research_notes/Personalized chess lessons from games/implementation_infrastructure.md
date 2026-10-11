# Implementation Infrastructure for "blunderbot" (data sources, APIs, engines, ML inference, costs, libraries, licenses)

Research date: 2026-10-09. Versions and dates below were pulled live from npm, PyPI, GitHub, and HuggingFace APIs, or from primary docs, on that date. Where I say "live test", I sent the HTTP request myself on 2026-10-09 and am reporting what came back.

## Q1. Game import APIs (Lichess, Chess.com): endpoints, params, rate limits, auth, recent changes

### Takeaway
Lichess gives you the most to work with. `GET /api/games/user/{username}` streams NDJSON or PGN. It can include clocks in centiseconds and, for games that already have Lichess analysis, per-move evals, best moves, PVs, and Inaccuracy/Mistake/Blunder judgments. It is throttled to 20, 30, or 60 games/s depending on auth. Chess.com's PubAPI gives monthly archives with `%clk` clocks in the PGN and an optional `accuracies` field, but no per-move evals, so every Chess.com game needs its own engine analysis.

### Cited Findings
**Lichess: general**
- Lichess OpenAPI spec is at version 2.0.176, and the spec itself is licensed AGPL-3.0-or-later. All requests go to `https://lichess.org` unless noted otherwise. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)
- Rate-limit policy: "All requests are rate limited using various strategies… Only make one request at a time. If you receive an HTTP response with a 429 status… waiting one minute before retrying will be sufficient, but some limits may require longer." The docs publish no numeric per-endpoint quotas. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml), [lichess.org/api](https://lichess.org/api)
- Third-party mirrors list numbers like "~15 req/min anonymous / ~60 per token" or "20 req/s". Lichess's own docs don't confirm any of them, so they are unverified. — [mintlify mirror](https://www.mintlify.com/lichess-org/lila/api/rate-limits), [jentic](https://jentic.com/apis/lichess.org/lichess-org-api-reference)
- Auth: you can use a personal access token, or "Login with Lichess" via OAuth2 Authorization Code + PKCE. Lichess "supports unregistered and public clients (no client authentication, choose any unique client id)". Only `S256` is accepted. "Access tokens are long-lived (expect one year)". "Refresh tokens are not supported". Tokens can be up to 512+ chars. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)
- The spec lists a JS/TS client (`devjiwonchoi/equine`) and the Python client `berserk`. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)

**Lichess: `GET /api/games/user/{username}` (export games of a user)**
- Returns PGN or NDJSON depending on the `Accept` header (`application/x-chess-pgn` vs `application/x-ndjson`). Games come most recent first by default (`sort=dateDesc|dateAsc`). Some users have more than 500,000 games, so stream the response. — [api-games-user-username.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-user-username.yaml)
- Throttle: "Anonymous request: 20 games per second; OAuth2 authenticated request: 30 games per second; Authenticated, downloading your own games: 60 games per second". — [same](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-user-username.yaml)
- Query params:
  - Range and count: `since` and `until` (ms timestamps, minimum 1356998400070), `max`.
  - Filters: `vs`, `rated`, `perfType` (comma list, e.g. `blitz,rapid,classical`), `color`, `analysed` (only games with or without computer analysis).
  - Content toggles: `moves` (default true), `pgnInJson`, `tags`, `clocks` (PGN `[%clk]` or JSON `clocks` in centiseconds), `evals` (PGN `[%eval]` or JSON `analysis`), `accuracy` (JSON only), `opening`, `division` (middlegame/endgame start plies, JSON only).
  - Game state: `ongoing` (delayed 3–60 s), `finished`.
  - Extras: `literate` (text annotations like "Mistake. Best move was h6." plus variations), `lastFen`, `withBookmarked`.
  - — [same](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-user-username.yaml)
- Live test (NDJSON, `evals=true&clocks=true&accuracy=true&division=true&analysed=true`) on an analysed blitz game:
  - Top-level keys: `id, rated, variant, speed, perf, createdAt, lastMoveAt, status, source, players, initialFen, winner, moves, clocks, analysis, arenaTour, clock, division`.
  - Each `analysis[]` entry has `eval` (cp) or mate. Bad moves also carry `best` (UCI), `variation` (SAN PV), and `judgment: {name: "Mistake"|"Blunder"|"Inaccuracy", comment}`.
  - `players.white.analysis` = `{inaccuracy, mistake, blunder, acpl, accuracy, phases:{opening, middlegame, endgame}}`.
  - `clocks` are centisecond integers. `division` = `{middle: 22, end: 33}`.
  - — live test of `https://lichess.org/api/games/user/german11?max=1&...` (spec: [GameJson.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/schemas/GameJson.yaml))
- Live-test anomaly worth flagging to implementers: requests sent with curl's default User-Agent got `404 {"error":"Not found"}` for several existing users. The same request with a custom UA (`blunderbot-research/0.1`) or a browser UA returned 200. — live test against `https://lichess.org/api/games/user/german11?max=1` (I found no documentation of this; see Gaps)
- `POST /api/games/export/_ids` takes up to 300 comma-separated game IDs per request (PGN or NDJSON), with the same content options. — [api-games-export-_ids.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-export-_ids.yaml)
- Opening Explorer moved to host `explorer.lichess.org`, and its spec now declares `security: OAuth2`, so a token is required. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml), [openingexplorer/lichess.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/openingexplorer/lichess.yaml)
- An "External engine" API (alpha) lets an engine running outside the browser serve analysis to the Lichess analysis board. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)

**Lichess: cloud eval `GET /api/cloud-eval`**
- Params: `fen` (X-FEN), `multiPv` (default 1), `variant`. Returns 404 if the position isn't cached. Up to 5 variations. Spec text: "about 320 million positions in the database… Use this endpoint to fetch a few positions here and there. If you want to download a lot of positions, get the full list" from the DB export. No auth (`security: []`). — [api-cloud-eval.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/analysis/api-cloud-eval.yaml)
- The "320 million" figure is probably stale. The eval dump currently holds 416,442,401 positions (see Q2). — [database.lichess.org](https://database.lichess.org/)
- Live test for the position after 1.e4 with multiPv=3 returned `depth: 60, knodes: 119133` and 3 PVs (e5 +22, c6 +27, c5 +32 cp). — live test `https://lichess.org/api/cloud-eval?fen=…&multiPv=3`

**Chess.com: Published-Data API (PubAPI)**
- Endpoints:
  - `https://api.chess.com/pub/player/{username}/games/archives` lists monthly archive URLs.
  - `https://api.chess.com/pub/player/{username}/games/{YYYY}/{MM}` returns that month's games as JSON.
  - `…/games/{YYYY}/{MM}/pgn` returns a multi-game PGN download (`application/x-chess-pgn`).
  - — [chess.com PubAPI](https://www.chess.com/news/view/published-data-api)
- Game fields documented: `url, fen, pgn, start_time (daily), end_time, time_control, rules, eco, tournament, match, accuracies (white/black, "if they were previously calculated")`, and white/black objects (`username, rating, result, @id`). The docs say nothing about per-move evals. — [chess.com PubAPI](https://www.chess.com/news/view/published-data-api)
- Rate limiting: "Your serial access rate is unlimited". Parallel requests "may" be blocked with `429 Too Many Requests`. Abnormal activity can get an app blocked entirely. They recommend a recognizable User-Agent with contact info. No numeric limits are published. — [chess.com PubAPI](https://www.chess.com/news/view/published-data-api)
- Caching: responses carry `ETag` and `Last-Modified`, and support `If-None-Match`/`If-Modified-Since` → 304. The page contradicts itself on refresh rate ("at most once every 12 hours" vs "24 hours"). Use `Accept-Encoding: gzip` (saves "up to 80%"). Page header says "Updated: Apr 22, 2026". The data is read-only, there's no explicit data license, and Chess.com IP (piece sets, move-classification glyphs) must be respected. — [chess.com PubAPI](https://www.chess.com/news/view/published-data-api)
- Live test:
  - `hikaru` has 154 monthly archives, and the list includes the current month (`2026/10`).
  - `games/2026/09` returned 509 games. Keys: `url, pgn, time_control, end_time, rated, accuracies, tcn, uuid, initial_setup, fen, time_class, rules, white, black, eco, tournament`.
  - PGN movetext includes clocks to tenths of a second (`{[%clk 0:04:53.5]}`). `accuracies` were present (`{white: 85.6, black: 94.12}`). There are no per-move evals.
  - Response headers: `etag`, `last-modified`, `cache-control: public, max-age=5`.
  - — live test `https://api.chess.com/pub/player/hikaru/games/2026/09`

### Inferences
- Lichess import strategy:
  - Request NDJSON with `clocks=true&evals=true&opening=true&division=true&accuracy=true`. Paginate incrementally using `since` set to the last import timestamp.
  - For games that come back with `analysis`, you can reuse Lichess's judgments and skip your own engine pass. You'd still want your own MultiPV analysis for motif classification, because Lichess gives only one PV, and only on mistakes.
  - Run one stream per user at a time, use a descriptive User-Agent, and on 429 back off for at least 60 s.
- With Lichess OAuth (PKCE, no client secret, ~1-year tokens, no refresh), you get 30 games/s for third-party export and 60 games/s for the user's own games. A 1,000-game import takes about 17–50 s of streaming. Better Auth would need a generic OAuth2 provider config for Lichess (I didn't verify whether a built-in one exists).
- Chess.com import: fetch the archive list, then each month serially, using conditional GETs (ETag) for incremental sync. Parse `%clk` for time-pressure features. Every move needs your own engine analysis.

### Gaps
- Numeric Lichess rate limits per endpoint are not published. Neither are cloud-eval or tablebase quotas.
- Why default-curl-UA requests get 404 from Lichess game export: I found no doc. The practical advice is to always send a descriptive UA.
- I could not see the full text of the Chess.com "Breaking change: Player's clubs endpoint" notice. I found no 2025–2026 changes to the games-archive endpoints.

## Q2. Bulk data: Lichess open database, puzzle DB, eval dump (sizes, columns, licenses)

### Takeaway
Everything relevant on database.lichess.org is CC0: games, puzzles, and evals. Broadcasts are the exception (CC BY-SA 4.0). The 6.16M-puzzle CSV (307 MB compressed) has theme tags and Glicko ratings, which makes it a ready-made puzzle bank indexed by motif and rating. The 416M-position eval dump (22.4 GB compressed) can act as a local eval cache for opening and common positions.

### Cited Findings
- License: "Database exports are released under the Creative Commons CC0 license. Use them for research, commercial purpose, publication, anything you like." Broadcast games (1,263,913) are released under CC BY-SA 4.0. — [database.lichess.org](https://database.lichess.org/)
- **Puzzles:**
  - 6,157,341 puzzles, in `lichess_db_puzzle.csv.zst`, last updated 2026-10-05. Live HEAD: `content-length: 307234795` (~307 MB compressed).
  - Columns: `PuzzleId,FEN,Moves,Rating,RatingDeviation,Popularity,NbPlays,Themes,GameUrl,OpeningTags,DailyDate`.
  - Moves are UCI. The FEN is the position *before* the opponent's move, so you apply the first move and the solution starts at move 2. All player moves are "only moves" (exception: mate in 1).
  - Popularity runs from -100 to 100. `OpeningTags` is only set for puzzles starting before move 20. `DailyDate` is a ms timestamp.
  - — [database.lichess.org](https://database.lichess.org/#puzzles)
- Puzzle generation: ">100 years of CPU time… went through 600,000,000 analysed games… re-analyzed interesting positions with Stockfish NNUE at 40 meganodes". Puzzles were then auto-tagged by the tagger, and ratings come from Glicko2 (each attempt counted as a game). The generator and tagger code is open at ornicar/lichess-puzzler. Theme names and descriptions are in `translation/source/puzzleTheme.xml` in lila. — [database.lichess.org](https://database.lichess.org/#puzzles), [lichess-puzzler](https://github.com/ornicar/lichess-puzzler/tree/master/tagger), [puzzleTheme.xml](https://github.com/lichess-org/lila/blob/master/translation/source/puzzleTheme.xml)
- The Lichess API docs say "Our collection of puzzles is in the public domain". — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)
- **Evals:**
  - 416,442,401 positions, in `lichess_db_eval.jsonl.zst`, updated 2026-10-05. Live HEAD: `content-length: 22423777007` (~22.4 GB compressed).
  - Schema: `{fen (pieces, side, castling, ep only), evals:[{knodes, depth, pvs:[{cp|mate, line}]}]}`. Evals were "produced by, and for, the Lichess analysis board, running various flavours of Stockfish within user browsers".
  - The DB recommends picking the highest-depth eval and its first PV.
  - — [database.lichess.org](https://database.lichess.org/#evals)
- **Games:**
  - 8,220,312,882 standard rated games in total, as monthly files. September 2026 is 29.2 GB (live HEAD 29,224,520,887 bytes) with 89,616,462 games.
  - "About 6% of the games include Stockfish analysis evaluations". Clocks (`%clk`) have been present since April 2017. Uncompressed files are ~7.1× larger. `.zst` can be partially decompressed and streamed (`zstdcat … | script`).
  - — [database.lichess.org](https://database.lichess.org/)
- Openings dataset: crowdsourced `lichess-org/chess-openings` on GitHub. — [database.lichess.org](https://database.lichess.org/)

### Inferences
- Puzzle bank: import the puzzle CSV into Postgres (`puzzles` table). Index `Themes` (text[] with GIN) and `Rating`. Lessons can then serve "fork, rating 1400–1600" puzzles immediately, which bootstraps spaced repetition before you have enough user-derived positions. CC0 means no attribution obligations, though attribution is good practice.
- Eval cache: loading the full 416M-position dump is heavy, since 22.4 GB compressed expands to far more. A cheaper option is to filter it to positions within the first ~15–20 plies, or to look up opening positions through `/api/cloud-eval` before running Stockfish.
- Monthly game dumps (~30 GB/month compressed) are only needed if you train your own models (e.g., Maia fine-tunes) or compute population baselines ("how often do 1500s miss this motif").

### Gaps
- I didn't check exact uncompressed sizes or row counts per theme for the puzzle and eval files.

## Q3. Chess libraries (TS/JS and Python): versions and licenses, and what they mean for a closed or open app

### Takeaway
For a closed-source web client, the permissive choices are chess.js (BSD-2) for rules/PGN and react-chessboard (MIT) or cm-chessboard (MIT) for the board UI. chessops and chessground (the Lichess stack) are GPL-3.0-or-later, and shipping them in a browser bundle counts as conveying GPL code. python-chess is GPL-3.0+, but using it server-side only does not trigger source disclosure.

### Cited Findings
| Package | Latest version (date) | License | Source |
|---|---|---|---|
| `chess.js` | 1.4.0 (2025-06-14) | BSD-2-Clause | [npm](https://www.npmjs.com/package/chess.js), [GitHub](https://github.com/jhlywa/chess.js) |
| `chessops` | 0.15.1 (2026-07-12) | GPL-3.0-or-later | [npm](https://www.npmjs.com/package/chessops), [GitHub](https://github.com/niklasf/chessops) |
| `@lichess-org/chessground` | 10.4.2 (2026-10-03) | GPL-3.0-or-later | [npm](https://www.npmjs.com/package/@lichess-org/chessground), [GitHub](https://github.com/lichess-org/chessground) |
| `chessground` (old unscoped name) | 9.2.1 (2025-05-15) | GPL-3.0-or-later | [npm](https://www.npmjs.com/package/chessground) |
| `react-chessboard` | 5.12.1 (2026-08-16) | MIT | [npm](https://www.npmjs.com/package/react-chessboard), [GitHub](https://github.com/Clariity/react-chessboard) |
| `cm-chessboard` | 8.15.3 (2026-10-01) | MIT | [npm](https://www.npmjs.com/package/cm-chessboard), [GitHub](https://github.com/shaack/cm-chessboard) |
| `chess` (python-chess, PyPI) | 1.11.2 (2025-02-25) | GPL-3.0+ | [PyPI](https://pypi.org/project/chess/), [GitHub](https://github.com/niklasf/python-chess) |
| `python-chess` (PyPI, legacy name) | 1.999 (2020 stub) | GPL-3.0+ | [PyPI](https://pypi.org/project/python-chess/) |

- chessground moved from the unscoped `chessground` npm name (last published 9.2.1) to `@lichess-org/chessground` (10.x, releases v10.4.0–10.4.2 in late Sep/early Oct 2026). — [npm chessground](https://www.npmjs.com/package/chessground), [GitHub releases](https://github.com/lichess-org/chessground/releases)
- chessops features: FEN/SAN/UCI read and write, bitboard `SquareSet`, legal move generation, setup validation, Chess960, 8 variants, attacks/rays (Hyperbola Quintessence), PGN parsing. — [chessops README](https://github.com/niklasf/chessops)
- chess.js describes itself as "a TypeScript chess library used for chess move generation/validation, piece placement/movement, and check/checkmate/stalemate detection". — [chess.js README](https://github.com/jhlywa/chess.js)
- GPL-3.0 definition: "To 'convey' a work means any kind of propagation that enables other parties to make or receive copies. Mere interaction with a user through a computer network, with no transfer of a copy, is not conveying." — [GPL-3.0 text](https://www.gnu.org/licenses/gpl-3.0.txt)
- AGPL-3.0 §13: if you modify the Program, you must offer all users interacting with it remotely over a network the Corresponding Source. — [AGPL-3.0 text](https://www.gnu.org/licenses/agpl-3.0.txt)
- Precedent: CSSLab's Maia platform frontend (GPL-3.0) uses `chessground ^8.3.5`, `@react-chess/chessground`, and `chess.ts`. — [maia-platform-frontend package.json](https://github.com/CSSLab/maia-platform-frontend/blob/main/package.json)

### Inferences
- JavaScript sent to the browser is a copy transferred to the user, so it is "conveying". If chessground or chessops ends up in the web bundle, the client code it combines with must be offered under GPL-compatible terms, with source available. Avoid both in the client if the app stays closed. If you're happy to open-source the frontend under GPL/AGPL, chessground is the most polished board (it's what Lichess uses).
- Server-side GPL use (python-chess or chessops in a worker, Stockfish binary on the server) is not conveying, so there's no source obligation. AGPL code (lila-tablebase, Maia-3 inference code) does carry a source obligation if you modify it and expose it over a network.
- Suggested TS stack for a closed app: chess.js for PGN, SAN, and legality in both web and server, plus react-chessboard for the UI (React-native, fits TanStack Start). Python workers use python-chess for motif detection, which is the richest API for attacks, pins, and SEE-like helpers.

### Gaps
- I ran no performance benchmarks comparing chess.js and chessops (perft speed).

## Q4. Running Stockfish: server binary vs browser WASM, throughput, cost, GPL, fishnet

### Takeaway
Stockfish 19 (2026-09-05) is current. One modern cloud vCPU does roughly 0.9–1.2M nodes/s, so analysing a 40-move game (~80 positions) at ~1M nodes/position takes about 70–90 vCPU-seconds. On-demand cloud CPU puts that at roughly $0.001–0.002 per game. Browser WASM builds exist: Chess.com's stockfish.js v19 and Lichess's stockfish-web. Multi-threading there requires cross-origin isolation (COOP/COEP).

### Cited Findings
**Versions**
- Stockfish 19 was released 2026-09-05 (tag `sf_19`). It claims up to +44 Elo vs SF18.
  - Universal binaries auto-detect CPU features (no more choosing AVX2 vs AVX-512).
  - New SFNNv16 net architecture; the secondary small net is retired.
  - New native targets: WebAssembly, RISC-V, and LoongArch.
  - Stricter FEN/UCI validation: invalid input prints `info string CRITICAL ERROR` and the process terminates.
  - — [SF19 release](https://github.com/official-stockfish/Stockfish/releases/tag/sf_19)
- Stockfish 18 was released 2026-01-31. It added SFNNv10 with "Threat Inputs", up to +46 Elo vs SF17, and a "Shared Memory" implementation that lets multiple Stockfish processes share network weights ("the most efficient version for cloud analysis and high-concurrency testing"). — [SF18 release](https://github.com/official-stockfish/Stockfish/releases/tag/sf_18)
- Engine license is GPL-3.0. README: "whenever you distribute Stockfish in some way, you MUST always include the license and the full source code (or a pointer to where the source code can be found) to generate the exact binary you are distributing." — [Stockfish README](https://github.com/official-stockfish/Stockfish#terms-of-use)

**Browser builds**
- `stockfish` npm (nmrugg/stockfish.js) is at 19.0.0 (2026-09-15), GPL-3.0, "(c) 2026, Chess.com, LLC". Five flavors:
  - Large multi-threaded (~94 MB; needs cross-origin isolation headers).
  - Large single-threaded.
  - Lite multi-threaded (~1.6 MB, "quite a bit weaker").
  - Lite single-threaded (~1.6 MB). The README recommends this one for most projects ("still far stronger than any human").
  - ASM-JS fallback (~3 MB, very slow).
  - Runs in modern browsers and Node.js.
  - — [stockfish.js README](https://github.com/nmrugg/stockfish.js), [npm stockfish](https://www.npmjs.com/package/stockfish)
- `@lichess-org/stockfish-web` 0.5.0 (GitHub release 2026-09-05) ships WASM builds `sf_19`, `sf_19_smallnet` (nn-61e7af4bb97d), and Fairy-Stockfish 14. It is "optimized for lichess.org… not straight-forward to load and use". The README points users to stockfish.js for a simpler option. License conflict: the npm metadata says AGPL-3.0-or-later, while the GitHub repo is tagged GPL-3.0. The older `lila-stockfish-web` npm package (0.0.11, 2025-04) is AGPL-3.0-or-later. — [stockfish-web README](https://github.com/lichess-org/stockfish-web), [npm](https://www.npmjs.com/package/@lichess-org/stockfish-web), [npm lila-stockfish-web](https://www.npmjs.com/package/lila-stockfish-web)
- Precedent: the Maia platform frontend serves a small SF17 WASM (`public/stockfish/sf17-79.wasm`, 472 KB) via `lila-stockfish-web`, and sets `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers in `next.config.js`. — [maia-platform-frontend repo tree](https://github.com/CSSLab/maia-platform-frontend), [next.config.js](https://github.com/CSSLab/maia-platform-frontend/blob/main/next.config.js)

**Throughput benchmarks (native)**
- A community benchmark (amdw/enginecloud, GCP, `stockfish bench`) ran Stockfish 18 in March 2026. Averaged NPS with threads = vCPU count, as I computed from its `sfbench.csv`:
  - c4d-standard-16: 19.4M (1.22M per vCPU)
  - c4d-standard-32: 37.6M (1.17M/vCPU)
  - c4d-standard-96: 107.9M (1.12M/vCPU)
  - c3d-standard-16 (AMD Genoa EPYC 9B14): 15.8M (0.99M/vCPU)
  - c4-standard-16: 18.0M (1.13M/vCPU)
  - c2d-standard-16: 13.2M (0.82M/vCPU)
  - Single-thread NPS was ~1.7–1.8M on c4d and ~1.29M on c3d.
  - The author concludes: "at least 1 MNPS of search speed per core with c4d and the avx512icl binary". The c4 family underperformed c3d. Hash size had no measurable effect on bench NPS.
  - — [SF_BENCHMARKS.md](https://github.com/amdw/enginecloud/blob/main/stockfish/benchmarks/SF_BENCHMARKS.md), [sfbench.csv](https://github.com/amdw/enginecloud/blob/main/stockfish/benchmarks/sfbench.csv)
- Caveat from the same source: "NPS values aren't necessarily comparable across different engines, or even different versions of the same engine." — [SF_BENCHMARKS.md](https://github.com/amdw/enginecloud/blob/main/stockfish/benchmarks/SF_BENCHMARKS.md)
- Price point: AWS `c7a.xlarge` (4 vCPU, AMD EPYC Genoa) is $0.20528/hr on-demand, i.e. ~$0.051 per vCPU-hour. — [Vantage c7a.xlarge](https://instances.vantage.sh/aws/ec2/c7a.xlarge)

**How Lichess distributes analysis (fishnet)**
- fishnet v2.15.0 (2026-10-07), GPL-3.0, is a volunteer-run distributed Stockfish client (Fairy-Stockfish for variants).
  - Requirements: about 64 MiB RAM per CPU core, outgoing HTTP only. "Almost all processors will be able to meet the requirement of ~2 meganodes in 6 seconds".
  - Faster clients get the user queue (humans waiting), slower ones the system queue. fishnet does not use GPUs.
  - "You do not need to request a key (nor our permission) to run private instances."
  - — [fishnet README](https://github.com/lichess-org/fishnet)
- fishnet source: the server sends a per-batch `nodes` limit (with separate `classical` and `sf16`/NNUE values) plus optional `depth` and `multipv`. The client estimates batch time as "60 positions at 1_450_000 nodes each", with a 7-minute deadline, and says top-end clients finish in ≤35 s. — [fishnet src/stats.rs](https://github.com/lichess-org/fishnet/blob/main/src/stats.rs), [src/api.rs](https://github.com/lichess-org/fishnet/blob/main/src/api.rs)

### Inferences
- Cost model, my own arithmetic from the cited figures:
  - A 40-move game has 80 plies, about 80 positions.
  - At 1M nodes/position: 80M nodes ÷ ~1.0M NPS/vCPU ≈ 80 vCPU-s ≈ 0.022 vCPU-h ≈ $0.0011 per game at $0.051/vCPU-h.
  - At Lichess's ~1.45M nodes/position: ~116M nodes ≈ $0.0016 per game.
  - Importing 200 games per user costs about $0.22–0.33 on-demand; spot or dedicated hosts would be cheaper.
  - A 16-vCPU box (~19M NPS) analyses roughly 600–850 games/hour at 1–1.45M nodes/position.
- Recommended pattern:
  - Run a fixed node budget per position, not a fixed depth, so throughput is predictable.
  - Do a cheap first pass (e.g., 300k–1M nodes, MultiPV 1) on every position. Then do a deeper MultiPV 3–5 re-search only on positions flagged as mistakes, which feed motif classification and puzzle extraction.
  - Run one single-threaded Stockfish process per core (SF18+ shares NNUE weights across processes), not one multi-threaded process per game.
  - Reuse Lichess analysis and cloud evals when they're available.
- Browser-side analysis (stockfish.js lite single-threaded) works well for interactive "explore this position" on the lesson page, and costs nothing on the server. Batch analysis of imported games belongs on the server, where it's deterministic, runs while the user is offline, and can be stored.
- GPL: running the Stockfish binary server-side involves no distribution. Serving stockfish.js WASM to browsers *is* distribution: include the license and a source link. Because it runs as a separate program in a Web Worker speaking UCI over `postMessage`, it's commonly treated as an aggregate rather than a combined work. That's my inference, not legal advice.

### Gaps
- I found no SF19-specific NPS benchmarks on cloud CPUs, and no Graviton/ARM per-vCPU numbers for SF18/19.
- I found no published measurement of WASM vs native NPS ratio or browser-side time-per-game.
- I found no authoritative "time to depth 18–20" figures. Node counts to reach a given depth vary heavily by position, so per-position node budgets are the better unit.

## Q5. Running Maia / Maia-2 / Maia-3 / Lc0 (human-move prediction), plus ONNX/JS options and licenses

### Takeaway
There are now three generations:
- **Maia-1:** 9 lc0 nets, Elo 1100–1900, GPL-3.0, run in lc0 with `go nodes 1`.
- **Maia-2:** a single 23.3M-param PyTorch model conditioned on both players' Elo, MIT-licensed, `pip install maia2`.
- **Maia-3:** "Chessformer" transformers (5M/23M/79M) on HuggingFace, AGPL-3.0 code. The Maia-2 README recommends it for new projects, and it reports 57.1% move-matching vs Maia-2's 53.25%.

CSSLab's own web platform already runs a Maia-3 ONNX model (45.7 MB) in the browser with onnxruntime-web, which gives a working JS-side reference implementation.

### Cited Findings
**Maia-1**
- Nine final models for Elo 1100–1900, stored as Leela (lc0) nets (`maia-1100.pb.gz` … `maia-1900.pb.gz`, release v1.0, 2021-01-14). "The Maias are not a full chess framework… you need to load them with lc0… you want to disable searching, a nodes limit of 1 is what we use (`go nodes 1`)." Repo license is GPL-3.0. — [maia-chess README](https://github.com/CSSLab/maia-chess), [releases](https://github.com/CSSLab/maia-chess/releases/tag/v1.0)
- lc0 status:
  - Latest stable is v0.32.1 (2025-11-23). v0.33.0-rc0 came out 2026-09-05. GPL-3.0.
  - v0.33 adds or improves ONNX backends (onnx-trt with bf16, onnx-coreml, onnx-migraphx), CUDA graph support, and a `threads` option on the BLAS backend (for CPU).
  - — [lc0 v0.33.0-rc0](https://github.com/LeelaChessZero/lc0/releases/tag/v0.33.0-rc0), [lc0 repo](https://github.com/LeelaChessZero/lc0)

**Maia-2**
- PyPI `maia2` 0.11.0 (2026-07-15), Python ≥3.10,<3.13. Runs on CUDA, Apple MPS, or CPU via `device="auto"`. v0.10.0 updated to PyTorch 2.8.0. The repo license is MIT. — [PyPI maia2](https://pypi.org/project/maia2/), [maia2 releases](https://github.com/CSSLab/maia2/releases), [maia2 repo](https://github.com/CSSLab/maia2)
- API:
  - `model.from_pretrained(type="rapid"|"blitz", device=...)`.
  - `inference.inference_batch(data, model, batch_size=1024, ...)`.
  - `inference.inference_each(model, prepared, fen, elo_self, elo_oppo)` returns `(move_probs dict, white_expected_score)`. The README notes that score is "not a calibrated win probability".
  - — [maia2 README](https://github.com/CSSLab/maia2)
- Architecture and results (NeurIPS 2024):
  - 23.3M parameters, vs 92M total for nine Maia-1 experts.
  - ResNet encoder (12 blocks, 256 channels) plus 2 skill-aware attention blocks.
  - Input is the current position only, an 18×8×8 tensor. Elo goes in as categorical buckets from <1000 to >2000 in 100-pt steps.
  - Top-1 accuracy 53.25% vs Maia 1500's 51.39%, Leela 39.35%, Stockfish ~38.4%.
  - Trained on 169M Lichess rapid games (9.1B positions), ~13 days on 2×A100.
  - The paper reports no inference latency.
  - — [Maia-2 paper (arXiv HTML)](https://arxiv.org/html/2409.20553v2)

**Maia-3**
- The `CSSLab/maia3` repo was created 2026-05-22 and is AGPL-3.0. It is UCI-engine inference code: `pip install .`, then the `maia3-5m` / `maia3-23m` / `maia3-79m` commands, which auto-download from HuggingFace.
  - The README recommends the 5M model for "First try, CPU, chess GUIs" and 79M for "Best accuracy".
  - UCI options: `Elo`, `SelfElo`, `OppoElo`, `Temperature`, `TopP`, `MultiPV` (shows likely human moves).
  - — [maia3 README](https://github.com/CSSLab/maia3)
- HuggingFace repos `UofTCSSLab/Maia3-5M`, `-23M`, and `-79M` (created 2026-05-17) each contain a single `.pt` checkpoint. The model cards declare no license field. — [HF Maia3-5M](https://huggingface.co/UofTCSSLab/Maia3-5M), [HF API](https://huggingface.co/api/models/UofTCSSLab/Maia3-79M)
- Paper "Chessformer: A Unified Architecture for Chess Modeling" (arXiv 2605.19091, submitted 2026-05-18). "57.1% move-matching accuracy… significantly surpassing the previous state of the art with fewer than a quarter of the parameters."
  - Venue conflict: the maia-chess README says "ICML '26", while the maia3 BibTeX and the arXiv listing say ICLR.
  - — [arXiv 2605.19091](https://arxiv.org/abs/2605.19091), [maia-chess README](https://github.com/CSSLab/maia-chess), [maia3 README](https://github.com/CSSLab/maia3)
- The Maia-2 README now says: "Maia-3 is recommended for new projects." — [maia2 README](https://github.com/CSSLab/maia2)

**JS/ONNX inference**
- CSSLab's `maia-platform-frontend` (GPL-3.0, Next.js) runs Maia in the browser:
  - Uses `onnxruntime-web ^1.23.0` with the model at `public/maia3/maia3_simplified.onnx` (45,683,686 bytes).
  - Runs in a Web Worker (`public/maia-worker.js`).
  - `src/lib/engine/maia.ts` passes tokens plus `eloSelfs`/`eloOppos` Float32 arrays and reads 4,352 move logits and 3 value logits (loss/draw/win) per position, with batch evaluation supported.
  - — [maia-platform-frontend tree](https://github.com/CSSLab/maia-platform-frontend), [maia.ts](https://github.com/CSSLab/maia-platform-frontend/blob/main/src/lib/engine/maia.ts), [package.json](https://github.com/CSSLab/maia-platform-frontend/blob/main/package.json)
- `onnxruntime-node` and `onnxruntime-web` are both at 1.30.0 (2026-09-14), MIT. PyPI `onnxruntime` is 1.30.0 (requires Python ≥3.11). — [npm onnxruntime-node](https://www.npmjs.com/package/onnxruntime-node), [npm onnxruntime-web](https://www.npmjs.com/package/onnxruntime-web), [PyPI onnxruntime](https://pypi.org/project/onnxruntime/)
- Other CSSLab repos to know about: `maia-individual` (AGPL-3.0, personalized per-player Maia), `maia2-skill-adaptation` (Apache-2.0), `chess-similarpos` ("Retrieval of Similar Chess Positions"). — [CSSLab org](https://github.com/CSSLab)

### Inferences
- "How likely is a player of this Elo to miss this?" means calling Maia with `SelfElo` set to the user's rating and reading P(best move) or P(the blunder actually played). A low P(best) means the move is hard for that level, so the user shouldn't be penalized much. A high P(best) that the user still missed is a high-priority weakness.
- Inference options for blunderbot, in order of simplicity:
  - **(a)** Python worker running `maia2` (MIT, the cleanest license for a closed app) or `maia3` (AGPL code; fine if unmodified or if you publish modifications) on CPU, batching all positions of a game in one call.
  - **(b)** Export to ONNX and run in `onnxruntime-node` inside the Bun/TS worker, using the Maia platform's `maia3_simplified.onnx` as the reference. Check the license first: the platform repo is GPL-3.0, and the weights have no stated license.
  - **(c)** Run it in the browser via onnxruntime-web, as maiachess.com does, for interactive lesson pages.
- Size: a 23M-param model is small enough that CPU batch inference over ~80 positions/game should take seconds, not minutes. GPU isn't required for per-user import volumes. This is an inference from model size; there's no published benchmark.
- Licensing caution: Maia-1 weights sit in a GPL-3.0 repo, Maia-3 code is AGPL-3.0, and Maia-3 HF weights carry no explicit license. Contact CSSLab before commercial use of Maia-3 weights.

### Gaps
- No published CPU/GPU inference latency for Maia-2 or Maia-3. You'd need to benchmark locally.
- No official ONNX export script or published ONNX artifact license. The only ONNX found lives inside the GPL-3.0 platform repo.
- I found no lc0 BLAS CPU speed numbers for the Maia-1 nets (6–12 block nets).

## Q6. Mixing the TS app (Bun, Hono, tRPC, Drizzle, Postgres) with Python ML: job queues, workers, storage

### Takeaway
pg-boss (MIT, v12.37.1) is the best fit. It officially supports Bun, runs on the existing Postgres (no Redis), has a Drizzle adapter for enqueueing inside transactions, and exposes "SQL support for non-Node.js runtimes". That lets a Python worker claim jobs from the same queue tables. Add pgvector (PostgreSQL License, v0.8.7) for position embeddings.

### Cited Findings
- pg-boss 12.37.1 (published 2026-10-08), MIT.
  - Requirements: "Node 22.12 or higher, or Bun"; PostgreSQL 13+.
  - Features: built on `SKIP LOCKED`; "Create jobs in an existing db transaction, including adapters for popular ORMs such as Drizzle…"; LISTEN/NOTIFY low-latency delivery; job dependency workflows; cron/RRULE scheduling; priority queues; dead-letter queues; retries with exponential backoff; rate-limit/debounce/concurrency queue policies; "SQL support for non-Node.js runtimes for most operations".
  - Extras: `@pg-boss/dashboard` and `@pg-boss/proxy` (HTTP proxy). It can also run embedded on PGlite.
  - — [pg-boss README](https://github.com/timgit/pg-boss), [npm pg-boss](https://www.npmjs.com/package/pg-boss)
- Graphile Worker 0.18.0 (2026-09-08), MIT: "Job queue for PostgreSQL running on Node.js". The README doesn't mention Bun. — [graphile/worker](https://github.com/graphile/worker), [npm](https://www.npmjs.com/package/graphile-worker)
- BullMQ 6.3.11 (2026-10-01), MIT. It's Redis-backed, which would add a Redis dependency. — [npm bullmq](https://www.npmjs.com/package/bullmq)
- pgvector v0.8.7 is distributed under a PostgreSQL-style license ("Portions Copyright (c) 1996-2026, PostgreSQL Global Development Group"). GitHub's API reports it as NOASSERTION. — [pgvector LICENSE](https://github.com/pgvector/pgvector/blob/master/LICENSE), [pgvector repo](https://github.com/pgvector/pgvector)
- `@electric-sql/pglite` 0.5.8 (Apache-2.0) is embedded Postgres, useful for tests. — [npm pglite](https://www.npmjs.com/package/@electric-sql/pglite)
- Data volumes to plan for:
  - Lichess analysis JSON gives one eval per ply plus best/variation/judgment on mistakes ([live test above](https://lichess.org/api/games/user/german11)).
  - Chess.com gives clocks in PGN ([live test](https://api.chess.com/pub/player/hikaru/games/2026/09)).
  - The puzzle CSV has 6.16M rows ([database.lichess.org](https://database.lichess.org/#puzzles)).
- Prior art for embeddings: CSSLab `chess-similarpos` ("Retrieval of Similar Chess Positions"). — [CSSLab/chess-similarpos](https://github.com/CSSLab/chess-similarpos)

### Inferences
- Proposed pipeline, as pg-boss queues:
  1. `import.user` (TS/Bun): stream Lichess NDJSON or Chess.com monthly JSON. Upsert `games`. Enqueue `analyze.game` per new game, in the same Drizzle transaction.
  2. `analyze.game` (worker pool of Stockfish processes): per-position fixed-node eval. Write `positions` rows (fen, ply, eval_cp/mate, best_uci, pv, clock_cs). Skip or seed positions using Lichess `analysis`, cloud-eval, or the eval dump.
  3. `detect.mistakes`: compute win-probability loss per move. Insert `mistakes` (position_id, played, best, wp_loss, severity, phase, time_left).
  4. `classify.motif` (Python, python-chess): tag fork/pin/hanging-piece/back-rank/etc. Align the vocabulary with Lichess puzzle theme names (from `puzzleTheme.xml`) so user mistakes map directly onto CC0 puzzles.
  5. `predict.human` (Python Maia-2/3, or onnxruntime-node): store P(best | user Elo) and P(played | user Elo).
  6. `aggregate.weaknesses`: per user × motif × phase × time-control rates.
  7. `generate.lesson`: pick puzzles from the user's own mistakes plus the Lichess bank, then schedule SRS (e.g., an FSRS/SM-2 `review_cards` table).
- The Python side can either poll pg-boss tables via its SQL API, or expose a FastAPI `/infer` batch endpoint that a TS worker calls. The first avoids an extra network service. The second keeps the queue logic in TS.
- Suggested core tables:
  - `games` (source, external_id unique, pgn, time_control, user_color, ratings, played_at)
  - `positions` (game_id, ply, fen, eval, best, pv jsonb, multipv jsonb, clock)
  - `mistakes`
  - `motif_tags` (mistake_id, tag)
  - `user_weakness_stats`
  - `puzzles` (Lichess import, themes text[] GIN)
  - `review_cards`
  - `position_embeddings vector(N)` (pgvector HNSW) for "show me similar positions".
  - Key evals by normalized FEN (first 4 fields) so they dedupe across users.

### Gaps
- I didn't verify whether Graphile Worker runs under Bun, or whether pg-boss's Python/SQL interface is documented well enough to claim jobs from Python without the Node client. Check pgboss.io docs before relying on it.
- I found no published Postgres schema for this exact use case. The schema above is my design inference.

## Q7. Syzygy tablebases: sizes, and the Lichess tablebase API

### Takeaway
Self-hosting 3-4-5-piece Syzygy (~1 GB) is trivial and covers most practical endgame "missed win/draw" detection. 6-piece is ~160 GB and 7-piece ~18.4 TB. Lichess's public tablebase API (now at `tablebase.lichess.org`) serves up to 7-piece (partial 8-piece via "Op1"). It returns 14-day cacheable responses, but no rate limits are published.

### Cited Findings
- Syzygy (DTZ+WDL) sizes by piece count, per the table on Wikipedia:
  - 3-piece: 25.8 KB
  - 4-piece: 4.3 MB
  - 5-piece: 979.6 MB, so ≤5 pieces totals ≈ 0.98 GB
  - 6-piece: 160.2 GB
  - 7-piece: 18.4 TB (generated 2018 by Ronald de Man and Bojun Guo)
  - The same table lists the new Prophet TB DTM tables (6-man ~1.2 TB, 2026). Lichess adopted Prophet as its DTM source, replacing Gaviota.
  - — [Wikipedia: Endgame tablebase](https://en.wikipedia.org/wiki/Endgame_tablebase), [lila-tablebase commit](https://github.com/lichess-org/lila-tablebase/commit/6d2cd6a0bd8908a0b59bcd9aede3a20a04f89780)
- The API spec now says: "The hostname for these endpoints is `tablebase.lichess.org` and not `lichess.org`". Endpoints are `/standard`, `/atomic`, `/antichess`, and `/standard/mainline`. Params are `fen` (underscores allowed) and `dtc=never|auxiliary|always`. No auth. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml), [tablebase/standard.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/tablebase/standard.yaml)
- Response fields:
  - Position level: `dtz`, `precise_dtz`, `dtm`, `dtc`, `dtw`, `checkmate/stalemate/insufficient_material`, and `category` (win, unknown, syzygy-win, maybe-win, cursed-win, draw, blessed-loss, …).
  - `moves[]` best-first, each with uci/san/dtz/dtm/zeroing/category.
  - `/standard/mainline` returns the DTZ mainline, or 404 for non-TB positions.
  - The README still shows the old host `tablebase.lichess.ovh`.
  - — [lila-tablebase README](https://github.com/lichess-org/lila-tablebase)
- Live test: `https://tablebase.lichess.org/standard?fen=…` returned 200 with `cache-control: max-age=1209600` (14 days). Moves now also include a `conversion` boolean. — live test
- lila-tablebase is AGPL-3.0, built on the Rust `shakmaty-syzygy` crate. A "partial 8-piece tablebase" (Op1) blog post is linked from the README. — [lila-tablebase](https://github.com/lichess-org/lila-tablebase)
- Rate limits: the general Lichess policy applies (one request at a time, back off 60 s on 429). I found no tablebase-specific quota. — [lichess-api.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/lichess-api.yaml)

### Inferences
- Load the ≤5-piece Syzygy files (~1 GB) on the analysis workers. Stockfish's `SyzygyPath` option and python-chess's `chess.syzygy` can both probe them. For 6–7-piece positions, call the Lichess API sparingly and cache aggressively; the 14-day `max-age` suggests caching is expected. That makes "you had a tablebase win and played a drawing move" lessons exact.
- Self-hosting 6-piece (160 GB SSD) is feasible later if endgame lessons become a focus.

### Gaps
- No published numeric rate limit for `tablebase.lichess.org`.
- I didn't verify the exact current size of the ≤5-piece download set from syzygy-tables.info, because the site blocked automated access.

## Q8. LLM integration for natural-language explanations (short)

### Takeaway
Ground the LLM in structured engine facts and have it only verbalize them; it should never search for the analysis itself. Inputs: FEN, the move played, best move plus PV (SAN), win-probability loss, motif tags, Maia P(best | Elo), and clock. A ~1.5k-input / ~200-output explanation costs about $0.00025 on the cheapest current Claude model and about $0.005–0.01 on mid/top tiers, halved again via the Batch API for offline lesson generation.

### Cited Findings
- Claude pricing per 1M tokens, input/output, from the Claude API skill's model table cached 2026-10-06:
  - Claude Haiku 5.5 (`claude-haiku-5-5`): $0.10 / $0.50 for prompts ≤100K tokens.
  - Claude Sonnet 5.5 (`claude-sonnet-5-5`): $2 / $10.
  - Claude Opus 5.5 (`claude-opus-5-5`): $4 / $20.
  - The Batch API runs asynchronously at 50% of the cost. Cache reads cost ~0.1× input, and cache writes ~1.25×.
  - — [Anthropic pricing docs](https://platform.claude.com/docs/en/about-claude/pricing)
- Lichess already ships templated mistake text ("Mistake. O-O was best.") plus the PV in `analysis[].judgment` and `variation`, and the `literate=true` PGN option. These can serve as a non-LLM baseline. — [api-games-user-username.yaml](https://github.com/lichess-org/api/blob/master/doc/specs/tags/games/api-games-user-username.yaml), live test

### Inferences
- Per-explanation cost, my arithmetic for 1,500 input + 200 output tokens, excluding thinking tokens:
  - Haiku 5.5 ≈ $0.00025
  - Sonnet 5.5 ≈ $0.005
  - Opus 5.5 ≈ $0.010
  - Batch roughly halves each. Caching a fixed system prompt plus motif glossary cuts input cost further.
  - 1,000 explanations ≈ $0.25 (Haiku) to $10 (Opus).
- Pattern:
  - Precompute every chess fact deterministically (engine, python-chess motif detectors, Maia).
  - Pass them as a JSON block.
  - Instruct the model to reference only provided moves and lines, and return structured output (`{headline, explanation, key_idea, related_motif}`).
  - Validate any SAN it mentions against legal moves using chess.js before display.
  - Generate explanations lazily, the first time a lesson card is shown, or in nightly batches.

### Gaps
- No measured hallucination rates for LLM chess explanations grounded in engine data. You'd need an eval set with SAN-legality and claim-consistency checks.
