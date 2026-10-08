# Adapting KiCad Designs for the LPKF ProtoMat E44

How to take a KiCad PCB design and make it something the Innovation Lab's
ProtoMat E44 can mill reliably. The guide works through a real example, the
`pic_programmer` demo that ships with KiCad, from the stock design (about 270
DRC errors against the E44 rules) to a board that passes DRC.

Screenshots are from **KiCad 10.0**. Menu names may differ slightly in older
versions.

> **Status: draft.** Sections marked **TODO** still need checking against how
> the lab runs the machine.

- [1. Why designs need adapting](#1-why-designs-need-adapting)
- [2. The rules at a glance](#2-the-rules-at-a-glance)
- [3. The example: pic_programmer](#3-the-example-pic_programmer)
- [4. Step 1: Add the E44 rules](#4-step-1-add-the-e44-rules)
- [5. Step 2: Set the board defaults](#5-step-2-set-the-board-defaults)
- [6. Step 3: Run DRC to see what needs changing](#6-step-3-run-drc-to-see-what-needs-changing)
- [7. Step 4: Make E44 versions of your footprints](#7-step-4-make-e44-versions-of-your-footprints)
- [8. Step 5: Fix tracks, vias, text and pours](#8-step-5-fix-tracks-vias-text-and-pours)
- [9. Step 6: Reroute what no longer fits](#9-step-6-reroute-what-no-longer-fits)
- [10. Step 7: Final DRC](#10-step-7-final-drc)
- [11. Things DRC won't tell you](#11-things-drc-wont-tell-you)
- [12. DRC message reference](#12-drc-message-reference)
- [13. Exporting for CircuitPro](#13-exporting-for-circuitpro)
- [14. Pre-flight checklist](#14-pre-flight-checklist)

Related files:

- [`../e44.kicad_dru`](../e44.kicad_dru): KiCad custom rules
- [`../E44_PCB_Design_Rules_Poster.pdf`](../E44_PCB_Design_Rules_Poster.pdf): one-page summary poster
- [`../examples/pic_programmer/`](../examples/pic_programmer/): the adapted example project

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
- **Holes aren't plated.** A board house plates every hole so the top and
  bottom copper are joined. On the E44 they aren't (see
  [section 11](#11-things-drc-wont-tell-you)).

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
| End mill | **TODO** confirm: poster says 1.0, `e44.kicad_dru` comment says 2.0 | Copper rubout (clearing large areas) |
| Contour router | 1.0 | Board outline, slots, cutouts |
| Spiral drills | 0.7, 0.8, 1.0, 1.5, 2.0 | All round holes and vias |

## 3. The example: pic_programmer

`pic_programmer` is one of the demo projects installed with KiCad (look in
KiCad's `demos` folder, or **File → Open Demo Project** from the KiCad project
manager). It's a two-layer, through-hole board, which makes it a good
match for the E44, but it was designed for a board house.

![The stock pic_programmer board in the KiCad 10 PCB editor](images/original-board.png)

Copy the demo to your own folder before changing it. The finished version is in
[`../examples/pic_programmer/`](../examples/pic_programmer/) so you can compare
your result.

## 4. Step 1: Add the E44 rules

1. Open the board in the PCB Editor.
2. **File → Board Setup… → Design Rules → Custom Rules**.
3. Paste the contents of [`e44.kicad_dru`](../e44.kicad_dru).
4. Check the message pane under the editor. It must say **No errors found**.
   The "share the same condition" lines are information only.
5. Click **OK**.

![Board Setup, Custom Rules page with the E44 rules pasted in](images/board-setup-custom-rules.png)

KiCad stores these rules in `<project name>.kicad_dru` next to your
`.kicad_pro` file. Copying `e44.kicad_dru` into your project folder and
renaming it to match the project (here `pic_programmer.kicad_dru`) does the
same thing.

> **Always check the message pane.** If there's an error anywhere in the file,
> KiCad ignores **every** custom rule, and DRC then passes boards it shouldn't.
> An earlier version of `e44.kicad_dru` had line breaks inside its `condition`
> strings, which KiCad can't read:
>
> ![The rule checker reporting "Unterminated delimited string"](images/rules-syntax-error.png)
>
> Keep each `condition "..."` on a single line if you edit the rules.

## 5. Step 2: Set the board defaults

The custom rules catch problems when you run DRC. Setting KiCad's defaults to
match stops you creating those problems while you work. All of these are in
**File → Board Setup…**.

### Constraints

**Design Rules → Constraints**:

| Setting | Value |
|---|---|
| Minimum clearance | 0.35 |
| Minimum track width | 0.5 |
| Minimum annular width | 0.6 |
| Minimum via diameter | 1.9 |
| Copper to edge clearance | 1 |
| Minimum drill size | 0.7 |

![Board Setup, Constraints page with the E44 values](images/board-setup-constraints.png)

The custom rules take priority over these, which is how 1.0 mm holes are
allowed a 0.5 mm annular ring even though the minimum here is 0.6.

### Net classes

**Design Rules → Net Classes**. Set **every** net class, not just `Default`,
to at least 0.35 clearance, 0.5 track width and a 1.9 / 0.7 via.

![Board Setup, Net Classes page](images/board-setup-net-classes.png)

pic_programmer has a `POWER` class for GND and VCC with 0.8 mm tracks, which
is fine, but its clearance was 0.28 mm, so it had to be raised to 0.35.

### Pre-defined sizes

**Design Rules → Pre-defined Sizes**. These appear in the track width and via
size drop-downs while routing:

- Tracks: 0.5, 0.8, 1, 1.5
- Vias: 1.9 / 0.7

![Board Setup, Pre-defined Sizes page](images/board-setup-predefined-sizes.png)

### Violation severity

**Design Rules → Violation Severity**. The E44 makes no solder mask or
silkscreen, so set these to **Ignore**. Otherwise they bury the errors that
matter (pic_programmer reports nearly 100 of them):

- Solder mask aperture bridges items with different nets
- Silkscreen clearance
- Silkscreen clipped by solder mask
- Silkscreen clipped by board edge

![Board Setup, Violation Severity page with the silkscreen checks set to Ignore](images/board-setup-violation-severity.png)

## 6. Step 3: Run DRC to see what needs changing

**Inspect → Design Rules Checker**, then **Run DRC**.

![DRC dialog on the stock board, reporting 269+ errors](images/drc-before.png)

Every red arrow on the board is a problem:

![The stock board covered in DRC markers](images/drc-before-markers.png)

For the stock pic_programmer the errors were:

| Problem | Count | Fixed in |
|---|---|---|
| Annular ring too small | ~200 | [Step 4](#7-step-4-make-e44-versions-of-your-footprints) |
| Pad hole not in the drill set | 38 | [Step 4](#7-step-4-make-e44-versions-of-your-footprints) |
| Via hole not in the drill set | 6 | [Step 5](#8-step-5-fix-tracks-vias-text-and-pours) |
| Track narrower than 0.5 mm | 11 | [Step 5](#8-step-5-fix-tracks-vias-text-and-pours) |
| Copper closer than 0.35 mm | 16 | [Step 6](#9-step-6-reroute-what-no-longer-fits) |
| Copper closer than 1.0 mm to the edge | 1 | [Step 5](#8-step-5-fix-tracks-vias-text-and-pours) (zone refill) |

Nearly all of these come from footprints, so start there.

## 7. Step 4: Make E44 versions of your footprints

Footprints are where most of the work is. Stock KiCad footprints assume
plated board-house holes and tight pad spacing.

### 7.1 Make a project library

Don't edit footprints one at a time on the board: the next **Update PCB from
Schematic** overwrites them. Instead keep E44 copies in a project library:

1. **Preferences → Manage Footprint Libraries… → Project Specific
   Libraries**.
2. Click the folder button and create a new library, for example `E44.pretty`
   in the project folder, with nickname `E44`.

![Footprint Libraries dialog with the E44 project library added](images/footprint-libraries.png)

3. Open **Tools → Footprint Editor**, open each stock footprint the board
   uses, and **File → Save As…** into the `E44` library. Adding `_E44` to
   the name makes them easy to tell apart.
4. Edit the copy (sections 7.2–7.6 below) and save it.

This builds up a reusable set of E44-ready footprints. The 21 made for
pic_programmer are in
[`../examples/pic_programmer/E44.pretty`](../examples/pic_programmer/E44.pretty).
**TODO:** decide whether the lab keeps a shared E44 library in this repository.

### 7.2 Hole sizes

Every round hole must be 0.7, 0.8, 1.0, 1.5 or 2.0 mm. KiCad's hole sizes
already include room for the lead, so round each one to the nearest drill in
the set, going **up** unless the bigger hole no longer fits on the pin pitch.

| Stock KiCad hole | E44 drill | pic_programmer parts |
|---|---|---|
| 0.75 | 0.8 | TO-92 transistors |
| 0.8 | 0.8 | Resistors, diodes, small capacitors, DIP sockets |
| 0.9 | 1.0 | 5 mm LEDs |
| 1.0 | 1.0 | DSUB-9, ZIF socket |
| 1.1 | 1.0 | TO-220 regulator. Rounded **down**: a 1.5 mm hole needs a 2.7 mm pad, which doesn't fit at 2.54 mm pitch. Check the part's lead size against 1.0 mm |
| 1.2, 1.27, 1.3 | 1.5 | Electrolytics, inductor, trimmer, terminal block |
| 2.0 | 2.0 | ZIF socket mounting pegs |
| Over 2.0 | Routed cutout | Mounting holes (see 7.4) |

### 7.3 Pad sizes and annular rings

The annular ring is the copper left around a hole:
`(pad size − hole size) / 2`. It must be at least 0.6 mm, or 0.5 mm on a
1.0 mm hole. That gives these minimum pad sizes:

| Hole | Minimum pad |
|---|---|
| 0.7 | 1.9 |
| 0.8 | 2.0 |
| 1.0 | 2.0 |
| 1.5 | 2.7 |
| 2.0 | 3.2 |

For an oval pad only the narrow side has to meet the minimum. Double-click a
pad in the Footprint Editor (or on the board) to open **Pad Properties** and
change **Pad size X / Y** and the hole **Diameter**. The pic_programmer DIP
sockets kept their long 2.4 mm pads and only grew the narrow side:

![Pad Properties for a DIP socket pad, before (2.4 x 1.6) and after (2.4 x 2.0)](images/pad-properties.png)

Bigger pads use up the space between them, so check the gap stays at least
0.35 mm. On a 2.54 mm pitch, 2.0 mm pads leave 0.54 mm, which passes, but no
track can fit through.

| pic_programmer footprint | Stock hole / pad | E44 hole / pad |
|---|---|---|
| Resistors, diodes, C_Disc, C_Axial | 0.8 / 1.6 | 0.8 / 2.0 |
| DIP-8/14/18/28 LongPads (ICs and sockets) | 0.8 / 2.4 × 1.6 | 0.8 / 2.4 × 2.0 |
| LED_D5.0mm | 0.9 / 1.8 | 1.0 / 2.0 |
| TO-220-3 (U3) | 1.1 / 1.9 × 2.0 | 1.0 / 2.0 |
| DSUB-9 (J1) | 1.0 / 1.6 | 1.0 / 2.0 |
| 40-pin ZIF socket (P3) | 1.0 / 1.6 × 2.8 | 1.0 / 2.0 × 2.8 |
| ZIF socket pegs | 2.0 / 2.8 | 2.0 / 3.2 |
| CP_Axial (C1, C2) | 1.2 / 2.4 | 1.5 / 2.7 |
| L_Radial (L1) | 1.3 / 2.6 | 1.5 / 2.7 |
| Trimmer (RV1) | 1.27 / 2.03 | 1.5 / 2.7 |
| Terminal block (P1) | 1.3 / 3.0 | 1.5 / 3.0 |

### 7.4 Holes bigger than 2.0 mm

The drills stop at 2.0 mm, but the 1.0 mm contour router cuts anything drawn
on **Edge.Cuts**, including internal cutouts. So for mounting holes and other
large holes:

1. Delete the pad.
2. Draw a circle of the hole's diameter on the **Edge.Cuts** layer in its
   place.

pic_programmer had three kinds: the six M4 mounting holes (4.3 mm), the DSUB
connector's mounting holes (3.2 mm) and the TO-220 tab hole (3.5 mm).

![E44 version of the DSUB-9 footprint, with 2.0 mm pads and the mounting holes drawn on Edge.Cuts](images/footprint-editor-dsub.png)

The copper-to-edge rule applies to these cutouts too, so nearby copper must be
at least 1.0 mm away.

### 7.5 Swap footprints that can't be fixed

Some footprints can't meet the rules however you size the pads. The
pic_programmer transistors used a TO-92 footprint with 1.27 mm pin pitch:
2.0 mm pads would overlap. The fix is a different footprint, KiCad's
`TO-92_Wide` (same triangle, 2.54 mm pitch). TO-92 leads bend easily to fit.

![The E44 TO-92_Wide footprint](images/footprint-editor-to92.png)

The wider footprint is bigger, so when you place it check it doesn't run into
its neighbours. Here the solder jumper next to Q3 had to move 1.27 mm.

### 7.6 Surface-mount parts

The finest pitch that works is 0.95 mm (SOT-23-5). SOIC (1.27 mm) and larger
are fine. TSSOP, MSOP, QFN and similar won't mill: choose a SOIC or
through-hole version, or use a breakout board.

The 0.35 mm clearance applies between pads **inside** a footprint too.
pic_programmer's only SMD part, the solder jumper JP1, has a 0.3 mm gap
between its pads. In the E44 copy each pad was moved 0.1 mm outwards to make
it 0.5 mm, which is still easy to bridge with solder.

### 7.7 Put the E44 footprints on the board

Once the library is ready:

1. **Edit → Change Footprints…** swaps the footprints on the board for the E44
   versions, keeping position, rotation and connections. Use it once per
   footprint type, or select several parts first.
2. In the **Schematic Editor**, change each symbol's **Footprint** field to
   the `E44:` version as well, otherwise the next **Update PCB from
   Schematic** puts the stock footprints back.
3. Run DRC with **Test for parity between PCB and schematic** ticked to check
   the board and schematic agree.

## 8. Step 5: Fix tracks, vias, text and pours

- **Thin tracks.** Select them, press **E** and set the width to at least 0.5.
  pic_programmer had 11 tracks of 0.35–0.43 mm. **Edit → Edit Track & Via
  Properties…** can change all tracks of a net class at once.
- **Vias.** Change every via to 1.9 / 0.7 with **Edit → Edit Track & Via
  Properties…**. Better still, avoid vias (see
  [section 11](#11-things-drc-wont-tell-you)).
- **Copper text.** Text on a copper layer is milled like a track, and KiCad's
  default text strokes (0.3 mm on this board) are thinner than the 0.5 mm
  track minimum. DRC doesn't check this. Move labels to the silkscreen layer
  or delete them. pic_programmer had 19 copper labels: the front ones moved to
  F.Silkscreen, and the two mirrored bottom ones were removed because they
  repeat the front ones.
- **Copper pours (zones).** Open each zone's properties and set **Clearance**
  to at least 0.35 and **Minimum width** to at least 0.5, then refill with
  **Edit → Fill All Zones**. Pour ground on spare area wherever you can: every
  bit of bare board has to be milled away, so a pour saves machine time and
  tool wear.

## 9. Step 6: Reroute what no longer fits

After the footprint changes DRC still listed about 30 clearance errors. Most
were tracks that used to run between 2.54 mm pins, which there's no longer
room for (2.0 mm pads leave 0.54 mm, and a 0.5 mm track needs 0.35 mm clearance
each side, 1.2 mm in total). The rest were tracks to the wider TO-92s.

![Close-up of the PIC sockets before and after: tracks no longer squeeze between pins](images/closeup-sockets.png)

There are two ways to fix this:

- **By hand**, with the interactive router (**Route → Route Single Track**),
  for a handful of tracks. Delete the failing track and route it again around
  the pins.
- **With an autorouter**, for many. pic_programmer was rerouted with
  [Freerouting](https://freerouting.app), which reads the net class widths
  and clearances from KiCad:
  1. Delete every track on each net that has an error, so those nets start
     clean. (For pic_programmer that was 9 nets.)
  2. Delete the copper pour for now. Freerouting treats a pour as solid copper
     and won't route through it, so nothing can use that layer.
  3. **File → Export → Specctra DSN…**
  4. Open the `.dsn` in Freerouting, run the autorouter, and save the result
     as a Specctra session (`.ses`) file.
  5. **File → Import → Specctra Session…** to bring the new tracks into
     KiCad.
  6. Redraw the pour, refill it, and run DRC.

Always check autorouted tracks by eye: they're correct, but they don't know
which parts of the board you'd like to keep clear.

![Close-up of the transistor area before and after: wider TO-92s and new routes](images/closeup-transistors.png)

![Close-up of the serial connector area before and after](images/closeup-connector.png)

## 10. Step 7: Final DRC

Run DRC again, with **Refill all zones before performing DRC** ticked. The
board is ready when there are no errors and no unconnected items.

![DRC dialog on the adapted board: 0 violations, 0 unconnected](images/drc-after.png)

![The adapted pic_programmer board](images/adapted-board.png)

The copper the E44 will leave behind on each side (black is copper):

| Bottom (B.Cu) | Top (F.Cu) |
|---|---|
| ![Bottom copper of the adapted board](images/copper-bottom.png) | ![Top copper of the adapted board](images/copper-top.png) |

## 11. Things DRC won't tell you

- **Holes aren't plated.** On a board-house PCB every hole joins top and
  bottom copper. On the E44 they don't, so:
  - A **via** needs a wire or rivet pushed through and soldered both sides.
    The rerouted pic_programmer uses none.
  - A **top-layer track to a through-hole pad** must be soldered on the top
    side as well. That's easy for a resistor, but hard or impossible under a
    DIP socket, a ZIF socket or a connector body. The adapted pic_programmer
    still has top tracks to its sockets, which are fine for a plated board
    but awkward on the E44.

  **TODO:** document the lab's double-sided practice (rivets, through-plating,
  board flipping and alignment). Until then, prefer single-sided boards, or
  keep top-layer tracks away from pads you can't reach from the top.
- **Copper text width** isn't checked (see [Step 5](#8-step-5-fix-tracks-vias-text-and-pours)).
- **Inside corners** of the board outline and cutouts can't be sharper than
  R0.5, because the router is 1.0 mm. Round them off, or allow for it in
  anything that has to fit.
- **Lead sizes.** The drill table is a starting point. Check the datasheet
  lead size of any part you round down (like the TO-220 here).

## 12. DRC message reference

| DRC message | Cause | Fix |
|---|---|---|
| `E44 clearance 0.35mm` | Copper too close together | Reroute, or shrink pads in the E44 footprint |
| `E44 min track 0.5mm` | Track narrower than 0.5 mm | Select the track, press **E**, set width to 0.5 or more |
| `E44 copper to edge 1.0mm` | Copper within 1.0 mm of Edge.Cuts | Move the part or track inwards, or refill the zone |
| `E44 annular ring 0.6mm` | Pad too small for its hole | Enlarge the pad ([7.3](#73-pad-sizes-and-annular-rings)) |
| `E44 annular ring 0.5mm on 1.0mm holes` | Pad on a 1.0 mm hole smaller than 2.0 mm | Enlarge the pad to 2.0 mm |
| `E44 slot narrower than 1.0mm router` | Oval hole under 1.0 mm wide | Widen the slot |
| `E44 pad drill not in tool set` | Pad hole isn't 0.7/0.8/1.0/1.5/2.0 | Change the hole ([7.2](#72-hole-sizes)), or make it a cutout ([7.4](#74-holes-bigger-than-20-mm)) |
| `E44 via drill not in tool set` | Via hole isn't in the set | Change the via to 1.9 / 0.7 |

## 13. Exporting for CircuitPro

**TODO:** Gerber and drill export settings for LPKF CircuitPro (layers,
Excellon format and units, drill map), and how to bring the files to the lab.

## 14. Pre-flight checklist

- [ ] E44 custom rules added, and the rule checker says **No errors found**
- [ ] Every net class: clearance ≥ 0.35, track ≥ 0.5, via 1.9 / 0.7
- [ ] Silkscreen and solder mask checks set to Ignore
- [ ] Footprints come from the E44 library, in both the board and the schematic
- [ ] Every hole is 0.7, 0.8, 1.0, 1.5 or 2.0 mm; larger holes are Edge.Cuts cutouts
- [ ] No SMD parts finer than 0.95 mm pitch
- [ ] No copper text
- [ ] Board outline closed, inside corners R0.5 or larger
- [ ] Ground pour on spare area
- [ ] As few vias and top-side joints as possible
- [ ] DRC: 0 errors, 0 unconnected, schematic parity passes
- [ ] Gerbers and drill files exported
