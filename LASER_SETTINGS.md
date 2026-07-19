# Laser Settings — 100W CO2, 3mm Cast Acrylic

Recommended starting settings for cutting a playable record on a **100W CO2 laser** from
**3mm cast acrylic**. These are calibration *starting points*, not guaranteed values — the
groove pass in particular must be dialed in per machine, tube, and lens.

> Read the notebook layer mapping below first, then §6 (**verify physical scale**) before you
> put material in the machine — a wrong-scale disc is unplayable no matter the power settings.

## 1. Layer → operation mapping

`Audio to Vector.ipynb` (cell 28) writes every path onto one of five **color-coded layers**.
Laser software (LightBurn, RDWorks) assigns cut/engrave parameters per color, so these map
straight onto laser operations:

| Layer | Color | Geometry | Operation |
|---|---|---|---|
| `outer_grove` | Cyan | Silent lead-in spiral | **Score** (shallow — do *not* cut through) |
| `audio` | Blue | Audio-modulated spiral (the sound) | **Score** — same as above |
| `inner_grove` | Magenta | Silent + locked inner groove/circle | **Score** — same as above |
| `cut_lines` | Red | Center hole + outer edge | **Cut through** |
| `label` | Green | Label circle + text | **Engrave** (cosmetic) |

**Cyan + blue + magenta are one continuous spiral groove.** The three colors exist only for
data-splitting (`numGroovesPerFile`) and troubleshooting — give them all the *same* groove
setting. So there are really three settings groups: **groove score**, **perimeter cut**,
**label engrave**.

## 2. Starting settings

LightBurn-style units (speed mm/s, power % of the 100W tube). Single pass unless noted.

| Operation | Layers | Speed | Power (Max) | Passes | Air | Focus |
|---|---|---|---|---|---|---|
| **Groove score** | cyan / blue / magenta | ~120 mm/s | ~15% | 1 | low | **defocus ~+2mm** |
| **Perimeter cut** | red | ~20 mm/s | ~65% | 1 | ON | at surface |
| **Label text** (raster/fill) | green | ~300 mm/s | ~20% | 1 | low | at surface |
| **Label circle** (score) | green | ~150 mm/s | ~18% | 1 | low | at surface |

## 3. Why these values — the nuances that actually matter

- **The groove is make-or-break.** The goal is a shallow (~0.1–0.2mm), *consistent* trench the
  stylus rides in — not a cut. Audio is encoded as *lateral* wiggle (the notebook modulates
  the groove's radius), so the depth only needs to keep the stylus seated. Too deep = drag and
  skipping; too shallow = the stylus jumps out.

- **Defocus the groove pass ~1–2mm.** A slightly defocused beam produces a wider, rounded
  U-groove that cradles the stylus far better than a sharp, in-focus V. It also lets you run a
  *stable* ~15% power (comfortably above the tube's ~10–12% firing floor) while staying shallow,
  instead of fighting an unstable near-threshold beam.

- **Keep curve depth consistent (Min Power).** Burn depth ∝ power ÷ speed. On tight audio
  curves the head decelerates, dwells longer, and burns deeper. Use LightBurn's **Min Power**
  and set it a few % *below* Max Power so power scales down with speed; tune Min until
  slow-curve depth matches the straightaways. **Do not set Min = Max** — that makes curves the
  *deepest* part of the groove.

- **Commanded speed ≠ actual speed.** The groove is thousands of tiny segments, so the head
  rarely reaches high commanded speeds — it spends its time accelerating and decelerating. A
  moderate, *sustainable* ~120 mm/s gives more even depth than commanding 300 mm/s the machine
  never actually hits.

- **Perimeter cut.** A 100W tube severs 3mm cast acrylic in one pass around 20 mm/s @ ~65%.
  Using ~65% (not 100%) at moderate speed gives a flame-polished edge with less flare-up. If it
  doesn't fully release, add a second pass or drop 5 mm/s rather than cranking power.

- **Label is cosmetic.** Raster/fill the text (frosts to a clean white on cast acrylic) and
  score the circle. Skip it entirely with `label = False` in cell 6 if you don't want it.

## 4. Cut order (important)

Run operations **inside-out, cutting the perimeter dead last**:

1. Label engrave (green)
2. Groove score (cyan / blue / magenta)
3. Center hole cut (red)
4. **Outer edge cut (red) — LAST**

If the outer edge is cut early, the disc detaches and shifts, misaligning everything cut
after it.

## 5. Material prep & safety

- **Cast, not extruded acrylic.** Cast engraves to a consistent frosty groove and cuts a
  flame-polished edge; extruded engraves gummy and inconsistent — bad for a playable groove.
- **Masking:** peel the mask off the **top** (groove/engrave) face for a clean groove; leave the
  mask on the **bottom** (cut) face to reduce flame staining along the edge.
- **Air assist** ON for the perimeter cut; low for the groove/engrave.
- **Ventilation** — acrylic fumes must be extracted. (Never put PVC in a CO2 laser; it releases
  chlorine gas and corrodes the machine.)

## 6. ⚠️ Verify physical scale BEFORE cutting

`CLAUDE.md` flags an unresolved `SCALE_NUM = 72` pitch/scale issue in the pipeline. After
importing the SVG/DXF, **measure the geometry in your laser software and scale it so the outer
edge = 7.000" and the center hole = 0.286"** before doing anything else. A wrong scale means the
wrong groove pitch and playback timing — an unplayable disc that no laser power setting can fix.

## 7. Calibration procedure

Ties into the project's existing test-tone workflow (`Audio Sample.ipynb` produces an
Ortofon-style sweep):

1. On **scrap 3mm cast**, run a **power × speed grid** for the groove score only (e.g. power
   10–20%, speed 80–160 mm/s), each cell a short arc.
2. Play each arc with your stylus. Pick the **shallowest groove that tracks without skipping** —
   lock that as your groove setting.
3. Cut a full **`frequency_sweep.svg`** disc (the calibration sweep) with the locked groove +
   perimeter settings. Play it to confirm tracking. Tune tone by adjusting `gain_db` /
   `cutoff_freq` **in the notebook**, not by changing laser power.
4. Only then cut a real track.
