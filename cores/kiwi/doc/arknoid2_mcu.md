# Arkanoid 2: MCU firmware and error $0E (COIN OVERFLOW)

There is no known dump of the Arkanoid 2 MCU (an i8742), so the core runs the custom firmware in
`firmware/arknoid2.s`. Occasionally a game stopped right after Start with error $0E, shown as
COIN OVERFLOW. This note describes where that comes from, the change to the firmware and how it was
checked. The closed issue "kiwi/arkanoid2: Coin overflow triggers too easily" (number 1488 in
jotego/jtcores) may be related.

## Protocol

The sub CPU (ROM b08__13.3e) talks to the MCU at C000 (data) and C001 (command and status). Status
bit 0 is set while a byte is waiting to be read (OBF), bit 1 while the MCU has not taken the last
written byte yet (IBF), and bits 4-7 carry the coin report. The commands, as also listed in MAME's
simulation (`doc/arkanoid2.cpp`):

| Command      | Meaning                                                                       |
|:-------------|:------------------------------------------------------------------------------|
| C1           | read the credits, then the inputs (two bytes)                                 |
| 54, 41, data | add the data byte to the credits and read the new count; at Start the game sends minus the number of players |
| 15           | take one credit (continue)                                                    |
| 80, 84, 88   | coin lockouts (test mode only)                                                |

Sub CPU helpers: `$0B7A` writes a command and `$0B73` a data byte, and both then wait until the MCU
has taken it (IBF clear). `$0B99` waits until a byte is ready (OBF set) and reads it. All three stop
with error $07 if the MCU does not respond. Test mode uses its own versions without a time limit
(`$097C`, `$0970`, `$0963`).

## Where error $0E comes from

The sub CPU raises it in two places. Once per frame, the C1 poll (`$0CA1`) sends C1, waits
(`$474B`), reads the credits and checks them:

    0CA9: CD 99 0B   CALL $0B99        ; credits
    0CAC: 32 99 E3   LD   ($E399),A
    ...
    0D0C: 3A 99 E3   LD   A,($E399)
    0D0F: FE 64      CP   $64
    0D11: 38 07      JR   C,$0D1A
    0D13: 3E 0E      LD   A,$0E        ; 100 credits or more: error $0E
    0D15: 32 00 E0   LD   ($E000),A
    ...
    0D6E: CD 99 0B   CALL $0B99        ; inputs, the second byte after C1

At Start it takes the credits for the players:

    0BF1: 3E 54      LD   A,$54
    0BF3: CD 7A 0B   CALL $0B7A        ; command 54
    0BF6: 3E 41      LD   A,$41
    0BF8: CD 7A 0B   CALL $0B7A        ; command 41
    0BFB: 79         LD   A,C
    0BFC: CD 73 0B   CALL $0B73        ; data: minus the number of players
    0BFF: CD 99 0B   CALL $0B99        ; the reply is read right away
    0C02: 4F         LD   C,A
    0C03: E7         RST  $20          ; wait one frame, the C1 poll runs meanwhile
    0C04: 3A 99 E3   LD   A,($E399)
    0C07: B9         CP   C
    0C08: 28 07      JR   Z,$0C11
    0C0A: 3E 0E      LD   A,$0E        ; reply differs from the polled credits: error $0E
    0C0C: 32 00 E0   LD   ($E000),A

The main CPU then shows the error screen and the game stops.

## What happened

The main loop of the firmware put the inputs byte into DBB whenever DBB was empty (label `CKCMD`).
After the C1 poll had read its two bytes, DBB was filled again straight away, so an inputs byte that
nobody had asked for was waiting in DBB, with OBF set, when the game started.

At `$0BFF` the sub CPU reads as soon as OBF is set, and OBF was already set by that byte. Two things
then happen at about the same time:

- the firmware answers the data byte: from `IN A,DBB` to `OUT DBB,A` it runs 12 instruction cycles,
  about 30 µs with the MCU at 6 MHz (15 clocks per cycle);
- the sub CPU (Z80 at 6 MHz) reads the data register about 163 T-states (27 µs) after it sees IBF
  clear, and it polls IBF every 45 T-states (7.5 µs), so the read comes 27-35 µs after the MCU took
  the byte.

If `OUT DBB,A` comes first, the sub CPU reads the new credit count and the check at `$0C04` passes.
If the sub CPU comes first, it reads the earlier inputs byte, the check at `$0C04` fails and the game
stops with error $0E. Which one comes first depends on where both CPUs are in their loops, which fits
an error that only appears now and then. The C1 poll is not affected because the sub CPU waits
before reading. MAME's simulation computes each reply when it is read, so this cannot happen there.

## The change

The firmware now only puts a byte into DBB when the sub CPU is going to read it:

| Command      | Reply                                                     |
|:-------------|:----------------------------------------------------------|
| C1           | the credits, then the inputs once the credits are read    |
| 54, 41       | none (they announce a data byte)                          |
| data after 41| the new credit count                                      |
| 15           | none                                                      |
| others       | the inputs (test mode reads one byte after 80/84/88, `$0554`) |

With DBB empty, `$0BFF` waits for the new credit count. Other details:

- `PEND` (R3) remembers that the inputs byte has to follow the credits of a C1 reply.
- The inputs routine (`CKCMD`, now the subroutine `SENDBTN`) used to clear the status nibble right
  after initialization. As it no longer runs then, the status is cleared after the last
  initialization handshake, so $5A does not stay in the coin code bits.
- With 15 and no credits left, the firmware used to reply with the inputs; now it does not reply.
- The coin routines moved to the second page of the program memory (`ORG $100`): conditional jumps
  only reach targets in their own 256-byte page and the code no longer fits in the first one.

The sub CPU always reads the second byte after C1 (`$0D6E`, and `$0542` in test mode), so the inputs
byte that follows the credits is always taken. The only other path after C1 is the tilt reply ($EE),
after which the sub CPU resets the MCU. Command 15 is not followed by a read (`$0C61`).

## Checks

- Tools: The Macroassembler AS 1.42 Beta (Bld 312), `asl -cpu 8042` and `p2bin`, as called by
  `jtframe mra`. With them the previous firmware rebuilds byte for byte (249 bytes), and
  `jtframe mra kiwi` reproduces the asm_md5 of the released Arkanoid 2 MRAs.
- The new firmware assembles with no errors or warnings to 0x16C bytes.
- Tested on MiSTer with the new firmware: coins and Start work, and error $0E has not appeared
  since. It works without problems in our tests.
