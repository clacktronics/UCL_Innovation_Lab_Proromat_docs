# pic_programmer, adapted for the E44 (option A: one-off copy)

KiCad's `pic_programmer` demo project (by Jean-Pierre Charras, distributed
with KiCad) adapted to the E44 rules by editing the pads on the board, as
worked through in option A of the [manual](../../manual/README.md). Open
`pic_programmer.kicad_pro` in KiCad 10.

What changed from the stock demo:

- E44 custom rules in `pic_programmer.kicad_dru` (same as `../../e44.kicad_dru`).
- Board Setup constraints, net classes and pre-defined sizes set to the E44
  values. Silkscreen, solder mask and "footprint doesn't match library" checks
  set to Ignore.
- Pads edited on the board in groups from the Search panel's Drills tab:
  holes changed to the drill set and pads enlarged, keeping each pad's shape.
  No footprint library was changed.
- The TO-92 transistors swapped for `TO-92_Wide` with Change Footprints. The
  schematic still names the stock footprint, so the parity check notes the
  difference; that's expected on a one-off copy.
- Holes over 2.0 mm replaced with Edge.Cuts circles inside each footprint.
- Copper text moved to silkscreen, tracks widened to 0.5 mm, and tracks that
  no longer fit rerouted with Freerouting. The board has no vias.

DRC result in KiCad 10.0: 0 violations, 0 unconnected items.

Note that some top-layer tracks still connect to socket pins. See "Things DRC
won't tell you" in the manual before milling it.
