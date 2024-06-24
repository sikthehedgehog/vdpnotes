# VDP implementation tidbits

- There's no centralized way to access the VRAM bus, everything that needs it puts their address directly on it and the VDP relies on its timing signals to coordinate who has access to the bus.
- The F flag is the internal "vblank interrupt pending" flag exposed directly on the status port. There are also hblank and external interrupt pending flags, but those aren't visible from outside.
- DMA length is stored _inverted_, then incremented as DMA progresses. This probably saved some gates, I guess.
- CRAM data is internally stored as `bbbgggrrr` in mode 5 and `xxxbbggrr` in mode 4. This should be observable when switching between modes.

## Commands and register writes

- The `CD0` to `CD4` flags are stored in latches like the rest of the command, but `CD5` is strictly logic instead (setting that bit immediately kickstarts the logic that enables the DMA operation).
- Register writes are triggered when `CD1` and `CD0` become `10` after doing a write. The value is copied from the latched command (instead of the data bus).
- There's no centralized register file, instead internally there's an "enable" for each latch spread all over the VDP.
- The register number is decoded in two parts. First bits 4:3 are decoded (`00`, `01`, `01` if mode 5 only, `11` if mode 5 only) and then bits 2:0 separately (eight different lines), a combination of those two is then used to detect the individual registers. The "if mode 5 only" check is used to lock out registers `$0B` onwards (VDP will ignore those writes in mode 4).
- Registers `$0B` and above are locked out outside mode 4 (writes don't go through). When a register write _does_ go through however, the whole register is written, including the bits not relevant to the current video mode (which should be observable when switching to the other mode).

## FIFO

- Byte writes go straight to the FIFO. Each FIFO entry has two flags to keep track of which bytes are valid (upper byte and lower byte flags). Both MD and SMS mode can issue byte writes.
    + For some reason I can't get the VDP to actually treat byte writes in MD mode as byte writes, however (it still writes words). I may be still missing something here.
- The flushing mechanism accounts for both 8-bit and 16-bit VRAM buses by having an extra bit for the flush index. On 8-bit buses the whole index is incremented by +1, and on 16-bit buses it's incremented by +2.
- FIFO FULL is set when the two indices are identical after a write, and reset when the indices stop being identical.
- FIFO EMPTY is set when the two indices are identical after flushing, and reset when the indices stop being identical.
- VDP reset forces the above flags to be EMPTY = 1 and FULL = 0.

## DMA operation

- Internally, length is stored _inverted_ (NOT'd) and then incremented until all bits are set. End result is the same, it probably saved a few gates.
- Usually, during memory writes the CD flags stored in the FIFO are used. DMA fill uses the live command flags however (makes sense, since fill doesn't use the FIFO, though not sure where the fill data comes from yet).

## Sprite system

- Sprite line buffer has a 8px wide data bus and VDP draws 8px at a time.
- Internally there's a mux which changes the order in which sprites are drawn (front to back or back to front). It's hardwired to "front to back" by tying the select line to ground (leftover from an old design?).

## Mode 4 implementation

- In mode 5, the VDP reads two 8×1 tiles for each of plane A and B, and has to pick which of them to read when picking a pixel to display. They took advantage of this in mode 4 and the mux for plane A will also do the bit shuffling to convert from planar to chunky.
- Sprite rendering seems to have been shoehorned into the Mega Drive (i.e. sprites are rendered into the linebuffer).
- The upper byte of the command (`CD4:2` and `A16:A4`) are permanently cleared while in mode 4 (the latch's reset line is held active). This should be observable if switching from mode 5 to mode 4 to mode 5.

## Mode 5 in SMS mode

Surprisingly mode 5 seems to be usable in SMS mode as the VDP explicitly makes some accommodations for it. What is confirmed so far:

- VDP commands (except register writes) have been extended to 3 bytes. This allows the Z80 to address all of video memory.
- When writing data, it goes directly to the FIFO (no intermediate latch), and each entry keeps track of whether each byte holds valid data. The FIFO index has a "bit -1" to keeps track of partial writes (i.e. only one byte written).
- It's possible to read all 10 bits of the status port by reading one of the mirrors of the VDP control port. If address bit 1 is clear (e.g. port `$BD` instead of `$BF`) data bits 1-0 will be replaced with bits 9-8.
    + This _only_ happens in mode 5, otherwise the mirrors behave as usual.

Data writes and reads have to be examined yet, but the FIFO actively keeps track of whether one or both bytes have been written and behaves differently in SMS mode, so there's an adjustment here as well.

It isn't known yet whether DMA transfer works in SMS mode, but if it does, source address will be measured in bytes instead of words (unlike MD mode), since the source address is used as-is on the address bus.
