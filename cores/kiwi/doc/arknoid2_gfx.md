# Arkanoid 2: graphics ROM layout (gfx1)

## Symptom

When the playfield is rebuilt (at the start of a round and after a life is lost), one frame showed
solid-colour blocks (white, red, peach, yellow) over the playfield on MiSTer. MAME shows a blank
playfield in that frame.

## What the game does

The bricks are drawn with sprites, one per brick cell: sprite slots 0x6F-0x158 (234 cells,
13 columns x 18 rows). The round setup first fills all cells with sprite code 0x2000, which the game
uses for an empty cell. Main CPU code:

    4184: 21 6F C0     LD   HL,$C06F
    4187: 11 70 C0     LD   DE,$C070
    418A: 01 E9 00     LD   BC,$00E9
    418D: 36 00        LD   (HL),$00
    418F: ED B0        LDIR              ; code low bytes = $00
    4191: 21 6F D0     LD   HL,$D06F
    4194: 11 70 D0     LD   DE,$D070
    4197: 01 E9 00     LD   BC,$00E9
    419A: 36 20        LD   (HL),$20
    419C: ED B0        LDIR              ; code high bytes = $20

During that frame the cells are at their grid positions on screen. One frame later the cells with a
brick get the brick's code and the empty cells are parked at y=0, off screen.

## The PCB

`sch/arknoid2/gfx.kicad_sch` shows how the tile code read from VRAM becomes the graphics ROM address:

| Code bit | Latch             | Goes to                                                       |
|:---------|:------------------|:--------------------------------------------------------------|
| 0-7      | A16 (74LS273)     | ROM A5-A12                                                    |
| 8        | A15 (74LS174) Q1  | ROM A13                                                       |
| 9        | A15 Q0            | ROM A14                                                       |
| 10       | A15 Q4            | ROM pin 1 (A15), through JP6                                  |
| 11       | A15 Q5            | ROM pin 22, through JP5                                       |
| 12       | A15 Q3            | chip enable: selects one of the two banks of four ROM sockets |
| 13       | A15 Q2            | not connected                                                 |

Arkanoid 2 has its four ROMs (b08-01..04 at 13a, 10a, 7a, 4a) in one bank; the other bank is empty.
So on the board code 0x2000 addresses tile 0, which is blank, and the empty cells are invisible.
MAME gets the same result because it wraps the code to the number of tiles.

## In the core

`jtkiwi_draw.v` builds the ROM address from all 14 code bits (`rom_addr = { code, ... }`), so code
0x2000 reads offset 0x100000 of the 2 MB gfx1 region. The MRA filled everything after the 512 kB of
ROMs with 0xFF, so these sprites were drawn with pen 15 in every pixel for one frame.

## Change

`cfg/mame2mra.toml` repeats the four ROMs over the whole gfx1 region for arknoid2 and its clones,
instead of padding the region with 0xFF. Code bit 13 then wraps to the same tiles as on the PCB.

Code bit 12 selects the empty bank on the PCB, so strictly the second and fourth copies stand in for
that empty bank. The game does not seem to show anything with bit 12 set: in a MAME trace of about
3400 frames (a game started and left to run until game over) the only such code was 0x3FFF in one
sprite slot at game over, parked off screen. Repeating the ROMs keeps the change simple.

Another way to get the same result would be to ignore code bit 13 in the core for this board, similar
to the existing `drtoppel` option for the tilemap code.

## Checks

- `jtframe mra kiwi` before and after the change: only the four Arkanoid 2 MRAs change.
- The ROM image these MRAs produce was compared byte by byte with the MRA used for testing.
- Tested on MiSTer: the blocks no longer appear, and the game works without problems in our tests.
- The short blank screen and redraw that remain are the game's own round setup: the routine at
  `$2B7E` clears all sprites and tiles, and the round is then drawn over a few frames. The same
  routine runs at the start of a round and after a life is lost, and MAME shows the same frames.
