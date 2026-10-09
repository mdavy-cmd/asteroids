# Space Invaders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `invaders.html`, a faithful single-file recreation of the 1978 arcade *Space Invaders*, and rebuild the README as an "Arcade Classics" collection.

**Architecture:**
- A persistent 224×256 one-byte-per-pixel framebuffer stands in for the arcade's video RAM. Sprites are ORed in with a collision test, overwritten or erased, using the same rules as the ROM.
- Game logic runs in fixed 60 Hz steps and follows the ROM's task order: VBLANK tasks, then the line-96 task, then the main loop.
- Rendering converts the framebuffer to an image tinted by the overlay bands, then scales it to the canvas.

**Tech Stack:** Plain HTML, CSS and JavaScript (canvas 2D, Web Audio). No libraries. Verified with the Playwright MCP browser tools against `python3 -m http.server`.

**Spec:** [`docs/superpowers/specs/2026-10-09-space-invaders-design.md`](../specs/2026-10-09-space-invaders-design.md).
The ROM facts and graphics appendices sit next to it. The commented disassembly used to check fine details is at
`/tmp/invaders-research/dl/Code.txt` (computerarcheology.com).

## Global Constraints

- One self-contained file, `invaders.html`. No libraries, no build step, no network requests.
- Coordinates are portrait screen pixels: x runs right over 0–223, y runs down over 0–255. Timers count frames at 60 Hz.
- Code style matches `asteroids.html`:
  - 4-space indentation and `// --- NAME ---` section markers;
  - comment density similar to that file;
  - game state in top-level variables;
  - `step()` advances one frame;
  - the `sfx()` gate means sound plays only in a game.
- High score is stored under the `localStorage` key `invadersHighScore`.
- Controls:
  - ← → or A D move;
  - Space fires (a fresh press per shot, following the ROM's fire-bounce rule);
  - Enter or Space starts a game from attract mode;
  - M toggles sound.
- Overlay colours:
  - red for y 32–62;
  - green for y 184–239 at full width;
  - green for y 240–255 only within x 16–133;
  - white everywhere else.
- Left out:
  - two-player mode;
  - coins and settings switches;
  - the TAITO COP message;
  - the WrapRef column bug;
  - the attract gags.

## Review Focus

Each of these risks has a matching check added to the task that owns the code:

- **Simultaneous left and right:** holding both arrow keys must move right, as in the ROM, and must not jitter. Check added to Task 2.
- **Fire held through a shot's flight:** holding fire must not auto-repeat. A new shot needs fire released while no shot is on screen. Check added to Task 3.
- **Tab hidden, then shown:** the loop must clamp elapsed time to 250 ms (as Asteroids does) and audio must suspend. Check added to Task 6.
- **Last invader killed while the cannon is exploding:** the round must still end and the next rack appear. Check added to Task 6.
- **Window resize to odd sizes:** the canvas must keep its 7:8 shape with sharp, evenly sized pixels. Check added to Task 1.

## Shared checks setup (used by every task)

Run once per session:

```bash
cd /Users/mdavy/source/repos/asteroids && python3 -m http.server 8765   # run in background
```

Then navigate the Playwright browser to `http://localhost:8765/invaders.html`. Each check is a function passed to
`browser_evaluate`. It runs synchronously in the page, so the `requestAnimationFrame` loop can't interleave with
it. Every check returns an object of named booleans, and all of them must be `true`. To silence bombs while
testing something else, set `enableAlienFire = false; alienFireDelay = Infinity;`.

Checks rely on a helper defined in the page:

```js
// Starts a game and skips the "PLAY PLAYER<1>" prompt so game tasks are running
function testStart() { startGame(); while (!tasksOn) step(); }
```

---

### Task 1: Page, graphics data, framebuffer and rendering

**Files:**
- Create: `invaders.html`

**Interfaces:**
- **Produces:**
  - Constants: `W = 224`, `H = 256`, `FRAME_MS`.
  - `screen`: `Uint8Array(W*H)`.
  - Sprite objects `{ w, h, cols }`, where each column value has bit `h-1` at the top:
    - `SPR.invaders[type][frame]`, where type 0 = octopus, 1 = crab, 2 = squid;
    - `SPR.invaderExplosion`, `SPR.player`, `SPR.playerExplosion[0..1]`, `SPR.playerShot`, `SPR.shotExplosion`;
    - `SPR.bombs[type][0..3]`, where type 0 = rolling, 1 = plunger, 2 = squiggly;
    - `SPR.bombExplosion`, `SPR.saucer`, `SPR.saucerExplosion`, `SPR.shield`.
  - `FONT[ch]`: 8×8 sprites.
  - Drawing helpers:
    - `drawSprite(spr, x, y) → boolean`: OR in; true if any target pixel was already lit;
    - `writeSprite(spr, x, y)`: overwrite the w×h block;
    - `eraseSprite(spr, x, y)`: AND-NOT;
    - `clearRect(x, y, w, h)`;
    - `isLit(x, y)`;
    - `drawText(str, x, y)`: writes 8×8 cells, advancing 8 px per character.
  - Display: `resize()`, `render()`, and the `loop()` fixed-step driver calling `step()`.
  - HUD functions: `drawHud()` (header, scores, credit, ships), `drawScores()`, `drawShips()`.

- [ ] **Step 1: Extract the ROM bytes** from `graphics.md` into hex strings, one byte per column, using a throwaway
  script outside the repo. The shield is 2 bytes per column, lower byte first. The font is 8 bytes per character.
- [ ] **Step 2: Write the page skeleton:** CSS (portrait 7:8 canvas, sized like Asteroids), the canvas, the key
  reminder, constants, graphics data, framebuffer helpers, overlay table, `render()`, `resize()` and `loop()`.
  Also a stub `step()` that draws the HUD once.
- [ ] **Step 3: Check the helpers and decoding:**

```js
() => {
    clearRect(0, 0, W, H);
    const r = {};
    r.orNoHit = drawSprite(SPR.playerShot, 10, 10) === false;
    r.orHit = drawSprite(SPR.playerShot, 10, 10) === true;
    eraseSprite(SPR.playerShot, 10, 10);
    r.erased = !screen.some(v => v);
    writeSprite(SPR.invaders[0][0], 0, 0);
    const row = y => Array.from({ length: 16 }, (_, x) => isLit(x, y) ? "#" : ".").join("");
    r.octopusTop = row(0) === "......####......";
    r.octopusBottom = row(7) === "..##........##..";
    writeSprite(SPR.shield, 40, 40);
    r.shieldSize = SPR.shield.w === 22 && SPR.shield.h === 16;
    clearRect(0, 0, W, H);
    drawText("A", 0, 0);
    r.glyphBlankTopRow = ![0,1,2,3,4,5,6,7].some(x => isLit(x, 0));
    return r;
}
```

- [ ] **Step 4: Check the display.** Take a screenshot and confirm:
  - the HUD header, "CREDIT 00" and the green line show;
  - the overlay colours land on the right rows.

  Then resize the viewport to 500×700 and 1280×720, and confirm the canvas keeps its 7:8 aspect ratio with sharp,
  even pixels (Review Focus).
- [ ] **Step 5: Commit:** `git add invaders.html && git commit -m "Add Space Invaders page, ROM graphics and framebuffer"`

### Task 2: Rack and cannon

**Files:** Modify `invaders.html`.

**Interfaces:**
- **Produces:**
  - Rack state: `alive` (`Uint8Array(55)`, index = row*11+col, row 0 at the bottom), `refX`, `refY` (cell top of
    row 0), `rackDX`, `rackDY`, `rackRight`, `animFrame`, `cursor`, `waitOnDraw`, `numAliens`.
  - Rack functions: `newRack(startY)`, `drawInvader()` (DrawAlien), `advanceCursor()` (CursorNextAlien),
    `moveRack()` (MoveRefAlien), `rackBump()`, `countInvaders()`.
  - Player state: `player = { x, timer, hit, blowDelay, blowChanges, blowFrame }`, plus `playerOK`,
    `enableAlienFire` and `alienFireDelay`. Player functions: `updatePlayer()` (GameObj0), `resetPlayer()`.
  - Game flow: `mode`, `tasksOn`, `startGame()` (a temporary version that skips the prompt; Task 6 adds the prompt),
    `runGameTasks()`.

- [ ] **Step 1: Write the checks** (they fail until Step 2):

```js
() => {
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const r = {};
    const moves = [];
    const orig = moveRack;
    moveRack = () => { moves.push(frame); orig(); };
    for (let i = 0; i < 200; i++) step();
    r.buildThenStep = moves[0] - (frame - 200) === 55;   // first step after the 55-frame build
    r.fullRackStep = moves[1] - moves[0] === 55;
    alive.fill(0); alive[0] = 1; moves.length = 0;
    for (let i = 0; i < 10; i++) step();
    r.lastAlienEveryFrame = moves.length >= 9 && moves.every((f, i) => i === 0 || f - moves[i - 1] === 1);
    // a bounce off the left wall with one alien left gives +3
    rackRight = false; rackDX = -2; refX = 6;
    for (let i = 0; i < 6; i++) step();
    r.rightStepIsThree = rackDX === 3 && rackRight;
    moveRack = orig;
    return r;
}
```

```js
() => {
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const r = {};
    let guard = 0;
    while (rackRight && guard++ < 5000) step();             // until the right edge is detected
    const y0 = refY, x0 = refX;
    const orig = moveRack; let after = null;
    moveRack = () => { orig(); after = { x: refX, y: refY }; };
    while (!after && guard++ < 6000) step();
    moveRack = orig;
    r.drop8 = after.y === y0 + 8;
    r.stepLeftOnDrop = after.x === x0 - 2;
    // cannon: right wins when both keys are held, and the limits are 16..185
    keys.ArrowLeft = keys.ArrowRight = true;
    while (player.timer > 0) step();
    const px = player.x; step(); step();
    r.rightWins = player.x === px + 2;
    for (let i = 0; i < 300; i++) step();
    r.rightLimit = player.x === 185;
    keys.ArrowRight = false;
    for (let i = 0; i < 300; i++) step();
    r.leftLimit = player.x === 16;
    keys.ArrowLeft = false;
    return r;
}
```

- [ ] **Step 2: Implement** the rack (ROM `DrawAlien`, `CursorNextAlien`, `MoveRefAlien`, `RackBump`) and the
  cannon (`GameObj0`: movement, drawing, the alien-fire delay). Each frame, `runGameTasks()` calls `drawInvader()`,
  then `updatePlayer()`, then `advanceCursor()`, then `rackBump()` and `countInvaders()`.
- [ ] **Step 3:** Run both checks; every value must be `true`. Take a screenshot showing the rack marching.
- [ ] **Step 4: Commit:** `git commit -am "Add the marching rack and the cannon"`

### Task 3: Player shot, invader hits, shields and scoring

**Interfaces:**
- **Produces:**
  - `shot = { status, x, y, sprite, timer, collided }`, where status follows the ROM: 0 idle, 1 requested,
    2 moving, 3 exploding, 4 removed after an alien hit, 5 alien exploding.
  - `fireBounce`, `invaderExplosion = { timer, x, y }`, `shotCount`.
  - `playerFire()` (PlrFireOrDemo), `updatePlayerShot()` (GameObj1), `playerShotHit()` (PlayerShotHit),
    `endOfShot()` (EndOfBlowup).
  - `drawShields()`, `drawBottomLine()`, `addScore(points)` (wraps at 10000), `checkExtraShip()`.
  - Score state: `score`, `highScore`, `ships`, `extraShipAvailable`.

- [ ] **Step 1: Write the checks:**

```js
() => {
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const r = {};
    while (player.timer > 0) step();
    for (let i = 0; i < 60; i++) step();                     // let the rack finish building
    // aim under column 3, bottom row
    player.x = refX + 16 * 3 - 4;
    const before = score, deadBefore = 55 - alive.reduce((a, b) => a + b);
    keys.Space = true; step(); keys.Space = false;
    let guard = 0;
    while (alive[3] && guard++ < 200) step();
    r.killedColumn3Bottom = alive[3] === 0 && 55 - alive.reduce((a, b) => a + b) === deadBefore + 1;
    r.scored10 = score === before + 10;
    // the rack freezes while the explosion shows (16 frames)
    const c = cursor, x = refX;
    for (let i = 0; i < 14; i++) step();
    r.frozen = cursor === c && refX === x;
    // holding fire through a shot does not auto-repeat (Review Focus)
    for (let i = 0; i < 60; i++) step();
    keys.Space = true; step();
    r.fired = shot.status !== 0;
    for (let i = 0; i < 120; i++) step();                   // shot ends while fire is still held
    r.noRepeat = shot.status === 0;
    keys.Space = false; step(); keys.Space = true; step(); keys.Space = false;
    r.fireAfterRelease = shot.status !== 0;
    return r;
}
```

```js
() => {
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const r = {};
    while (player.timer > 0) step();
    alive.fill(0);                                          // nothing to hit, so the shot meets the shield
    const shieldPixels = () => { let n = 0; for (let y = 192; y < 208; y++) for (let x = 32; x < 54; x++) n += screen[y * W + x]; return n; };
    const p0 = shieldPixels();
    player.x = 32 + 2 - 8;                                  // shot column x = 34, inside the left pillar
    keys.Space = true; step(); keys.Space = false;
    for (let i = 0; i < 40; i++) step();
    r.holeBitten = shieldPixels() < p0;
    r.holeWithinExplosion = p0 - shieldPixels() <= 8 * 8 + 4;
    r.scoreWraps = (() => { score = 9990; addScore(30); return score === 20; })();
    score = 1490; extraShipAvailable = true; const s0 = ships; addScore(10); checkExtraShip();
    r.extraShipOnce = ships === s0 + 1 && !extraShipAvailable;
    addScore(1000); checkExtraShip();
    r.noSecondExtra = ships === s0 + 1;
    return r;
}
```

- [ ] **Step 2: Implement** `GameObj1` (states 1–5 with the 16-frame explosion at x−3, y+2) and
  `PlrFireOrDemo` (fire bounce cleared only while the shot is idle). Also implement `PlayerShotHit`:
  - the miss line at y ≤ 32;
  - the saucer zone at y ≤ 42;
  - the rack-area test `sy >= refY + 6` → miss;
  - row = `ceil((refY - sy + 6) / 16) - 1`, col = `sx > refX ? ceil((sx - refX) / 16) - 1 : 0`, index = row×11 + col;
  - the alien explosion, written over its 16×16 block for 16 frames.

  Add `EndOfBlowup` (shot count, saucer direction), shields at x 32/77/122/167 and y 192, the green line at y 239,
  scoring and the extra cannon.
- [ ] **Step 3:** Run both checks; every value must be `true`. Take a screenshot of a bitten shield.
- [ ] **Step 4: Commit:** `git commit -am "Add the player's shot, invader hits, shields and scoring"`

### Task 4: Alien shots, cannon death, lives and invasion

**Interfaces:**
- **Produces:**
  - `bombs[0..2]`: rolling, plunger, squiggly. Each is
    `{ type, active, blowing, blowCnt, steps, x, y, frameIdx, sprite, colPtr, skipFirst }`.
  - `bombSpeed`, `skipPlunger`.
  - `reloadRate(score) → moves`, `handleBomb(b, others)` (HandleAlienShot), `updateBombs()` (one per frame, phase
    `frame % 3`), `findColumn(x) → 1-based column`, `lowestInColumn(col) → index | -1`.
  - `hitPlayer()` (sets `player.hit`), `invaded()` and `invadedFlag`.
  - `gameOver()`: a stub that sets `mode = "gameover"`; Task 6 fills it in.

- [ ] **Step 1: Write the checks:**

```js
() => {
    const r = {};
    r.reload = [[0,48],[299,48],[300,16],[1099,16],[1100,11],[2099,11],[2100,8],[3099,8],[3100,7],[9990,7]]
        .every(([s, R]) => reloadRate(s) === R);
    testStart();
    // record plunger columns: the first 6 shots follow 1 7 1 1 1 4 (with a full rack every column has aliens)
    const cols = [];
    const orig = handleBomb;
    handleBomb = (b, others) => {
        const was = b.active; orig(b, others);
        if (!was && b.active && b.type === 1) cols.push((b.x - 7 - refX) / 16 + 1);
    };
    let guard = 0;
    while (cols.length < 6 && guard++ < 20000) { step(); if (player.hit) { player.hit = false; resetPlayer(); player.timer = 0; } }
    handleBomb = orig;
    r.plungerColumns = cols.join(" ") === "1 7 1 1 1 4";
    return r;
}
```

```js
() => {
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const r = {};
    while (player.timer > 0) step();
    const s0 = ships;
    hitPlayer();
    for (let i = 0; i < 6; i++) step();
    r.frozenAfter5 = playerOK === false;
    let n = 6; while (player.hit) { step(); n++; }
    r.explosion60 = n >= 60 && n <= 61;
    r.shipLost = ships === s0 - 1;
    let hidden = 0; while (player.timer > 0) { step(); hidden++; }
    r.respawn128 = hidden >= 127 && hidden <= 128;
    alienFireDelay = 48; let wait = 0; while (!enableAlienFire) { step(); wait++; }
    r.fireDelay48 = wait >= 47 && wait <= 48;
    // invaders reaching y 216 end the game even with cannons left
    enableAlienFire = false; alienFireDelay = Infinity;
    refY = 208; let guard = 0;
    while (mode === "playing" && guard++ < 2000) step();
    r.invadedGameOver = mode === "gameover" && invadedFlag;
    return r;
}
```

- [ ] **Step 2: Implement** `GameObj2`/`3`/`4` (the shot part) and `HandleAlienShot`:
  - firing rules: reload by the other two shots' step counts, rolling aims via `findColumn`, plunger and squiggly use
    their tables, the lowest live alien fires, from x+7, y+10;
  - movement: one frame advance per move, speed 4, or 5 when `numAliens <= 8`;
  - endings: bottom at y ≥ 228, player band at y 210–218, explosion at x−2, y+2 visible while blowCnt is 3..1;
  - resets: the plunger stops with one alien left, the rolling shot skips its first run after a reset.

  Also add the cannon explosion (60 frames, `PLAYER_EXPLOSION` toggling, `playerOK` cleared at the first change),
  lives and invasion.
- [ ] **Step 3:** Run both checks; every value must be `true`.
- [ ] **Step 4: Commit:** `git commit -am "Add alien shots, cannon explosions, lives and invasion"`

### Task 5: Saucer

**Interfaces:**
- **Produces:**
  - `saucer = { start, active, hit, hitTimer, x, dx }`, `tillSaucer`.
  - `timeSaucer()` (TimeToSaucer), `updateSaucer()` (the saucer half of GameObj4), `removeSaucer()`,
    `saucerPoints() → points`.
  - Constants `SAUCER_SCORES` (15 entries) and `SAUCER_DIRECTION_BITS` (256 characters).

- [ ] **Step 1: Write the checks:**

```js
() => {
    const r = {};
    shotCount = 8;  r.shot8 = saucerPoints() === 300;
    shotCount = 23; r.shot23 = saucerPoints() === 300;
    shotCount = 1;  r.shot1 = saucerPoints() === 50;
    shotCount = 15; r.shot15 = saucerPoints() === 100;
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    // the timer only runs once the rack is below its first height
    for (let i = 0; i < 2000; i++) step();
    r.noSaucerBeforeDrop = !saucer.start && refY === 128 || refY > 128;
    // direction comes from the ROM bit table after each shot removal
    shotCount = 0; saucer.active = false; endOfShot();
    r.dir1 = SAUCER_DIRECTION_BITS[1] === "1" ? saucer.x === 9 && saucer.dx === 2 : saucer.x === 192 && saucer.dx === -2;
    // a shot reaching the saucer row scores its table value after the hit sequence
    alive.fill(1);
    saucer.start = true; saucer.active = false; bombs[2].steps = 0;
    let guard = 0; while (!saucer.active && guard++ < 10) step();
    r.launched = saucer.active;
    shotCount = 7;                                       // this shot will be the 8th: 300 points
    const s0 = score;
    saucer.hit = true; shot.status = 4;
    for (let i = 0; i < 120; i++) step();
    r.scored300 = score === (s0 + 300) % 10000;
    r.cleared = !saucer.active;
    return r;
}
```

- [ ] **Step 2: Implement** TimeToSaucer (1,536 frames, only while the 8-bit `refL` is below `$78`) and the launch
  rule (≥ 8 aliens, no squiggly shot in flight). The saucer moves 2 px per run and is removed at x < 8 or x ≥ 193.
  Add the hit timeline (32 → explosion at 31, score text and points at 24, clear at 0) and the direction and score
  tables.
- [ ] **Step 3:** Run the check; every value must be `true`. Take a screenshot with the saucer on screen (red band).
- [ ] **Step 4: Commit:** `git commit -am "Add the mystery saucer"`

### Task 6: Game flow and attract mode

**Interfaces:**
- **Produces:**
  - `script` (the current generator), `attractScript(startAt)`, `startScript()`, `gameOverScript()`,
    `typeText(str, x, y)` (a generator, 6 frames per letter), `wait(frames)` (a generator).
  - Round flow: `nextRack()` and `rackCount`.
  - Demo: `demo` flag, `demoCmd` and `DEMO_COMMANDS`.

- [ ] **Step 1: Write the checks:**

```js
() => {
    const r = {};
    localStorage.removeItem("invadersHighScore"); highScore = 0;
    mode = "attract"; script = attractScript(0); tasksOn = false;
    pressed.Enter = true; step();
    r.started = mode === "playing" && !tasksOn;
    let n = 0; while (!tasksOn && n < 400) { step(); n++; }
    r.prompt176 = n >= 175 && n <= 178;
    enableAlienFire = false; alienFireDelay = Infinity;
    // next rack: kill everything, 48-frame pause, new rack lower down
    for (let i = 0; i < 60; i++) step();
    alive.fill(0);
    let w = 0; while (refY !== 152 && w < 200) { step(); w++; }
    r.nextRackHeight = refY === 152 && rackCount === 1;
    r.pause48 = w >= 48 && w <= 52;
    // last invader killed while the cannon explodes (Review Focus)
    while (player.timer > 0) step();
    alive.fill(0); alive[0] = 1; hitPlayer(); alive[0] = 0;
    let g = 0; while (rackCount === 1 && g++ < 1000) step();
    r.roundEndsAfterExplosion = rackCount === 2;
    // game over saves the high score and returns to attract
    score = 1230; ships = 1; while (player.timer > 0) step();
    hitPlayer(); g = 0; while (mode !== "attract" && g++ < 2000) step();
    r.backToAttract = mode === "attract";
    r.highSaved = Number(localStorage.getItem("invadersHighScore")) === 1230;
    // attract cycle with demo runs without errors for a long stretch
    for (let i = 0; i < 6000; i++) step();
    r.attractStillRunning = mode === "attract";
    return r;
}
```

- [ ] **Step 2: Implement** the scripts, each a generator:
  - **Attract cycle:** wait 64 frames, type "PLAY" and "SPACE  INVADERS", wait 64, show the score table (icons
    drawn, lines typed), wait 128, run the demo until its cannon is hit and finishes exploding, then show "PUSH" /
    "ONLY 1PLAYER  BUTTON" for 128 frames, and loop.
  - **Start:** "PLAY PLAYER<1>" for 176 frames, with the P1 score blinking.
  - **Game over:** type "GAME OVER", wait 128 frames, clear, then return to attract at the PUSH screen.

  Also add round end and `nextRack()` (the table of starting heights; reset everything the ROM mirror resets), and
  the demo (DemoCommands, constant fire, no scoring). Add the Asteroids-style `visibilitychange` audio pause and the
  250 ms lag clamp (Review Focus).
- [ ] **Step 3:** Run the check; every value must be `true`. Watch an attract cycle by screenshot at the score
  table and during the demo.
- [ ] **Step 4: Commit:** `git commit -am "Add attract mode, demo, rounds and game over"`

### Task 7: Sound

**Interfaces:**
- **Produces:**
  - `sound` with `unlock`, `pause`, `toggleMute`, `tone`, `noise`, `fleet(note)`, `shot()`, `invaderHit()`,
    `playerDeath()`, `saucerHit()`, `extraShip()` and `setSaucer(on)`.
  - `timeFleetSound()`, `fleetCount`, `fleetNote` and `fleetDelay(numAliens)`.

- [ ] **Step 1: Write the check:**

```js
() => {
    const r = {};
    r.delays = fleetDelay(55) === 52 && fleetDelay(50) === 52 && fleetDelay(49) === 46 && fleetDelay(8) === 16 && fleetDelay(1) === 5;
    const calls = [];
    const orig = sound.fleet; sound.fleet = n => calls.push([frame, n]);
    testStart(); enableAlienFire = false; alienFireDelay = Infinity;
    const start = frame;
    for (let i = 0; i < 200; i++) step();
    sound.fleet = orig;
    r.firstNoteAt64 = calls[0][0] - start === 64;
    r.interval52 = calls[1][0] - calls[0][0] === 52;
    r.cycles = calls.slice(0, 4).map(c => c[1]).join("") === "0123".slice(0, Math.min(4, calls.length));
    // no game sound in attract mode
    const hits = []; const oh = sound.invaderHit; sound.invaderHit = () => hits.push(frame);
    mode = "attract"; script = attractScript(0); tasksOn = false;
    for (let i = 0; i < 3000; i++) step();
    sound.invaderHit = oh;
    r.silentAttract = hits.length === 0;
    return r;
}
```

- [ ] **Step 2: Implement** the synthesized sounds, modelled on Asteroids' `sound` object, and wire up the
  triggers:
  - the fleet march every `fleetDelay` frames while `playerOK`, starting 64 frames into a rack;
  - the shot sound when a shot starts;
  - the invader-hit sound at a kill;
  - the death sound at a hit;
  - the saucer warble while the saucer flies;
  - the saucer-hit sound at the explosion;
  - the extra-cannon chime.
- [ ] **Step 3:** Run the check; every value must be `true`. Then play by key presses and confirm the console
  shows no errors.
- [ ] **Step 4: Commit:** `git commit -am "Add synthesized sound"`

### Task 8: README and screenshots

**Files:**
- Modify `README.md`.
- Rename `docs/screenshot.png` to `docs/asteroids.png`.
- Create `docs/invaders.png`.

- [ ] **Step 1:** `git mv docs/screenshot.png docs/asteroids.png`, then capture `docs/invaders.png` from a game
  in progress: rack mid-screen, bitten shields, a bomb in flight, and the saucer if it can be timed.
- [ ] **Step 2: Rewrite `README.md`:**
  - the title "Arcade Classics" and an intro;
  - a shared Play section;
  - an Asteroids section: the existing text word for word, headings dropped one level, and the screenshot path
    updated;
  - a Space Invaders section with controls, scoring, "How close to the arcade is it?" (from the ROM, approximated,
    left out), a code tour and sources;
  - a shared disclaimer.
- [ ] **Step 3:** Check that every relative link and image path in the README resolves (`ls` each path).
- [ ] **Step 4: Commit:** `git add -A README.md docs && git commit -m "Rebuild README as Arcade Classics and add Space Invaders docs"`

### Task 9: Final review

- [ ] Run all the Task 1–7 checks again on the finished file.
- [ ] Dispatch one fresh reviewer on the whole branch diff (spec + fact sheet in hand) and fix what it confirms.
- [ ] Use superpowers:finishing-a-development-branch.
