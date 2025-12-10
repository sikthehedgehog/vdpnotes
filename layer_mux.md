# Layer multiplexer

The layer multiplexer is what picks which plane to show at a given pixel based on layer priority and transparent pixels. It can be thought of being split into three major sections:

* Transparency check
* Priority encoder
* Color bus multiplexer

On top of this, the logic that decides whether a pixel is shadowed or highlighted in S/H is computed in parallel using the same information passed to the layer multiplexer.

## Transparency check

You'd think that this would be a simple "color index == 0" check, but each layer has its own particularities on top of that:

- Plane A works as expected in mode 5, but in mode 4 it can only turn transparent when the sprite is opaque (color 0 is otherwise opaque!).
- Plane B is forced transparent in mode 4.
- Sprite plane is forced transparent for the two special S/H colors (palette 3 color 14/15) when S/H is enabled. The shadow and highlight logic itself is processed in parallel.

Plane A's quirk is because color 0 in mode 4 will come from whichever palette the tile uses, *not* the background color (in fact, mode 4 has no notion of a background plane at all, only a border color). This is similar to the behavior of the Game Boy Color.

This is surprisingly underdocumented, it seems most documentation simply does it by *omission* (since color 0 has no special behavior beyond its interaction with the sprite layer). On top of that, nearly every game uses the same shade for color 0 in both palettes, and the hardware limitations discourage the use of the second palette for the background in the first place.

If you want an example of this behavior, look at Battle OutRun: it uses palette 0 color 0 for the ground beyond the road and palette 1 color 0 as the background for the HUD.

## Priority encoder

Picking the topmost layer at any given pixel is pretty much a priority encoder, however actual priority encoders are slow (worst case is when the bottommost option is picked, as the chain has to check all the other options first). It also doesn't work with the way VDP does multiplexers.

Instead what happens is that the VDP checks for every possible condition in which a layer end up on top, for each of the three layers. This involves looking both at the priority bits as well as whether each layer is transparent (as determined above). These are a lot of conditions to check but they can be done in parallel, and some of the conditions are shared across all three layers.

Background plane is implied by none of the other three layers being picked.

## Color bus multiplexer

We're not done yet. The output of the priority encoder then is modified as follows:

- Background plane is enabled when all the other layers are transparent.
- If we're outside the viewport area (i.e. this pixel ends up in the border area), all three planes are disabled and background plane is also enabled.
- If test register 0 bit 6 is set, it has the same effect as being outside the viewport area.
- *After* all the above, test register 0 bits 8:7 can be used to override the state of each plane:
	+ 01: sprite plane is forced on
	+ 10: plane A is forced on
	+ 11: plane B is forced on
	+ Any non-00 value forces background off  
	
All this is then stored in latches and used one dot later in a multiplexer which picks which plane is the source of the color to send to the color bus.

*Note* that the way multiplexers work on the VDP is that there's a separate enable for each input and only one should be set to select an output. Test register 0 bits 8:6 can break this assumption and result in two layers being enabled at the same time: this is what leads to the somewhat unstable AND behavior exploited by Overdrive 2 (which is internally a bus fight).
