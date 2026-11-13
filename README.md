# Tiny Chess — a 4 KB Python chess engine

A compact chess engine written for the make.fun `tiny-chess-01`
competition ("build the strongest chess engine in 4 KB of Python").
It plays the house gauntlet through the `agent/v1` env protocol:
it asks the env for the game in play, searches the position, and
sends its move back in UCI.

## How it plays

- **Board:** 0x88 (128 squares), signed pieces, incremental
  evaluation so make/unmake is cheap.
- **Search:** negamax alpha-beta with iterative deepening
  (depths 1–6), a fixed node budget per depth (no clock — the
  trials replay byte-for-byte), quiescence over captures and
  promotions (and every reply to a check), a transposition table
  (Zobrist hashing), killer moves, MVV-LVA capture ordering, and
  null-move pruning.
- **Evaluation:** material plus the standard Michniewski
  piece-square tables (precomputed into a lookup table at
  startup), a bishop-pair bonus, and a draw score for repetitions
  inside the search.
- **Time:** resigns when the root score is below −1600 so lost
  games do not burn the trial wall clock.

## Files

- `main.py` — the entrypoint (2,742 bytes). It is the packed
  form of `src.py`, produced by the kit's `makefun compress`
  (an LZMA stream plus the `makefun-compress/v1` stub). It is
  the only file the engine reads or runs.
- `src.py` — the readable source the packed entrypoint was
  built from. It is kept beside the entry for the transcript
  gate (`makefun bundle` sends it under `packs/`); it is not
  committed, so it does not count against the snapshot cap.
- `LICENSE` — MIT. `.makefun/owner.json` — the entry wallet.

## Running it

The engine needs the env URL:

```sh
MAKEFUN_ENV_URL=http://127.0.0.1:8080 python3 -P -s main.py
```

It loops on `POST /game`, plays every game to the end, and
exits when the env reports `done`. It uses only the Python
standard library.

## Rebuilding the entrypoint

After any change to `src.py`, re-pack it as a command of its
own (the transcript gate requires the compress run to come
after the last write to the source):

```sh
node <kit>/cli/makefun.mjs compress src.py --out main.py
```

then commit `main.py`.

## Local results (practice instances)

- trial-1 (formula piece-square tables): Elo 989
  (1 win, 1 draw, 6 losses).
- trial-2 (Michniewski tables): Elo 1253
  (2 wins, 2 draws, 4 losses — including a checkmate win over
  the nodes-100k house engine and a draw against nodes-2M).

The practice floor to enter the final is 1154; the house
baseline (sunfish's 4 KB engine) averages about 1325.

## Verification

Move generation is checked against exact perft counts on
five positions (startpos, Kiwipete, en-passant/promotion,
castling, and a promotion position, depths 1–3), make/unmake
round-trips and the incremental evaluation (including the
bishop-pair bonus) and Zobrist hash are checked against
direct recomputation over random games, and the search finds
a mate-in-1 (Qxf7) in the test position. The engine reaches
depth 4 in a typical middlegame at the 100,000-node budget
and depth 5–6 in quieter positions.
