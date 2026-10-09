# Space Invaders: design

**Date:** 2026-10-09
**Status:** Approved in conversation (all four sections); awaiting written-spec review

## Intent

The repo is becoming a collection of faithful, single-file recreations of classic arcade games. Asteroids
(`asteroids.html`) is the first. This adds the second: a recreation of the 1978 Taito/Midway *Space Invaders*
in `invaders.html`, built "in a similar way":

- One self-contained HTML file: plain JavaScript on a `<canvas>`, no libraries, nothing to install or build.
- As faithful to the arcade as Asteroids is: behaviour, timings and graphics are taken from the ROM disassembly
  wherever possible. Anything approximated or left out is listed as such in the README.
- Same code style and layout as `asteroids.html`.

**Success means:** someone who knows the arcade game recognises it at once, in looks, sound, rhythm and quirks
such as the speed-up, the 300-point saucer shot and shields being bitten away. Someone reading the code finds it as
approachable as the Asteroids code, with constants they can check against the ROM.

## Reference material

Two appendices hold the exact values, with a ROM address or source for each one:

- [`2026-10-09-space-invaders-rom-facts.md`](2026-10-09-space-invaders-rom-facts.md): a fact sheet covering
  coordinates, timing, the rack, the player, alien shots, the saucer, shields, scoring, sound triggers, the HUD,
  the colour overlay and attract mode.
- [`2026-10-09-space-invaders-rom-graphics.md`](2026-10-09-space-invaders-rom-graphics.md): every sprite, the
  shield and the font, as raw ROM bytes.

The data was rebuilt from computerarcheology.com's commented disassembly. Its CRC32s match MAME's `invaders`
ROM set exactly. Values in this spec are given in portrait screen pixels (x right 0–223, y down 0–255) and in
frames at 60 Hz. Where this spec and the fact sheet disagree, the fact sheet wins.

## Scope

**In:** one-player game, attract mode with demo, all gameplay rules below, synthesized sound, high score saved in
the browser, colour overlay, README rebuilt for the collection.

**Left out**, and listed as such in the README:
- two-player mode
- coins and the settings switches; the game uses the default settings: 3 cannons, extra cannon at 1,500
- the hidden "TAITO COP" message
- the rolling shot's column-finding bug when the reference alien's H ≥ $80 (WrapRef $1590)
- the alternate attract cycle's gags: the upside-down Y in "PLAy", and "INSERT CCOIN"

## 1. Structure and screen

### File layout

`invaders.html` mirrors `asteroids.html`:
- A short `<style>` block, the canvas, and a one-line key reminder underneath.
- The script starts with constants in arcade units, each commented with its ROM source.
- Sections marked with `// --- NAME ---` comments:
  - graphics
  - game state
  - controls
  - sound
  - helpers
  - screen
  - game flow
  - player
  - player shot
  - invaders
  - alien shots
  - saucer
  - collisions
  - attract mode
  - game loop
  - rendering
- Game state is held in top-level variables, so it can be inspected and changed from the browser console. That
  includes `step()`, which advances exactly one frame.

### Graphics data

Sprites and the font are transcribed from the ROM as hex strings, one byte per column with bit 7 at the top, in
the same order as the ROM. The shield is 2 bytes per column, lower byte first. This keeps the data compact and
easy to check against the disassembly. It is the same idea as Asteroids' `FONT_DATA` strings.

### The screen

- The screen is a persistent 224×256 framebuffer (`Uint8Array`, one byte per pixel), never cleared between
  frames, just like the arcade's video RAM.
- Each moving object erases its previous image and draws its new one. Shields are just pixels that stay on
  screen.
- Drawing helpers copy the arcade's routines:
  - `drawSprite`: ORs a sprite in and returns whether any target pixel was already lit. This is the collision
    test, as in DrawSprCollision $1491.
  - `writeSprite`: overwrites a block, as in DrawSprite $15D3. Invaders use it, which is why they erase the
    16×16 block they move through.
  - `eraseSprite`: AND-NOT, as in EraseShifted $1452.
  - `clearRect`.
  - `drawText`: byte-aligned 8×8 characters.
- When a collision happens, the game works out what was hit from where it happened (saucer row, rack area,
  another shot, or anything else such as a shield or the green line), following the ROM's rules (fact sheet §3–5).

### Display

- On each display refresh, the framebuffer is converted to an image, and each lit pixel is coloured according to
  its position:
  - **red** for y 32–62;
  - **green** for y 184–239 at full width, and for y 240–255 only within x 16–133;
  - **white** everywhere else.
- These band boundaries come from MAME's `invaders.lay` layout. They are not in the ROM, so the README lists
  them as approximated.
- The image is drawn onto the visible canvas with image smoothing off, so pixels stay sharp at any size.
- The canvas keeps the arcade's portrait 7:8 aspect ratio and is sized to fit the window, the same way the
  Asteroids canvas is.

### Timing and input

- Fixed 60 Hz simulation steps, drawn at the display's refresh rate, using the same loop as Asteroids. The real
  hardware ran at 59.54 Hz; we use 60.
- **Keys:** ← → (or A D) move, Space fires, Enter (or Space) starts, M toggles sound.
- **Fire:** a shot needs a fresh press, so fire must be released between shots, as in the arcade.
- **Both directions held:** right wins, as in the arcade.
- **High score:** saved under the `localStorage` key `invadersHighScore`, separate from Asteroids.

## 2. Gameplay rules

The values come from the fact sheet, which gives the ROM source for each.

### Rack

- **Layout:** 5 rows × 11 columns.
  - Bottom two rows: octopus, 10 points.
  - Middle two rows: crab, 20 points.
  - Top row: squid, 30 points.
- **Cells:** 16 px apart both across and down. The cell for column c, row r (row 0 at the bottom) is at
  x = refX + 16c, y = refY − 16r.
- **Starting position:** every rack starts at x = 24.
  - The bottom row's y by round is: 128, 152, 168, 176, 176, 176, 184, 184, 184.
  - From round 10 the list repeats from its second entry: rounds 10–17 match rounds 2–9.
- **Building:** a new rack builds itself one invader per frame, bottom-left to top-right.
- **Marching:**
  - One live invader is redrawn per frame, at the current reference position, with the current animation frame.
  - When the cursor wraps past the last invader, the reference position moves by the step and the animation frame
    flips.
  - So with N live invaders a full step takes N frames.
  - The step is 2 px. If exactly one invader is left, a bounce off the left wall makes its rightward step 3 px.
- **Edges:** found by testing the framebuffer for lit pixels in screen column x = 213 when moving right, or x = 9
  when moving left, over y 40–223. A hit reverses direction from the next step and adds an 8 px drop to that step.
- **Freezes:** the rack freezes for the 16 frames of an invader explosion, and while the cannon is exploding or
  waiting to reappear.
- **Reaching the bottom:** if an invader reaches the cannon's row (y ≥ 216), the game is over at once, even with
  cannons in reserve.

### Cannon and shot

- **Movement:** 1 px per frame, cell x from 16 to 185, at y 216. Each new cannon starts at x 16.
- **Shot:**
  - Only one at a time. It starts at cell x + 8, y 212, and moves up 4 px per frame.
  - You can't fire while an invader explosion is showing, or while the cannon is dead or not yet back.
- **Shot explosion:** at the top of the screen, and on hitting a shield or a bomb, the shot's 8×8 explosion lasts
  16 frames. It erodes whatever it overlaps.
- **Cannon explosion:**
  - It lasts 60 frames: the intact cannon for 5 frames, then the two explosion images alternating every 5 frames.
  - The next cannon appears 128 frames later.
  - Bombs resume 48 frames after it appears.
  - The same 128 + 48 frame delay applies at the start of every rack, while the rack marches.

### Alien shots (bombs)

- **Types:** rolling, plunger and squiggly; at most one of each on screen.
- **Movement:** they take turns on a 3-frame cycle, so each moves once every 3 frames. Each move is 4 px, or 5 px
  once 8 or fewer invaders remain, until the next rack. The animation advances one of 4 frames per move.
- **Choosing a column:**
  - Rolling: the column above the cannon's tip.
  - Plunger and squiggly: the ROM's fixed column tables, whose pointers carry over from shot to shot.
  - The lowest live invader in the chosen column fires. If the column is empty, there is no shot that turn.
- **Reload:** a new bomb may start only when each of the other two is idle or has made more than R moves. R is 48,
  16, 11, 8 or 7, depending on the score thresholds in the fact sheet.
- **Restrictions:**
  - The squiggly can't fire while the saucer is flying.
  - Once a plunger shot finishes with one invader left, plungers stop for the rest of the rack.
  - The rolling shot skips its first chance after each reset.
- **Explosion:** the 6×8 bomb explosion is visible for 9 frames and erodes what it overlaps, including the green
  line.
- **Hitting the cannon:** a bomb that collides within the cannon's band (fact sheet §4) destroys it.
- **Shot against bomb:** when the player's shot and a bomb collide, both explode.

### Saucer

- **When it appears:** a timer sets it off every 1,536 frames. The timer runs only once the rack is below round 1's
  starting height, and it resets each rack. A saucer launches only if no squiggly is in flight and at least 8
  invaders remain.
- **Flight:** it flies along y 40–47, moving 2 px every 3 frames.
- **Direction:**
  - Bit 0 of the ROM byte at $0800 + n, where n is the player's shot count for the rack. The 256 bits are in the
    fact sheet appendix.
  - 1 means it enters at x 9 moving right; 0 means it enters at x 192 moving left.
- **Score:**
  - The k-th shot of the rack scores entry (k mod 15) of the table: 50, 50, 100, 150, 100, 100, 50, 300, 100, 100,
    100, 50, 150, 100, 100. So the 8th shot scores 300, then the 23rd, 38th and every 15th after that.
  - Shot counts carry over across lives and reset at each rack.
- **After a hit:**
  1. The explosion and the saucer-hit sound start.
  2. About 21 frames later the score text (" 50", "100", "150" or "300") replaces the explosion, and the points
     are added.
  3. About 72 frames after the hit the score text is cleared.

### Shields

- **Position:** four shields, each 22×16, at y 192–207, with left edges at x 32, 77, 122 and 167.
- **Damage:**
  - Damage comes from the explosion sprites (above).
  - Invaders erase shield pixels as they pass, because they overwrite a 16×16 block.
- **Refresh:** intact shields at every new rack. They are not restored when the cannon dies.

### Scoring, lives and rounds

- **Points:** 10, 20 or 30 per invader. The score is 4 digits and wraps past 9,999.
- **Cannons:**
  - 3 cannons, plus one extra cannon at 1,500 points, once per game.
  - The lives digit counts the cannon in play.
  - The icons show the reserve.
- **Losing the last cannon:** when its explosion ends, the game is over.
- **End of a round:** when the last invader dies, there is a 48-frame pause, or a wait for a cannon explosion to
  finish. Then the play area (y 32–239) is cleared and the next rack, the green line and fresh shields are drawn.

## 3. Screens and sound

### HUD (positions from the ROM; fact sheet §9)

- **Header:** ` SCORE<1> HI-SCORE SCORE<2> ` at y 8.
- **Scores:** 4-digit scores at y 24. P1 is at x 24 and HI at x 88. P2 is left blank, as in a one-player arcade
  game.
- **Green line:** at y 239.
- **Bottom row** (y 240):
  - the lives digit at x 8;
  - reserve cannon icons every 16 px from x 24;
  - `CREDIT 00` at x 136.

### Attract mode

This is the arcade's plain cycle, looping until Enter or Space is pressed. Pressing either starts a game from any
point in the cycle.

1. Wait 64 frames. Type "PLAY" at (96, 64), about 6 frames per letter, then "SPACE  INVADERS" at (56, 88).
2. Wait 64 frames. Show "*SCORE ADVANCE TABLE*" at (32, 120), with the saucer, squid, crab and octopus icons at x 64,
   y 136/152/168/184. Then type "=? MYSTERY", "=30 POINTS", "=20 POINTS" and "=10 POINTS" at x 80. Wait 128 frames.
3. Demo game: the rack, shields and green line, with no sound and no scoring. The cannon fires constantly and moves
   by the ROM's DemoCommands table, advancing one step per shot. The demo ends when the cannon is hit.
4. The arcade's "credit inserted" screen: "PUSH" at (96, 96) and "ONLY 1PLAYER  BUTTON" at (32, 120), for 128
   frames. Then the cycle restarts.

### Start and end of a game

- **Start:** "PLAY PLAYER<1>" at (56, 112) for 176 frames, with the P1 score blinking every 4 frames. Then the rack
  builds and play begins.
- **End:** "GAME OVER" typed at (72, 56), then a 128-frame wait. The high score is saved, and the game returns to
  attract mode at step 4 (the "PUSH" screen), as the arcade returns to its credit screen.

### Sound

- **Synthesis:** Web Audio, built the same way as in Asteroids. A `sound` object is unlocked on the first key press
  and suspended while the tab is hidden. M toggles mute, and the `sfx()` gate means nothing plays outside a game.
- **Fleet march:**
  - Four descending bass notes, cycling.
  - The interval between notes comes from the ROM's table, indexed by live invaders: 52 frames at 50 or more,
    falling to 5 frames with one left. Each note is held 4 frames.
  - The first note comes 64 frames into a rack.
  - It is silent while the cannon is dead or waiting to reappear, while the saucer-hit sound plays, and when no
    invaders remain.
- **Saucer:** a continuous warbling tone while it flies and hasn't been hit.
- **Player shot:** sounds while the shot is flying or exploding.
- **Invader hit:** at the kill, lasting until its explosion ends.
- **Player death:** from the hit until the explosion ends.
- **Saucer hit:** from the explosion until the score text clears.
- **Extra cannon:** a chime. Its length is approximated, because the ROM times it in main-loop passes rather than
  frames.
- **Accuracy:** the triggers and timings follow the ROM. The sounds themselves are imitations of the cabinet's
  analogue circuits, so the README lists them as approximated.

## 4. Docs and testing

### README (rebuilt as a collection)

- **Title and intro:** retitled **Arcade Classics**, with a short intro: faithful single-file recreations of
  classic arcade games, nothing to install.
- **Play:** a shared section. Open any game's `.html` file, or run `python3 -m http.server` and visit
  `/asteroids.html` or `/invaders.html`.
- **Games:** one section per game, each with a screenshot, controls, scoring, "How close to the arcade is it?",
  code tour and sources.
  - The existing Asteroids text moves under its own heading word for word, with its headings dropped one level.
  - The Space Invaders section follows the same pattern. Its accuracy part lists what comes from the ROM, what is
    approximated (the overlay boundaries, the sounds, 60 Hz rather than 59.54 Hz) and what is left out (as in
    Scope).
- **Screenshots:** rename `docs/screenshot.png` to `docs/asteroids.png`, and capture a new `docs/invaders.png` from
  the running game.
- **Disclaimer:** one shared disclaimer covering *Asteroids* (Atari) and *Space Invaders* (Taito), noting that the
  on-screen text is part of each recreated arcade screen.

### Testing

The repo has no test framework, and adding one would break the "nothing to install" promise. So verification
happens in a real browser via Playwright: the browser tools during development, with no test files committed,
just as Asteroids has none. Because game state and `step()` are top-level, checks set up a situation, advance an
exact number of frames and check the result:

- **Rack timing:** with 55 invaders a full step takes 55 frames; with one left it moves every frame, with a 3 px
  rightward step after a left bounce. The 8 px drop at each edge, and the starting heights for rounds 1–10, are
  also checked.
- **Saucer:** the score sequence by shot count, including 300 on shots 8 and 23, and the entry direction from the
  bit table.
- **Shields:** a bomb or shot leaves a hole matching its explosion sprite, and an invader passing over a shield
  clears its 16×16 block.
- **Rules:**
  - Invaders reaching y 216 end the game with cannons in reserve.
  - The extra cannon arrives at 1,500, once.
  - The score wraps past 9,999.
  - The high score survives a reload.
  - The bomb reload rate follows the score table.
- **Errors:** the console shows no errors through attract mode, a demo, and a full game.
- **Looks and feel:** screenshots compared by eye with arcade screenshots (layout, sprite shapes, overlay
  colours), plus a few rounds played with simulated key presses.
