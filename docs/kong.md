# KONG CLIMB

A single-file HTML5 tribute to the 1981 girder-climbing arcade classic — built from
scratch. It recreates the *mechanics* of the iconic first stage (25 m): run, jump,
climb, dodge barrels, grab the hammer, rescue the captive at the top.

All pixel art, the girder layout, and every sound are original creations. No ROMs or
original game assets were used or extracted.

## Run it

```sh
python3 -m http.server 8123 --directory kong-climb
# open http://localhost:8123/kong.html
```

(Opening `kong.html` directly also works.)

## Controls

| Key | Action |
| --- | --- |
| ← / → | Run |
| ↑ / ↓ | Climb ladders |
| SPACE or Z | Jump (fixed arc, no mid-air steering — like the original) |
| ENTER | Start / restart |
| P | Pause |
| M | Mute sound |

## Gameplay (matches the 1981 spec)

- The ape hurls barrels that zigzag down the girders, randomly taking ladders on the
  way down. Every 6th barrel is *wild* — faster, and it descends every ladder.
- **Jump over a barrel: 100 pts**, doubling per chain (200/400/800/1600).
- **Hammer** (two pickups per screen, ~10 s): smash barrels for **300**, fireballs
  for **500**. You cannot climb ladders while holding it.
- **Oil drum** at the bottom left ignites burned barrels; every third burn releases a
  **fireball** that stalks you across girders and ladders (more appear on later levels).
- Items left on the girders: hard hat **100**, lunch box **200**, toolbox **300**.
- **Bonus timer** starts at 5000 and drains; reaching 0 costs a life, clearing the
  stage banks it as score.
- 3 lives, extra life at 7000 (then every 30000). Each level: faster barrels, more
  ladder descents, more fireballs. High score persists in `localStorage`.
- Rescue: climb the broken-off ladder to the top platform and reach the captive —
  the girders flash, the ape drops, next level.

## Tech

- Single `index.html`, zero dependencies, canvas at 224×256 (the classic arcade
  resolution) scaled 3× with crisp pixels.
- Fixed 60 Hz simulation step with rAF rendering.
- WebAudio-synthesized sound effects (no audio files): footsteps, heartbeat that
  quickens as the bonus drains, jump, smash, ignition, death, jingles.
- `window.KC` exposes a small debug/testing hook (state, player, barrels, spawners).

## Verification

Tested end-to-end in a headless browser: movement, jumping, ladder climbing, barrel
physics (roll → edge bounce → girder-end drop → ladder descent), jump-over scoring
chains, hammer smashing, drum ignition and fireball spawning/AI, death/respawn,
stage-clear cutscene with bonus tally, level progression, game over, and hi-score
persistence — all with zero console errors.
