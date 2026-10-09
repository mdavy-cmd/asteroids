# Space Invaders (Midway 1978, MAME set "invaders") - raw graphics data

## Where this comes from

All bytes come from the commented listing at https://computerarcheology.com/Arcade/SpaceInvaders/Code.html.
I rebuilt the 8 KB ROM image from the hex column of that listing. As a check, I computed the CRC32 of each 2 KB part
and compared them with the CRCs MAME uses for the `invaders` set (734f5ad8 for .h, 6bfaca4a for .g, 0ccead96 for .f,
14e538b0 for .e). All four match exactly, after zeroing $1000-$13FF. That range holds the old Taito diagnostic routine,
which the listing shows only for reference. So every byte below is the real Midway ROM data. Each entry gives its ROM address.

## How the hardware stores pixels, and how I turned it into portrait

* The video RAM runs from $2400 to $3FFF. Each raw scan line is 32 bytes (256 pixels), and there are 224 lines. Bit 0 of a byte is the first
  pixel on the line. The monitor is turned 90 degrees anticlockwise (computerarcheology Hardware page: "the first byte is lower left. First
  'row' ends upper left").
* Portrait coordinates are x to the right (0-223) and y downwards (0-255). For screen address A and bit b:
  `x = (A - 0x2400) >> 5` and `y = 255 - (((A - 0x2400) & 31) * 8 + b)`.
* Sprites and characters are stored as a list of bytes. **Each ROM byte is one portrait column**, and the bytes run from left to right.
  **Bit 7 is the top pixel and bit 0 is the bottom pixel** of an 8-pixel-tall column.
  Shield is the only exception: it has 2 bytes per column. Byte 2j is the lower 8 pixels and byte 2j+1 is the upper 8 pixels.
* In the arrays below, every sprite is converted to **row-major portrait form**:
  * `rows`: one string per pixel row, top row first. `'#'` means lit and `'.'` means dark. `rows[y][x]`.
  * `bits`: the same rows as integers. The **most significant bit (bit w-1) is the leftmost pixel**.
  * `raw`: the original ROM bytes, in column order, if you'd rather draw them the way the hardware does.
* The game positions sprites with a 16-bit "pixel number". In the disassembly H is called Xr and L is called Yr.
  The sprite's top-left corner in portrait is `x = H - 0x20`, `y = 248 - L`. L counts up from the bottom of the screen,
  so the sprite's bit-0 row is at y = 255 - L. The low 3 bits of L go to the hardware shifter.
* Every sprite cell is 8 px tall. Where a sprite has blank rows or columns, they are padding that really exists in the ROM.
  The game depends on that padding: for example, the saucer's 4 blank columns erase its trail as it moves 2 px at a time.

## Notes

* Invader sprites are drawn **without OR**: the routine DrawSprite at $15D3 writes 2 bytes per column. That means a 16-wide
  x 16-tall block is overwritten, made of the 8 sprite rows plus 8 blank rows above them.
* Shots and their explosions are ORed in, with collision testing (routine $1491). They are erased with AND-NOT (routine $1452).
* The player shot is 1 byte, $0F. Only the bottom 4 rows of its 8-row cell are lit.
* Alien shots have 4 animation frames of 3 bytes each. The frame advances by one on every move (code at $05D4).
* `alienExplosion` is shown for 16 frames, `playerShotExplosion` for 16, `alienShotExplosion` for 3 shot ticks (9 frames),
  and `saucerExplosion` for 7 saucer ticks (21 frames). Details are in facts.md.
* Character codes: $00-$19 = A-Z, $1A-$23 = 0-9, $24 '<', $25 '>', $26 space, $27 '=', $28 '*', $29 upside-down Y
  (attract-mode "PLAy"), $38 '?', $3F '-'. The font starts at $1E00 with 8 bytes per character (DrawChar at $08FF).
  Codes $2A-$37 and $39-$3E are not characters, because those ROM slots were reused for messages and other data.
* Character cells are 8x8. The glyphs use rows 1-7, and row 0 is blank. Characters are always drawn byte-aligned
  (DrawSimpSprite at $1439).

```js
// Portrait, row-major. rows[y][x] === '#' means lit. bits[y] has the MSB as the leftmost pixel.
const SPRITES = {
  // ROM $1C00, 16 bytes. 16x8 px (portrait). 10 pts, rack rows 0-1 (bottom two), anim frame 0
  alienA_octopus_f0: { w: 16, h: 8, romAddr: 0x1C00,
    raw: [0x00, 0x00, 0x39, 0x79, 0x7A, 0x6E, 0xEC, 0xFA, 0xFA, 0xEC, 0x6E, 0x7A, 0x79, 0x39, 0x00, 0x00],
    rows: [
      '......####......',
      '...##########...',
      '..############..',
      '..###..##..###..',
      '..############..',
      '.....##..##.....',
      '....##.##.##....',
      '..##........##..',
    ],
    bits: [0x03C0, 0x1FF8, 0x3FFC, 0x399C, 0x3FFC, 0x0660, 0x0DB0, 0x300C] },
  // ROM $1C30, 16 bytes. 16x8 px (portrait). 10 pts, anim frame 1
  alienA_octopus_f1: { w: 16, h: 8, romAddr: 0x1C30,
    raw: [0x00, 0x00, 0x38, 0x7A, 0x7F, 0x6D, 0xEC, 0xFA, 0xFA, 0xEC, 0x6D, 0x7F, 0x7A, 0x38, 0x00, 0x00],
    rows: [
      '......####......',
      '...##########...',
      '..############..',
      '..###..##..###..',
      '..############..',
      '....###..###....',
      '...##..##..##...',
      '....##....##....',
    ],
    bits: [0x03C0, 0x1FF8, 0x3FFC, 0x399C, 0x3FFC, 0x0E70, 0x1998, 0x0C30] },
  // ROM $1C10, 16 bytes. 16x8 px (portrait). 20 pts, rack rows 2-3, anim frame 0
  alienB_crab_f0: { w: 16, h: 8, romAddr: 0x1C10,
    raw: [0x00, 0x00, 0x00, 0x78, 0x1D, 0xBE, 0x6C, 0x3C, 0x3C, 0x3C, 0x6C, 0xBE, 0x1D, 0x78, 0x00, 0x00],
    rows: [
      '.....#.....#....',
      '...#..#...#..#..',
      '...#.#######.#..',
      '...###.###.###..',
      '...###########..',
      '....#########...',
      '.....#.....#....',
      '....#.......#...',
    ],
    bits: [0x0410, 0x1224, 0x17F4, 0x1DDC, 0x1FFC, 0x0FF8, 0x0410, 0x0808] },
  // ROM $1C40, 16 bytes. 16x8 px (portrait). 20 pts, anim frame 1
  alienB_crab_f1: { w: 16, h: 8, romAddr: 0x1C40,
    raw: [0x00, 0x00, 0x00, 0x0E, 0x18, 0xBE, 0x6D, 0x3D, 0x3C, 0x3D, 0x6D, 0xBE, 0x18, 0x0E, 0x00, 0x00],
    rows: [
      '.....#.....#....',
      '......#...#.....',
      '.....#######....',
      '....##.###.##...',
      '...###########..',
      '...#.#######.#..',
      '...#.#.....#.#..',
      '......##.##.....',
    ],
    bits: [0x0410, 0x0220, 0x07F0, 0x0DD8, 0x1FFC, 0x17F4, 0x1414, 0x0360] },
  // ROM $1C20, 16 bytes. 16x8 px (portrait). 30 pts, rack row 4 (top), anim frame 0
  alienC_squid_f0: { w: 16, h: 8, romAddr: 0x1C20,
    raw: [0x00, 0x00, 0x00, 0x00, 0x19, 0x3A, 0x6D, 0xFA, 0xFA, 0x6D, 0x3A, 0x19, 0x00, 0x00, 0x00, 0x00],
    rows: [
      '.......##.......',
      '......####......',
      '.....######.....',
      '....##.##.##....',
      '....########....',
      '......#..#......',
      '.....#.##.#.....',
      '....#.#..#.#....',
    ],
    bits: [0x0180, 0x03C0, 0x07E0, 0x0DB0, 0x0FF0, 0x0240, 0x05A0, 0x0A50] },
  // ROM $1C50, 16 bytes. 16x8 px (portrait). 30 pts, anim frame 1
  alienC_squid_f1: { w: 16, h: 8, romAddr: 0x1C50,
    raw: [0x00, 0x00, 0x00, 0x00, 0x1A, 0x3D, 0x68, 0xFC, 0xFC, 0x68, 0x3D, 0x1A, 0x00, 0x00, 0x00, 0x00],
    rows: [
      '.......##.......',
      '......####......',
      '.....######.....',
      '....##.##.##....',
      '....########....',
      '.....#.##.#.....',
      '....#......#....',
      '.....#....#.....',
    ],
    bits: [0x0180, 0x03C0, 0x07E0, 0x0DB0, 0x0FF0, 0x05A0, 0x0810, 0x0420] },
  // ROM $1CC0, 16 bytes. 16x8 px (portrait). drawn over the dead alien for 16 frames
  alienExplosion: { w: 16, h: 8, romAddr: 0x1CC0,
    raw: [0x00, 0x08, 0x49, 0x22, 0x14, 0x81, 0x42, 0x00, 0x42, 0x81, 0x14, 0x22, 0x49, 0x08, 0x00, 0x00],
    rows: [
      '.....#...#......',
      '..#...#.#...#...',
      '...#.......#....',
      '....#.....#.....',
      '.##.........##..',
      '....#.....#.....',
      '...#..#.#..#....',
      '..#..#...#..#...',
    ],
    bits: [0x0440, 0x2288, 0x1010, 0x0820, 0x600C, 0x0820, 0x1290, 0x2448] },
  // ROM $1C60, 16 bytes. 16x8 px (portrait). also used for reserve-ship icons
  player: { w: 16, h: 8, romAddr: 0x1C60,
    raw: [0x00, 0x00, 0x0F, 0x1F, 0x1F, 0x1F, 0x1F, 0x7F, 0xFF, 0x7F, 0x1F, 0x1F, 0x1F, 0x1F, 0x0F, 0x00],
    rows: [
      '........#.......',
      '.......###......',
      '.......###......',
      '...###########..',
      '..#############.',
      '..#############.',
      '..#############.',
      '..#############.',
    ],
    bits: [0x0080, 0x01C0, 0x01C0, 0x1FFC, 0x3FFE, 0x3FFE, 0x3FFE, 0x3FFE] },
  // ROM $1C70, 16 bytes. 16x8 px (portrait). player blow-up frame A
  playerExplosion0: { w: 16, h: 8, romAddr: 0x1C70,
    raw: [0x00, 0x04, 0x01, 0x13, 0x03, 0x07, 0xB3, 0x0F, 0x2F, 0x03, 0x2F, 0x49, 0x04, 0x03, 0x00, 0x01],
    rows: [
      '......#.........',
      '...........#....',
      '......#.#.#.....',
      '...#..#.........',
      '.......##.##....',
      '.#...#.##.#.#...',
      '...########..#..',
      '..##########.#.#',
    ],
    bits: [0x0200, 0x0010, 0x02A0, 0x1200, 0x01B0, 0x45A8, 0x1FE4, 0x3FF5] },
  // ROM $1C80, 16 bytes. 16x8 px (portrait). player blow-up frame B (shown first)
  playerExplosion1: { w: 16, h: 8, romAddr: 0x1C80,
    raw: [0x40, 0x08, 0x05, 0xA3, 0x0A, 0x03, 0x5B, 0x0F, 0x27, 0x27, 0x0B, 0x4B, 0x40, 0x84, 0x11, 0x48],
    rows: [
      '...#.........#..',
      '#.....#....##..#',
      '...#....##......',
      '......#.......#.',
      '.#..#.##..##...#',
      '..#....###...#..',
      '...#########....',
      '..##.#######..#.',
    ],
    bits: [0x1004, 0x8219, 0x10C0, 0x0202, 0x4B31, 0x21C4, 0x1FF0, 0x37F2] },
  // ROM $1C90, 1 bytes. 1x8 px (portrait). lit pixels are the bottom 4 rows of the 8-row cell
  playerShot: { w: 1, h: 8, romAddr: 0x1C90,
    raw: [0x0F],
    rows: [
      '.',
      '.',
      '.',
      '.',
      '#',
      '#',
      '#',
      '#',
    ],
    bits: [0x0, 0x0, 0x0, 0x0, 0x1, 0x1, 0x1, 0x1] },
  // ROM $1C91, 8 bytes. 8x8 px (portrait). drawn at shot x-3, 2 px lower than shot cell
  playerShotExplosion: { w: 8, h: 8, romAddr: 0x1C91,
    raw: [0x99, 0x3C, 0x7E, 0x3D, 0xBC, 0x3E, 0x7C, 0x99],
    rows: [
      '#...#..#',
      '..#...#.',
      '.######.',
      '########',
      '########',
      '.######.',
      '..#..#..',
      '#..#...#',
    ],
    bits: [0x89, 0x22, 0x7E, 0xFF, 0xFF, 0x7E, 0x24, 0x91] },
  // ROM $1CEE, 3 bytes. 3x8 px (portrait).
  rollingShot_f0: { w: 3, h: 8, romAddr: 0x1CEE,
    raw: [0x00, 0xFE, 0x00],
    rows: [
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '...',
    ],
    bits: [0x2, 0x2, 0x2, 0x2, 0x2, 0x2, 0x2, 0x0] },
  // ROM $1CF1, 3 bytes. 3x8 px (portrait).
  rollingShot_f1: { w: 3, h: 8, romAddr: 0x1CF1,
    raw: [0x24, 0xFE, 0x12],
    rows: [
      '.#.',
      '.#.',
      '##.',
      '.##',
      '.#.',
      '##.',
      '.##',
      '...',
    ],
    bits: [0x2, 0x2, 0x6, 0x3, 0x2, 0x6, 0x3, 0x0] },
  // ROM $1CF4, 3 bytes. 3x8 px (portrait).
  rollingShot_f2: { w: 3, h: 8, romAddr: 0x1CF4,
    raw: [0x00, 0xFE, 0x00],
    rows: [
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '...',
    ],
    bits: [0x2, 0x2, 0x2, 0x2, 0x2, 0x2, 0x2, 0x0] },
  // ROM $1CF7, 3 bytes. 3x8 px (portrait).
  rollingShot_f3: { w: 3, h: 8, romAddr: 0x1CF7,
    raw: [0x48, 0xFE, 0x90],
    rows: [
      '.##',
      '##.',
      '.#.',
      '.##',
      '##.',
      '.#.',
      '.#.',
      '...',
    ],
    bits: [0x3, 0x6, 0x2, 0x3, 0x6, 0x2, 0x2, 0x0] },
  // ROM $1CE2, 3 bytes. 3x8 px (portrait).
  plungerShot_f0: { w: 3, h: 8, romAddr: 0x1CE2,
    raw: [0x04, 0xFC, 0x04],
    rows: [
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '###',
      '...',
      '...',
    ],
    bits: [0x2, 0x2, 0x2, 0x2, 0x2, 0x7, 0x0, 0x0] },
  // ROM $1CE5, 3 bytes. 3x8 px (portrait).
  plungerShot_f1: { w: 3, h: 8, romAddr: 0x1CE5,
    raw: [0x10, 0xFC, 0x10],
    rows: [
      '.#.',
      '.#.',
      '.#.',
      '###',
      '.#.',
      '.#.',
      '...',
      '...',
    ],
    bits: [0x2, 0x2, 0x2, 0x7, 0x2, 0x2, 0x0, 0x0] },
  // ROM $1CE8, 3 bytes. 3x8 px (portrait).
  plungerShot_f2: { w: 3, h: 8, romAddr: 0x1CE8,
    raw: [0x20, 0xFC, 0x20],
    rows: [
      '.#.',
      '.#.',
      '###',
      '.#.',
      '.#.',
      '.#.',
      '...',
      '...',
    ],
    bits: [0x2, 0x2, 0x7, 0x2, 0x2, 0x2, 0x0, 0x0] },
  // ROM $1CEB, 3 bytes. 3x8 px (portrait).
  plungerShot_f3: { w: 3, h: 8, romAddr: 0x1CEB,
    raw: [0x80, 0xFC, 0x80],
    rows: [
      '###',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '.#.',
      '...',
      '...',
    ],
    bits: [0x7, 0x2, 0x2, 0x2, 0x2, 0x2, 0x0, 0x0] },
  // ROM $1CD0, 3 bytes. 3x8 px (portrait).
  squigglyShot_f0: { w: 3, h: 8, romAddr: 0x1CD0,
    raw: [0x44, 0xAA, 0x10],
    rows: [
      '.#.',
      '#..',
      '.#.',
      '..#',
      '.#.',
      '#..',
      '.#.',
      '...',
    ],
    bits: [0x2, 0x4, 0x2, 0x1, 0x2, 0x4, 0x2, 0x0] },
  // ROM $1CD3, 3 bytes. 3x8 px (portrait).
  squigglyShot_f1: { w: 3, h: 8, romAddr: 0x1CD3,
    raw: [0x88, 0x54, 0x22],
    rows: [
      '#..',
      '.#.',
      '..#',
      '.#.',
      '#..',
      '.#.',
      '..#',
      '...',
    ],
    bits: [0x4, 0x2, 0x1, 0x2, 0x4, 0x2, 0x1, 0x0] },
  // ROM $1CD6, 3 bytes. 3x8 px (portrait).
  squigglyShot_f2: { w: 3, h: 8, romAddr: 0x1CD6,
    raw: [0x10, 0xAA, 0x44],
    rows: [
      '.#.',
      '..#',
      '.#.',
      '#..',
      '.#.',
      '..#',
      '.#.',
      '...',
    ],
    bits: [0x2, 0x1, 0x2, 0x4, 0x2, 0x1, 0x2, 0x0] },
  // ROM $1CD9, 3 bytes. 3x8 px (portrait).
  squigglyShot_f3: { w: 3, h: 8, romAddr: 0x1CD9,
    raw: [0x22, 0x54, 0x88],
    rows: [
      '..#',
      '.#.',
      '#..',
      '.#.',
      '..#',
      '.#.',
      '#..',
      '...',
    ],
    bits: [0x1, 0x2, 0x4, 0x2, 0x1, 0x2, 0x4, 0x0] },
  // ROM $1CDC, 6 bytes. 6x8 px (portrait). drawn at shot x-2, 2 px lower than shot cell
  alienShotExplosion: { w: 6, h: 8, romAddr: 0x1CDC,
    raw: [0x4A, 0x15, 0xBE, 0x3F, 0x5E, 0x25],
    rows: [
      '..#...',
      '#...#.',
      '..##.#',
      '.####.',
      '#.###.',
      '.#####',
      '#.###.',
      '.#.#.#',
    ],
    bits: [0x08, 0x22, 0x0D, 0x1E, 0x2E, 0x1F, 0x2E, 0x15] },
  // ROM $1D64, 24 bytes. 24x8 px (portrait). 4 blank columns each side (self-erasing when moving 2 px)
  saucer: { w: 24, h: 8, romAddr: 0x1D64,
    raw: [0x00, 0x00, 0x00, 0x00, 0x04, 0x0C, 0x1E, 0x37, 0x3E, 0x7C, 0x74, 0x7E, 0x7E, 0x74, 0x7C, 0x3E, 0x37, 0x1E, 0x0C, 0x04, 0x00, 0x00, 0x00, 0x00],
    rows: [
      '........................',
      '.........######.........',
      '.......##########.......',
      '......############......',
      '.....##.##.##.##.##.....',
      '....################....',
      '......###..##..###......',
      '.......#........#.......',
    ],
    bits: [0x000000, 0x007E00, 0x01FF80, 0x03FFC0, 0x06DB60, 0x0FFFF0, 0x0399C0, 0x010080] },
  // ROM $1D7C, 24 bytes. 24x8 px (portrait).
  saucerExplosion: { w: 24, h: 8, romAddr: 0x1D7C,
    raw: [0x00, 0x22, 0x00, 0xA5, 0x40, 0x08, 0x98, 0x3D, 0xB6, 0x3C, 0x36, 0x1D, 0x10, 0x48, 0x62, 0xB6, 0x1D, 0x98, 0x08, 0x42, 0x90, 0x08, 0x00, 0x00],
    rows: [
      '...#..#.#......#.#..#...',
      '....#........##....#....',
      '.#.#...####...##........',
      '......#######..###..#...',
      '.....###.#.#.#..###..#..',
      '...#...#####...##.......',
      '.#......#.#...##...#....',
      '...#...#...#....#.......',
    ],
    bits: [0x128148, 0x080610, 0x51E300, 0x03F9C8, 0x0754E4, 0x11F180, 0x40A310, 0x111080] },
  // ROM $1BA0, 16 bytes. 16x8 px (portrait). attract animation
  splashAlienPullingUpsideDownY_f0: { w: 16, h: 8, romAddr: 0x1BA0,
    raw: [0x00, 0x03, 0x04, 0x78, 0x14, 0x13, 0x08, 0x1A, 0x3D, 0x68, 0xFC, 0xFC, 0x68, 0x3D, 0x1A, 0x00],
    rows: [
      '..........##....',
      '...#.....####...',
      '...#....######..',
      '...###.##.##.##.',
      '...#..#########.',
      '..#.#...#.##.#..',
      '.#...#.#......#.',
      '.#...#..#....#..',
    ],
    bits: [0x0030, 0x1078, 0x10FC, 0x1DB6, 0x13FE, 0x28B4, 0x4502, 0x4484] },
  // ROM $1BD0, 16 bytes. 16x8 px (portrait). attract animation
  splashAlienPullingUpsideDownY_f1: { w: 16, h: 8, romAddr: 0x1BD0,
    raw: [0x00, 0x00, 0x03, 0x04, 0x78, 0x14, 0x0B, 0x19, 0x3A, 0x6D, 0xFA, 0xFA, 0x6D, 0x3A, 0x19, 0x00],
    rows: [
      '..........##....',
      '....#....####...',
      '....#...######..',
      '....##.##.##.##.',
      '....#.#########.',
      '...#.#...#..#...',
      '..#...#.#.##.#..',
      '..#...##.#..#.#.',
    ],
    bits: [0x0030, 0x0878, 0x08FC, 0x0DB6, 0x0BFE, 0x1448, 0x22B4, 0x234A] },
  // ROM $1F80, 16 bytes. 16x8 px (portrait). attract animation
  splashAlienPushingY_f0: { w: 16, h: 8, romAddr: 0x1F80,
    raw: [0x60, 0x10, 0x0F, 0x10, 0x60, 0x30, 0x18, 0x1A, 0x3D, 0x68, 0xFC, 0xFC, 0x68, 0x3D, 0x1A, 0x00],
    rows: [
      '..........##....',
      '#...#....####...',
      '#...##..######..',
      '.#.#.####.##.##.',
      '..#...#########.',
      '..#.....#.##.#..',
      '..#....#......#.',
      '..#.....#....#..',
    ],
    bits: [0x0030, 0x8878, 0x8CFC, 0x57B6, 0x23FE, 0x20B4, 0x2102, 0x2084] },
  // ROM $1FB0, 16 bytes. 16x8 px (portrait). attract animation
  splashAlienPushingY_f1: { w: 16, h: 8, romAddr: 0x1FB0,
    raw: [0x00, 0x60, 0x10, 0x0F, 0x10, 0x60, 0x38, 0x19, 0x3A, 0x6D, 0xFA, 0xFA, 0x6D, 0x3A, 0x19, 0x00],
    rows: [
      '..........##....',
      '.#...#...####...',
      '.#...##.######..',
      '..#.#.###.##.##.',
      '...#..#########.',
      '...#.....#..#...',
      '...#....#.##.#..',
      '...#...#.#..#.#.',
    ],
    bits: [0x0030, 0x4478, 0x46FC, 0x2BB6, 0x13FE, 0x1048, 0x10B4, 0x114A] },
  // ROM $1D20, 44 bytes. 22x16 px (portrait). 2 bytes per column: byte 2j = lower 8 px, byte 2j+1 = upper 8 px
  shield: { w: 22, h: 16, romAddr: 0x1D20,
    raw: [0xFF, 0x0F, 0xFF, 0x1F, 0xFF, 0x3F, 0xFF, 0x7F, 0xFF, 0xFF, 0xFC, 0xFF, 0xF8, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF0, 0xFF, 0xF8, 0xFF, 0xFC, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x7F, 0xFF, 0x3F, 0xFF, 0x1F, 0xFF, 0x0F],
    rows: [
      '....##############....',
      '...################...',
      '..##################..',
      '.####################.',
      '######################',
      '######################',
      '######################',
      '######################',
      '######################',
      '######################',
      '######################',
      '######################',
      '#######.......########',
      '######.........#######',
      '#####...........######',
      '#####...........######',
    ],
    bits: [0x03FFF0, 0x07FFF8, 0x0FFFFC, 0x1FFFFE, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3FFFFF, 0x3F80FF, 0x3F007F, 0x3E003F, 0x3E003F] },
};

// 8x8 font, portrait, row-major (rows[0] = top). Keys are the characters; `code` is the ROM character code.
const FONT = {
  'A': { code: 0x00, romAddr: 0x1E00, raw: [0x00, 0x1F, 0x24, 0x44, 0x24, 0x1F, 0x00, 0x00], rows: ['........', '...#....', '..#.#...', '.#...#..', '.#...#..', '.#####..', '.#...#..', '.#...#..'] },
  'B': { code: 0x01, romAddr: 0x1E08, raw: [0x00, 0x7F, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00], rows: ['........', '.####...', '.#...#..', '.#...#..', '.####...', '.#...#..', '.#...#..', '.####...'] },
  'C': { code: 0x02, romAddr: 0x1E10, raw: [0x00, 0x3E, 0x41, 0x41, 0x41, 0x22, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#......', '.#......', '.#......', '.#...#..', '..###...'] },
  'D': { code: 0x03, romAddr: 0x1E18, raw: [0x00, 0x7F, 0x41, 0x41, 0x41, 0x3E, 0x00, 0x00], rows: ['........', '.####...', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.####...'] },
  'E': { code: 0x04, romAddr: 0x1E20, raw: [0x00, 0x7F, 0x49, 0x49, 0x49, 0x41, 0x00, 0x00], rows: ['........', '.#####..', '.#......', '.#......', '.####...', '.#......', '.#......', '.#####..'] },
  'F': { code: 0x05, romAddr: 0x1E28, raw: [0x00, 0x7F, 0x48, 0x48, 0x48, 0x40, 0x00, 0x00], rows: ['........', '.#####..', '.#......', '.#......', '.####...', '.#......', '.#......', '.#......'] },
  'G': { code: 0x06, romAddr: 0x1E30, raw: [0x00, 0x3E, 0x41, 0x41, 0x45, 0x47, 0x00, 0x00], rows: ['........', '..####..', '.#......', '.#......', '.#......', '.#..##..', '.#...#..', '..####..'] },
  'H': { code: 0x07, romAddr: 0x1E38, raw: [0x00, 0x7F, 0x08, 0x08, 0x08, 0x7F, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '.#...#..', '.#####..', '.#...#..', '.#...#..', '.#...#..'] },
  'I': { code: 0x08, romAddr: 0x1E40, raw: [0x00, 0x00, 0x41, 0x7F, 0x41, 0x00, 0x00, 0x00], rows: ['........', '..###...', '...#....', '...#....', '...#....', '...#....', '...#....', '..###...'] },
  'J': { code: 0x09, romAddr: 0x1E48, raw: [0x00, 0x02, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00], rows: ['........', '.....#..', '.....#..', '.....#..', '.....#..', '.....#..', '.#...#..', '..###...'] },
  'K': { code: 0x0A, romAddr: 0x1E50, raw: [0x00, 0x7F, 0x08, 0x14, 0x22, 0x41, 0x00, 0x00], rows: ['........', '.#...#..', '.#..#...', '.#.#....', '.##.....', '.#.#....', '.#..#...', '.#...#..'] },
  'L': { code: 0x0B, romAddr: 0x1E58, raw: [0x00, 0x7F, 0x01, 0x01, 0x01, 0x01, 0x00, 0x00], rows: ['........', '.#......', '.#......', '.#......', '.#......', '.#......', '.#......', '.#####..'] },
  'M': { code: 0x0C, romAddr: 0x1E60, raw: [0x00, 0x7F, 0x20, 0x18, 0x20, 0x7F, 0x00, 0x00], rows: ['........', '.#...#..', '.##.##..', '.#.#.#..', '.#.#.#..', '.#...#..', '.#...#..', '.#...#..'] },
  'N': { code: 0x0D, romAddr: 0x1E68, raw: [0x00, 0x7F, 0x10, 0x08, 0x04, 0x7F, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '.##..#..', '.#.#.#..', '.#..##..', '.#...#..', '.#...#..'] },
  'O': { code: 0x0E, romAddr: 0x1E70, raw: [0x00, 0x3E, 0x41, 0x41, 0x41, 0x3E, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '..###...'] },
  'P': { code: 0x0F, romAddr: 0x1E78, raw: [0x00, 0x7F, 0x48, 0x48, 0x48, 0x30, 0x00, 0x00], rows: ['........', '.####...', '.#...#..', '.#...#..', '.####...', '.#......', '.#......', '.#......'] },
  'Q': { code: 0x10, romAddr: 0x1E80, raw: [0x00, 0x3E, 0x41, 0x45, 0x42, 0x3D, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#...#..', '.#...#..', '.#.#.#..', '.#..#...', '..##.#..'] },
  'R': { code: 0x11, romAddr: 0x1E88, raw: [0x00, 0x7F, 0x48, 0x4C, 0x4A, 0x31, 0x00, 0x00], rows: ['........', '.####...', '.#...#..', '.#...#..', '.####...', '.#.#....', '.#..#...', '.#...#..'] },
  'S': { code: 0x12, romAddr: 0x1E90, raw: [0x00, 0x32, 0x49, 0x49, 0x49, 0x26, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#......', '..###...', '.....#..', '.#...#..', '..###...'] },
  'T': { code: 0x13, romAddr: 0x1E98, raw: [0x00, 0x40, 0x40, 0x7F, 0x40, 0x40, 0x00, 0x00], rows: ['........', '.#####..', '...#....', '...#....', '...#....', '...#....', '...#....', '...#....'] },
  'U': { code: 0x14, romAddr: 0x1EA0, raw: [0x00, 0x7E, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '..###...'] },
  'V': { code: 0x15, romAddr: 0x1EA8, raw: [0x00, 0x7C, 0x02, 0x01, 0x02, 0x7C, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '.#...#..', '..#.#...', '...#....'] },
  'W': { code: 0x16, romAddr: 0x1EB0, raw: [0x00, 0x7F, 0x02, 0x0C, 0x02, 0x7F, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '.#...#..', '.#.#.#..', '.#.#.#..', '.##.##..', '.#...#..'] },
  'X': { code: 0x17, romAddr: 0x1EB8, raw: [0x00, 0x63, 0x14, 0x08, 0x14, 0x63, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '..#.#...', '...#....', '..#.#...', '.#...#..', '.#...#..'] },
  'Y': { code: 0x18, romAddr: 0x1EC0, raw: [0x00, 0x60, 0x10, 0x0F, 0x10, 0x60, 0x00, 0x00], rows: ['........', '.#...#..', '.#...#..', '..#.#...', '...#....', '...#....', '...#....', '...#....'] },
  'Z': { code: 0x19, romAddr: 0x1EC8, raw: [0x00, 0x43, 0x45, 0x49, 0x51, 0x61, 0x00, 0x00], rows: ['........', '.#####..', '.....#..', '....#...', '...#....', '..#.....', '.#......', '.#####..'] },
  '0': { code: 0x1A, romAddr: 0x1ED0, raw: [0x00, 0x3E, 0x45, 0x49, 0x51, 0x3E, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#..##..', '.#.#.#..', '.##..#..', '.#...#..', '..###...'] },
  '1': { code: 0x1B, romAddr: 0x1ED8, raw: [0x00, 0x00, 0x21, 0x7F, 0x01, 0x00, 0x00, 0x00], rows: ['........', '...#....', '..##....', '...#....', '...#....', '...#....', '...#....', '..###...'] },
  '2': { code: 0x1C, romAddr: 0x1EE0, raw: [0x00, 0x23, 0x45, 0x49, 0x49, 0x31, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.....#..', '...##...', '..#.....', '.#......', '.#####..'] },
  '3': { code: 0x1D, romAddr: 0x1EE8, raw: [0x00, 0x42, 0x41, 0x49, 0x59, 0x66, 0x00, 0x00], rows: ['........', '.#####..', '.....#..', '....#...', '...##...', '.....#..', '.#...#..', '..###...'] },
  '4': { code: 0x1E, romAddr: 0x1EF0, raw: [0x00, 0x0C, 0x14, 0x24, 0x7F, 0x04, 0x00, 0x00], rows: ['........', '....#...', '...##...', '..#.#...', '.#..#...', '.#####..', '....#...', '....#...'] },
  '5': { code: 0x1F, romAddr: 0x1EF8, raw: [0x00, 0x72, 0x51, 0x51, 0x51, 0x4E, 0x00, 0x00], rows: ['........', '.#####..', '.#......', '.####...', '.....#..', '.....#..', '.#...#..', '..###...'] },
  '6': { code: 0x20, romAddr: 0x1F00, raw: [0x00, 0x1E, 0x29, 0x49, 0x49, 0x46, 0x00, 0x00], rows: ['........', '...###..', '..#.....', '.#......', '.####...', '.#...#..', '.#...#..', '..###...'] },
  '7': { code: 0x21, romAddr: 0x1F08, raw: [0x00, 0x40, 0x47, 0x48, 0x50, 0x60, 0x00, 0x00], rows: ['........', '.#####..', '.....#..', '....#...', '...#....', '..#.....', '..#.....', '..#.....'] },
  '8': { code: 0x22, romAddr: 0x1F10, raw: [0x00, 0x36, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#...#..', '..###...', '.#...#..', '.#...#..', '..###...'] },
  '9': { code: 0x23, romAddr: 0x1F18, raw: [0x00, 0x31, 0x49, 0x49, 0x4A, 0x3C, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '.#...#..', '..####..', '.....#..', '....#...', '.###....'] },
  '<': { code: 0x24, romAddr: 0x1F20, raw: [0x00, 0x08, 0x14, 0x22, 0x41, 0x00, 0x00, 0x00], rows: ['........', '....#...', '...#....', '..#.....', '.#......', '..#.....', '...#....', '....#...'] },
  '>': { code: 0x25, romAddr: 0x1F28, raw: [0x00, 0x00, 0x41, 0x22, 0x14, 0x08, 0x00, 0x00], rows: ['........', '..#.....', '...#....', '....#...', '.....#..', '....#...', '...#....', '..#.....'] },
  ' ': { code: 0x26, romAddr: 0x1F30, raw: [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00], rows: ['........', '........', '........', '........', '........', '........', '........', '........'] },
  '=': { code: 0x27, romAddr: 0x1F38, raw: [0x00, 0x14, 0x14, 0x14, 0x14, 0x14, 0x00, 0x00], rows: ['........', '........', '........', '.#####..', '........', '.#####..', '........', '........'] },
  '*': { code: 0x28, romAddr: 0x1F40, raw: [0x00, 0x22, 0x14, 0x7F, 0x14, 0x22, 0x00, 0x00], rows: ['........', '...#....', '.#.#.#..', '..###...', '...#....', '..###...', '.#.#.#..', '...#....'] },
  'Y_upsideDown': { code: 0x29, romAddr: 0x1F48, raw: [0x00, 0x03, 0x04, 0x78, 0x04, 0x03, 0x00, 0x00], rows: ['........', '...#....', '...#....', '...#....', '...#....', '..#.#...', '.#...#..', '.#...#..'] },
  '?': { code: 0x38, romAddr: 0x1FC0, raw: [0x00, 0x20, 0x40, 0x4D, 0x50, 0x20, 0x00, 0x00], rows: ['........', '..###...', '.#...#..', '....#...', '...#....', '...#....', '........', '...#....'] },
  '-': { code: 0x3F, romAddr: 0x1FF8, raw: [0x00, 0x08, 0x08, 0x08, 0x08, 0x08, 0x00, 0x00], rows: ['........', '........', '........', '........', '.#####..', '........', '........', '........'] },
};
```

## Text encoding helper

```js
// Converts ASCII to the ROM's character codes. In the ROM, '^' stands for the upside-down Y ($29).
const CHARCODE = c => c >= 'A' && c <= 'Z' ? c.charCodeAt(0) - 65
                    : c >= '0' && c <= '9' ? 0x1A + c.charCodeAt(0) - 48
                    : ({'<':0x24,'>':0x25,' ':0x26,'=':0x27,'*':0x28,'^':0x29,'?':0x38,'-':0x3F})[c];
```
