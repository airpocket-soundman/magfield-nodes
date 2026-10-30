# MAGFIELD NODE — Design Notes (v3: J2 on back side, centerline layout)

Magnetic-field visualization node board. ~100 nodes are placed in a grid; each node
reads its local magnetic field vector with a BMM150 magnetometer and shows the field
direction as an RGB LED color, so field distortion over a 10 m × 10 m area becomes
visible as a pattern of light.

- Board: **32 × 32 mm, 2-layer, 1.6 mm**, JLCPCB standard rules
  (track/gap ≥ 0.15 mm, via 0.3 mm drill / 0.6 mm pad)
- Finish: **standard HASL** (v2: the v1 card-edge fingers are gone, so no ENIG
  is needed — HASL is the cheaper standard finish)
- Battery holder on **bottom** side; all SMT parts on top **except J2**, which
  moved to the bottom side alongside the holder in v3 (see J2 section) —
  assembly is single-side for the top, plus the one bottom-side connector
- Files are KiCad 7 format (schematic 20230121 / PCB 20221018). KiCad 8 opens them directly.
- Panelization: single-board files only. When ordering, use JLCPCB
  **"Panel by JLCPCB"** (e.g. a 5 × 4 or 10 × 10 grid with mouse bites / V-cuts);
  panelizing is where the SMT cost saving is — bare-PCB cost barely changes.

## Architecture

```
 LIR2032 cell (BT1, bottom)
   │  CELL_P / CELL_N
 DW01A (U3) + FS8205A dual MOSFET (Q1)   over-discharge / over-current protection
   │  VBAT_RAW ───────────────► J2 pin 2 (dock charges here via Grove cable,
   │                             load switch OFF while charging)
 slide switch SW1 (SPDT, top)   load switch: VBAT_RAW → VBAT
   │
   ├─ VBAT rail (3.0–4.2 V): ATtiny1616 (U1), WS2812B-MINI LED (D1)
   └─ XC6206 3.0 V LDO (U4) → 3V0 rail: BMM150 (U2) only
        I2C (PA1=SDA, PA2=SCL on the ATtiny, pull-ups 4.7 kΩ to 3V0)
        DRDY → PC0, INT → PC1
```

### Power design

- The cell connects through the DW01A/FS8205A protection pair in the negative leg,
  the standard DW01A application circuit (R1 100 Ω + C1 100 nF supply filter,
  R2 1 kΩ on the current-sense input).
- **VBAT_RAW** is the protected cell output. It is routed directly to J2 pin 2,
  so the external dock charges the cell **with the load switch off**:
  no MCU/LED current flows during charging, charge-termination detection in the dock
  stays clean, and nothing on the node needs charge management.
- The slide switch is the only path from VBAT_RAW to the **VBAT** load rail.
- ATtiny1616 runs directly from VBAT (rated 1.8–5.5 V, so a full 4.2 V LIR2032 is fine).
- BMM150 absolute maximum is 3.6 V, so it is fed from a 3.0 V LDO (3V0 rail).
  I2C pull-ups go to 3V0; ATtiny V_IH at 4.2 V supply is 0.6·VDD ≈ 2.52 V < 3.0 V, OK.
- WS2812B-MINI runs from VBAT. Its datasheet rating starts around 3.5 V, so LED
  brightness/color near end-of-discharge (≈3.0–3.3 V) must be verified on the bench
  (see Risks). DIN is driven from PA4 through 300 Ω; 100 nF decoupling at the LED.

## Dock connector J2 — Grove-style JST PH 4-pin (v2, replaces card-edge fingers)

**Change (2026-10-30, v2):** the v1 card-edge fingers were replaced by one
4-pin JST PH SMD connector along the south edge. Rationale: fingers
forced an ENIG (gold) finish on every node PCB and expensive card-edge
sockets on the dock; a PH connector is keyed by its housing, costs ~$0.24,
lets the node PCB use standard HASL, and the dock side becomes a plain
Grove/PH cable instead of a precision socket.

**Change (2026-10-30, v3):** J2 moved to the **back side** (B.Cu, the same
side as the battery holder), centered on the board's vertical centerline
**(x = 16.0 mm**, footprint origin (16.0, 26.9)), mating face still pointing
off the south edge. The battery holder shifted 0.6 mm north (center
(16.0, 11.9)) to make room. Coexistence check: the holder is a metal clip
whose outline is the Ø20 mm cell itself — cell south rim at y = 21.9 mm vs.
the J2 housing north face at y = 22.45 mm, ≈0.55 mm plan clearance; no pads
collide (holder pads end at y = 17.4, J2 pads start at y = 22.3). They fit,
but a sample fit-check before fab is on the TODO list (Risks #12).

**Pin mapping after the back-side flip (verified 2026-10-30):** KiCad's
footprint flip is a true mirror — pads keep their numbers and nets. On the
back side the pads sit at pin 1 = GND (13.0, 29.75), pin 2 = VBAT_RAW
(15.0, 29.75), pin 3 = UPDI (17.0, 29.75), pin 4 = ID (19.0, 29.75).
In board coordinates pin 1 moved from the east end (v2, top side) to the
west end (v3, back side); viewed from the back — the side you mate from —
the order reads the same as the front view did in v2. A standard keyed
PH/Grove cable therefore still lands pin 1 → pin 1 on the charge board:
**1 = GND, 2 = VBAT_RAW, 3 = UPDI, 4 = ID**, unchanged electrically.

- Part: **JST S4B-PH-SM4-TB(LF)(SN)**, side-entry SMD, 2.0 mm pitch —
  **LCSC C265102** (verified on LCSC; $0.2424 @50+). Footprint is the KiCad
  standard-library `JST_PH_S4B-PH-SM4-TB_1x04-1MP_P2.00mm_Horizontal`,
  generated from the JST ePH datasheet. The two solder-retention (MP) tabs
  are tied to GND.
- Cable side: PH housing PHR-4 with crimp contacts (or a pre-made Grove
  4-pin cable) from the dock.

| J2 pin | Net | Notes |
|---|---|---|
| 1 | GND | |
| 2 | VBAT_RAW (B+) | charge input from dock; battery side of protection, **before** the load switch (same net the v1 fingers used) |
| 3 | UPDI | dock can reprogram inserted nodes |
| 4 | ID / NC | spare — **left unconnected** on the node (no ID-resistor footprint fitted; the dock may use it later for node detection) |

Trade-off vs. v1: the PH housing is polarized, so reverse insertion is
mechanically impossible; the mirror-safe finger trick is no longer needed.
The node is now connected by a short cable instead of sliding into a slot.

## Programming

- UPDI pads **J1** on the top side: UPDI / VBAT / GND (2.54 mm pads, pogo or header).
  Located mid-west edge at (5.00, 15.60), rotated 90° (moved from the NW
  corner on 2026-10-30 to free the MH1 mounting hole — see "Mounting
  holes").
- UPDI is also routed to **J2 pin 3**, so the dock can reprogram nodes.
- ATtiny pin map used: PA0 = UPDI, PA1 = SDA, PA2 = SCL, PA4 = LED data,
  PC0 = BMM150 DRDY, PC1 = BMM150 INT. All other GPIOs are unconnected (marked NC).

## Footprint sources

| Part | Footprint | Source |
|---|---|---|
| U1 ATtiny1616-MNR | VQFN-20-1EP 3×3 mm P0.4, EP 1.7×1.7 | Pad geometry matches KiCad library `Package_DFN_QFN:VQFN-20-1EP_3x3mm_P0.4mm_EP1.7x1.7mm` (verified against the installed KiCad 7 footprint library); pin order verified against Microchip DS40002204 (pinout + Table 5-1 port multiplexing, cross-checked both ways) |
| U2 BMM150 | **Custom WLCSP-12**, created for this design | Ball map from Bosch BMM150 datasheet/flyer: 0.4 mm grid, balls on checkerboard positions — A1 PS, A3 SCL, A5 CSB, B2 VDDIO, B4 SDA, C1 SDO, C5 GND, D2 INT, D4 DRDY, E1/E3 GND, E5 VDD. Ball pads Ø0.24 mm. No library footprint exists, so it was drawn from the datasheet |
| U3 DW01A, Q1 FS8205A | SOT-23-6 | Pad geometry from KiCad library (centers ±1.1375, pads 1.325×0.6). **Pin arrangement assumed** — see Risks |
| U4 XC6206P302MR | SOT-23 | KiCad library geometry; pinout 1=GND, 2=VOUT, 3=VIN per the LCSC-hosted XC6206P datasheet |
| D1 WS2812B-MINI-V3/W | **Custom** 4-pad (2.0×1.85 mm body class) | Pad order is **assumed** — see Risks |
| BT1 MY-2032-10 | **Custom** | KiCad library has no MYOUNG holder; pads 4.5×3.2 mm at ±10.5 mm, Ø24 mm courtyard — **verify against the MYOUNG drawing before fab** |
| SW1 slide switch | Generic MSK12C02-class SPDT | A = VBAT_RAW (in), COM = VBAT (out), B unconnected; two mounting pads |
| J1 | Custom pads (UPDI test pads) | — |
| J2 | JST PH S4B-PH-SM4-TB 4-pin side-entry SMD | KiCad standard library footprint (generated from JST ePH datasheet) |
| 0603 passives | KiCad library R_0603 / C_0603 pad geometry | — |

The B.Cu layer is a solid GND zone (filled, 738 mm²) except where bottom-side
routing (VBAT_RAW trunk, CELL_N) and the battery-holder pads cross it.

## BOM / LCSC mapping — standardized 2026-10-30 (all-LCSC + JLCPCB-SMT)

Every line was verified on 2026-10-30 against **both** LCSC (listed and in
stock) and the **JLCPCB SMT assembly library** (assembly-supported, in
stock), preferring JLCPCB **Basic > Preferred > Extended**. The schematic
carries an `LCSC` and an `MPN` property for every assembled part; the
quote BOM (`../quote/node-bom-jlcpcb.csv`) is generated from them.
Stock figures are a 2026-10-30 snapshot — re-check at order time.

| Ref | Part (MPN) | LCSC | JLCPCB class | Notes |
|---|---|---|---|---|
| U1 | Microchip ATtiny1616-MNR | C507118 | Extended | in stock both |
| U2 | Bosch BMM150 | C171681 | Extended | JLCPCB flags **X-ray inspection required** (WLCSP) — the baseline quote carried a $51 X-ray line; unavoidable |
| D1 | Worldsemi WS2812B-MINI-V3/W | C527089 | Extended | **Standard PCBA only** (not Economic); MSL 5 → JLCPCB baking fee applies; unavoidable |
| U3 | KUU DW01A | C42459937 | Extended | The 9-digit number is real — verified on LCSC and matched in the baseline quote ($2.4486/102 pcs). Supports 0V-battery charging, over-discharge cutoff 2.4 V. (Fortune DW01A-G C61503 is out of stock at JLCPCB and lacks 0V-charge.) |
| Q1 | TECH PUBLIC FS8205A | C2830320 | Extended | common-source dual N-FET, in stock both (C32254 elsewhere is FS8205 without "A", out of stock) |
| U4 | Torex XC6206P302MR-G (3.0 V LDO, SOT-23) | C9972 | Extended | Genuine Torex; pinout 1=GND, 2=VOUT, 3=VIN matches the routed footprint (see CPL section). The baseline quote matched a clone listing (C5148691) — C9972 is the pinout-verified original |
| SW1 | SHOU HAN MSK12C02 (SPDT slide switch) | C431540 | Extended | The older C16168 listing is delisted at LCSC; C431540 is the live number. This fixes the baseline quote's only unmatched line |
| BT1 | MYOUNG MY-2032-10 holder | C964827 | Extended | cell itself is NOT stocked at LCSC — buy separately (≈$1.26 PowerStream / AliExpress) |
| J2 | JST S4B-PH-SM4-TB(LF)(SN) | C265102 | Extended | $0.2424 @50+ |
| C1,C2,C6,C7 | Samsung CL10B104KB8NNNC 100 nF 50 V X7R 0603 | C1591 | Extended | See exception note below |
| C3 | Samsung CL10A106KP8NNNC 10 µF 10 V X5R 0603 | C19702 | **Basic** | 10 V rating is fine on the 4.2 V-max rail |
| C4,C5 | Samsung CL10A105KB8NNNC 1 µF 50 V X5R 0603 | C15849 | **Basic** | |
| R1 | UNI-ROYAL 0603WAF1000T5E 100 Ω | C22775 | **Basic** | |
| R2 | UNI-ROYAL 0603WAF1001T5E 1 kΩ | C21190 | **Basic** | |
| R3 | UNI-ROYAL 0603WAF3000T5E 300 Ω | C23025 | **Basic** | |
| R4,R5 | UNI-ROYAL 0603WAF4701T5E 4.7 kΩ | C23162 | **Basic** | |

**100 nF exception:** the canonical JLCPCB **Basic** 100 nF 0603 (Yageo
CC0603KRX7R9BB104, C14663) showed **OutOfStock at LCSC retail** on
2026-10-30 (checked twice), failing the all-LCSC-in-stock rule. C1591 is
the identical spec (100 nF / 50 V / X7R / 0603) with ~7.8 M pieces at LCSC
and JLCPCB SMT stock, so it was chosen despite Extended class. All other
passives are JLCPCB Basic. No BOM function lacked an assembly-supported
option — no circuit part had to be substituted.

**Panel note (from the baseline quote):** with "Panel by JLCPCB", JLCPCB's
preset set PCB quantity to 20 panels while PCBA was quoted at 17 — check
the panel count on the final order so bare-PCB and assembly quantities
agree with the decided 3×2 panel plan.

## CPL / JLCPCB assembly data

`../quote/node-cpl-jlcpcb.csv` is generated from `kicad-cli pcb export pos`
(mm), **in the same absolute coordinate frame as the gerber export**
(board outline spans X 0…32 mm, Y −32…0 mm in that frame). The previous
CPL used board-corner Y-down coordinates, which mirrored the overlay
against the gerbers in JLCPCB's preview — that is fixed.

**Rotation basis:** all footprints sit at 0° on the board (verified from
the board file — the layout is fully axis-aligned), so every CPL rotation
is 0.0 as a board fact, not a placeholder. For the standard KiCad library
footprints (0603, SOT-23, SOT-23-6, VQFN-20, JST PH) KiCad's drawn
orientation matches the JLCPCB/EasyEDA library orientation at rotation 0
(pin 1 top-left / NW). For the custom footprints, pin-1 (or key pad)
directions in board coordinates (viewed from the top, +X east, +Y south)
are:

| Part | Key pad position (board view) |
|---|---|
| D1 (WS2812B-MINI) | pad 1 = VDD at **NW**; pads run 1 NW → 2 SW → 3 SE (GND) → 4 NE (DIN) |
| U2 (BMM150) | ball **A1 at NW** corner; rows A→E run west→east, columns 1→5 north→south |
| U1 (VQFN-20) | pin 1 on the west edge, northernmost west-side pad; numbering CCW from there |
| U4 (SOT-23) | pin 1 (GND) NW, pin 2 (VOUT) SW, pin 3 (VIN) east-center |
| SW1 (MSK12C02) | pad 1 (VBAT_RAW in) west, pad 2 (COM → VBAT) center, pad 3 east (NC); mount tabs on the south side |
| J2 (bottom side) | pin 1 (GND) at the **west** end in board coordinates (see v3 pin-mapping note); pins 1→4 run west→east |
| BT1 (bottom side) | pad 1 (cell +, VBAT_RAW) contacts at east+west ends (symmetric), pad 2 (CELL_N) at center — 0°/180° equivalent |

At order time, confirm D1 / U2 / J2 in JLCPCB's assembly preview against
this table (a 10-second visual check); if a preview shows one rotated,
correct only that line's Rotation by the observed multiple of 90°.

## Validation performed (KiCad 7.0.11, kicad-cli + pcbnew)

KiCad 7's CLI has no `sch erc` / `pcb drc` subcommands (those are KiCad 8), so
validation was done with the netlist exporter, pcbnew, and scripted geometry checks:

- **Schematic loads in KiCad 7** and `kicad-cli sch export netlist` succeeds.
  The exported netlist was checked pin-by-pin against the design netlist:
  **90 pins / 22 parts, 0 mismatches, 33 nets**. The only single-pin net is `ID`
  (intentional). All unused pins carry explicit no-connect markers in the schematic
  (they appear as `unconnected-(...)` nets in the netlist).
- **PCB loads in pcbnew**: 22 footprints, 55 tracks/vias. GND zone fills (738 mm²).
- **Clearance check** (scripted, all routed copper vs. foreign pads/tracks/vias,
  pads vs. pads, both layers, 0.15 mm rule): **0 violations**. This check caught and
  fixed 3 real shorts during design (CELL_N crossing the VBAT_RAW trunk, two via
  collisions).
- **Gerbers + drill exported** to `gerbers/` with `kicad-cli pcb export`.
- **Routing status: COMPLETE (2026-10-30).** All 44 ratsnest connections from
  the v1 draft were routed (Freerouting passes + scripted hand-fixes via pcbnew).
  Final verification with **KiCad 8.0.9 `kicad-cli pcb drc --severity-all`** on
  the delivered `magfield-node.kicad_pcb`: **0 errors, 0 unconnected items**.
  Remaining warnings are only the accepted classes listed below. Gerbers + drill
  were re-exported to `gerbers/`, and final renders are in
  `iterations/final-front.png` / `iterations/final-back.png`.
- **v2 rework (2026-10-30)**: card-edge fingers → J2 PH connector (see above).
  Parts moved for the connector (Q1, C1, R2, U3 shifted west/north), affected
  nets re-routed (Freerouting + extensive scripted hand-finishing around the
  U1/U2/BT1 keep-outs). Final verification on the delivered board with
  KiCad 8.0.9 `kicad-cli pcb drc --severity-all`: **0 errors, 0 unconnected**,
  only the accepted warning classes below. Gerbers + drill re-exported;
  renders in `iterations/final2-front.png` / `iterations/final2-back.png`.
- **v3 rework (2026-10-30)**: centerline alignment (J2 and LED D1 both on
  x = 16.0), J2 flipped to the back side, holder shifted 0.6 mm north,
  board name "MAGFIELD NODE v1" re-set on the front silk in the area freed
  by J2, and a full silkscreen cleanup (all top-edge reference labels moved
  inside the outline; holder silk circle reduced to the true Ø20 mm cell
  outline; back-side texts moved off the pads). Full rip-up and re-route
  (Freerouting + scripted hand-finishing; the BMM150 escapes, the UPDI
  bridge to J2 and the battery runs are hand-placed). Final verification
  on the delivered board with KiCad 8.0.9 `kicad-cli pcb drc
  --severity-all`: **0 errors, 0 unconnected, 0 silkscreen warnings** —
  only the library classes below remain. Gerbers + drill re-exported;
  renders in `iterations/final3-front.png` / `iterations/final3-back.png`;
  3D appearance images regenerated (`iterations/appearance-top.png` /
  `appearance-bottom.png`; connector and holder models are the same
  approximations as in v2 — the real side-entry J2 is smaller than the
  model shown).
- **Parts standardization (2026-10-30)**: every BOM line now has a
  concrete LCSC part verified in stock at LCSC **and** in the JLCPCB SMT
  assembly library (see the BOM section). Schematic-only change: `LCSC`
  and `MPN` properties set on all 21 assembled symbols, U4 value updated
  to XC6206P302MR-G, SW1 value updated to MSK12C02. The PCB was not
  touched (no part's footprint or pinout changed). Re-verified after the
  edit: ERC identical to before (118 warnings / 0 errors, same classes),
  netlist exports cleanly (22 comps), KiCad 8.0.9 DRC on the delivered
  PCB still **0 errors / 0 unconnected** (22 library-class warnings =
  the accepted set below). Quote BOM regenerated from the schematic,
  CPL regenerated in the gerber coordinate frame (see CPL section),
  gerbers + drill re-exported and `node-gerbers.zip` refreshed.

## Accepted DRC warnings (final board)

Only these warning classes remain; each is cosmetic or a library-metadata
artifact, not an electrical issue:

- `lib_footprint_mismatch` (17) / `lib_footprint_issues` (5): the project's
  custom footprints (BMM150 WLCSP-12, MY-2032-10 holder, slide switch) are
  not in the standard KiCad libraries, so they differ from / lack library
  counterparts; +1 mismatch in v2 is the J2 library footprint with nets
  assigned to its pads.
- Silkscreen classes (`silk_over_copper`, `silk_edge_clearance`,
  `silk_overlap`): **zero in v3** — the v1/v2 warnings (7 / 5 / 1) were all
  eliminated by the v3 silk cleanup, so nothing remains to accept here.

## Design-rule note: board-edge clearance

v1 set `min_copper_edge_clearance = 0.05 mm` in the `.kicad_pro` solely so
the card-edge fingers could reach the board edge. With the fingers removed
in v2, the exception is gone: the setting is restored to the standard
**0.3 mm**, and the delivered v2 board passes DRC under it. All rules follow
JLCPCB standard 2-layer capability (≥0.15 mm clearance, via 0.3/0.6 mm,
hole clearance 0.25 mm).

## Mounting holes (added 2026-10-30, post-final3 revision)

User-requested mechanical fixing: two M2 screw holes on the **NW–SE
diagonal**.

- **MH1 = (3.15, 3.60)** (NW), **MH2 = (29.30, 29.30)** (SE). Footprint
  `MountingHole_2.2mm_M2`: NPTH ⌀2.2 mm, no annular copper, excluded from
  BOM and position files (verified absent from the POS export).
- Each hole carries a **circular keepout rule area r = 2.5 mm on F.Cu+B.Cu**
  (no tracks / vias / zone fill; the B.Cu GND pour is carved out around
  both holes). This also covers the M2 screw-head/washer zone (⌀ ≈ 4.5 mm).
- Measured minimum copper distance after routing: **MH1 2.64 mm** (UPDI
  via at (6.09, 3.62)), **MH2 3.06 mm** (SW1 mount pad). DRC keepout rule:
  clean.
- **Why NW–SE:** NE is unusable (BMM150 sensor corner + R4/R5 + dense
  escapes — and magnetic hygiene forbids crowding it). SW and SE were both
  clear but are not a diagonal. NW was blocked only by the J1 test pads
  and two local runs, so J1 was relocated instead (below).
- **Back-side check:** both holes and their keepouts are clear of the BT1
  holder lands and the cell circle (MH2 centre is 21.9 mm from the cell
  centre, r = 10 mm). A screw head/washer on the back lands on bare board
  at both holes — nothing can crush the cell; the closest back copper to
  MH2 is 3.06 mm away (> washer radius 2.25 mm).
- **J1 relocation (the only footprint moved; J1 is hand-soldered, NOT in
  BOM/CPL):** (2.60, 5.00) rot 0° → **(5.00, 15.60) rot 90°**; pads now
  UPDI (2.46, 15.60) west / VBAT (5.00, 15.60) / GND (7.54, 15.60) east.
  Local reroutes to free the NW corner: the VBAT bridge via moved east to
  (7.62, 8.40); the SDA west-edge detour was replaced by a B-layer hop
  (vias (6.90, 0.86) / (6.85, 5.48)); J1's UPDI feed runs on B.Cu north of
  the BT1 west pad. Sensor corner and LED feed geometry untouched.
- **CPL/BOM impact: none.** POS export diffed against
  `quote/node-cpl-jlcpcb.csv`: all 21 assembled parts identical in
  position and rotation; BOM unchanged.
- Verification after the change: **DRC 0 errors / 0 unconnected** (22
  warnings — the same pre-existing lib-footprint classes listed below),
  gerbers + drill regenerated (`gerbers/`, `node-gerbers.zip`; the drill
  file contains both ⌀2.200 holes at the exact coordinates), renders
  `iterations/final4-front.png` / `final4-back.png`. Scripts:
  `work/apply_holes.py`, `work/fix_holes2.py`. Pre-change board backed up
  at `work/pre-holes-backup.kicad_pcb`.

## Magnetic hygiene (BMM150 self-interference)

The BMM150 measures µT-class fields while the board itself carries the
largest pulsed current in the system — the WS2812B LED (tens of mA PWM).
Layout choices made (v3, during the re-route):

- **Sensor corner kept quiet.** U2 sits in the NE corner; nothing routes
  underneath or around it except its own pin escapes and the 3V0 feed
  (LDO output, DC, mA-class for the sensor itself). No battery or LED
  current passes the sensor corner.
- **LED loop confined to the north edge.** D1 is on the centerline at the
  north edge, 11.5 mm from U2's center. Its VBAT feed runs along the north
  edge strip (y ≈ 1.4–2.3) and its GND return enters the ground zone
  directly at the pad, so feed and return stay adjacent along the north
  edge, minimising loop area. The return is the zone rather than a
  dedicated paired trace — accepted on a 2-layer board, since the zone
  carries the return immediately under/beside the feed strip.
- **Battery trunk kept west/south.** VBAT_RAW runs down the west edge on
  B.Cu (x ≈ 2.7–4.3) and along the south-center — the runs nearest U2 are
  the F.Cu segment ending ≈7 mm west of the sensor and the B.Cu charge
  feed at x = 23.25, ≈2.7 mm SW of the sensor's corner. That feed carries
  current only while docked and charging, when the node is off and not
  sensing.
- **What was not achievable:** a fully paired (track-over-track) LED
  supply/return — the GND zone return is the practical equivalent here;
  and the charge feed's ≈2.7 mm approach to the sensor corner (harmless
  per above, but noted).

Firmware mitigations (requirements for the node firmware):

- Sample the field **before** updating the LED, or during the LED-off
  phase of the PWM cycle, so the measurement never coincides with the
  LED's current edge.
- **Per-node offset calibration**: each node measures its own residual
  offset (LED off, known state) and stores it; apply as a correction.
- In continuous-on LED mode the LED current is constant, so its magnetic
  offset is a **fixed calibration term** — measure it once per brightness
  setting and subtract it like any other offset.

## Firmware requirements (notes for the node firmware)

- **LED modes.** Firmware shall support (1) a **continuous-on mode**
  (steady brightness; ~1 h runtime at high brightness on the 40 mAh
  LIR2032 is acceptable) and (2) a **flashing / pulsed power-saving
  mode**. See the magnetic-hygiene section for how each mode interacts
  with field sampling.
- **Sensitivity mapping parameters in EEPROM.** The field→color mapping
  shall be parameterised by values stored in EEPROM, settable at
  programming time: **baseline subtraction** (offset), **gain**,
  **deadband**, and **averaging** (sample count / window). Defaults are
  compiled in; EEPROM values override them when present.

## Risks / TODO-verify before fab (honest list)

1. ~~Routing is incomplete~~ — **Resolved 2026-10-30**: routing finished and
   KiCad 8 DRC passes with 0 errors / 0 unconnected (see Validation above).
   Items 2–6 below were checked against the schematic as delivered: its LCSC
   fields carry only the BOM-verified parts (BMM150 C171681, ATtiny1616
   C507118, WS2812B C527089, DW01A C42459937, FS8205A C2830320, MY-2032-10
   C964827, and — added in v2 — J2 JST PH C265102), so none of items 2–6 were
   resolved by schematic edits — all remain open verification tasks before fab.
2. **FS8205A pinout assumed** (1=G1, 2=S1, 3=D1, 4=D2, 5=S2, 6=G2 — the common
   SOT-23-6 arrangement for this part family). Verify against the TECH PUBLIC
   datasheet for C2830320 before fab; if it differs, only the schematic symbol
   pin mapping changes.
3. **MY-2032-10 land pattern unverified** — no datasheet drawing was available;
   verify pad positions against the MYOUNG datasheet or a sample part.
4. **WS2812B-MINI pad order assumed** (1=VDD, 2=DOUT, 3=VSS, 4=DIN CCW from
   top-left). Verify against the Worldsemi/OPSCO datasheet.
5. ~~LDO LCSC number unverified~~ — **Resolved 2026-10-30** (standardization
   pass): U4 is Torex XC6206P302MR-G, **LCSC C9972**, in stock at LCSC and
   in the JLCPCB SMT library; its SOT-23 pinout (1=GND, 2=VOUT, 3=VIN)
   matches the routed footprint as placed (verified from the board file).
6. **Slide switch footprint geometry** — part fixed 2026-10-30: SHOU HAN
   MSK12C02, **LCSC C431540** (in stock at LCSC and JLCPCB SMT). The custom
   footprint was drawn for exactly this part (3 signal pads + mount tabs,
   7-pin MSK12C02 class), so no footprint swap was needed; like the other
   custom footprints (items 3, 4), verify the pad geometry against a
   sample part or the datasheet drawing before fab.
7. **BMM150 escape routing**: 0.4 mm-pitch WLCSP checkerboard — diagonal ball
   spacing is 0.566 mm, so 0.15 mm tracks fit between balls. The breakout is
   now fully routed and DRC-clean (2026-10-30); JLCPCB standard process handles
   this pitch, but assembly of WLCSP at JLCPCB should still be confirmed for
   the order.
8. **WS2812B low-voltage behavior**: rated ≈3.5 V+; on a sagging LIR2032
   (≈3.0–3.4 V at end of discharge) color/brightness may degrade. Bench-verify;
   fallback is running the LED from the 3V0 rail (worse) or accepting dimmer output.
9. **Full-charge voltage**: VBAT_RAW/VBAT reach 4.2 V. BMM150 is protected by the
   LDO, but note the dock must apply charge voltage only to **J2 pin 2**
   (VBAT_RAW); the PH housing is polarized, but a mis-wired dock cable could
   still put charge voltage on the UPDI pin (pin 3).
10. **DW01A protection point**: with R2 = 1 kΩ the over-current threshold is set
    by the FS8205A Rds(on) per the standard application circuit; fine for LED-class
    loads, but confirm against the cell's discharge rating if the LED is driven hard.
11. **J2 PH connector (new in v2)**: LCSC C265102 verified as JST
    S4B-PH-SM4-TB(LF)(SN), and the footprint is KiCad's standard library land
    pattern generated from the JST ePH datasheet — still, confirm the land
    pattern against the JST drawing and test-fit a real PHR-4 housing / Grove
    cable before fab, including cable bend space on the dock side.
12. **J2 on the back side (new in v3)**: J2 now shares the back with the
    battery holder. Plan clearance between the Ø20 mm cell rim and the J2
    housing is only ≈0.55 mm, and the holder's true outline is itself
    unverified (item 3 — LCSC describes MY-2032-10 as a metal clip, so the
    cell rim should be the real extent, but no dimension drawing was
    obtained). Sample-fit a real holder + cell + connector before fab; if
    they clash, the holder must move further north or J2 returns to the
    front. Note the node now needs bottom-side assembly for this one part.
