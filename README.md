# Tiny Chess — a 4 KB Python chess engine

A compact chess engine for the make.fun `tiny-chess-01`
competition ("build the strongest chess engine in 4 KB of
Python"). It plays the house gauntlet through the `agent/v1`
env protocol: it asks the env for the game in play, searches
the position, and sends its move back in UCI.

## How it plays

- **Board:** 0x88 (128 squares), signed pieces, incremental
  evaluation so make/unmake is cheap.
- **Search:** negamax alpha-beta with iterative deepening
  (depths 1–7), principal-variation search (zero-window
  re-searches on moves after the first), a cumulative node
  budget of 200,000 nodes
  per move (no clock — trials replay byte-for-byte),
  quiescence over captures and promotions (and every reply
  to a check) with delta pruning, a transposition table
  (Zobrist hashing, 2^18 entries), killer moves, MVV-LVA capture ordering,
  and null-move pruning.
- **Evaluation:** material plus the standard Michniewski
  piece-square tables (precomputed into a lookup table at
  startup), a bishop-pair bonus, and a draw score for
  repetitions inside the search.
- **Time:** resigns when the root score is below −1600 so
  lost games do not burn the trial wall clock.

## Files

- `main.py` — the entrypoint (2,614 bytes). It is the packed
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

- Baseline (Michniewski tables, 100k nodes/move): Elo 1253.
- Previous engine (cumulative 200k-node budget + delta
  pruning): trial-8 Elo 1396, trial-9 Elo 1326, mean 1361 —
  above the ~1325 house baseline (sunfish). Practice floor
  to enter the final is 1154.
- Current engine (PVS + depth 7 + 2^18-entry TT): trial-10
  (instance 0, the graded instance) Elo 1396, 3W 2D 3L —
  beats the 1326 trial-9 baseline on the graded instance.
  Trial-11 (instance 2) was interrupted by a runtime
  restart and did not complete. Trial-10 wall 1736s of the
  1800s budget — close to the wall, watch it.

## Verification

Move generation is checked against exact perft counts on
five positions (depths 1–3), make/unmake round-trips and
the incremental evaluation and Zobrist hash against direct
recomputation over random games, and the search finds a
mate-in-1 (Qxf7) in the test position.
