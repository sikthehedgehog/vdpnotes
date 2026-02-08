# VDP implementation tidbits

- The F flag is the internal "vblank interrupt pending" flag exposed directly on the status port. There are also hblank and external interrupt pending flags, but those aren't visible from outside.
- DMA length is stored _inverted_, then incremented as DMA progresses. This probably saved some gates, I guess.
- CRAM data is internally stored as `bbbgggrrr` in mode 5 and `xxxbbggrr` in mode 4. This should be observable when switching between modes.

## VRAM

- There's no centralized way to access the VRAM bus, everything that needs it puts their address directly on it and the VDP relies on its timing signals to coordinate who has access to the bus.
- 16-bit data seems to be stored little endian internally. VDP has to fake having a big endian interface in mode 5 to prevent confusion when using the 68000.
- With an 8-bit bus (mode 4 and mode 5 64KB), `/SE0` is always asserted. With a 16-bit bus (mode 5 128KB), it constantly alternates between `/SE0` and `/SE1`. This lets the VDP share the same 8-bit port to read data from both VRAM halves.
    + (`/SE0` and `/SE1` are the two VRAM selects, for context)

## Commands and register writes

- The `CD0` to `CD4` flags are stored in latches like the rest of the command, but `CD5` is strictly logic instead (setting that bit immediately kickstarts the logic that enables the DMA operation).
- Register writes are triggered when `CD1` and `CD0` become `10` after doing a write. The value is copied from the latched command (instead of the data bus).
- There's no centralized register file, instead internally there's an "enable" for each latch spread all over the VDP.
- The register number is decoded in two parts. First bits 4:3 are decoded (`00`, `01`, `01` if mode 5 only, `11` if mode 5 only) and then bits 2:0 separately (eight different lines), a combination of those two is then used to detect the individual registers. The "if mode 5 only" check is used to lock out registers `$0B` onwards (VDP will ignore those writes in mode 4).
- Registers `$0B` and above are locked out outside mode 4 (writes don't go through). When a register write _does_ go through however, the whole register is written, including the bits not relevant to the current video mode (which should be observable when switching to the other mode).
- Autoincrement is either 1 in mode 4 or register `$0F`'s value in mode 5… in theory. In practice both are always added together, so that means that if you switch to mode 5, set the register, then switch back to mode 4, the actual autoincrement will be 1 more than the register's value.

## FIFO

- Byte writes go straight to the FIFO. Each FIFO entry has two flags to keep track of which bytes are valid (upper byte and lower byte flags). Both MD and SMS mode can issue byte writes.
    + For some reason I can't get the VDP to actually treat byte writes in MD mode as byte writes, however (it still writes words). I may be still missing something here.
- The flushing mechanism accounts for both 8-bit and 16-bit VRAM buses by having an extra bit for the flush index. On 8-bit buses the whole index is incremented by +1, and on 16-bit buses it's incremented by +2.
- FIFO FULL is set when the two indices are identical after a write, and reset when the indices stop being identical.
- FIFO EMPTY is set when the two indices are identical after flushing, and reset when the indices stop being identical.
- VDP reset forces the above flags to be EMPTY = 1 and FULL = 0.

## DMA operation

- Internally, length is stored _inverted_ (NOT'd) and then incremented until all bits are set. End result is the same, it probably saved a few gates.
- Usually, during memory writes the CD flags stored in the FIFO are used. DMA copy uses the live command flags however (makes sense, since copy doesn't use the FIFO), I need to recheck if this is the case for DMA fill as well.

## Scroll layers

- There's a single PLA entry to trigger all the tile reads for a plane (I think `w507`? for plane A, `w508` for plane B): it's loaded into a shift register and the `1` travels across it to trigger each of the eight latches in sequence.
- On top of that after all eight bytes have been fetched it'll then copy everything to a separate latch (probably to make room for scrolling?). This latch is the one used to output pixels to screen.
- It's continuously latching tiles on a fixed period of time even if there isn't tile data on the VRAM bus. It doesn't matter since the garbage is going to be hidden in the border area anyway.

## Sprite system

- Sprite line buffer has a 8px wide data bus and VDP draws 8px at a time.
- Internally there's a mux which changes the order in which sprites are drawn (front to back or back to front). It's hardwired to "front to back" by tying the select line to ground (leftover from an old design?).
- The VDP checks the VRAM address on the bus to see when a write to the sprite cache happens, but expects the data to come from the FIFO. This breaks with DMA copy, as it will *not* update the cache that way because it by-passes the FIFO.

## Mode 4 implementation

- In mode 5, the VDP reads two 8×1 tiles for each of plane A and B, and has to pick which of them to read when picking a pixel to display. They took advantage of this in mode 4 and the mux for plane A will also do the bit shuffling to convert from planar to chunky.
- Plane B is "disabled" by forcing its layer transparent. Technically it's still functional, but without any data fetches it would have only shown garbage anyway.
- Sprite rendering seems to have been shoehorned into the Mega Drive (i.e. sprites are rendered into the linebuffer).
- The upper byte of the command (`CD4:2` and `A16:A4`) are permanently cleared while in mode 4 (the latch's reset line is held active). This should be observable if switching from mode 5 to mode 4 to mode 5.

## Mode 5 in SMS mode

Surprisingly mode 5 seems to be usable in SMS mode as the VDP explicitly makes some accommodations for it. What is confirmed so far:

- VDP commands (except register writes) have been extended to 3 bytes. This allows the Z80 to address all of video memory.
- When writing data, it goes directly to the FIFO (no intermediate latch), and each entry keeps track of whether each byte holds valid data (sadly, the whole entry is wasted on a single byte).
    +  Unlike in MD mode, byte writes *do* work properly in SMS mode. The bottom bit of the current address is used to determine whether it's an upper or lower byte, instead of CPU bus signals.
- It's possible to read all 10 bits of the status port by reading one of the mirrors of the VDP control port. If address bit 1 is clear (e.g. port `$BD` instead of `$BF`) data bits 1-0 will be replaced with bits 9-8.
    + This _only_ happens in mode 5, otherwise the mirrors behave as usual.

Data reads have to be examined yet, but I would be surprised if it's any different than for data writes.

It isn't known yet whether DMA transfer works in SMS mode (probably not), but if it does, source address will be measured in bytes instead of words (unlike MD mode), since the source address is used as-is on the address bus. DMA copy and fill work as expected, as they don't involve the CPU bus.

## Unused logic

There's a bunch of logic in the VDP that either goes nowhere or has been otherwise nullified.

- `w17`: not connected. Seems to have something to do with refresh and FIFO being full.

- `w262`: not connected. It's a combination of `w300` (flushing second byte of a word write to 8-bit VRAM) and `w248` (DMA fill in progress).

- `w703` and `w704`: they increment and decrement `l371`, respectively. The former is `w702` AND 1, the latter is `w702` AND 0, which means that in practice `w702` is increment and there's no way to decrement. Who knows what the logic was originally supposed to be?

- `w777` and `l427`: the former is latched by the latter, but the latter's output is not connected anywhere. (TODO: describe what `w777`'s logic is)

- `w942` to `w950`: looks like at one point it was possible to select whether sprites would overwrite or underwrite other sprites drawn earlier, where `w950` selected which of the two to use (`w942` to `w949` are the muxes for each pixel). In practice `w950` is hardwired to always select underwrite.

- The behavior of window plane differs between modes 4 and 5. In mode 5, both window registers are latched every scanline; in mode 4, the horizontal register is latched every scanline and the vertical register at the top of the screen. This matches the behavior of scroll, but… window plane is not supported in mode 4 anyway (VDP explicitly disables window fetches if not in mode 5). What gives?
	+ Note that this behavior *is* technically observable if you switch between video modes, though you'll have to fight sync alignment issues as well if you attempt that and you can only see it for one scanline anyway. It's "unused" if you stick to mode 4, however.

- The logic for plane B tilemap fetches checks for the window plane just like plane A does. This is nullified by the fact that the "in window" check logic explicitly limits itself to when plane A fetches happen.
