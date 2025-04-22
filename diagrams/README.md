# VDP logic diagram

An attempt to document the VDP logic as a diagram showing how each wire and latch is connected. Work in progress.

## File format

The PNG files are provided for convenience.

The `.dia` files are the source files and they're meant to be opened with
[Dia](https://en.wikipedia.org/wiki/Dia_(software)), while the `.png` images
are exported from those. The export is done manually so it's possible that I
accidentally forget to update something on a commit.

## Conventions

By general rule, boxes with solid borders introduce an intentional delay (i.e. waiting for a certain clock cycle), while boxes with dotted borders don't (i.e. only propagation delay). Rounded boxes refer to signals from other parts of the VDP circuit.

The boxes only include a loose description while the original wire names are still included in them. Refer to the Verilog file for exact details.