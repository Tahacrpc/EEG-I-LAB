# HackEEG Shield: PCB Manufacturing-Readiness Audit

**Date:** 2026-10-03
**Mode:** Read-only. Nothing in the schematic, PCB, libraries, project settings or design rules was modified or saved. No ECO was run, no polygon was repoured, DRC was not run, and no manufacturing outputs were generated.
**Tooling:** Live Altium MCP bridge (eda-agent 0.6.1, script 2026.10.01.1, Altium Designer 26.10.1.5), plus offline parsing of the EAGLE source and manufacturer datasheets.
**Evidence folder:** `C:\EEG\analysis\mfg_audit_2026-10-03\` (`evidence\` holds the raw MCP dumps and the comparison script; `datasheet_text\` holds text extracted from each datasheet)
**Related earlier reports:** `VERIFICATION_REPORT.md` (EAGLE→Altium migration audit) and `POST_REPAIR_REPORT.md` (R1/R2a/R3/R6). This audit does not repeat them; it continues from the open items they left.

---

## 0. Verdict

**Not ready to order.** Connectivity is correct and the footprints are consistent. Several board-level fabrication details that came through the EAGLE import would change the Gerbers if outputs were generated today:

| # | Blocker | Status |
|---|---|---|
| H1 | Silkscreen artwork (component outlines, header pin numbers, labels) is on **Mechanical 7/8**, not on Top/Bottom Overlay | Verified |
| H2 | **All 166 vias are untented** in Altium. The EAGLE rules and the 2014 production Gerbers both show tented vias | Verified |
| H3 | Polygon-connect rule with a **1 mil thermal spoke**, applied to the 8 small Top/Bottom pours | Rule verified; which polygons belong to the class is inferred |
| H4 | Board-edge copper clearance rule is **0 mil** in Altium (EAGLE: 10 mil). The inner AGND/DVDD plane outlines run 8–10 mil from the edge | Rule verified; actual copper-to-edge distance not measured |
| H5 | **No current DRC result.** Only 1 stored violation from an earlier run; the imported rule set will also flood a fresh DRC with false errors | Verified |
| M1 | Layer stack is **missing the third dielectric** (DVDD–Bottom): 1.30 mm total instead of the intended ≈1.53 mm | Verified against the EAGLE layer setup |

None of these is an electrical connectivity defect. Section 6 lists the exact actions; each one that changes the design or produces output needs your approval.

---

## 1. Active project and documents (task 1)

| Item | Value (from `proj_get_path` and `app_list_documents`) |
|---|---|
| Project | `C:\EEG\hackeeg-shield\hackeeg-shield-master\eagle\Imported hackeeg-shield.PrjPcb\hackeeg-shield.PrjPcb` |
| PCB | `...\Imported hackeeg-shield.PrjPcb\hackeeg-shield.PcbDoc` (loaded, not modified) |
| Schematics | `hackeeg-shield_0.SchDoc`, `_1.SchDoc`, `_2.SchDoc` (loaded, not modified) |
| Board revision | EAGLE source `hackeeg-shield.brd` = **v1.5.0**, upstream commit `10d9b28` (hash-verified in the earlier report) |

**Integrity check after the audit:** SHA-256 of all five Altium documents plus the EAGLE `.sch`/`.brd` were recomputed. They are **identical** to `SHA256_post_repair.txt` (for example PcbDoc `71769974…7d45`, PrjPcb `143dce90…a85a`). This session changed no design file.

The only change to Altium state was switching the active (focused) document from `_0.SchDoc` to the PcbDoc. Several read tools compile the project in memory; no compile result was saved.

---

## 2. Layer stack (task 2)

### 2.1 Copper layers: verified

| Order | Altium layer | Name | Role (EAGLE `stack-up.txt`) | Copper | Content |
|---|---|---|---|---|---|
| 1 | TopLayer | Top | Signal | 1.378 mil (35 µm) | Routing plus 6 small pours |
| 2 | MidLayer1 | AGND | Ground | 35 µm | Full-board AGND pour (5345 mm²) |
| 3 | MidLayer14 | DVDD | Digital 3.3 V | 35 µm | Full-board DVDD pour (5348 mm²) |
| 4 | BottomLayer | Bottom | Signal | 35 µm | Routing plus 3 AVSS pours |

The inner layers are signal layers carrying solid polygon pours, not negative plane layers. That matches EAGLE. All 166 vias run TopLayer→BottomLayer (`pcb_get_vias`), so they are **through vias**.

> Tool note: `pcb_get_fab_stats` reports "166 buried vias" and "min track width 2 mil". Both are artifacts. The vias are verified Top→Bottom, and the 2 mil "tracks" are 396 component keepout outlines (`IsKeepout=true`, §4.1).

### 2.2 Dielectrics: defect (M1)

| Gap | Altium | EAGLE `layerSetup (1*2*15*16)`, `mtIsolate` |
|---|---|---|
| Top–AGND | Core 9.055 mil (0.23 mm), Er 4.8 | 0.23 mm |
| AGND–DVDD | Core 36.61 mil (0.93 mm), Er 4.8 | 0.93 mm |
| DVDD–Bottom | **none / 0** | 0.23 mm (15th `mtIsolate` entry) |
| **Total (with 4×35 µm Cu)** | **≈1.30 mm** | **≈1.53 mm** (close to a standard 1.6 mm build) |

The importer kept only the first two `mtIsolate` entries. **Gerber copper and drill data are not affected.** Anything derived from the stack is affected: board thickness on fab drawings or stack-up exports, impedance calculations, and via aspect ratio. Before ordering, set the stack to the fab house's standard symmetric 4-layer build (design change, needs approval) or specify the stack-up directly to the fab.

The earlier report noted that the importer also created 30 mid-layers and 16 plane layers that are unused. `pcb_get_layer_stackup` now reports `layer_count: 4`, so check this in the Layer Stack Manager.

---

## 3. Design rules and scopes (task 3)

There are 45 rules (`pcb_get_design_rules`). **No rule is scoped by net or net class**; the only user net class is `All Nets`.

### 3.1 Rules that match the EAGLE fab intent (MF_Standard_4_Layer DRU)

| Rule | Altium | EAGLE | Board actual |
|---|---|---|---|
| Clearance (All/All, two rules, priorities 2/3) | 5 mil | mdWireWire/Pad/Via 5 mil | — |
| Clearance, polygon class Isolate_12 | 12 mil | polygon isolate 0.3048 mm | AGND/DVDD pours |
| Width min | 5 mil | msWidth 5 mil | min copper **6 mil** |
| Hole size min | 10 mil | msDrill 10 mil | min **15 mil**, max 126 mil, 11 distinct sizes |
| Hole-to-hole | 10 mil | mdDrill 10 mil | not measured (needs DRC) |
| Routing via | 10/18 mil min | — | vias 15/23, 16/24, 20/30, 23–24/35–36 mil |
| Solder mask expansion (All) | 2 mil | mlMin/MaxStopFrame 2 mil | see §5 |
| Paste expansion (All) | 0 mil | mlCreamFrame 0 | — |
| Pad classes SMDSolder_OFF / SMDPaste_OFF | 0 mil | EAGLE pad stop/cream off | members not enumerated |

### 3.2 Rules that are wrong or are importer defaults

| Rule | Value | Problem | Severity |
|---|---|---|---|
| `mdCopperDimension` (Board Clearance, All) | **0 mil** | EAGLE `mdCopperDimension` = **10 mil**. Pours and copper are not kept back from the edge (H4) | High |
| `PolygonConnect_For_PolygonThermal_ON_Width10000_Gap100000` (P2) | Relief, **conductor 1 mil**, gap 10 mil, 4 spokes, not vias | 1 mil spokes cannot be fabricated (H3) | High |
| `Width` | Min 5 / **Max 10** / Pref 5 mil | 474 copper tracks are 11–35 mil (power nets), so DRC will flag all of them. EAGLE had no maximum | Medium (false DRC) |
| `AssemblyTestPointUsage` / `FabricationTestPointUsage` | "One Required" for All | Every net will fail; the board has no testpoint strategy | Medium (false DRC) |
| `RoutingCorners`, `Fanout_*`, `RoutingTopology`, `DiffPairsRouting`, `Height` 1000 mil | Defaults | Routing-time only, or harmless | Low |
| `MinimumSolderMaskSliver` | 10 mil, **disabled** | Mask slivers are unchecked; check them in CAM | Medium |
| `PlaneClearance` / `PlaneConnect` | 20 mil / relief | Not used (no negative planes) | Low |
| `UnpouredPolygon` | Allow modified: No, shelved: No | Flags any polygon that needs a repour. Repour state not checked (repour forbidden) | Info |

How the polygon-connect classes map: the class names encode the EAGLE polygon wire width (`Width100000` = 10 mil, `160000` = 16 mil, `80000` = 8 mil, `10000` = **1 mil**). EAGLE polygon widths (from `.brd`): DVDD (L15) 10 mil, AGND (L2) 16 mil, AVSS (L16, one pour) 8 mil, and **AVDD, IOREF, BOARD_ADDR_0, VCC_5V_FILTERED, AVSS ×2 (L1) and AVSS ×2 (L16) all 1 mil**. So 8 of the 11 pours almost certainly fall under the 1 mil spoke rule. **Confirming membership needs Altium's Object Class Explorer.** This MCP bridge does not enumerate class members.

---

## 4. Copper, polygons, clearances and DRC (tasks 4, 5)

### 4.1 Copper geometry: verified

* 3166 tracks in total. **1863 are on copper layers**, and 396 of those are 2 mil **keepout** outlines owned by components (all `IsKeepout=true`, no net), imported from EAGLE tRestrict. Keepouts do not reach the Gerbers.
* Real copper width histogram (mil → count): 6→36, 8→609, 10→348, 11→1, 12→151, 14→78, 16→147, 24→74, 35→23. **Minimum copper width is 6 mil**, which is at or above the 5 mil rule.
* Annular ring: vias **4 mil** minimum (23/15 mil vias). THT pads 10 mil minimum. The 4 mil via ring equals EAGLE `rlMinViaOuter` = 4 mil (restring 25 % of drill, clamped to 4–20 mil).
* Unrouted connections: **0** (`pcb_get_unrouted_nets`, not reanalysed).
* No removed pad shapes (726 pads/vias checked), no invalid regions (19), no mirrored free text (480 checked).
* No component origin outside the board outline (145 checked). No pad or via within 15 mil of the board edge (726 checked).
* **Via antenna:** 1 AVSS via at (1186, 477) mil connects on only one layer. That is harmless at EEG frequencies, but it is a dangling stub (Low, L6).

### 4.2 Polygons

| Layer | Net (count) | Notes |
|---|---|---|
| Top | AVDD, AVSS ×2, VCC_5V_FILTERED, BOARD_ADDR_0, IOREF | Small local pours; EAGLE width 1 mil → H3 |
| MidLayer1 | AGND | Outline 10 mil inside the board edge (EAGLE: 15–3994 × 15–2095 mil vs dimension 5–4005 × 5–2105), wire width 16 mil |
| MidLayer14 | DVDD | Outline 8 mil inside the edge (13–3997 × 13–2097), wire width 10 mil |
| Bottom | AVSS ×3 | 2 in the 1 mil class, 1 in the 8 mil class |

**H4, board-edge clearance.** EAGLE poured these planes with a 10 mil copper-to-dimension rule. In Altium the equivalent rule is 0 mil. How close the poured inner-layer copper gets to the routed edge **was not measured**: the bridge does not expose region geometry, and a repour or Gerber output is outside read-only scope. If the copper follows the outline, it sits roughly 8–10 mil from the edge on **both** inner planes. Some fabs need 10–20 mil, and exposed AGND and DVDD at the same routed edge is a short risk. A DRC with Board Clearance set to 10 mil, or a Gerber check, settles it.

### 4.3 Stored DRC violations

`pcb_get_clearance_violations` (reads stored markers, does not run DRC): **1 violation**.

| Rule | Location | Objects | Assessment |
|---|---|---|---|
| Clearance (Same Net) | (1568, 1440) mil, Bottom | AVSS track, 35 mil × 75.5 mil, vs a Bottom "Polygon Region (0 holes)" with an empty net | The three Bottom polygon regions all report `InPolygon=true` and an empty region net. The only Bottom pours are AVSS, so this is most likely an AVSS track against its own AVSS pour: a same-net artifact, not a short. **Needs confirmation in a fresh DRC.** |

**These results are stale and incomplete.** They come from an earlier run in this Altium session. A fresh DRC was not run because that writes violation objects into the open document, and you required approval for that (H5). With the current rules, expect: about 474 width-maximum hits, testpoint-usage hits on every net, and the same-net hit above.

### 4.4 Silkscreen: defect in output mapping (H1)

| Layer | Content (verified counts) |
|---|---|
| TopOverlay | 148 text objects (designators), **0 tracks, 0 arcs** |
| BottomOverlay | 4 text objects |
| Mechanical7 "tPlace" | 1042 tracks plus arcs, 173 free texts (**board labels**: VREFP, BIASINV, START, SCLK, MISO, MOSI, 5V, GND, Ext. Sync, header pin numbers 0–53, …) and component outlines |
| Mechanical8 "bPlace" | 5765 primitives (bottom artwork and logos), 2 texts |
| Mechanical3 "tDocu" | 243 tracks (documentation, not silkscreen) |

EAGLE tPlace/bPlace **are** the silkscreen layers; the 2014 GTO/GBO Gerbers contain them. A default Altium Gerber setup prints only TopOverlay/BottomOverlay, so the board would be built **without component outlines, pin-1 marks, jumper labels or header numbering**. Silk-to-mask and silk-to-silk DRC rules also check only Overlay layers, so none of this artwork is currently checked against exposed copper.

Fix options (all need approval): (a) in the Gerber/OutJob setup, add Mechanical 7 to the top silk film and Mechanical 8 to the bottom silk film, or (b) move these primitives to the Overlay layers. Option (b) changes the PcbDoc.

### 4.5 Silkscreen and mask DRC coverage

`SilkToSolderMaskClearance` (10 mil, IsPad) and `SilkToSilkClearance` (10 mil) apply only to Overlay primitives. Silk-over-pad was **not verified** for the Mechanical 7/8 artwork.

---

## 5. Footprints, pads, drills and solder mask (tasks 5, 6)

### 5.1 Schematic ↔ PCB footprint and pad consistency: verified

* All 142 schematic components have a footprint name **identical** to their PCB footprint (`sch_fp.json` vs `pcb_get_components`).
* Every schematic pin has a PCB pad, with no duplicate pad names. The only extra PCB pads are the 3 fiducial pads (U$2/U$5/U$6), which have no schematic pin as expected. There are 6 free NPTH mounting holes (MH1–MH6, 126 mil, pad = hole, no copper).
* PCB-only components U$1, U$4, U$7 (logos) are intentional.
* IC9 comment is empty on the PCB but "TXS0102DCUR" in the schematic. Metadata only.

### 5.2 Independent schematic↔PCB connectivity: verified, 0 differences (task 3 of the brief)

Method: Altium's compiled schematic connectivity for **all 142 components** (`proj_get_connectivity_many`, 551 pins) was compared with the live PCB pad nets (`obj_query ePadObject`, 554 component pads). Each net was treated as a set of pads, so names did not matter. Script: `evidence\compare.py`; output: `evidence\compare_output.txt` (re-run from the archived copies).

| Result | Count |
|---|---|
| Schematic nets split across PCB nets | **0** |
| PCB nets merging schematic nets | **0** |
| Pins netted on one side only | **0** |
| Unconnected pins (single-pin schematic net with no PCB net) | IC1-27/29/31/60/64, JP3-5…8 (deliberate, same as EAGLE) |
| **Net-name-only differences** | 94: 14 active-low notation (`!X` ↔ overbar), 20 auto-names (`N$x` ↔ `Net<C>_<p>`), 60 unused header pins |

Correction to the task brief: no Altium netlist **export file** was generated in this session. Both netlists were read live through MCP query tools and saved as JSON under `evidence\`. The earlier session's Protel netlist (`altium_exports\hackeeg-shield_SCH_Protel_final.NET`) reached the same 183/183 result.

### 5.3 Pinouts against manufacturer datasheets: verified

| Ref | Part | Datasheet (revision, page) | Package | Symbol/pad map | Result |
|---|---|---|---|---|---|
| IC1 | ADS1299IPAG | TI SBAS499C, Pin Functions pp. 6–7 | TQFP-64 PAG | 64/64 pin numbers and names | ✅ (one name difference: pin 54 is **AVDD1** in the datasheet, "AVDD" in the symbol; same net, cosmetic) |
| IC2 | TPS72325DBVT | TI SLVS346E, Table 4-1 p. 3 | SOT-23-5 DBV | GND1 IN2 EN3 NR4 OUT5 | ✅ |
| IC3 | TPS73225DBVT | TI SBVS037S, Table 4-1 p. 3 | SOT-23-5 DBV | IN1 GND2 EN3 NR4 OUT5 | ✅ |
| IC6 | TPS60403DBVT | TI SLVS324C, Table 6-1 p. 3 | SOT-23-5 DBV | OUT1 IN2 CFLY−3 GND4 CFLY+5 | ✅ |
| IC7, IC8 | OPA376DBV | TI SBOS406G, Pin Functions p. 3 | SOT-23-5 DBV | OUT1 V−2 +IN3 −IN4 V+5 | ✅ |
| IC9 | TXS0102DCUR | TI SCES640L, Table 4-1 p. 3 | VSSOP-8 DCU | B2 1, GND 2, VCCA 3, A2 4, A1 5, OE 6, VCCB 7, B1 8 | ✅ |
| IC10, IC11 | TXB0108PWR | TI SCES643L, Table 4-1 p. 4 | TSSOP-20 PW | A1…A8 = 1, 3–9; VCCA 2; OE 10; GND 11; B8…B1 = 12–18, 20; VCCB 19 | ✅ |
| IC5 | 24AA256UID (SOIC) | Microchip DS20005215D, Table 2-1 p. 5 | SOIC-8 | A0 1, A1 2, A2 3, VSS 4, SDA 5, SCL 6, **NC 7**, VCC 8 | ✅ (pin 7 is named "WP" in the symbol but is **NC** in the datasheet; it is tied to AGND, which is harmless. L3) |

Circuit-level checks from these datasheets:
* TPS723 EN tied to IN (−5 V): the datasheet describes EN as bipolar, enabled when driven below the negative enable threshold (SLVS346E p. 3). ✅
* TXS0102 OE pulled to VCCA (R4 10 k); internal 10 kΩ pull-ups on both ports (SCES640L §7.3.5 p. 16), so no external I²C pull-ups are needed. ✅
* TXB0108 OE pulled to VCCA (R2/R3). No external pull resistors on its I/Os (TXB0108 needs > 50 kΩ if any are used, SCES643L §7.3.5). Unused channels have both sides tied to GND. ✅

### 5.4 Land-pattern geometry

| Item | Result |
|---|---|
| IC1 TQFP64 vs TI PAG mechanical drawing (SBAS499C p. 81): 0.50 mm pitch, leads 0.17–0.27 mm wide, tip-to-tip 11.8–12.2 mm, body 10 mm | Pads 59 × 11 mil (1.50 × 0.28 mm) at 0.50 mm pitch, row centres 460 mil (11.68 mm) apart, so the pads span 10.18–13.18 mm and cover the full foot length with toe margin. **Consistent.** TI gives no land pattern in this document, so an IPC-7351 check is still open |
| 360 SMD pad sizes, 194 THT drills, 148 THT diameters | Identical to EAGLE (earlier report) |
| **46 EAGLE auto-diameter THT pads** (open item from the earlier report) | Recomputed with the EAGLE DRU restring (0.25 × drill, clamped 4–20 mil): **42/46 match exactly**. JP15 pins 1–4 are 48 mil in Altium vs 41.3 mil computed (larger, 10 mil ring, 31 mil gap at 2 mm pitch). That is the conservative direction; **no action needed** |
| SOT-23-5, VSSOP-8, TSSOP-20, SOIC-8, 0603/0805/1206/1210, connector land patterns vs IPC/manufacturer recommendations | **Not verified** (only pin mapping and EAGLE equivalence were checked) |

### 5.5 Solder mask and paste: not verified (M2) except vias

* **Vias (H2):** `audit_tented_via_ratio` finds **0 of 332 via surfaces tented**. EAGLE `mlViaStopLimit` = 25 mil tents every via with a drill under 25 mil, which is **all 166 vias** (15–24 mil drills). The 2014 production solder-mask Gerber (`camfiles\…v1.3.1.zip`, GTS) has **no via-sized apertures**; its only circles are 52–72 mil THT and 134 mil holes. That independently confirms tented vias were built before. In Altium, 166 via openings would expose via copper beside fine-pitch pads (IC1 0.5 mm, IC10/IC11 TSSOP) and next to sensitive analog nets.
* **Pads:** the MCP pad read reports a mask/paste expansion of 0 on every pad sampled (fiducial, SOT-23, THT). This is most likely the local override field, with the rule values (2 mil / 0 mil) actually in effect, but it **cannot be confirmed** from the bridge. Members of the `SMDSolder_OFF` / `SMDPaste_OFF` classes were not enumerated. The 2014 Gerbers show 4 mil per-side mask expansion, but that is board revision v1.3.1 with a different DRU, so it is not a valid reference for v1.5.0.
* Fiducials: 39 mil copper, rule mask expansion 2 mil. Typical practice is a 2–3× copper mask opening; check this in CAM.
* `MinimumSolderMaskSliver` is disabled.

---

## 6. ADS1299 power, ground and analog inputs (task 2 of the brief)

Source: TI SBAS499C pin table (pp. 6–7) compared with live connectivity.

| Function | Pins | Net | Datasheet requirement | Result |
|---|---|---|---|---|
| AVDD | 19, 21, 22, 56, 59 + AVDD1 54 | AVDD | 1 µF to AVSS (59: to AVSS pin 58) | ✅ connected. Caps: C7/C8, C11/C12, C33 go AVDD–AVSS. **C9/C10 go AVDD–AGND** (L4) |
| AVSS | 20, 23, 32, 57, 58 + AVSS1 53 | AVSS | — | ✅ |
| VREFN / VREFP | 25 / 24 | AVSS / VREFP | ≥10 µF VREFP→VREFN | ✅ C22 10 µF + C23 0.1 µF (VREFN = AVSS) |
| VCAP1–4 | 28, 30, 55, 26 | N$3, N$4, N$5, N$8 | 100 µF / 1 µF / 1 µF ∥ 0.1 µF / 1 µF to AVSS | ✅ C17 100 µF, C18 1 µF, C19 1 µF + C20 0.1 µF, C21 1 µF, all to AVSS |
| DVDD | 48, 50 | DVDD | 1 µF to DGND | ✅ C13 1 µF + C14 0.1 µF to AGND (= DGND) |
| DGND, DAISY_IN | 33, 49, 51 / 41 | AGND | — | ✅ single-ground design; DAISY_IN grounded |
| IN1P…IN8N | 16…1 | AIN1P…AIN8N | — | ✅ all 8 channels via 4.99 kΩ (R34–R49) + 4.7 nF to AGND (C34–C49) from JP16 |
| SRB1 / SRB2 | 17 / 18 | **"SRB2" / "SRB1"** | — | ⚠ names crossed (item A below) |
| RESV1 | 31 | none | **"connect directly to DGND"** | ⚠ floating (item B below) |
| BIASREF | 60 | none | Bias amp non-inverting input | ⚠ floating, valid with BIASREF_INT = 1 (L2) |
| NC / Reserved | 27, 29 / 64 | none | leave open | ✅ |
| GPIO1–3 | 42, 44, 45 | BOARD_ADDR_0–2 | ≥10 kΩ to DGND if unused | ✅ 10 kΩ pull-downs (R19–R21), JP3 straps to DVDD for ADDR_0/1 |
| CLKSEL | 52 | CLKSEL | strap via ≥10 kΩ | ✅ driven from the host through IC10 (not strapped) |
| Supply limits | — | AVSS = −2.5 V in bipolar mode | AVSS to DGND ≥ −3 V (abs. max, p. 7) | ✅ |

### Item A: SRB1/SRB2 names crossed

**Library naming or design defect? Neither. It is a net-label naming inconsistency inherited from the upstream design.**
* The symbol pin names are correct against the datasheet (17 = SRB1, 18 = SRB2), and PCB pads 17/18 carry the same nets as the schematic.
* The net **labelled** "SRB1" goes to chip pin **SRB2**, through JP6 "SRB1-BIAS" pin 2 and JP8 "SRB1-REF_ELEC" pin 2. The net labelled "SRB2" goes to chip pin **SRB1** through JP7 "SRB2-REF_ELEC" pin 3.
* The earlier report showed this is identical in the upstream EAGLE schematic and board, so it was not introduced by the migration.
* **Impact:** connectivity is the tested upstream configuration. The risk is documentation and firmware: anyone using the JP6/JP7/JP8 labels or net names to choose MISC1.SRB1 or CHnSET.SRB2 will select the opposite pin.
* **Design change needed?** No, for reproducing the tested board. Recommended: annotate the documentation (`docs\connectors.md`, `configuration.md`), and optionally rename the nets in a later revision.

### Item B: RESV1 (pin 31) floating

**This is a real datasheet non-compliance, not a library artifact.** SBAS499C p. 6: "RESV1 … Reserved for future use, connect directly to DGND." Pin 31 has no net in the schematic or on the PCB, and it is the same in upstream EAGLE.
* **Design defect?** Yes against the datasheet; the functional impact is unknown. This is the upstream, field-used design, and no failure attributable to it is documented.
* **Design change needed for this build?** Not required to reproduce the tested board. **Recommended for any new revision:** a short trace from pad 31 to the adjacent DGND/AGND pad 33 (or a via to the AGND plane). That is a schematic + PCB change and needs your approval.

### Other ADS1299 observations (upstream design choices, no change required)
* **L2 BIASREF floating:** valid only if firmware sets CONFIG3 bit 3 BIASREF_INT = 1 ("BIASREF signal (AVDD + AVSS)/2 generated internally", SBAS499C p. 48; also p. 68). Confirm in firmware.
* **L4 C9/C10 return to AGND:** in unipolar mode (JP5 ties AVSS to AGND) this is identical to the datasheet. In bipolar mode these two AVDD caps decouple to AGND instead of AVSS (datasheet p. 6). Which IC1 pins they serve was not traced.
* **L5 input filter:** the RC caps go single-ended to AGND. SBAS499C §12.1 p. 72 recommends a differential C0G capacitor across differential inputs. Design choice.

---

## 7. Findings register (task 6)

Severity: **High** = must be resolved or consciously accepted before generating fab outputs. **Medium** = affects documentation, DRC usefulness or assembly. **Low/Info** = observation.

| ID | Sev. | Finding | Evidence | Verified? | Design change needed? |
|---|---|---|---|---|---|
| H1 | High | Silkscreen artwork on Mech 7/8, not Overlay | `obj_count`/`obj_query` per layer; 2014 GTO | Verified | Output mapping (no PcbDoc change) **or** move layers (PcbDoc change) |
| H2 | High | 166/166 vias untented; EAGLE intent tented | `audit_tented_via_ratio`; EAGLE `mlViaStopLimit` 25 mil; 2014 GTS | Verified | Yes: via tenting (via property or mask rule). Or accept untented as a deliberate choice |
| H3 | High | 1 mil thermal-spoke polygon-connect rule; 8 pours likely in that class | Rule descriptor; EAGLE polygon widths | Rule verified; membership inferred | Yes: set the conductor width to ≥ 8–10 mil (or direct connect), then **repour** |
| H4 | High | Board-edge clearance 0 mil (EAGLE 10 mil); inner planes 8–10 mil from edge | Rule; EAGLE polygon outlines | Rule verified; copper distance not measured | Probably: rule to ≥10 mil + repour, after DRC/Gerber confirmation |
| H5 | High | No current DRC; imported rules produce false positives | Stored violations = 1; rule table | Verified | Rule cleanup (M1) + DRC run, both need approval |
| M1 | Medium | Stack missing DVDD–Bottom dielectric (1.30 vs ≈1.53 mm) | `pcb_get_layer_stackup`; EAGLE `mtIsolate` | Verified | Yes (Layer Stack Manager), or specify the stack-up to the fab directly |
| M2 | Medium | Pad mask/paste expansions and class membership unverified | Pad reads report 0; rules 2/0 mil | Unverified | Unknown; verify in CAM |
| M3 | Medium | Width-max 10 mil and testpoint-usage rules are importer defaults | Rule table; 474 tracks > 10 mil | Verified | Rule edit (non-electrical) |
| M4 | Medium | BOM not procurement-ready: no MPN, voltage rating or dielectric (e.g. C17 100 µF 1210, C22 10 µF, C34–C49 4.7 nF) | Component parameters | Verified | Parameter additions (metadata) |
| M5 | Medium | ADS1299 RESV1 floating vs datasheet | SBAS499C p. 6; connectivity | Verified | Optional now; recommended next revision |
| L1 | Low | SRB1/SRB2 net names crossed vs pins | §6 A | Verified | No (documentation) |
| L2 | Low | BIASREF floating, requires BIASREF_INT = 1 | SBAS499C p. 48 | Verified (hardware); firmware not checked | No |
| L3 | Low | 24AA256UID pin 7 is NC per datasheet, labelled "WP", tied to GND | DS20005215D p. 5 | Verified | No |
| L4 | Low | C9/C10 AVDD caps return to AGND | Connectivity; SBAS499C p. 6 | Verified | No (bipolar-mode consideration) |
| L5 | Low | Input caps single-ended, not differential | SBAS499C p. 72 | Verified | No |
| L6 | Low | AVSS via at (1186, 477) connected on one layer only | `audit_find_via_antennas` | Verified | Optional cleanup |
| L7 | Low | Stored same-net clearance hit, AVSS track vs AVSS pour region | `pcb_get_clearance_violations` | Likely benign | Confirm in fresh DRC |
| L8 | Info | 94 net-name-only differences, topology identical | `compare_output.txt` | Verified | No |
| L9 | Info | JP15 THT pads 48 mil vs EAGLE-auto 41.3 mil | §5.4 | Verified | No |
| L10 | Info | JP3 has no strap for BOARD_ADDR_2 (pins 5–8 NC) | Connectivity | Verified (same as upstream) | No |
| L11 | Info | `pcb_get_fab_stats` misreports buried vias and a 2 mil minimum track | §2.1, §4.1 | Verified tool artifact | No |
| L12 | Info | ADS1299 pin 54 named AVDD in the symbol, AVDD1 in the datasheet | SBAS499C p. 6 | Verified | No |

**No connectivity defect, footprint mismatch or pin-mapping error was found.**

---

## 8. Still unverified (needs DRC, CAM output or GUI)

1. Fresh DRC results under a corrected rule set: clearances, hole-to-hole, short circuits, polygon state.
2. Actual poured copper-to-edge distance on the inner AGND and DVDD planes (H4).
3. Membership of the polygon classes (H3) and pad classes `SMDSolder_OFF` / `SMDPaste_OFF` (M2).
4. Effective per-pad solder-mask and paste openings, mask slivers, and fiducial mask openings (Gerber review).
5. Silk-over-pad or silk-over-mask-opening for the Mech 7/8 artwork once it is mapped to silkscreen.
6. Polygon repour state: whether stored pours match the current rules (repour forbidden in this audit).
7. IPC-7351 land-pattern review for SOT-23-5, VSSOP-8, TSSOP-20, SOIC-8, chip passives and connectors.
8. Fab-house capability match: 5/5 mil trace/space, 4 mil via annular ring, 15 mil minimum drill, chosen stack-up. Confirm against the selected fab's published 4-layer limits.
9. Firmware setting CONFIG3.BIASREF_INT and SRB register usage (outside the PCB).

---

## 9. Required actions before manufacturing (all need your approval)

| Step | Action | Type | Changes design? |
|---|---|---|---|
| 1 | **Decide via tenting** (H2). Recommended: tent all vias top and bottom to match EAGLE and the 2014 build | Design property / rule | Yes |
| 2 | **Fix the polygon-connect rule** for the 1 mil class: conductor width ≥ 8–10 mil (H3). Confirm class members first | Rule | Yes |
| 3 | **Set Board Clearance (`mdCopperDimension`) to 10 mil** (H4), matching EAGLE | Rule | Yes |
| 4 | **Repour all polygons** after steps 2–3 | Polygon repour | Yes |
| 5 | Clean up importer rules: remove or relax Width max (e.g. 40 mil) and disable the testpoint-usage rules (M3) | Rule | Yes (non-electrical) |
| 6 | **Correct the stack-up** to the fab's 4-layer build, adding the DVDD–Bottom dielectric (M1) | Layer stack | Yes |
| 7 | **Run DRC** (all rules, report to `C:\EEG\analysis\`) and classify every violation | Analysis output | No design change; writes violation markers |
| 8 | **Map Mech 7 → top silk and Mech 8 → bottom silk** in the Gerber/OutJob setup (H1), or move the artwork to the Overlay layers | Output config, or PcbDoc | Output config: no / move: yes |
| 9 | **Generate Gerber X2 + NC drill into `C:\EEG\analysis\`** (not the project folder) and review: mask openings, silkscreen, copper-to-edge, thermals, drill table | Output | No |
| 10 | Add MPN, voltage rating and dielectric to the BOM (M4) | Parameters | Metadata |
| 11 | Optional, next revision: tie RESV1 (pin 31) to DGND (M5) | Schematic + PCB | Yes |
| 12 | Documentation: note the SRB1/SRB2 label crossing (L1) and the BIASREF_INT = 1 requirement (L2) | Docs | No |

Recommended order: 7 (baseline DRC, read-only apart from violation markers) → 1–6 → 4 → 7 again → 8 → 9 → final review.

---

## 10. Sources

* TI ADS1299 datasheet SBAS499C (Jan 2017): https://www.ti.com/lit/ds/symlink/ads1299.pdf
* TI TPS723 SLVS346E: https://www.ti.com/lit/ds/symlink/tps723.pdf
* TI TPS732 SBVS037S: https://www.ti.com/lit/ds/symlink/tps732.pdf
* TI TPS60403 SLVS324C: https://www.ti.com/lit/ds/symlink/tps60403.pdf
* TI OPA376 SBOS406G: https://www.ti.com/lit/ds/symlink/opa376.pdf
* TI TXS0102 SCES640L: https://www.ti.com/lit/ds/symlink/txs0102.pdf
* TI TXB0108 SCES643L: https://www.ti.com/lit/ds/symlink/txb0108.pdf
* Microchip 24AA256UID DS20005215D: https://ww1.microchip.com/downloads/en/DeviceDoc/24AA256UID-256K-I2C-Serial-EEPROM-with-EUI48-EUI64-20005215D.pdf
* EAGLE source: `hackeeg-shield.brd` (v1.5.0, embedded DRU "MF_Standard_4_Layer"), `stack-up.txt`, `camfiles\advanced-circuits\hackeeg-shield-cam-files-2014-02-17-0758-v1.3.1.zip` (older revision, used only as evidence of build practice)

## 11. Files produced by this audit

| Path | Content |
|---|---|
| `reports\PCB_MANUFACTURING_AUDIT.md` | This report |
| `mfg_audit_2026-10-03\evidence\compare.py`, `compare_output.txt` | Name-independent schematic↔PCB comparison and its output |
| `mfg_audit_2026-10-03\evidence\pcb_pads.json` | 560 PCB pads (component, pad, net, size, hole) |
| `mfg_audit_2026-10-03\evidence\pcb_tracks_all.json` | 3166 tracks (layer, width, net, keepout owner) |
| `mfg_audit_2026-10-03\evidence\sch_connectivity_all_142.json`, `sch_component_info_all.json`, `sch_fp.json` | Schematic pin nets, component metadata, footprints |
| `mfg_audit_2026-10-03\datasheet_text\*.txt` | Text extracted from the 8 datasheets cited above |
