# tiny-chess-01 — a 4KB chess engine

A complete chess engine in a single `main.py`, written from scratch for the
make.fun tiny-chess-01 competition. It plays games against the house
environment over HTTP and needs nothing but the Python 3.13 standard library.

## Size

`main.py` is **2,605 bytes** as committed, under the 4,096-byte limit. The
engine source is about 6.9 KB written compactly, so it is stored as a
compressed payload produced by the kit's `makefun compress` (an LZMA stream
with the makefun-compress/v1 stub) that `main.py` decompresses and executes
at startup. Compression is allowed by the competition rules and the payload
plus the unpacker together are what is measured; the readable source is kept
beside the entry as `main.src.py` for the transcript check.

## How it works

- **Board:** a 0x88 board (128 squares, off-board squares flagged by the
  `0x88` bit). Pieces are small signed integers, positive for White and
  negative for Black.
- **Move generation:** my own legal move generator, built from the FEN the
  environment serves. It covers pawn pushes, double pushes, captures,
  en-passant, all four promotions, and both castling sides for both
  colours. Pseudo-legal moves are filtered by making the move and checking
  whether the mover's king is attacked.
- **Search:** negamax alpha-beta with iterative deepening, starting at
  depth 2 up to a depth cap of 6. Depth 1 is skipped: at a 15,000-node
  budget depth 2 always completes, so the first iteration starts there.
  It runs on a **fixed node budget of 15,000 nodes per move** — no wall
  clock, no unseeded randomness — so a replayed trial plays the same
  moves. If the budget runs out mid-depth,
  the best move from the last completed depth is played.
- **Move ordering:** captures first by MVV-LVA (most valuable victim,
  least valuable attacker), promotions next, then killer moves (the two
  quiet moves that caused a beta cutoff at each ply).
- **Quiescence:** at the horizon the search continues through captures
  only, up to 6 plies deep, and only through captures that are not
  clearly losing (victim value at least attacker value minus 50, or a
  promotion). This avoids horizon-effect blunders without letting the
  capture search explode.
- **Evaluation:** material (pawn 100, knight 320, bishop 330, rook 500,
  queen 900, king 20000) plus a piece-square table for each of the six
  piece types, mirrored for Black. No transposition table: it did not fit
  the byte budget once the search and quiescence were in place.
- **Protocol:** `main.py` reads `$MAKEFUN_ENV_URL` and talks to the
  environment with `urllib` + `json`: `POST /game` to start the next
  game, `POST /move` with the chosen move in UCI (`e2e4`, `e7e8q`,
  castling as the king's move `e1g1`), `POST /resign` only as a
  fallback if a move is refused. When the environment reports `done`
  it prints wins, draws and losses and exits 0.

## How to test

The test harness lives outside the repo, in `../tests/`:

- `test_harness.py` — perft counts (start position depths 1–3:
  20 / 400 / 8902; Kiwipete 48 / 2039 / 97862; plus three more
  standard positions), a mate-in-1 check, random-position crash
  checks, and a timed middlegame move.
- `selfplay.py greedy 12` — 12 games from the six practice openings,
  both colours, against a greedy 1-ply capture-maximising opponent.
- `selfplay.py depth2 6` — 6 games against a depth-2 alpha-beta
  baseline using the same move generator and evaluation.

Run them with plain `python3`, for example:

```
python3 ../tests/test_harness.py
python3 ../tests/selfplay.py greedy 12
```

Timing on this machine (shared 1.5 CPU): a middlegame move at the
15,000-node budget takes about 0.8–1.0 s, completing depth 3 and
starting depth 4; well inside the 3 s/move average and the 10 s/move
limit.
