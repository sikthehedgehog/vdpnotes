# VDP test registers

Ports `$C00018` and `$C0001C` seem to be used to access test features for the VDP (likely used during manufacturing to look for defects). There are multiple registers: you select a register by writing to `$C00018` (register number in bits 11:8, argument in bits 7:0), and then reading or writing data at `$C0001C`.

(yes, I know I'm missing here some stuff that is already known from empirical testing, will eventually get around that some day)

## Test register $0

Writing to `$C0001C`:

* Bit 5 changes whether the VRAM address is multiplexed (row/column) or output as-is.
    - `0` = alternate between row and columns on `AD[7:0]` (VRAM address bus)
    - `1` = put the whole address on `{RD[7:0],AD[7:0]}` (VRAM data and address bus)
* Bit 15 seems to be unused.

## Test register $1

Writing to `$C0001C`:

* Bits 6:4 seem to be used to force VDP to do memory fetches, though I'm not sure how it works exactly. The following combinations are known to select something:
  - `000`: nothing (normal)
  - `001`: plane A tilemap table
  - `010`: plane B tilemap table
  - `011`: ??? (checked by `w382`)
  - `100`: ??? (checked by `w383`)
  - `101`: hscroll table
* Bit 7: when set, the HL pin will force a hscroll table fetch.
* Bit 8: when set, the HL pin will force a plane tilemap fetch.
* Bit 9: when set, the HL pin will force a ??? fetch.
* Bit 10: when set, the HL pin will force a ??? fetch.
* Bits 15:11 seem to be unused.
