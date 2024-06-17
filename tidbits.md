# VDP design tidbits

- DMA length is stored _inverted_, then incremented as DMA progresses. This probably saved some gates, I guess.

## Mode 4 implementation

- In mode 5, the VDP reads two 8×1 tiles for each of plane A and B, and has to pick which of them to read when picking a pixel to display. They took advantage of this in mode 4 and the mux for plane A will also do the bit shuffling to convert from planar to chunky.
- Sprite rendering seems to have been shoehorned into the Mega Drive (i.e. sprites are rendered into the linebuffer).

## Mode 5 in SMS mode

Surprisingly mode 5 seems to be usable in SMS mode as the VDP explicitly makes some accommodations for it. What is confirmed so far:

- VDP commands (except register writes) have been extended to 3 bytes. This allows the Z80 to address all of video memory.
- It's possible to read all 10 bits of the status port by reading one of the mirrors of the VDP control port. If address bit 1 is clear (e.g. port `$BD` instead of `$BF`) data bits 1-0 will be replaced with bits 9-8.
    + This _only_ happens in mode 5, otherwise the mirrors behave as usual.

Data writes and reads have to be examined yet, but the FIFO actively keeps track of whether one or both bytes have been written and behaves differently in SMS mode, so there's an adjustment here as well.

It isn't known yet whether DMA transfer works in SMS mode, but if it does, source address will be measured in bytes instead of words (unlike MD mode).
