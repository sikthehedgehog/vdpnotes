# Mega Drive VDP registers

## General notes

- *None* of the registers have default values! Any semblance to a default value is either some funny physical effect in play, or the result of a setting being ignored under common circumstances (e.g. window table address doesn't matter if window is hidden).
- Registers $0B to $17 can only be written while mode 5 is enabled.
- Registers $18 to $1F don't exist at all and the VDP will ignore writes to them.
- Registers $20 to $3F don't exist, they're mirrors of registers $00 to $1F.

## Register $00

- Bit 5 ("LCB"): left column blank. Hides the 8px column at the left of the viewport (extends left border by 8px). Works in both mode 4 and 5!
- Bit 4 ("IE1"): when 1, enable the hblank interrupt.
- Bit 2 ("M4"): should be 1. It's AND'd with bits 2:1 of each RGB component. Likely used to mask video output when trying to play a SG-1000 game (as modes 0-3 are not supported).
- Bit 0 ("ES"): select sync source (0 = internal, 1 = external). When external, /CSYNC pin is an input and used as the sync source.
	+ Enabling external sync seems to reduce the available sprite bandwidth.
	+ /HSYNC can also be set as an input (see register $0C) and even though it's optional it does seem to have an effect. Improved timing, perhaps?

### In mode 4:

- Bit 7 ("VSI"): vscroll inhibit. When 1, the vscroll value for plane A columns 24-31 (X coordinate 192-255) is forced to 0.
- Bit 6 ("HSI"): hscroll inhibit. When 1, the hscroll value for plane A rows 0-1 (Y coordinate 0-15) is forced to 0.
- Bit 3 ("SS"): sprite shift. When 1, it shifts all the sprites to the left by 8px.
- Bit 1 ("M3"): unused in mode 4.

### In mode 5:

- Bit 7: a broken version of vscroll inhibit. It still applies to plane A in columns 12-15 (same X coordinate range after accounting for the column width difference).
- Bit 6: unused in mode 5.
- Bit 3: when 0, /HSYNC works normally. When 1, it makes /HSYNC toggle every time /CSYNC pulses. Purpose unknown, it doesn't make much sense?
- Bit 1 ("M3"): when 0, HV counter port is freerunning (constantly updated). When 1, HV counter port is frozen and only updated when there's a falling edge on the /HL pin (lightgun pulse).

## Register $01

- Bit 6 ("DISP"): when 0, display rendering is enabled. When 1, it forces passive scan.
- Bit 5 ("IE0"): when 1, enable vblank interrupt.
- Bit 2 ("M5"): select video mode (0 = mode 4, 1 = mode 5).

### In mode 4:

- Bit 7: unused in mode 4.
- Bit 4: unused in mode 4.
- Bit 3: unused in mode 4.
- Bit 1 ("SZ"): select sprite size (0 = 8×8, 1 = 8×16)
- Bit 0 ("MAG"): this mode 4 function is not implemented in the Mega Drive VDP.

### In mode 5:

- Bit 7: select VRAM memory setup.
	+ 0: VRAM is 64KB large with an 8-bit data bus. DMA transfer speed is about half to that of CRAM and VSRAM.
	+ 1: VRAM is 128KB large with a 16-bit data bus. DMA transfer speed is identical to that of CRAM and VSRAM.
- Bit 4 ("M1"): when 1, DMA commands are allowed (CD5 is not ignored).
- Bit 3 ("M2"): select vertical resolution (0 = V28, 1 = V30). Note that V30 mode is only implemented on 50Hz systems (it'll never generate vsync in 60Hz and V counter will wrap around only on overflow).
- Bit 1: unused in mode 5.
- Bit 0: ??? (no idea what it does, but it's doing *something* in mode 5 when external sync is enabled and SPA/B is an input)

## Register $02

Base address for the plane A tilemap table.

### In mode 4:

- Bits 3:1: plane A table address bits 13:11.  

Games are supposed to set the other bits to 1, but Mega Drive VDP ignores them.

### In mode 5:

- Bits 6:3 ("SA16:13"): plane A table address bits 16:13.

## Register $03

### In mode 4:

Unused in mode 4, but games are supposed to set it to $FF.

### In mode 5:

Base address for the window tilemap table.

- Bits 6:0 ("WD16:11"), in H32: window table address bits 16:11.
- Bits 6:1 ("WD16:12"), in H40: window table address bits 16:12.  

Window tilemap is 32 tiles wide in H32 and 64 tiles wide in H40.

## Register $04

### In mode 4:

Unused in mode 4, but games are supposed to set it to $FF.

### In mode 5:

Base address for the plane B tilemap table.

- Bits 3:0 ("SB16:13"): plane B table address bits 16:13.

## Register $05

Base address for the sprite table.

### In mode 4:

- Bits 6:1 ("A13:8"): sprite table address bits 13:8.

### In mode 5:

- Bits 7:0, in H32 ("AT15:9"): sprite table address bits 16:9.
- Bits 7:1, in H40 ("AT15:10"): sprite table address bits 16:10.

## Register $06

Selects which bank of tiles to use for sprites.

### In mode 4:

- Bit 2: sprite tile address bit 13.

### In mode 5:

- Bit 5: sprite tile address bit 16.

This only applies when using 8×8 tiles. When in interlaced mode 2 (8×16 tiles), the upper bit of the tile number is used instead (all 2048 tiles can be addressed directly).

## Register $07

Selects the background plane color.

Note: the /YS pin only goes low if the color index is 0. This means that if you pick any other value in bits 3:0 of this register that it will never go low when the background plane is shown (this is relevant for the 32X).

### In mode 4:

- Bits 3:0 ("C3:0"): background color index.

Background color is always taken from palette 1.

### In mode 5:

- Bit 7: when test register 0 bit 7 = 1, this is output as the highlight flag.
- Bit 6: when test register 0 bit 6 = 1, this is output as the shadow flag.
- Bits 5:4 ("CPT1:0"): background color palette.
- Bits 3:0 ("COL3:0"): background color index.

## Register $08

Only used in mode 4, provides the hscroll value for plane A ("HS7:0"). This value is latched at the beginning of the scanline.

## Register $09

Only used in mode 4, provides the vscroll value for plane A ("VS7:0"). This value is latched at the top of the viewport.

Trying to set vscroll within the 224-255 range is the same as setting it within the 0-31 range (because the wraparound will kick in).

## Register $0B

- Bit 7: when 1, output the color bus over the "DRAM" bus.
	+ This should be set to 1 when using external CRAM!
- Bit 6: when 1, allows VDP to take over DRAM.
	+ Do *not* set this to 1 on a Mega Drive or it will crash immediately!
	+ System C/C2 uses this to let the CPU access external CRAM.
- Bits 5:4: determines how VRAM is used in a 128KB setup.
	+ X0: the full 128KB are available.
	+ 01: the first 64KB are available (VRAM address bit 16 is forced to 0).
	+ 11: the last 64KB are available (VRAM address bit 16 is forced to 1).
	+ Note that all this does is change VRAM address bit 16, all the other side effects apply as usual.
	+ The internal logical VRAM address is *not* affected, only the encoded one that goes to the `AD7:0` pins. This matters for the sprite cache checks.
- Bit 3 ("IE2"): when 1, enable external interrupt (falling edge on /HL pin).
- Bit 2 ("VSCR"): vscroll mode in mode 5.
	+ 0: full vscroll. Column 0's value is used for the entire plane. VSRAM is latched once at the beginning of the scanline.
	+ 1: per-column vscroll. Each column is mapped to its corresponding VSRAM address. VSRAM is read across the scanline as needed.
- Bits 1:0 ("HSCR", "LSCR"): hscroll mode in mode 5. Bit 1 is AND'd against scanline bits 7-3, bit 0 is AND'd against scanline bits 2-0.
	+ Unlike vscroll, this does _not_ change the timing of hscroll fetches (hscroll table is always read at the beginning of each scanline).

## Register $0C

- Bit 7 ("RS0"): select dot clock source (0 = internal DCLK, 1 = EDCLK).
- Bit 6: select what /VSYNC outputs.
	+ 0: /VSYNC goes low on vsync.
	+ 1: /VSYNC goes low when outputting the left half of the current dot (note: covers a bit less than half actually, but it's good enough to keep track of what the VDP is doing).
- Bit 5: /HSYNC pin direction (0 = output, 1 = input).
- Bit 4: SPA/B pin direction
	+ 0: SPA/B is an input. When the pin is pulled low, it forces /YS to go low as well (regardless of which color is being output).
	+ 1: SPA/B is an output. The pin is high when outputting a sprite pixel, and low otherwise. Can be used to complement the color bus, or possibly for layer priority use.
	+ The SPA/B pin is tied high on the Mega Drive.
- Bit 3 ("S/TE"): enable S/H.
- Bits 2:1 ("LSM1:0"): select progressive or interlaced mode.
	+ X0 = progressive (bit 2 is ignored).
	+ 01 = interlaced mode 1 (standard resolution, 8×8 tiles).
	+ 11 = interlaced mode 2 (double resolution, 8×16 tiles).
- Bit 0 ("RS1"): horizontal resolution (0 = H32, 1 = H40).

On a Mega Drive, RS1 and RS0 must have the same value, otherwise it will not output the correct sync timings for a TV.

## Register $0D

Base address for the hscroll table (only used in mode 5).

- Bits 6:0 ("HS16:10"): hscroll table address bits 16:10.

## Register $0E

Note: only applies when *not* in interlaced mode 2.

- Bit 4: address bit 16 for plane B tiles
- Bit 0: address bit 16 for plane A tiles

Effectively acts as an extra bit for tile IDs in mode 5 when using 8×8 tiles. It's not used in interlaced mode 2 (8×16 tiles), since the top bit of the tile number is used instead (all 2048 tiles can be addressed directly).

[Nemesis insists that the plane B's bit is forced to 0 if plane A's bit is also 0](http://gendev.spritesmind.net/forum/viewtopic.php?p=18732#p18732), but there doesn't seem to be anything like that in the logic? (either he made a mistake while testing or there was some flaw in the VDP)

## Register $0F

Determines how much to increment the current address after every CPU access (whether read, write or DMA). This is true regardless of whether it's a byte or a word access.

- In mode 4: this register is ignored, the current address is always incremented by 1.
- In mode 5 ("INC7:0"): this register is an 8-bit unsigned number that's added to the current address.

## Register $10

Select plane A and B tilemap size in mode 5:
- Bits 5:4 ("VSZ1:0"): tilemap height
- Bits 1:0 ("HSZ1:0"): tilemap width

Valid values are:
- 00 = 32 tiles wide/tall
- 01 = 64 tiles wide/tall
- 10 = invalid
- 11 = 128 tiles wide/tall  

Further restrictions are that the tilemap must not exceed 8KB (excess offset bits will be discarded, i.e. it will be truncated vertically).  

The behavior of 10 depends on the axis:
- The height bits are AND'd against the top bits of the Y coordinate, so it gives funny wraparound behavior.
- The width bits are fed into a multiplexer used for bit shifting, and 01 has nothing assigned to it: a width of 01 will result in a 32×1 large tilemap.

## Register $11

Determines the position of the horizontal split between plane A and window.

- Bit 7 ("RIGT"): on which side of the split is window plane visible? (0 = left, 1 = right)
- Bits 4:0 ("WHP5:1"): position of the split (in 16px steps).

## Register $12

Determines the position of the vertical split between plane A and window.

- Bit 7 ("DOWN"): on which side of the split is window plane visible? (0 = above, 1 = below)
- Bits 4:0 ("WVP4:0"): position of the split (in 8px steps).

## Register $13

Bits 7:0 of the DMA length counter ("LG7:0").

Note: the register itself is used as counter, so its value will be modified after a DMA operation (at the end of the operation it will be 0).

## Register $14

Bits 15:8 of the DMA length counter ("LG15:8").

Note: the register itself is used as counter, so its value will be modified after a DMA operation (at the end of the operation it will be 0).

## Register $15

Bits 8:1 of the DMA source address.

Note: the register itself is used as the source address, so its value will be modified after a DMA operation (at the end of the operation it will point right after the source data).

## Register $16

Bits 16:9 of the DMA source address.

Note: the register itself is used as the source address, so its value will be modified after a DMA operation (at the end of the operation it will point right after the source data).

## Register $17

- Bit 7 ("DMD1"): 0 for a DMA transfer (source is ROM/RAM), 1 for a DMA copy or fill.
- Bit 6 ("DMD0"): depends on the type of DMA.
	+ If DMA transfer: source address bit 23.
	+ Otherwise: 0 = DMA fill, 1 = DMA copy.
- Bits 5:0: bits 22:17 of DMA source address.

Note: the register itself is used as the source address, *but this register is not incremented* (DMA source wraps around at 128KB boundaries).  
