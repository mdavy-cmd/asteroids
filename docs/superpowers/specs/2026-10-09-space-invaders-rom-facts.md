# Space Invaders (Taito/Midway 1978, MAME set `invaders`): fact sheet taken from the ROM

## Sources and how to read the references

Main source: the commented disassembly at https://computerarcheology.com/Arcade/SpaceInvaders/Code.html.
"$xxxx Label" points to a ROM address or label in that listing. "RAM $20xx" means a variable described on
RAMUse.html. "HW" means Hardware.html.

**Mirror $1Bxx→$20xx**: at power-up the ROM block $1B00-$1BFF is copied into RAM. The block $1B00-$1BBF is
copied again (CopyRAMMirror $01E4) at every new game, every new rack and every player switch. It is **not** copied
when the player dies. Most starting values below come from this block.

I rebuilt the 8 KB ROM image from the listing's hex column. Its CRC32s match MAME's `invaders` set exactly
(734f5ad8, 6bfaca4a, 0ccead96, 14e538b0; the unused $1000-$13FF area is zero). So derived tables, like the saucer
direction bits, are exact.

I checked a few items against outside sources: DIP defaults against MAME `src/mame/midw8080/mw8080bw.cpp`, and the
colour overlay against MAME `src/mame/layout/invaders.lay`. Anything not settled by the ROM is marked UNCONFIRMED.

## 0. Coordinates and timing (read this first)

**Converting to portrait.** Video RAM is $2400-$3FFF: 224 raw lines of 32 bytes each, with bit 0 as the first pixel.
The monitor is turned 90° anticlockwise (HW: "the first byte is lower left").
- In portrait, x runs right from 0 to 223 and y runs down from 0 to 255.
- For screen address A and bit b: `x = (A-$2400)>>5` and `y = 255 - (((A-$2400)&31)*8 + b)`.
- The top of a byte cell is therefore at `y = 248 - 8*((A-$2400)&31)`.

**The game's "pixel number".** Objects are positioned with a 16-bit pixel number. The disassembly calls H "Xr" and
L "Yr" (ConvToScr $1A47 computes address = $2000 + HL/8, and CnvtPixNumber $1474 sends L&7 to the shifter).
- `x = H - $20`.
- `y_top = 248 - L`, and the sprite's bottom row is at y = 255 - L. L counts **upwards** from the bottom of the
  screen.
- So in the disassembly, "ΔYr = -8" means 8 px down, and "ΔXr = +2" means 2 px right.

**Sprite bytes.** Each byte is one portrait column, bit 7 at the top. Bytes run left to right, 8 px tall
(DrawSprite $15D3 adds $20 per byte, so each byte lands on the next raw line, one column to the right).

**Interrupts.** There are two interrupts per frame (HW Interrupts):
- RST 1 at $0008 (ScanLine96), when the beam reaches line 96.
- RST 2 at $0010 (ScanLine224), at the start of VBLANK.

Each frame does the following:
- **ScanLine224 ISR:** isrDelay $20C0 -1 (this is the frame counter that all delay loops use); coin handling;
  TimeFleetSound $1740; shotSync ← $2032; DrawAlien $0100 (draws one alien); RunGameObjs from $2010 (objects 0-4);
  TimeToSaucer $0913.
- **ScanLine96 ISR:** RunGameObjs from $2020 (objects 1-4); CursorNextAlien $0141.

Objects 1-4 are called in both ISRs. They only act in one of them: CompYToBeam $1A06 compares bit 7 of the object's H
with $2072 (set to $80 in ScanLine224, $00 in ScanLine96). So objects at x ≥ 96 update in the VBLANK ISR, objects at
x < 96 update at line 96, and each object moves at most once per frame.

The main game loop ($081F-$0851) is not synchronised to the frame. It runs many times per frame and does collision
resolution, scoring, edge detection, fire input and sound selection.

**For a 60 Hz remake, one frame = one tick**, with these rates:
- the player moves 1 px per frame;
- the player shot moves 4 px per frame;
- one alien is redrawn per frame;
- each alien shot and the saucer move once every 3 frames (explained in item 4).

The nominal rate is 60 Hz (HW). MAME's timing constants give 4.992 MHz / 320 / 262 = 59.54 Hz (mw8080bw.h).

## 1. Rack layout

- **Size:** 5 rows × 11 columns = 55 aliens (InitAliens $01C0 counts $37; GetAlienCoords $017A uses 11 per row).
  Alien index = row×11 + column. Row 0 is the **bottom** row and column 0 is on the left. Live flags are at
  $2100-$2136 for player 1 and $2200 for player 2.
- **Type by row** (DrawAlien $0118-$012C uses offset (row&$FE)×8 into AlienSprA $1C00; scores come from
  AlienScoreValue $097C and table $1DA0):
  - rows 0-1 (bottom two): sprite A, the "octopus", 12 px wide ($1C00 / $1C30), **10 pts**;
  - rows 2-3: sprite B, the "crab", 11 px wide ($1C10 / $1C40), **20 pts**;
  - row 4 (top): sprite C, the "squid", 8 px wide ($1C20 / $1C50), **30 pts**.

  (The listing's comment "0,1 -> 32" at $0120 is wrong; the code gives offsets 0, 16 and 32.)
- **Spacing:** cells are on a 16 px grid both across and down (GetAlienCoords adds $10 per column to H and per row
  to L). Each alien is drawn in a 16-wide × 8-tall cell, which leaves an 8 px gap between rows.
- **Start position:** the reference alien (row 0, column 0) is at H=$38, L=$78 (NewGame $07EA `LD HL,$3878`;
  Mirror $1B09/$1B0A).
  - In portrait, column c, row r has its cell at **x = 24 + 16c**, **y_top = 128 - 16r**.
  - At the start of round 1: bottom row y 128-135, top row y 64-71, rack cells span x 24-199.
  - Every new rack restarts at H=$38 (x = 24) ($0A1E).
- **Starting height by round** (AlienStartTable $1DA3, indexed by the rack counter at $2xFE, code $0A09-$0A1C:
  count = (count&7)+1, then L = [$1DA2+count]):

  | Round | Ref L | Bottom row y | Top row y |
  |---|---|---|---|
  | 1 | $78 ($07EA) | 128-135 | 64-71 |
  | 2 | $60 | 152-159 | 88-95 |
  | 3 | $50 | 168-175 | 104-111 |
  | 4, 5, 6 | $48 | 176-183 | 112-119 |
  | 7, 8, 9 | $40 | 184-191 | 120-127 |

  Round 10 goes back to $60, and the cycle of 8 repeats (rounds 10-17 = rounds 2-9, and so on).
- **How a rack appears:** the cursor draws the aliens one per frame from index 0 (bottom-left) to 54, so the rack
  "builds" over 55 frames. Only then does the first step happen.
- **Starting direction:** right, ΔH = +2 (L_00D7 at $00D7 sets $21FB/$22FB = 2; Mirror $1B08 = 02).

## 2. Rack movement

- **One alien per frame.**
  - CursorNextAlien $0141 runs once per frame in ScanLine96. It moves the cursor to the next *live* alien; dead ones
    are skipped in the same call by the loop at $0154.
  - DrawAlien $0100 runs once per frame in ScanLine224 and redraws that one alien at the current reference position.
  - The flag waitOnDraw $2000 keeps the two in step.
  - When the cursor passes index 54, MoveRefAlien $01A1 moves the reference alien by (ΔH, ΔL) and flips the animation
    frame $2005 ($01B2-$01B8).
  - So with N live aliens, a full rack step takes N frames, rippling from bottom-left to top-right. A single
    remaining alien moves every frame.
- **Step size:** +2 px right / -2 px left (RackBump $1597: `LD B,$FE` at $15A5).
  - When **exactly one alien is left**, a bounce off the left wall gives **+3** for moving right ($18F1-$18F9).
    Moving left is always -2.
  - InitRack $00C2 turns a stored 3 back into 2.
- **Drop:** rackDownDelta $200E = $F8, i.e. **8 px down** (Mirror $1B0E). RackBump copies it into ΔL $2007.
  - MoveRefAlien adds ΔL *and* the new ΔH in the same step, then clears ΔL ($01A9-$01AC).
  - So the drop step also moves 2 px sideways in the new direction.
- **Edge detection:** RackBump $1597 runs in the main loop (PlyrShotAndBump $190A). It checks pixels, not
  coordinates:
  - It scans 23 bytes of video RAM ($15C5-$15D1) in a single portrait column.
  - Moving right, it scans from $3EA4: **x = 213, y 40-223**. Moving left, it scans from $2524: **x = 9, y 40-223**.
  - Any lit pixel there reverses the direction and sets the drop.
  - It takes effect on the *next* rack step; the current ripple finishes with the old delta.
  - Because it tests pixels, the turning point depends on which sprites (octopus, crab or squid) are at the edge.
- **Paused during an invader explosion: yes.**
  - A hit sets expAlienTimer $2003 = $10 ($152A).
  - AExplodeTime $1538 is called from DrawAlien and counts it down once per frame. While it runs, no alien is drawn
    and waitOnDraw is not cleared, so the cursor doesn't move.
  - The rack is frozen for **16 frames**. Then the explosion's 16×16 area is cleared (EraseSimpleSprite) and
    marching resumes.
- **Also frozen** while playerOK $2068 = 0, meaning the player is exploding or waiting to respawn
  (CursorNextAlien $0141-$0145).
- **Reaching the bottom:** if the next live alien's L is below $28 (that is, it would sit at y ≥ 216, the cannon's
  row), the game jumps to $1971. See item 7.

## 3. Player

- **Update rate:** the player object runs once per frame (only in ScanLine224's RunGameObjs, which starts at $2010).
  - **Speed: 1 px per frame** (MovePlayerRight $0381 / MovePlayerLeft $038E).
  - Right is tested first, so it wins if both are pressed ($0366-$036C).
- **X limits:** H $30-$D9 ($0382 `CP $D9`, $038F `CP $30`), so the cell's x runs from **16 to 185**.
  - The cannon's lit pixels are cell columns 2-14, so the visible extent is x 18-199.
  - Start: x = 16 (Mirror $1B1B = $30).
- **Y:** L = $20 (Mirror $1B1A), so rows **y 216-223**.
- **Shot:**
  - It starts at H = player H + 8 ($03FF), which is the cannon-tip column, x = player x + 8.
  - Its starting L is $28 (Mirror $1B29), so its 4 lit pixels are at y 212-215.
  - It moves **4 px per frame** (shotDeltaX, Mirror $1B2C = 4; GameObj1 $03BB acts once per frame).
- **Firing rules:**
  - Only one shot at a time: PlrFireOrDemo $1618 needs plyrShotStatus $2025 = 0, and the shot status is still 5
    while an alien explosion is showing.
  - You must release fire between shots (fireBounce $202D).
  - You can't fire while dying, or before the player object's start timer has run out ($161E-$1625).
- **Top of screen:**
  - PlayerShotHit $14E4: L ≥ $D8 counts as a miss (status 3). That is reached after 44 frames (40 + 4×44 = 216).
  - The explosion is the 8×8 sprite $1C91, drawn at H-3, L-2 ($03E7-$03F2).
  - blowUpTimer $2026 = $10 (Mirror $1B26). It is drawn on the first tick ($03DC) and erased when the timer hits 0,
    so it lasts **16 frames** (15 frames visible). The same applies when the shot hits a shield or an alien shot.
- **Player explosion:**
  - expAnimateCnt = $0C = 12 (Mirror $1B17) and the reload is 5 ($02A7), so it lasts 12 × 5 = **60 frames**.
  - For the first 5 frames the intact cannon stays on screen.
  - Then the image switches 11 times, every 5 frames, alternating $1C80 (first) and $1C70 (DrawPlayerDie $039B).
  - At 60 frames it is erased and the player block is reloaded from the mirror ($02AE-$02BE).
  - playerOK is cleared, alien fire is turned off and alienFireDelay is set to $30 at each image change
    ($029B-$02A3). The first change is 5 frames after the hit, so the rack keeps marching for those 5 frames.
- **Next ship:**
  - The reloaded block includes obj0 timer = $0080 (Mirror $1B10/$1B11), so the new cannon appears **128 frames
    after the explosion ends** (it is not drawn until then).
  - Aliens stay frozen until it appears (playerOK).
  - Aliens may fire again **48 frames** after it appears ($03B0-$03B8, alienFireDelay $206A = $30).
  - The same 128 + 48 frames apply at the start of every rack. During that time the aliens are already marching.
  - In 2-player games the timer is cleared on a player switch ($0318) and "PLAY PLAYER<n>" is shown instead.

## 4. Alien shots

- **Shot timing:** the three shots and the saucer take turns. obj2's extra timer $2032 is reloaded to 2 every time
  it runs ($0477), and it is counted down at *both* ISRs. It is copied to shotSync $2080 once per frame ($0072).
  Over a 3-frame cycle:
  - frame A (sync 2): the squiggly shot or the saucer runs (GameObj4 $0682 needs sync = 2);
  - frame B (sync 0): the rolling shot runs in the VBLANK ISR;
  - frame C (sync 1): the rolling shot runs at line 96, and the plunger runs (GameObj3 $04BC needs sync = 1).

  The movement code is also gated by CompYToBeam ($05C4). Result: **each alien shot moves once every 3 frames**.
- **Speed:**
  - Normally 4 px down per move (alienShotDelta $207E = $FC, Mirror $1B7E; this byte sits just after the
    "PLAY PLAYER<1>" text). That is 4 px per 3 frames.
  - With **8 or fewer aliens** it becomes 5 px per move ($FB, SpeedShots $08D8). This lasts until the next rack.
  - The RAM page says "-1 / -4"; the code and the ROM show -4 / -5.
- **Animation:** the image advances one of 4 frames per move ($05D4-$05E2).
- **Rolling shot** (GameObj2 $0476, sprite $1CEE): aims at the player.
  - Column = FindColumn $156F applied to player H+8 ($061B-$062C). In practice that is the column k whose cell
    satisfies refH+16k < target ≤ refH+16(k+1), where refH is the reference alien's H.
  - If the player is left of the rack, column 0 is used. A result of 12 or more is clamped to 11.
  - WrapRef $1590 has a bug: odd results if refH ≥ $80.
  - After each reset it skips its first chance to fire ($047D-$0489).
- **Plunger shot** (GameObj3 $04B6, sprite $1CE2): uses column table $1D00-$1D0F (16 entries; it resets at
  LSB $10, $04DC). The table values are 1-based columns:
  01 07 01 01 01 04 0B 01 06 03 01 01 0B 09 02 08.
- **Squiggly shot** (GameObj4 → $050F, sprite $1CD0): uses $1D06-$1D14 (15 entries; it resets at LSB $15 to $06,
  $0529): 0B 01 06 03 01 01 0B 09 02 08 02 0B 04 07 0A.
  - ColFireTable $1D00. The table pointers carry over from shot to shot ($0508, $0549).
  - $1D15-$1D1F is never used.
- **Choosing which alien fires:**
  - FindInColumn $062F takes the **lowest live alien** in the chosen column. If the column is empty, no shot is
    fired this time (the pointer still advances; RET NC at $05A8).
  - The new shot starts at alien H+7 and L-10 ($05A9-$05B4). That means x = cell+7..+9, starting 2 px below the
    alien's cell. It is first drawn one move lower.
- **Reload rule** (HandleAlienShot $0563, $057C-$0595): a new shot can start only if each of the *other two* shots
  is either idle (step count 0) or has already made **more than R moves**.
  - R comes from AShotReloadRate $170E, based on the score's high BCD byte. Tables $1CB8 = 02 10 20 30 and
    $1AA1 = 30 10 0B 08, then 07:

    | Score | R (moves) |
    |---|---|
    | 0-299 | 48 |
    | 300-1099 | 16 |
    | 1100-2099 | 11 |
    | 2100-3099 | 8 |
    | 3100 and up | 7 |

    One move = 3 frames.
  - No new shots at all while enableAlienFire $2069 = 0 ($0571-$0578).
- **Limits:**
  - At most **3** alien shots on screen, one of each type.
  - The **squiggly shot shares its slot with the saucer**: once the saucer is triggered and no squiggly is in
    flight, GameObj4 runs the saucer instead ($0689-$0695). So no squiggly shots while the saucer is on screen.
  - **Plunger:** when a plunger shot finishes and numAliens = 1, skipPlunger $206E is set ($04FC-$0505). GameObj3
    then returns at once ($04B7), so there are no plunger shots for the rest of that rack.
- **Endings:**
  - The shot ends when L < $15 (bottom; $05F6).
  - It also ends on any collision ($05FB) with shield, line or shots.
  - A collision while L is $1E-$26 counts as **hitting the player** (playerAlive $2015 = 0; $0600-$060F).
- **Shot explosion:**
  - The 6×8 sprite $1CDC is drawn at H-2, L-2 ($0651-$0664).
  - aShotBlowCnt = 4 (Mirror $1B3A/$1B4A/$1B5A). It is drawn when the count reaches 3 and erased when it reaches 0,
    so it is visible for 3 moves = **9 frames**.

## 5. Saucer (mystery ship)

- **Spawn timer** (TimeToSaucer $0913, once per frame in the ISR):
  - tillSaucer $2091 counts down from **$0600 = 1536 frames** (Mirror $1B91). It sets saucerStart $2083 and
    restarts at $600.
  - It only counts while refAlienYr $2009 < $78, i.e. after the rack has gone below round 1's starting height.
    In later rounds that is true from the start.
  - It is reset at every rack and player switch (mirror). The listing's comment says "game loops", but it runs in
    the ISR, so it counts frames.
- **Launch conditions** (GameObj4 $0689-$06A6): saucerStart is set, no squiggly shot is in flight, and
  **numAliens ≥ 8** (`CP $08`).
- **Speed and height:**
  - It moves **2 px every 3 frames** (it only runs at shotSync = 2 and is gated by CompYToBeam; $06BA-$06C1).
  - L = $D0 (Mirror $1B89): **y 40-47**. Lit pixels are y 41-47, at cell columns 4-19.
  - It is removed when H < $28 or H ≥ $E1 ($06CA-$06D2).
- **Direction** (EndOfBlowup $0456-$0474):
  - It is decided at every player-shot removal while no saucer is on screen.
  - HL = ($208F) gives $08nn, because $2090 holds $08 from Mirror $1B90. nn = an 8-bit count of player shots,
    incremented first. The code then does `LD A,(HL)` and `AND 1`.
  - So the direction is **bit 0 of the ROM byte at $0800+nn**, not bit 0 of the count (the RAM page has this
    wrong).
  - bit = 1: start at H $29 (x 9) moving +2 (right). bit = 0: start at H $E0 (x 192) moving -2 (left).
  - The default is H $29 moving +2. It comes from the mirror and from the saucer reset ($075F, which copies
    $1B83-$1B8C).
  - The 256 bits are listed in the appendix.
- **Score table** SaucerScrTab $1D54: 10 05 05 10 15 10 10 05 30 10 10 10 05 15 10 05. Each value × 16 in BCD gives
  50, 100, 150 or 300 points ($072C-$0733).
  - Pointer $208D starts at **$1D54** (Mirror $1B8D) at every rack and turn.
  - It goes up by 1 at every player-shot removal ($0447-$0453). It wraps from $1D63 back to $1D54. That compare is
    a bug ($044C `CP $63`): only 15 of the 16 entries are ever used.
  - The pointer moves on before the score is read, since the score is read 7 saucer ticks after the hit. So **the
    k-th shot of the rack scores entry (k mod 15)**:

    | k (mod 15) | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 0 |
    |---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
    | points | 50 | 50 | 100 | 150 | 100 | 100 | 50 | **300** | 100 | 100 | 100 | 50 | 150 | 100 | 100 |

    So **300 comes on the 8th shot, then the 23rd, 38th and every 15th after that**. This is the "23rd-shot"
    trick, because the saucer normally hasn't appeared by shot 8.
  - Shots fired in the same rack carry over across lives in a 1-player game. The counts reset at each new rack and
    at each player switch.
- **Hit detection:** a player-shot collision with L ≥ $CE sets saucerHit $2085 (PlayerShotHit $14F0 → $1579).
- **What happens after a hit:** saucerHitTime $2086 = $20 (Mirror $1B86) counts down once per saucer tick (every 3
  frames; $06D6-$06E9):
  - at $1F: explosion sprite $1D7C and the saucer-hit sound ($074B-$075C);
  - at $18 (about 21 frames later): the score text replaces the explosion. It is 3 characters, " 50", "100", "150"
    or "300" (SaucSoreStr $1D94/$1D97/$1D9A/$1D9D), drawn byte-aligned at the saucer's cell (x = H-32, y 40-47,
    24 px). The points are added at that moment ($070C-$0739);
  - at 0 (about 72 frames later): the area is cleared and the saucer reset ($06F9-$0704).

## 6. Shields

- **Count, position and size:** 4 shields (DrawShieldPl1 $01F8, C=4). Each is **22 columns × 16 rows** (44 bytes,
  ShieldImage $1D20; CopyShields $0221 uses `LD BC,$1602`).
  - The first is at screen $2806, so **x 32-53, y 192-207**.
  - Each next shield is 22 + 23 = 45 columns further right ($0239 adds $2E0 on top of the 22 lines drawn).
  - Left edges: **x = 32, 77, 122, 167**.
  - The bitmap is asymmetric: 5 solid columns left of the arch and 6 right of it. See graphics.md.
- **Damage:**
  - Shots are ORed in with a collision test (DrawSprCollision $1491) and erased with AND-NOT (EraseShifted $1452).
  - When a shot hits, its explosion sprite is ORed onto the shield, then erased with AND-NOT when its timer ends.
    For the player that is $1C91, 8×8, at H-3/L-2 ($03DF-$03F7). For the aliens it is $1CDC, 6×8, at H-2/L-2
    ($064E-$0669).
  - So **the shield loses exactly the explosion's pixel pattern** (plus the shot's own pixels).
  - The same thing makes holes in the bottom green line: alien shots end at L < $15 and their explosion reaches
    y 239.
- **Invaders erase shields: yes.**
  - DrawAlien uses DrawSprite $15D3, which **writes** (does not OR) 2 bytes per column. It overwrites a 16×16 block:
    the sprite rows plus 8 px above them.
  - The alien explosion (DrawSprite, then EraseSimpleSprite $1424 with 2 bytes × 16) also clears its block.
  - So descending invaders wipe out shield pixels as they pass through.
- **Refresh:**
  - New intact shields at every new rack ($0A2A/$0A33 → RestoreShields at $0804-$0811).
  - No refresh when the player dies.
  - In 2-player games each player's damaged shields are saved and restored at turn changes (RememberShields
    $147C, buffers $2142/$2242).

## 7. Scoring, lives, round flow

- **Points:** 10 / 20 / 30 by row ($1DA0). Saucer: 50 / 100 / 150 / 300.
- **Score format:** 4 BCD digits. Anything over 9999 wraps, because the carry is dropped ($09A2).
- **Starting lives:** 3 + DIP (IN2 bits 0-1), so 3-6 (GetShipsPerCred $08D1). MAME's default is **3**
  (mw8080bw.cpp `PORT_DIPNAME(0x03,0x00,Lives)`).
  - The digit shows the ships including the one in play. The icons show the reserve ships (RemoveShip $1A7F).
- **Extra ship:** once per player per game (flag $20E5/$20E6).
  - It is awarded when the score's high BCD byte is ≥ $15, i.e. **1500**. With DIP IN2 bit 3 set it is $10, i.e.
    1000 ($093D-$094E). MAME's default is **1500** (`PORT_DIPNAME(0x08,0x00,Bonus_Life)`).
  - An icon is added at x = 8+16N, where N is the new reserve count, and the digit is updated ($0955-$0968).
- **Invaders reach the cannon's row:** this is **game over for that player even with lives left**.
  - At $1971 the invaded flag is set. The player is blown up ($16E6-$16F9) and the reserve icons and digit are
    cleared to 0 ($16FF-$1706).
  - Then $1671 marks the player dead, updates the high score and prints "GAME OVER".
  - In a 2-player game, the other player carries on if still alive ($16BE-$16C6).
- **Losing the last ship:** when the explosion ends with a reserve of 0, $02DB → $166D → game over.
- **End of a round:**
  - The game loop checks numAliens $2082 = 0 ($082B), counted by CountAliens $15F3. The count drops at the moment of
    the kill.
  - **Pause:** $0A3C waits **48 frames** ($0A42, isrDelay = $30). If the player is exploding, it waits for the
    explosion to finish instead.
  - Then the playfield is cleared, the mirror reset (saucer, shots, timers and fleet sound all restart), the rack
    counter goes up, and a new rack and fresh shields are set up ($09EF-$0A39).
  - The player is reset to x 16 and stays hidden for 128 frames. There is no on-screen message between racks.
- **Game start:**
  - "PLAY PLAYER<1>" appears at x 56, y 112 for **176 frames** (PromptPlayer $088D, isrDelay = $B0). The player's
    score blinks, changing every 4 frames.
  - Then the rack builds, and the cannon appears 128 frames after that.
- **Game over:** "GAME OVER" is typed out at x 72, y 56, followed by a 128-frame wait ($16C9-$16D4). Then the game
  goes to the attract credit screen ($0B89).

## 8. Sound triggers

Port bits come from HW Output. Sound outputs are level bits: the code holds a bit on and later clears it.
- **Port 3 bits:** 0 = UFO (repeating), 1 = shot, 2 = player death ("flash"), 3 = invader hit, 4 = extended play
  (extra ship), 5 = amp enable.
- **Port 5 bits:** 0-3 = fleet notes 1-4, 4 = UFO hit, 5 = cocktail flip.

- **Fleet march (4 notes):**
  - The note bit in soundPort5 $2098 rotates 1→2→4→8→1 ($1799-$17A5).
  - TimeFleetSound $1740 runs once per frame and counts fleetSndCnt $2096 down. At 0 it plays the current note,
    reloads the counter from fleetSndReload $2097, and holds the note for **4 frames** ($1767). After that the fleet
    bits are cleared ($1743/$176D).
  - The game loop picks the reload value from the number of live aliens (FleetDelayExShip $1775; tables $1A11 and
    $1A21; the first entry whose alien count is ≤ the live count wins):

    | Aliens ≥ | 50 | 43 | 36 | 28 | 22 | 17 | 13 | 10 | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 |
    |---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
    | frames per note | 52 | 46 | 39 | 34 | 28 | 24 | 21 | 19 | 16 | 14 | 13 | 12 | 11 | 9 | 7 | 5 |

  - The march is **not locked to rack movement**: 55 aliens step every 55 frames, but the note plays every 52. The
    listing's comment at $1A11 says the same.
  - The first note comes 64 frames into a rack (Mirror $1B96 = $40).
  - It is silent while the player is dying or waiting (playerOK = 0, $1747) and when no aliens are left.
  - There is no march in the attract demo, because SplashDemo enters at $0072, after the TimeFleetSound call.
- **Saucer:** port 3 bit 0 is re-set on every game-loop pass while the saucer is on screen and not hit
  (CtrlSaucerSound $1804). It is cleared otherwise ($0707).
- **Player shot:** port 3 bit 1 is on while plyrShotStatus ≠ 0, i.e. while the shot is flying or exploding
  (ShotSound $172C).
- **Player death:** port 3 bit 2 is turned on in the game loop as soon as playerAlive ≠ $FF ($0844). It is also
  turned on when the invaders land ($16F1). All port-3 sounds go off when the explosion ends ($02C1).
- **Invader hit:** port 3 bit 3 at the kill (ScoreForAlien $0A67, game mode only). It is cleared when the explosion
  ends ($154E).
- **Saucer hit:** port 5 bit 4 at hit-timer $1F, which also mutes the fleet ($074B-$0753). It is cleared at timer 0
  ($06EA-$06F4).
- **Extra ship:** port 3 bit 4 ($0977), held for extraHold $2099 = $FF game-loop passes ($17AA). That count is not
  tied to frames, so its length in frames is **UNCONFIRMED**.
- **Amp:** port 3 bit 5 is set when play starts ($081A). All sound is turned off in attract ($0AEA).

## 9. HUD layout and colour overlay

Positions are the top-left of 8 px character cells (addresses are converted with the formula in section 0).
- **Header** " SCORE<1> HI-SCORE SCORE<2> ": $241E, 28 characters, **x 0-223, y 8-15** (DrawScoreHead $191A).
  - Character i sits at x = 8i: "SCORE<1>" x 8-71, "HI-SCORE" x 80-143, "SCORE<2>" x 152-215.
- **Score digits** (4 each, y 24-31; positions from init data $1BF6-$1BFF):
  - P1 at $271C: x 24-55;
  - HI at $2F1C: x 88-119;
  - P2 at $391C: x 168-199.

  In a 1-player game the P2 digits are blanked ($08E4).
- **Bottom row** (y 240-247):
  - lives digit at $2501: x 8-15;
  - reserve-ship icons from $2701: the player sprite, 16 px apart, at x 24, 40, 56, …; the area is cleared up to
    x 135 ($19E6-$1A02);
  - "CREDIT " at $3501: x 136-191;
  - 2-digit BCD credit count at $3C01: x 192-207 ($193C, $1947).
- **Green line:** 1 px at **y 239**, x 0-223 (DrawBottomLine $01CF: byte $2402 bit 0 for $E0 lines). It is redrawn
  every rack.
- **ClearPlayField $09D6** clears only y 32-239. The header, the scores and the bottom row stay.
- **Overlay:** this is not in the ROM; it was a physical gel. The source is MAME `src/mame/layout/invaders.lay`.
  - MAME's rectangles, in a 224×260 element space: white everywhere; **red y 32-62** (it was 32-64 until 2008,
    when it was changed for MAME Testers bug 02049, "red overlay … shows on top line of invaders heads");
    **green y 184-240, full width**; **green y 240-260 for x 16-134 only**.
  - Read as pixel rows: red ≈ y 32-61/63, green y 184-239, bottom 16 rows green only for x 16-133.
    - The saucer (y 41-47) is red.
    - The scores (y 24-31) and the first rack (y ≥ 64) are white.
    - The shields, the cannon and the line at y 239 are green.
    - In the bottom row, the reserve-ship icons are green; the lives digit (x 8-15) and CREDIT are white.
  - Caveat: MAME stretches the 260-tall element onto 256 lines, which puts the boundaries about 1-4 px higher.
    That would make the y-239 line green only for x 16-134.
  - Real Midway gels varied (Arcade-Museum forum; tobiasvl.github.io/blog/space-invaders). The exact physical
    boundaries are **UNCONFIRMED**.

## 10. Attract mode (code from $0AEA)

The sequence alternates between two versions. splashAnimate $20EC starts at 1 (no animation) and flips every cycle
($0BDA).
1. Wait 64 frames (OneSecDelay $0AB1).
2. "PLAY" is typed out at x 96, y 64, about 6 frames per letter (PrintMessageDel $0A93). In the animated version it
   reads "PLAy", with an upside-down Y (char $29). Then "SPACE  INVADERS" (two spaces) is typed at x 56, y 88.
3. Wait 64 frames. "*SCORE ADVANCE TABLE*" appears at once at x 32, y 120 (DrawAdvTable $1815). The saucer, squid,
   crab (frame 1) and octopus icons are drawn at x 64, y 136/152/168/184. Then "=? MYSTERY", "=30 POINTS",
   "=20 POINTS" and "=10 POINTS" are typed at x 80. Wait 128 frames.
4. Animated version only: a squid walks in at y 64 from x 222 to x 126, 1 px per frame. It then drags the
   upside-down Y off to the right (x 120→223). After 64 frames it comes back pushing an upright Y (x 223→119).
   After another 64 frames the alien is erased and the Y stays.
   Data: $1A95, $1BB0 and $1FC9; sprites $1BA0/$1BD0 and $1F80/$1FB0.
5. Demo game ($0B4A): rack, shields and line are drawn, with no sound. The demo cannon fires constantly and moves
   according to DemoCommands $1F74 (01=right, 02=left), which advance one step per shot. The demo ends when the
   cannon is hit.
6. Credit screen ($0B89):
   - "INSERT  COIN" at x 64, y 112. In the animated version an extra "C" is added at x 120, giving "INSERT CCOIN".
   - "<1 OR 2 PLAYERS>" at x 48, y 144.
   - If the coinage DIP (IN2 bit 7 = 0, MAME default On) is set: "*1 PLAYER  1 COIN" at y 168 and
     "*2 PLAYERS 2 COINS" at y 192.
   - Wait 128 frames. Animated version only: a squid walks along y 40 from x 2 to x 116 and drops a squiggly shot
     that blows away the extra "C" ($189E). Then the loop starts again.
- **Hidden message:** a button sequence during the demo prints "TAITO COP" ($199A).
- **With credits inserted:** "PUSH" appears at x 96, y 96, with "ONLY 1PLAYER  BUTTON" or "1 OR 2PLAYERS BUTTON" at
  x 32, y 120 (WaitForStart $0765).
- **TILT:** "TILT" at x 96, y 72.

## Appendix: data tables (exact ROM bytes)

- **Saucer direction bits.** Character n of this string is bit 0 of ROM[$0800+n]. After the n-th player shot of the
  rack/turn (n mod 256): 1 = enters from the left moving right, 0 = enters from the right moving left. Shot 1 uses
  index 1.
  `1010111010100011011111011100100100101111101000101110111110010111001000100111101001100001001100011110101001111011111001000001010110000001010011111010011001010011101000000000100000010111111000101010101010111011010010110000100100010001010100110010111011101011`
- **DemoCommands $1F74:** 01 01 00 00 01 00 02 01 00 02. The pointer runs $1F75-$1F7D, then wraps to $1F74
  ($165A-$1661).
- **Corrections to the listing's own comments:**
  - The alien-shot speed is -4/-5, not -1/-4.
  - The saucer direction comes from the ROM byte, not from the count's parity.
  - The saucer timer counts frames, not game loops.
  - The player object runs in the ScanLine224 ISR, not "mid-screen". The rate is once per frame either way.
  - The DrawAlien type comment is wrong.
