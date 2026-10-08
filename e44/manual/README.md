# Adapting KiCad Designs for the LPKF ProtoMat E44

A guide to taking a KiCad PCB design and making it something the Innovation
Lab's ProtoMat E44 can mill reliably.

> **Status: draft.** Sections marked **TODO** still need writing or checking
> against how the lab runs the machine.

- [1. Why designs need adapting](#1-why-designs-need-adapting)
- [2. The rules at a glance](#2-the-rules-at-a-glance)
- [3. Setting up your KiCad project](#3-setting-up-your-kicad-project)
- [4. Adapting footprints](#4-adapting-footprints)
- [5. Routing](#5-routing)
- [6. Board outline, slots and cutouts](#6-board-outline-slots-and-cutouts)
- [7. Running DRC and fixing errors](#7-running-drc-and-fixing-errors)
- [8. Exporting for CircuitPro](#8-exporting-for-circuitpro)
- [9. Pre-flight checklist](#9-pre-flight-checklist)

Related files:

- [`../e44.kicad_dru`](../e44.kicad_dru): KiCad custom rules
- [`../E44_PCB_Design_Rules_Poster.pdf`](../E44_PCB_Design_Rules_Poster.pdf): one-page summary poster

---

## 1. Why designs need adapting

A commercial board house etches copper chemically and can hold tracks and gaps
of 0.15 mm or less. The E44 cuts copper away mechanically: a V-shaped cutter
mills an isolation channel around every track and pad. This means:

- **Gaps are set by the cutter.** Copper closer together than the isolation
  channel can't be separated.
- **Holes come from a fixed set of drills.** A hole size that isn't in the
  tool rack can't be made.
- **Fine features are fragile.** Thin tracks and small annular rings can tear
  off during milling or when soldering.

KiCad's default libraries and settings assume a board house, so most designs
need some changes before they will mill well. The E44 is rated to 0.1 mm tracks
and 0.15 mm gaps, but these rules are deliberately conservative so boards work
first time.

## 2. The rules at a glance

All dimensions in mm.

| Rule | Value | Notes |
|---|---|---|
| Minimum track width | 0.5 | Set as the net class default |
| Minimum clearance | 0.35 | Copper to copper, including pads within a footprint. 0.3 mm isolation channel + 0.05 mm margin |
| Copper to board edge | 1.0 | From the Edge.Cuts outline to any copper |
| Minimum annular ring | 0.6 | 0.5 allowed on 1.0 mm holes |
| Drill sizes | 0.7, 0.8, 1.0, 1.5, 2.0 | Only these. Any other round hole fails DRC |
| Via | 0.7 hole / 1.9 pad | Set as the net class default |
| 2.54 mm headers | 1.0 hole / 2.0 pad | Leaves a 0.54 mm gap between pads |
| Slot width | 1.0 minimum | Cut with the 1.0 mm router, so inside corners are R0.5 |
| Finest SMD pitch | 0.95 | SOT-23-5 and SOIC are fine. No TSSOP or QFN |

### Tool set

| Tool | Size | Used for |
|---|---|---|
| Universal Cutter | 0.2–0.5 | Isolation milling. Isolation width set to 0.3 in CircuitPro |
| End mill | **TODO** confirm: poster says 1.0, `.kicad_dru` comment says 2.0 | Copper rubout (clearing large areas) |
| Contour router | 1.0 | Board outline, slots, cutouts |
| Spiral drills | 0.7, 0.8, 1.0, 1.5, 2.0 | All round holes and vias |

## 3. Setting up your KiCad project

Do this at the start of a new design, or before adapting an existing one.

### 3.1 Add the custom rules

1. Open the PCB in the PCB Editor.
2. **File → Board Setup → Design Rules → Custom Rules**.
3. Paste the contents of [`e44.kicad_dru`](../e44.kicad_dru).
4. Click **Check rule syntax**, then **OK**.

KiCad stores these rules in `<project name>.kicad_dru` next to your
`.kicad_pro` file. Copying `e44.kicad_dru` into your project folder
and renaming it to match the project has the same effect.

The custom rules catch problems at DRC time. The steps below set KiCad's
defaults so you don't create those problems in the first place.

### 3.2 Constraints

**Board Setup → Design Rules → Constraints**:

| Setting | Value |
|---|---|
| Minimum clearance | 0.35 |
| Minimum track width | 0.5 |
| Minimum annular width | 0.5 (the custom rules enforce 0.6 except on 1.0 mm holes) |
| Minimum via diameter | 1.9 |
| Minimum through hole | 0.7 |
| Copper to edge clearance | 1.0 |

### 3.3 Net classes

**Board Setup → Design Rules → Net Classes**, edit the `Default` class:

| Setting | Value |
|---|---|
| Clearance | 0.35 |
| Track width | 0.5 |
| Via size | 1.9 |
| Via hole | 0.7 |

If you have power nets, add a `Power` net class with wider tracks (for
example 1.0) and assign it in the schematic or with net class patterns.

### 3.4 Pre-defined sizes

**Board Setup → Design Rules → Pre-defined Sizes**. Adding these makes them
available in the track width and via size drop-downs while routing:

- Tracks: 0.5, 0.8, 1.0, 1.5
- Vias: 1.9 / 0.7

## 4. Adapting footprints

Footprints are where most of the work is. Stock KiCad footprints are designed
for plated, board-house holes and tight pad spacing.

### 4.1 Where to make the changes

Don't edit footprints one by one on the board: the next **Update PCB from
Schematic** can overwrite them. Instead:

1. Create a project footprint library (for example `E44.pretty`) via
   **Preferences → Manage Footprint Libraries → Project Specific Libraries**.
2. Copy the stock footprint into it and edit the copy.
3. Point the schematic symbol at the new footprint.

This also builds up a reusable set of E44-ready footprints. **TODO:** decide
whether the lab keeps a shared E44 footprint library in this repository.

### 4.2 Hole sizes

Every round hole must be 0.7, 0.8, 1.0, 1.5 or 2.0 mm.

To choose a size, take the component lead diameter from the datasheet, add
about 0.2 mm, and round **up** to the next drill in the set.

| Typical use | Drill |
|---|---|
| Small resistor / diode / capacitor leads, IC sockets, DIP ICs | 0.8 |
| 2.54 mm pin headers, larger axial parts | 1.0 |
| Larger leads (TO-220, terminal blocks, DC jacks) | 1.5 |
| Mounting pegs, M2 screws | 2.0 |

**TODO:** check typical parts against the lab's stock and fill in examples.

Holes larger than 2.0 mm (for example 3.2 mm for M3 screws) aren't in the drill
set. **TODO:** confirm with the lab whether these should be drawn on Edge.Cuts
so the 1.0 mm contour router cuts them out.

### 4.3 Pad sizes and annular rings

The annular ring is the copper left around a hole:
`(pad diameter − hole diameter) / 2`. It must be at least 0.6 mm, or 0.5 mm
on a 1.0 mm hole.

| Hole | Minimum pad |
|---|---|
| 0.7 | 1.9 |
| 0.8 | 2.0 |
| 1.0 | 2.0 |
| 1.5 | 2.7 |
| 2.0 | 3.2 |

Bigger pads use up spacing, so check the gap to the next pad stays at least
0.35 mm. On a 2.54 mm pitch, 2.0 mm pads leave 0.54 mm, which passes.

Common stock footprints that need enlarging (check the values in your KiCad
version):

- **`PinHeader_*_P2.54mm`**: 1.0 hole with a 1.7 pad. Enlarge pads to 2.0.
- **`DIP-*_W7.62mm`**: 0.8 hole with a 1.6 pad. Enlarge pads to 2.0.

Tip: in the footprint editor, oval pads (for example 2.0 × 2.4) give more
copper to solder to without closing the gap between neighbouring pins.

### 4.4 Slots and oval holes

Oval holes are cut with the 1.0 mm contour router, so they must be at least
1.0 mm wide. Footprints with narrower slots (some USB connectors and DC jacks)
need the slot widened or replaced with round holes.

### 4.5 Surface-mount parts

The finest pitch that works is 0.95 mm (SOT-23-5). SOIC (1.27 mm) and larger
are fine. TSSOP, MSOP, QFN and similar fine-pitch packages won't mill:
choose a SOIC or through-hole version of the part, or use a breakout board.

The 0.35 mm clearance also applies between pads *inside* a footprint. If DRC
flags pads within a stock footprint, narrow the pads slightly in a project
copy rather than ignoring the error.

## 5. Routing

- **Use 0.5 mm tracks as a minimum** and go wider whenever there is room. Wider
  tracks are stronger and survive soldering and rework better.
- **Pour ground where you can.** Every area of bare board must be milled away,
  which takes time and wears tools. A ground fill on spare area means less
  milling. Set the zone clearance to at least 0.35 mm, and the zone minimum
  width to at least 0.5 mm so thin slivers of copper aren't left behind.
- **Avoid routing between pads** of 2.54 mm parts. With 2.0 mm pads there's
  only 0.54 mm between them, which is not enough for a 0.5 mm track plus two
  0.35 mm clearances.
- **Keep vias to a minimum.** **TODO:** document how the lab makes vias and
  through-connections on double-sided boards (the E44 does not plate holes,
  so connections between layers need wires, pins or rivets).
- **Prefer single-sided** when the design allows it. **TODO:** document the
  double-sided workflow (board flipping and alignment).

## 6. Board outline, slots and cutouts

- Draw the outline on **Edge.Cuts** as one closed shape.
- The outline is cut with a 1.0 mm router, so **inside corners** can't be
  sharper than R0.5. Add a 0.5 mm (or larger) fillet to internal corners, or
  allow for the rounding when designing parts that fit into them.
- Internal cutouts must be at least 1.0 mm wide.
- Keep all copper at least 1.0 mm from Edge.Cuts. Check that connectors
  meant to sit at the board edge still meet this.

## 7. Running DRC and fixing errors

Run **Inspect → Design Rules Checker** with *Test for parity between PCB and
schematic* enabled. Fix every error before bringing the board to the lab.

| DRC message | Cause | Fix |
|---|---|---|
| `E44 clearance 0.35mm` | Copper too close together | Move tracks apart, or shrink pads in a project copy of the footprint |
| `E44 min track 0.5mm` | Track narrower than 0.5 mm | Select the track, press **E**, set width to 0.5 or more |
| `E44 copper to edge 1.0mm` | Copper within 1.0 mm of Edge.Cuts | Move the part or track inwards, or enlarge the board |
| `E44 annular ring 0.6mm` | Pad too small for its hole | Enlarge the pad (see [4.3](#43-pad-sizes-and-annular-rings)) |
| `E44 annular ring 0.5mm on 1.0mm holes` | Header pad smaller than 2.0 mm | Enlarge the pad to 2.0 mm |
| `E44 slot narrower than 1.0mm router` | Oval hole under 1.0 mm wide | Widen the slot (see [4.4](#44-slots-and-oval-holes)) |
| `E44 pad drill not in tool set` | Pad hole isn't 0.7/0.8/1.0/1.5/2.0 | Change the hole to a size in the set (see [4.2](#42-hole-sizes)) |
| `E44 via drill not in tool set` | Via hole isn't in the set | Change the via to 1.9 / 0.7 |

Tip: to change many identical footprints at once, use **Edit → Change
Footprints** to swap them all for your E44 version.

## 8. Exporting for CircuitPro

**TODO:** Gerber and drill export settings for LPKF CircuitPro (layers,
Excellon format and units, drill map), and how to bring the files to the lab.

## 9. Pre-flight checklist

- [ ] Custom rules pasted into Board Setup
- [ ] Net class defaults set (0.5 track, 0.35 clearance, 1.9 / 0.7 via)
- [ ] Every hole is 0.7, 0.8, 1.0, 1.5 or 2.0 mm
- [ ] Annular rings ≥ 0.6 mm (≥ 0.5 mm on 1.0 mm holes)
- [ ] No SMD parts finer than 0.95 mm pitch
- [ ] Board outline closed, inside corners R0.5 or larger
- [ ] All copper ≥ 1.0 mm from the board edge
- [ ] Ground pour on spare area
- [ ] DRC passes with no errors
- [ ] Gerbers and drill files exported
