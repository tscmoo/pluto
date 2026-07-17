# Pluto

Pluto is a StarCraft: Brood War AI. A single neural network, trained by
self-play reinforcement learning, plays the whole game — macro and micro,
all three races; there is no scripted play. It plays 1v1 melee through
[BWAPI](https://bwapi.github.io/).

**Status: binary beta.** This repo currently hosts binary releases for
testing — grab the latest from the [Releases](../../releases) page.

## Requirements

- StarCraft: Brood War 1.16.1 + BWAPI 4.4.0
- An NVIDIA GPU (GeForce GTX 10-series / Pascal or newer)
- NVIDIA driver 525 or newer (late 2022; ships CUDA 12.0)
- ~2.5 GB free GPU memory, ~1.2 GB disk space
- 64-bit Windows, or Wine (StarCraft is 32-bit; the inference engine is
  a separate 64-bit process)

## Install

Unpack the release zip into your StarCraft folder's `bwapi-data/AI/`
(the usual place for bots), so the files land like this:

```
bwapi-data/AI/pluto.dll                 32-bit BWAPI module
bwapi-data/AI/pluto/pluto_infer.exe     64-bit GPU inference engine
bwapi-data/AI/pluto/pluto_weights.bin   network weights
```

Point bwapi.ini at the DLL as usual: `ai = bwapi-data/AI/pluto.dll`

Register the bot as Zerg, Terran, Protoss, or Random — it plays all of
them.

The binaries are unsigned, so Windows SmartScreen or antivirus software
may warn on first run.

## How it runs

On game start, pluto.dll launches pluto_infer.exe (found next to the
DLL) and talks to it over stdio pipes. No network connections are made
and no files outside the StarCraft folder are touched. The engine
process exits with the game.

If the engine cannot start (no NVIDIA GPU, driver too old, missing
weights), the bot reports the reason in game chat and in the logs, then
leaves the game after ~10 seconds.

### Files written

| file | what |
| --- | --- |
| `pluto.log` | DLL log (StarCraft folder) |
| `pluto_infer.log` | engine log (StarCraft folder) |
| `bwapi-data/write/pluto_bandit_<opponent>.txt` | per-opponent build-order stats |

The bot picks its opening with a bandit over the per-opponent win/loss
counts; every build is listed with its mask and name, so you can edit
the numbers to steer it (the file is re-read every game). The standard
tournament write/ → read/ copy persists it across games.

## Notes for tournament hosts

- The bot uncaps game speed and paces itself by inference (~15 ms per
  model step on a modern GPU; one step every 6 frames).
- Games of the same matchup must not run in parallel on one install
  (standard tournament-manager behavior). Different opponents in
  parallel are fine.
- A mid-game engine failure exits the StarCraft process — it counts as
  a crash, deliberately, rather than leaving the game as a loss.
