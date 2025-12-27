# VDP test registers

Ports `$C00018` and `$C0001C` seem to be used to access test features for the VDP (likely used during manufacturing to look for defects). There are multiple registers: you select a register by writing to `$C00018` (register number in bits 11:8, argument in bits 7:0), and then reading or writing data at `$C0001C`.

(yes, I know I'm missing here some stuff that is already known from empirical testing, will eventually get around that some day)

## Port $C00018

Writing to this port selects what test register to use in port $C0001C.

- Bits 15:12 seem to be unused.
- Bits 11:8 select the test register. It's one of 0 to 8, or $F. Other values will not enable any test features.
- Bits 7:0 seem to select arguments for the current test register and their meaning changes depending on what's selected.

## Test register $0

This register is write-only.

Writing to `$C0001C`:

* Bit 0 only is useful when VDP is configured to output the color bus: it replaces color bus bit 6 with the layer priority.
* Bit 1 changes how DMA source and length increment works.
    - `0` = increment by 1.
    - `1` = increment by 256.
* Bit 2 apparently only takes effect in SMS mode (!!): when set, it exposes the missing HV counter bits in the upper byte of the data bus (which is normally unreachable to the Z80).
    - Bit 8 is X coordinate bit 0, inverted
    - Bits 10:9 are Y coordinate bits 9:8, inverted
* Bit 3 forces the line counter for the hblank interrupt to be loaded with the value of VDP register $0A *now*.
* Bit 5 changes whether the VRAM address is multiplexed (row/column) or output as-is.
    - `0` = alternate between row and columns on `AD[7:0]` (VRAM address bus)
    - `1` = put the whole address on `{RD[7:0],AD[7:0]}` (VRAM data and address bus)
* Bit 6 hides all layers (as if they were transparent). This happens before bits 8:7, which can override its effect (see below).
* Bits 8:7 override layer mux enable:
    - `00` = no override (normal)
    - `01` = force enable sprite plane
    - `10` = force enable plane A
    - `11` = force enable plane B
    - Note: must be `00` for background plane to display.
* Bits 11:9 override PSG output. When bit 9 is set, bits 11:10 select a PSG channel to output and the rest are muted.
* Bit 12 is not fully understood, but it seems to let you insert sprites into the list of sprites to render? The format of the data is as follows (to-do: check *when* the writes happen, order of the words may be wrong):
	- 1st word: tile ID in bits 10:0
	- 2nd word: X position in bits 8:0
	- 3rd word:
		+ Bit 0 = horizontal flip
		+ Bits 2:1 = palette number
		+ Bit 3 = priority flag
		+ Bits 5:4 = sprite width
		+ Bits 7:6 = sprite height
		+ Bits 13:8 = Y offset within sprite
* Bit 13 seems to give direct access to the sprite linebuffer when set (sprite rendering is disabled). Meant to be used in conjunction with test register $8.
* Bit 15 seems to be unused.

The way VDP implements multiplexers is that each input is gated and then merged. The layer mux select is a _huge_ string of this, and when bits 8:7 are used to forcefully enable a plane that shouldn't be displayed, it'll interfere with the plane that's supposed to show up (this seems to be a bus fight, in many later VDPs it tends to look like an AND but it may also show up as noise instead).

## Test register $1

This register is write-only.

Writing to `$C0001C`:

* Bit 0 has two effects:
    - It selects Z80/PSG clock. When `0` it's the usual 3.58MHz, while when `1` it passes the 68000 clock to it instead.
    - It also seems to force the use of EDCLK as if `RS0` had been set (on a Mega Drive, this would affect H32 mode's normal functionality).
* Bit 1 controls EDCLK pin direction:
    - `0`: EDCLK is input to VDP (used as DCLK when `RS0` is set).
    - `1`: EDCLK is output from VDP (it outputs the internally generated DCLK in H32 mode).
    - Actually it outputs the actual DCLK in use (either internal DCLK if `RS0` = `0` or external EDCLK if `RS0` = `1`), but it's impossible to receive external EDCLK on real hardware due to changing the pin direction. This _should_ work with the Nuked VDP core however, as you can connect directly to both `EDCLK_i` and `EDCLK_o` (ignoring the "pin direction").
* Bit 2: when set, the V counter is frozen by default, then it will increment every dot (2 DCLK) where the /BG pin is high.
* Bit 3: when set, the H counter is frozen by default, then it will increment every dot (2 DCLK) where the /INTAK pin is high.
* Bits 6:4 seem to be used to force VDP to do memory fetches, though I'm not sure how it works exactly. The following combinations are known to select something:
  - `000`: nothing (normal)
  - `001`: plane A tilemap table
  - `010`: plane B tilemap table
  - `011`: ??? (checked by `w382`)
  - `100`: ??? (checked by `w383`)
  - `101`: hscroll table
* Bit 7: when set, the HL pin will force a hscroll table fetch. Also forces window registers to take effect immediately instead of being latched on V counter increment only.
    - Actually not sure on this bit, I need to recheck what it's actually forcing to be fetched.
    - It seems to force VSRAM to be constantly latched as well? (see `w513`)
* Bit 8: when set, the HL pin will force a plane tilemap fetch.
* Bit 9: when set, the HL pin will force a plane A tile fetch.
* Bit 10: when set, the HL pin will force a plane B tile fetch.
* Bits 15:11 seem to be unused.

## Test register $2

Writing to `$C0001C` changes the V counter's value to bits 8:0 from the data bus.

Reading from `$C0001C` returns a bunch of flags in bits 7:0 of the data bus (to be documented).

## Test register $3

Writing to `$C0001C` changes the H counter's value to bits 8:0 from the data bus.

Reading from `$C0001C` returns a bunch of flags in bits 13:0 of the data bus (to be documented).

## Test register $4

Reading from `$C0001C` gives the current color output (note that for whatever reason all the bits are inverted, i.e. apply a NOT to it):

- Bits 10:8 = blue component
- Bits 7:5 = green component
- Bits 4:2 = red component
- Bit 1 = shadow
- Bit 0 = highlight

## Test register $7

Reading from `$C0001C` seems to return information about the current sprite being rendered. The returned information depends on bits 6:5 of `$C00018`.

- When `$C00018[6:5] == 0`:
	+ `$C0001C[10:0]` = tile number
- When `$C00018[6:5] == 1`:
	+ `$C0001C[10:9]` = always 0?
	+ `$C0001C[8:0]` = X coordinate
- When `$C00018[6:5] == 2`:
	+ `$C0001C[13:8]` = Y offset within sprite
	+ `$C0001C[7:6]` = sprite height
	+ `$C0001C[5:4]` = sprite width
	+ `$C0001C[3]` = priority flag
	+ `$C0001C[2:1]` = palette
	+ `$C0001C[0]` = horizontal flip  

(strictly speaking bits 13:11 always return the upper bits of the Y offset, but I get the impression that you weren't supposed to rely on that :P)

## Test register $8

Seems to give direct access to the sprite linebuffer when test register $0 bit 13 = 1. Sprite rendering is "disabled" when this bit is set (i.e. it still tries to render but it can't touch the linebuffer).

Bits 5:0 of `$C00018` select a 8px boundary within the linebuffer. Bits 7:6 of `$C00018` select a pair of pixels within the current 8px boundary. In other words:

- `$C00018[5:0]` → X coordinate bits 8:3
- `$C00018[7:6]` → X coordinate bits 2:1  

(in other words: drop the bottom bit of the X coordinate, then rotate it right by two places and put it in the low byte of `$C00018`)

Reading from `$C0001C` will return the pixel data from the linebuffer, on bits 6:0 for the even pixel and bits 14:8 for the odd pixel:

- `$C0001C[1:0]` = palette of even pixel
- `$C0001C[2]` = priority of even pixel
- `$C0001C[6:3]` = color of even pixel
- `$C0001C[9:8]` = palette of odd pixel
- `$C0001C[10]` = priority of odd pxel
- `$C0001C[14:11]` = color of odd pixel  

Writing to `$C0001C` seems to write the pair of pixels into the linebuffer instead (needs tests to confirm, the mechanism is spread all over so it's harder to verify just by reading the Verilog code).

## Test register $F

Writing to `$C0001C` will reset the VDP (as if its /RESET pin had been asserted), regardless of what you write. This register is write-only.
