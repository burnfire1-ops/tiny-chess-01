# Tiny Chess 01 — round 2

A 0x88 alpha-beta chess engine in Python (stdlib only, deterministic).

- Iterative deepening (depths 1-10) with a fixed per-move node budget,
  alpha-beta search, quiescence search (captures/promotions, all evasions
  in check).
- Transposition table (2^16 slots, 64-bit Zobrist key, best move, depth,
  score, bound), killer moves (2 per ply); move order is hash move, then
  captures by MVV-LVA, then killers.
- Evaluation: material plus piece-square tables (single king table),
  updated incrementally on make/unmake.
- Root move choice uses a stable sort by score only: ties keep the
  previous iteration's order (best move first). Sorting (score, move)
  tuples instead let double pawn pushes win ties, because the move
  integer carries flag bits (double-push = 1<<20); combined with the
  narrow-window root recording bounds as scores, that picked g4/f4/h4
  and lost every game by checkmate. The same stable-sort fix is applied
  to the in-search move ordering.
- Driver: talks to the env over HTTP (`/game`, `/move`) and falls back to
  the env's `legal_moves` if the search ever fails, so a game is never
  left unplayed.

Measured on this machine: ~40,000 nodes/s in middlegame positions, so the
150,000-node per-move budget runs about 3-4 s a move, well inside the
10 s move limit and the 1,800 s trial wall budget.

Build: `makefun compress main.src.py` produces the packed `main.py`.
