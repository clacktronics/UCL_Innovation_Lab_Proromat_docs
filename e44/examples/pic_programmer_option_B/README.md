# pic_programmer, adapted for the E44 (option B: footprint library)

KiCad's `pic_programmer` demo project (by Jean-Pierre Charras, distributed
with KiCad) after adapting it to the E44 rules, as worked through in the
[manual](../../manual/README.md) using option B. Open `pic_programmer.kicad_pro` in KiCad 10.

What changed from the stock demo:

- E44 custom rules in `pic_programmer.kicad_dru` (same as `../../e44.kicad_dru`).
- Board Setup constraints, net classes and pre-defined sizes set to the E44
  values. Silkscreen and solder mask checks set to Ignore.
- Every footprint replaced by an E44 version from the project library
  `E44.pretty`: holes changed to the drill set, pads enlarged, holes over
  2.0 mm turned into Edge.Cuts cutouts, and the TO-92 transistors swapped for
  `TO-92_Wide`. The schematic footprint fields point at the same footprints.
- Copper text moved to silkscreen; the duplicate mirrored bottom labels removed.
- Tracks that no longer fit rerouted with Freerouting. The board has no vias.

DRC result in KiCad 10.0: 0 violations, 0 unconnected items, schematic parity
passes.

Note that some top-layer tracks still connect to socket pins. See "Things DRC
won't tell you" in the manual before milling it.
