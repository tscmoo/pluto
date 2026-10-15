# Pluto

Pluto is a StarCraft: Brood War AI. A single neural network, trained by
self-play reinforcement learning, plays the whole game — macro and micro,
all three races; there is no scripted play. It plays 1v1 melee through
BWAPI (https://bwapi.github.io/).

**Status: binary releases.** This repo hosts binary releases of Pluto.
The current one is **Pluto v2.0** (model `md09b_1331000`), with CPU
inference — no GPU needed. Grab it from the Releases page. Earlier
releases: the CoG 2026 competition entry (`cog2026-2578600`) and beta 1
(`beta-2247200`).

## Requirements

- StarCraft: Brood War 1.16.1 + BWAPI 4.4.0
- 64-bit Windows, or Wine (StarCraft is 32-bit; the inference engine is
  a separate 64-bit process)
- An x86-64 CPU with AVX2 (Intel Haswell 2013+ / any AMD Zen). AVX-VNNI
  (Intel 12th gen 2021+ / AMD Zen 5) roughly doubles inference speed
  and is strongly recommended.
- 6+ cores recommended; ~2 GB free RAM, ~0.4 GB disk space

## Install

Unpack the release zip into your StarCraft folder's `bwapi-data/AI/`
(the usual place for bots), so the files land like this:

    bwapi-data/AI/pluto.dll                 32-bit BWAPI module
    bwapi-data/AI/pluto/pluto_infer.exe     64-bit CPU inference engine
    bwapi-data/AI/pluto/pluto_weights.bin   network weights (int8)

Point bwapi.ini at the DLL as usual: `ai = bwapi-data/AI/pluto.dll`

When upgrading, replace all three files: don't mix them with files from
an earlier release.

Register the bot as Zerg, Terran, Protoss, or Random — it plays all of
them. It announces itself in chat on game start:
"Pluto online (v2.0 md09b_1331000). gl hf!"

The binaries are unsigned, so Windows SmartScreen or antivirus software
may warn on first run.

## How it runs

On game start, pluto.dll launches pluto_infer.exe (found next to the
DLL) and talks to it over shared memory. No network connections are
made and no files outside the StarCraft folder are touched. The engine
process exits with the game.

The model takes one step every 6 game frames. The DLL gives the engine
up to 40 ms in the frame it observed and, if the answer isn't in by
then, waits for it at the next frame — so every decision is acted on
exactly when the model was trained to expect it. On a slow CPU the game
runs slower; the bot does not play worse. A step takes ~20 ms on a
current desktop CPU and ~60 ms on a 2021 6-core (i5-12500T) with 200
units on the map, up to ~3x that in big late-game fights. When the
stalls stay long for a sustained stretch, Pluto says so once in chat
("Pluto: this machine can't keep up ... sorry about the pace") and notes
it in its game record.

If the engine cannot start (missing files, unsupported CPU) or dies
mid-game, the bot reports the reason in game chat and in the logs
("Pluto: quitting in 10 seconds"), then exits the StarCraft process.
This looks like a crash, and is meant to: it is an error, not a game.
Check pluto.log and pluto_infer.log in the StarCraft folder.

### Files written

    pluto.log          DLL log     (StarCraft folder)
    pluto_infer.log    engine log  (StarCraft folder)
    bwapi-data/write/pluto_bandit_<opponent>.txt
                       per-opponent build-order stats and the running
                       record against that opponent

The bot picks its opening with a bandit over the per-opponent win/loss
counts; every build is listed by name, so you can edit the numbers to
steer it. The file is re-read every game (from write/ or
read/, whichever is newer, so the usual write/ -> read/ copy between
games works too); if you edit it, bump the `seq` line so your copy wins
and keep the closing `end` line. Below the stats, the file keeps one
JSON line per game: a start record (opponent, map, chosen build), an
end record (result, frame timing, the model's win-probability
trajectory) and any error or resign events.

Stats files written by releases before v2.0 use an older format and are
not read: that opponent starts fresh, and the first v2.0 game against
it replaces the file. Copy it elsewhere first if you want to keep the
old records.

## Notes

- The bot uncaps game speed and paces itself by inference (one model
  step every 6 frames, ~20-60 ms per step on the hardware above).
- Optional: run `pluto_infer.exe --bench` once per machine (~2-4 min).
  It times the thread counts on that hardware and writes the best one
  to `pluto/pluto_config.json`, which the engine then uses. Without it
  the engine uses up to 6 threads.
- `BWRL_DRAW=1` in the environment draws the model's win-probability
  graph on screen (off by default; nice for watching games).
- Pluto resigns: when its own win estimate stays at -0.90 or below
  (on a -1 to 1 scale) for ~10 seconds, it says "gg" and leaves the
  game. `BWRL_RESIGN_THRESHOLD=<number between -1 and 0>` moves the
  threshold (e.g. `-0.95` resigns later) and `BWRL_RESIGN_THRESHOLD=off`
  turns resigning off. A mid-game engine failure exits the StarCraft
  process instead — it counts as a crash, deliberately, rather than as
  a loss.
- Two games against the same opponent must not run in parallel on one
  install (they share the stats file above; tournament managers already
  avoid this). Different opponents in parallel are fine.
- Under a tournament manager with a per-frame time limit (the usual
  55 ms rule): the 40 ms budget is chosen for it — a step that finishes
  within the budget plus one frame period costs no frame more than the
  period, and only the overflow lands in one longer frame at the next
  step. On hardware where that still draws timeouts, set
  `BWRL_STRADDLE=1`: the DLL then never holds a frame past the budget,
  and a late decision slips to a later frame instead (the competition
  behaviour; the bot decides less often). `BWRL_FRAME_BUDGET_MS=<ms>`
  changes the budget for either mode.
