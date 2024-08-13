# VDP test registers

Ports `$C00018` and `$C0001C` seem to be used to access test features for the VDP (likely used during manufacturing to look for defects). There are multiple registers: you select a register by writing to `$C00018` (register number in bits 11:8, argument in bits 7:0), and then reading or writing data at `$C0001C`.

(yes, I know I'm missing here some stuff that is already known from empirical testing, will eventually get around that some day)

## Port $C00018

Writing to this port selects what test register to use in port $C0001C.

- Bits 15:12 seem to be unused.
- Bits 11:8 select the test register. It's one of 0 to 8, or $F. Other values will not enable any test features.
- Bits 7:0 seem to select arguments for the current test register and their meaning changes depending on what's selected.

## Test register $0

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
* Bits 8:7 override layer mux enable:
    - `00` = no override (normal)
    - `01` = force enable sprite plane
    - `10` = force enable plane A
    - `11` = force enable plane B
    - Note: must be `00` for background plane to display.
* Bits 11:9 override PSG output. When bit 9 is set, bits 11:10 select a PSG channel to output and the rest are muted.
* Bit 15 seems to be unused.

The way VDP implements multiplexers is that each input is gated and then merged. The layer mux select is a _huge_ string of this, and when bits 8:7 are used to forcefully enable a plane that shouldn't be displayed, it'll interfere with the plane that's supposed to show up (this seems to be a bus fight, in many later VDPs it tends to look like an AND but it may also show up as noise instead).

## Test register $1

Writing to `$C0001C`:

* Bit 0 has two effects:
    - It selects Z80/PSG clock. When `0` it's the usual 3.58MHz, while when `1` it passes the 68000 clock to it instead.
    - It also seems to force the use of EDCLK as if `RS0` had been set (on a Mega Drive, this would affect H32 mode's normal functionality).
* Bit 1 controls EDCLK pin direction:
    - `0`: EDCLK is input to VDP (used as DCLK when `RS0` is set).
    - `1`: EDCLK is output from VDP (it outputs the internally generated DCLK).
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

## Test register $F

Writing to `$C0001C` will reset the VDP (as if its /RESET pin had been asserted), regardless of what you write. This register is write-only.
