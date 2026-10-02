# HackEEG Shield: EAGLE-to-Altium Migration Audit

**Date:** 2026-10-03
**Scope:** Phases 1–4 (inspect, compare, repair plan, verification). **No design files were modified and no ECO was executed.**
**Analysis directory:** `C:\EEG\analysis\` (scripts in `scripts\`, raw extractions in `extract\`, reports in `reports\`, frozen copies in `backup_original\`)

---

## 0. Verdict

| Item | Result |
|---|---|
| EAGLE schematic vs EAGLE board | **Agree completely** (183/183 pad groups; same 145/142 components, values and packages, apart from EAGLE-internal value text) |
| Altium PCB vs EAGLE board | **Identical** at pad level: 145 components, 554 component pads, 187 nets, 183 connectivity groups, footprints, pad positions (≤0.0002 mm), pad sizes, drills, 166 vias, 6 mounting holes |
| Altium schematic vs EAGLE schematic | **One electrical defect.** The 6 ADS1299 AVDD pins (IC1-19, 21, 22, 54, 56, 59) are isolated from the AVDD net. Everything else is equivalent. |
| The 260 Altium differences | All 260 classified. **7 are the AVDD defect + 1 is its side effect (SUPPLY22).** The other 252 are non-electrical. |
| Is the proposed ECO safe? | **No.** Running it would disconnect the ADS1299 analog supply on the PCB. |
| Ready for further PCB validation? | **Not yet.** Ready once repair R1 is done and a re-compare shows zero connectivity differences (see §9). |

---

## 1. Sources and revision (Phase 1)

| Item | Location | Notes |
|---|---|---|
| EAGLE schematic | `hackeeg-shield-master\eagle\hackeeg-shield.sch` | EAGLE 9.5.2 XML, 3 sheets |
| EAGLE board | `hackeeg-shield-master\eagle\hackeeg-shield.brd` | EAGLE 9.5.2 XML, 4 copper layers (Top, AGND, DVDD, Bottom) |
| EAGLE libraries | `eagle\*.lbr` (14 files) | Project libraries. Parts from other libraries (SparkFun, adafruit, supply1/2) are embedded in the .sch/.brd |
| Altium project | `eagle\Imported hackeeg-shield.PrjPcb\hackeeg-shield.PrjPcb` | Text file, parsed directly |
| Altium schematics | `hackeeg-shield_0/1/2.SchDoc` | OLE binary with ASCII records, parsed directly |
| Altium PCB | `hackeeg-shield.PcbDoc` | OLE binary, parsed directly (Components6, Nets6, Pads6, Tracks6, Vias6, Arcs6, Polygons6, Regions6, Rules6, Classes6) |
| ECO report | `hackeeg-shield\Report\hackeeg-shield.PrjPcb And hackeeg-shield.XLS` | Change Order Report, 260 actions (= the "260 differences") |
| Second PcbDoc | `C:\EEG\hackeeg-shield yedek\hackeeg-shield.PcbDoc` | **Not the project document.** See §8 R6 |

**Source revision (verified):** `hackeeg-shield.sch`, `hackeeg-shield.brd` and all 14 `.lbr` files are byte-identical (SHA-256) to upstream `starcat-io/hackeeg-shield` master, commit `10d9b283a34635f8f919961d1702d4b782a48ac9` (2021-06-03, "Merge pull request #7 from starcat-io/fix/jumper-settings"). The local folder has no `.git`, so this identification relies on the hash match. `hackeeg-shield.xyrs` is not in upstream; it is a locally generated file.

**Preservation:** Frozen copies of all EAGLE sources plus a snapshot of the Altium SchDoc/PcbDoc/PrjPcb/LOG/XLS are in `backup_original\`. Baseline hashes are in `reports\SHA256_baseline.txt`. All parsing was read-only.

**Parseability:** All Altium documents could be parsed directly, so no ASCII export was needed for the analysis. §10 lists the exports that would independently confirm the results.

---

## 2. Method, and what counts as verified

* **EAGLE:** connectivity was taken from `<net>/<segment>/<pinref>` and mapped through each device's `<connect gate pin pad>` table to pad level. Board connectivity came from `<signal>/<contactref>`.
* **Altium PCB:** binary pad records (`Pads6`) were decoded for the owning component and net index, then resolved through `Components6` and `Nets6`.
* **Altium schematic:** connectivity was rebuilt **geometrically** from wire vertices, pin hotspots (location + length × orientation), junctions, net labels and power ports. Power ports and net labels were treated as global. That is valid here because the project has **0 ports, 0 sheet entries, 0 sheet symbols and 0 off-sheet connectors** (verified), so Automatic scope resolves to the same result as Global. This independent netlist **reproduces Altium's own ECO exactly** (same 6 pins, same NetIC1_19, same 60 dropped single-pin nets, same 34 renames), which cross-validates the parser.
* Connectivity was compared as **name-independent pad groups** first, and only then by net name. Nothing was inferred from symbol graphics.

---

## 3. Component counts

| Set | Count | Breakdown |
|---|---|---|
| EAGLE schematic parts | 204 | 142 physical + 51 AGND symbols + 7 AVSS symbols (`GND` deviceset, value "AVSS") + 1 AVDD symbol (SUPPLY22, `3.3V` deviceset, value "AVDD") + 3 frames |
| EAGLE board elements | 145 | 142 physical + 3 board-only artwork: U$1, U$4 (OSHW-LOGO-L), U$7 (ENDLESSKNOT-MEDIUM) |
| Altium schematic components | 143 | 142 physical + **SUPPLY22 imported as a real component** ← defect |
| Altium schematic power ports | 58 | 51 AGND + 7 AVSS (matches EAGLE; the AVDD symbol is missing because it became a component) |
| Altium PCB components | 145 | Same 145 designators as the EAGLE board |
| Altium PCB pads | 560 | 554 component pads + 6 free pads MH1–MH6 (= the 6 EAGLE board holes, 3.2 mm) |
| Altium PCB nets | 187 | Same 187 names as the EAGLE board signals |
| Schematic sheets | 3 | `_0` ADS1299/supplies, `_1` bias/reference op-amps, `_2` level shifters/headers |

Unique IDs: all 142 PCB components link (`SOURCEUNIQUEID`) to the matching schematic component. U$1/U$4/U$7 have no schematic counterpart, which is expected for board-only artwork.

---

## 4. Pin-to-net equivalence

Full table: `reports\pin_to_net_equivalence_all_pads.csv` (554 rows). Columns: EAGLE schematic net, EAGLE board net, Altium schematic net, Altium PCB net, status.

| Status | Pads |
|---|---|
| Identical net name in all four sources | 393 |
| Same group; EAGLE `N$x` auto-name becomes Altium `Net<Comp>_<pin>` | 50 |
| Single-pin stub net in EAGLE (unused header pins), dropped from Altium netlist | 60 |
| Same group; active-low `!NAME` becomes Altium overbar `N\A\M\E\` | 33 |
| Deliberately unconnected in EAGLE and Altium | 12 |
| **FAIL: IC1 AVDD pins isolated in Altium schematic** | **6** |

**ADS1299 (IC1), 64 pins** (`reports\ADS1299_IC1_pin_table.csv`):

| Function group | Pins | Result |
|---|---|---|
| Analog inputs IN1P…IN8N → AIN1P…AIN8N | 1–16 | ✅ identical, all 8 channels |
| SRB1/SRB2 | 17, 18 | ✅ identical (see §11, naming observation) |
| **AVDD** | **19, 21, 22, 54, 56, 59** | ❌ **unnamed island in Altium schematic** (PCB is correct) |
| AVSS, AVSS1, VREFN→AVSS | 20, 23, 25, 32, 53, 57, 58 | ✅ |
| VREFP | 24 | ✅ |
| VCAP1–4 (N$3, N$4, N$5, N$8) | 28, 30, 55, 26 | ✅ same groups, auto-named |
| DVDD | 48, 50 | ✅ |
| DGND→AGND, DAISY_IN→AGND | 33, 49, 51, 41 | ✅ |
| SPI: DIN/DOUT/SCLK/!CS | 34, 43, 40, 39 | ✅ (!SPI_CS in overbar notation) |
| !PWDN, !RESET, START, CLK, CLKSEL, !DRDY | 35, 36, 38, 37, 52, 47 | ✅ |
| GPIO1–4 | 42, 44, 45, 46 | ✅ |
| BIASOUT, BIASIN, BIASINV | 63, 62, 61 | ✅ |
| NC, RESV1, BIASREF (unconnected in source) | 27, 29, 64, 31, 60 | ✅ unconnected in all sources |

Bias/reference circuitry (IC7/IC8 OPA376, sheet 1), level shifters (IC9–IC11), EEPROM (IC5), regulators (IC2/IC3/IC6, ±2.5 V), the Arduino headers and the electrode connector all match at the pad-group level.

---

## 5. Footprints and pad mapping

* Schematic footprint links: 142/142 equal to the EAGLE package names. Each component has exactly one PCB model.
* PCB footprint names: 145/145 equal to the EAGLE packages.
* Pad name sets per component: 145/145 identical, with no duplicate pad names.
* Pad positions: 554/554 match EAGLE absolute pad centres, maximum deviation 0.0002 mm (rounding). Board origin is preserved.
* Pad geometry: 360/360 SMD sizes match. 194/194 THT drills match. 148 THT diameters match. 46 THT pads had an automatic diameter in EAGLE (restring from DRC rules), so the fixed Altium value **could not be compared** without the original DRU. See risks.
* Every Altium schematic pin designator exists as a pad on the linked footprint, and the pin→pad assignment equals EAGLE's `<connect>` table.
* Copper: 166 vias (EAGLE 166). Polygons: 11 copper pours with the same nets and layers as EAGLE. The 4 EAGLE `pour="cutout"` polygons (N$12, N$23, N$26, N$55; zero-area, 2 vertices) were imported as no-net cutout regions (KIND=1) on MID1/MID14.

---

## 6. Classification of all 260 differences

Detailed list: `reports\ECO_260_classification.csv`.

| # | ECO section | Count | Category | Electrical? | Action |
|---|---|---|---|---|---|
| A | Remove Pins From Nets (IC1-19/21/22/54/56/59 from AVDD) | 6 | Pin-to-net connectivity | **YES: critical** | **Do not execute.** Fix the schematic (R1) |
| B | Add Nets `NetIC1_19` | 1 | Pin-to-net connectivity | **YES** | Do not execute; disappears after R1 |
| C | Add Components `SUPPLY22` | 1 | Migration artifact (power symbol became a component) | Indirect | Do not execute; disappears after R1 |
| D | Remove Nets: 60 × `N$…` (unused stacking-header pins JP20–JP26) | 60 | Deliberately unconnected pins | No (0 tracks/vias/arcs on these nets; single pad each) | R2 (keep) or accept |
| E | Remove Nets: N$12, N$23, N$26, N$55 | 4 | Migration artifact (EAGLE cutout-polygon signals, no pads) | No | Accept |
| F | Remove Components U$1, U$4, U$7 | 3 | Component identity (PCB-only logos) | No, but destroys artwork | **Do not execute** (R3) |
| G | Change Net Names: 14 × `!X` → overbar | 14 | Net naming | No | Optional (R4) |
| H | Change Net Names: 20 × `N$x` → `Net<C>_<p>` | 20 | Net naming | No | Optional (R4) |
| I | Change Component Comments (U$2/5/6 FIDUCIAL, IC9 TXS0102DCUR) | 4 | Other artifact (EAGLE value text) | No | Optional (R5) |
| J | Change Component Parameters (EAGLE DeviceName/LibraryName/… attributes) | 142 | Other artifact | No | Optional (R5) |
| K | Add Component Classes (one per sheet) | 3 | Other artifact | No | Optional (R5) |
| L | Add Rules: Supply Nets AGND, AVSS | 2 | Other artifact | No | Optional (R5) |
| | **Total** | **260** | | | |

No footprint or pad-mapping differences exist, and no Unique ID mismatches exist for the 142 linked components.

Why D happens: the project has `NetlistSinglePinNets=0` ("Allow Single Pin Nets" off). In EAGLE, each unused header pin has a short wire stub, which makes it a one-pin net. Altium leaves single-pin nets out of the netlist, so the PCB nets look surplus. This is also the source of the single-pin-net ERC warnings.

None of the 34 renames can break PCB rules: all 45 rules are scoped `All`, by pad class or by polygon class. None uses `InNet`/`InNetClass`, and no user net classes exist.

---

## 7. Root cause: why Altium removes six ADS1299 pins from AVDD

**Verified facts:**

1. In EAGLE, net **AVDD** on sheet 1 has 5 segments. Four carry an `AVDD` net label. **Segment 3 has no label.** It contains IC1 pins `AVDD@1…@6` (pads 19, 21, 22, 56, 59, 54) and one supply symbol, **SUPPLY22**: library `SparkFun-Aesthetics`, deviceset **`3.3V`**, value **`AVDD`**. EAGLE joins all segments of a net by **name**, so these pins are on AVDD in EAGLE. The board agrees.
2. The importer turned the 58 `AGND`/`GND`(AVSS) supply symbols into Altium power ports, but turned **SUPPLY22 into an ordinary component**: LibRef `3.3V`, Comment `AVDD`, one Power-type pin named "3.3V" with no designator, and an empty PCB model.
3. Altium does not name a net from a component's power pin. The island therefore has no net identifier. Altium names it **`NetIC1_19`** after its first pin, and this net contains the 6 AVDD pins plus the SUPPLY22 pin.
4. On the PCB, those 6 pads are on **AVDD**, routed with AVDD tracks/vias (78 tracks, 6 vias) and the Top-layer AVDD pour. The ECO therefore proposes removing them from AVDD, creating NetIC1_19 and adding SUPPLY22.

**Consequence if the ECO were executed:** the 6 ADS1299 AVDD pads would move to an unnamed net while still touching AVDD copper. That means clearance/short violations, and on repour the AVDD polygon would pull away from them. Effectively, **the ADS1299 would lose its analog supply connection.**

**Why ERC did not catch it:** the island is a valid, fully wired multi-pin net. It is simply unnamed, and ERC has no rule that says "these pins should be on AVDD". This is a concrete case where **ERC = 0 errors does not mean the design is correct.**

---

## 8. Minimal repair plan (Phase 3), pending your approval

Only **R1** is needed for electrical equivalence. R2–R6 decide how to handle the non-electrical differences. Order: R6 → R1 → re-compare → R2/R3 → re-compare → R4/R5 (optional).

### R1: Restore the AVDD connection in the schematic (CRITICAL)
* **Affected:** `hackeeg-shield_0.SchDoc`, SUPPLY22, IC1 pins 19/21/22/54/56/59, net AVDD.
* **Change:** Delete component SUPPLY22. At its pin hotspot, place a **Power Port** with Net = `AVDD` (any style). The hotspot is at X = 1129, Y = 1045 in sheet units of 10 mil, i.e. **11290 mil, 10450 mil**. As an alternative, place a **Net Label `AVDD`** on any wire of that island; this matches how the other four EAGLE AVDD segments are drawn. Either one alone is enough.
* **Justification:** reproduces EAGLE's name-based joining of segment 3 to AVDD (§7). It adds no connection that does not exist in the EAGLE schematic and board.
* **Do not** use text "3.3V" for the port. That would create a separate global net "3.3V" and leave AVDD split.
* **Expected outcome:** Altium re-compare shows no "Remove Pins From Nets", no "Add Nets NetIC1_19" and no "Add Components SUPPLY22". The total drops from 260 to 252, and none of the remaining 252 is electrical.

### R2: Single-pin stub nets (choose one)
* **R2a (closest to the source, recommended):** Project → Project Options → Options → enable **Allow Single Pin Nets**. The 60 stubs then appear as nets, and the "Remove Nets" items turn into renames (e.g. `N$6 → NetJP20_9`).
* **R2b:** accept removal of the 60 PCB nets. This is electrically neutral: each net has one pad and 0 copper primitives.
* N$12/N$23/N$26/N$55 (no pads, no copper) are removed in both options.

### R3: Keep the PCB-only artwork U$1, U$4, U$7
* Untick these three in the ECO dialog. To make this permanent, set Project Options → Comparator → "Extra Components" to Ignore Differences.
* **Justification:** these are logos (2× OSHW, 1× endless knot) that exist only on the EAGLE board.

### R4: Net names (optional, cosmetic)
* Accept the 34 renames, or leave them. Active-low names follow Altium overbar notation. `N$` nets get Altium auto-names. If you want the EAGLE names kept on the PCB, place net labels with the `N$x` names in the schematic. This is not needed for correctness.

### R5: Metadata (optional)
* 4 comment changes, 142 parameter changes (EAGLE `DeviceName`, `LibraryName`, … attributes), 3 per-sheet component classes and 2 Supply Nets rules. All are non-electrical, so apply or ignore as you prefer. Applying them after R1 is harmless.

### R6: Settle which PcbDoc is the reference (before anything else)
* `C:\EEG\hackeeg-shield yedek\hackeeg-shield.PcbDoc` (saved 21:37) differs from the project PcbDoc (21:21). Connectivity is identical, but U$1/U$4/U$7 have **lost their designators** and 142 components have different source-library-reference fields. That looks like a partially updated copy.
* **Recommendation:** keep the project PcbDoc as the reference (it matches EAGLE exactly) and do not swap in the "yedek" file. Make a new, clearly named backup before running R1.

---

## 9. Verification status (Phase 4)

### 9.1 Verified facts
* Local EAGLE files = upstream master `10d9b28` (hash match).
* EAGLE schematic ≡ EAGLE board (connectivity, components, packages).
* Altium PCB ≡ EAGLE board (components, footprints, pads, pad positions/sizes/drills, nets, vias, holes, pour nets/layers).
* Altium schematic ≡ EAGLE schematic **except** the AVDD island (6 pins).
* All 260 ECO actions are classified, and the classification is complete.
* No PCB rule references a net name.
* The single No-ERC directive sits exactly on **IC1-60 BIASREF** (sheet 1), which is unconnected in the source design. The directive is consistent with the source.

### 9.2 Unresolved connectivity differences
* **1:** AVDD island (IC1-19/21/22/54/56/59). This is fixed by R1 and still needs a GUI re-compare to confirm.

### 9.3 Critical errors
* **The proposed ECO must not be executed as is.** Items A/B/C break the ADS1299 supply, and item F deletes board artwork.

### 9.4 Remaining risks and assumptions
* **Assumption:** the 83 ERC warnings are mostly the 60 single-pin stub nets plus importer-generated warnings. The ERC list was not available to check this (§10).
* **Not compared:** 46 THT pad diameters that EAGLE sizes automatically, and solder mask/paste expansions. EAGLE took these from DRU rules at CAM time, while Altium uses fixed values or Altium rules. Compare Gerbers before fabrication.
* **Not compared:** silkscreen, board outline, inner-plane pour parameters (isolate, thermals), polygon pour results after repour, and the layer stack/dielectrics (`eagle\stack-up.txt` vs Altium Layer Stack Manager).
* The importer created 30 mid-layers and 16 plane layers in the layer table. Only 4 copper layers are used, so remove the unused ones in the Layer Stack Manager before DRC/CAM.
* Altium connectivity was rebuilt by geometry. Because it reproduces Altium's own ECO exactly, confidence is high, but it is not a substitute for the Altium netlist export in §10.

### 9.5 Readiness
**Not ready yet.** The design is ready for further PCB validation (DRC, Gerber-vs-EAGLE-CAM comparison) once:
1. R6 and R1 are done,
2. Design → Compile shows no NetIC1_19,
3. a new Project → Show Differences / ECO shows **zero** connectivity items (no "Remove Pins From Nets", no "Add Nets"), and
4. the §10 Protel netlist confirms AVDD has 21 pads including IC1-19/21/22/54/56/59.

---

## 10. Altium GUI operations and exports

**Repair (after approval):**
1. Close the project. Copy the whole `Imported hackeeg-shield.PrjPcb` folder to a dated backup.
2. Open `hackeeg-shield_0.SchDoc`. Find SUPPLY22 (Edit → Find Similar, or the Navigator panel). Delete it.
3. Place → Power Port. In Properties set Net = `AVDD`. Place it on the former SUPPLY22 pin hotspot (11290 mil, 10450 mil) so it touches the island wire. Save.
4. Project → Project Options → Options: decide R2a (Allow Single Pin Nets). Comparator: decide R3 (Extra Components → Ignore).
5. Project → Compile PCB Project. Run Project → Validate (ERC). Navigator: confirm IC1 pin 19 is on **AVDD**.
6. Design → Update PCB Document. In the ECO dialog, **Validate Changes only**. Expect no "Remove Pins From Nets" and no "Add Nets". Untick U$1/U$4/U$7 if they are still listed. Execute only after reviewing.

**Exports that independently confirm this audit (save into `C:\EEG\analysis\altium_exports\`):**

| Export | How | Purpose |
|---|---|---|
| Schematic netlist | Design → Netlist For Project → **Protel** (`.NET`) | Pad-level SCH netlist; diff against `extract\eagle_sch.json` |
| PCB netlist | In PCB: Design → Netlist → Export Netlist from PCB (`.Net`) | Pad-level PCB netlist |
| ERC report | Messages panel → right click → Export (or Reports → ERC report) | Classify the 83 warnings |
| Comparison report | Project → Show Differences → Report (after R1) | Confirm zero connectivity items |
| Gerber + NC drill (Altium) | Fabrication Outputs | Compare with EAGLE CAM (`cam\hackeeg-shield.cam`) or the 2014 Advanced Circuits zip in `eagle\camfiles\` |

---

## 11. Source-design observations (outside migration scope; do not change during migration)

These belong to the original design and were carried over faithfully. They are listed only so they are not mistaken for migration errors later.
* **SRB net naming is crossed relative to the chip pins:** ADS1299 pin 17 (SRB1) is on net **"SRB2"**, and pin 18 (SRB2) is on net **"SRB1"**. Connectivity is preserved exactly; only the labels are swapped. Check against `docs\connectors.md` before using the net names in documentation.
* **IC1-31 RESV1 is unconnected.** The TI pin table describes RESV1 as reserved and to be connected to DGND. This is the tested upstream design, so do not change it; just keep it in mind for a future revision.
* **IC1-60 BIASREF is unconnected.** This is valid when the internal bias reference is selected (CONFIG3.BIASREF_INT = 1 in firmware). The firmware setting was not checked here.
* DGND and DAISY_IN are tied to **AGND** (single-ground design), and VREFN is tied to AVSS. All three are preserved.
* EAGLE values differ between the schematic and the board for IC7/IC8 (empty vs "OPA376DBV") and for the fiducials. This is EAGLE-internal text only.

---

## 12. File index (`C:\EEG\analysis\`)

| Path | Content |
|---|---|
| `reports\VERIFICATION_REPORT.md` | This report |
| `reports\ECO_260_classification.csv` | All 260 ECO actions with category, severity and recommendation |
| `reports\pin_to_net_equivalence_all_pads.csv` | 554 pads × 4 sources |
| `reports\ADS1299_IC1_pin_table.csv` | IC1 64-pin table |
| `reports\SHA256_baseline.txt` | Hashes of the frozen originals |
| `backup_original\` | Frozen copies of EAGLE sources and an Altium snapshot |
| `extract\` | JSON/TSV extractions (EAGLE sch/brd, Altium sch records/netlist, PCB, ECO) |
| `scripts\` | Read-only analysis scripts (`eagle_extract.py`, `eagle_compare.py`, `altium_pcb_extract.py`, `compare_pcb.py`, `pad_geometry.py`, `pad_size_check.py`, `altium_sch_records.py`, `altium_sch_netlist.py`, `compare_sch.py`, `compare_values_fp.py`, `pcb_copper_by_net.py`, `full_pin_table.py`, `ic1_table.py`, `eco_classify.py`) |

Re-run order: `eagle_extract.py → altium_pcb_extract.py → altium_sch_records.py → altium_sch_netlist.py → compare_*.py → full_pin_table.py → eco_classify.py` (run from `scripts\`, Python 3 + `olefile`, `xlrd`).
