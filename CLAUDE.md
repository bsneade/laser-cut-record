# CLAUDE.md

## Overview

This project turns an audio file into a **laser-cuttable spiral groove** that can be
etched onto a flat disc to produce a physically playable record. It is a Python/Jupyter
reimplementation of the Instructables ["Laser Cut Record"](https://www.instructables.com/Laser-Cut-Record/)
project, aiming to make that workflow more "push button."

The audio is filtered to mimic vinyl, then mapped onto a decreasing-radius spiral where
the instantaneous audio amplitude modulates the groove's radius. The result is exported
as **SVG** (for the laser cutter), **DXF**, and **PNG** (for reference).

## Repository layout

- `Audio Sample.ipynb` — generates a calibration test tone (a frequency sweep). Use it to
  produce a consistent source for dialing in laser settings, like an Ortofon test record.
- `Audio to Vector.ipynb` — the main pipeline: audio file → processed audio → spiral groove
  → SVG/DXF/PNG.
- `frequency_sweep.{wav,svg,dxf,png}` — committed example artifacts produced by the notebooks.
- `frequency_sweep_{riaa,lowpass,gain,waveform}.wav` — intermediate audio dumps from each
  processing stage (saved when `save_audio = True`).
- `old/` — an earlier render of an "Around The World" track (`.wav/.dxf/.svg/.jpg`).
- `README.md` — links to run `Audio Sample.ipynb` on Google Colab (Colab badge).

Outputs are **committed artifacts, not source** — the notebooks are the source of truth.

## Workflow

1. **(Optional) Make a test file:** run `Audio Sample.ipynb` to write `frequency_sweep.wav`
   — a logarithmic sweep used to calibrate the laser.
2. **Generate the groove:** set `fileName` in `Audio to Vector.ipynb`, run all cells, and it
   writes `<name>.svg`, `<name>.dxf`, and `<name>.png`.
3. **Cut:** send the SVG to the laser cutter. Output is split into multiple polylines
   (see `numGroovesPerFile`) so the laser isn't fed too much data at once.

## Audio → Vector pipeline (`Audio to Vector.ipynb`)

Built on `torchaudio` (audio processing) and `ezdxf` (vector/DXF generation):

1. **Load** the wav (`torchaudio.load`).
2. **Stereo → mono** (`torch.mean` across channels) — only one channel is cut.
3. **RIAA filter** (`riaa_biquad`) — applies the vinyl RIAA equalization curve.
4. **Low-pass filter** (`lowpass_biquad`, `cutoff_freq`) — caps frequency to what the laser
   can resolve without melting the material.
5. **Gain** (`gain_db`).
6. **Normalize** volume to `amplitude_s = amplitude / dpi * SCALE_NUM`.
7. **Generate spiral:** audio modulates groove radius (`radCalc = radius + audio_data[index]`);
   radius decreases each angular step to form the spiral. The disc is built as:
   silent outer groove → audio grooves → silent inner locked groove, plus center hole,
   outer cut line, and a text label.
8. **Export** PNG (reference), SVG (for cutting), and DXF.

`indexIncr = samplingRate * 60 / rpm / THETA_COUNT` controls how many audio samples are
consumed per angular step (i.e. the playback timing).

## Key parameters

All live in the Configuration / Constants cells of `Audio to Vector.ipynb`.

### Calculation
| Param | Default | Meaning |
| --- | --- | --- |
| `samplingRate` | 44100 | Sample rate of the incoming audio (Hz) |
| `rpm` | 45 | Playback speed (33.3 / 45 / 78) |
| `amplitude` | 10 | Groove amplitude, in pixels |
| `minDist` | 6.0 | Min pixel spacing between path points (prevents cutter stalling) |
| `spacing` | 10 | Starting pitch — space between grooves (pixels) |

### Geometry (inches)
| Param | Default | Meaning |
| --- | --- | --- |
| `innerHole` | 0.286 | Center hole diameter |
| `diameter` | 7 | Record diameter |
| `outerRad` | 3.4 | Radius of outermost groove (`6.8 / 2`) |
| `innerRad` | 2.25 | Radius of innermost groove |
| `labelRad` | 2 | Label radius |

### Laser (tune per cutter)
| Param | Default | Meaning |
| --- | --- | --- |
| `dpi` | 1200 | Cutter resolution |
| `numGroovesPerFile` | 20 | Grooves per output polyline/file split (lower = less data per send) |
| `cutoff_freq` | 2500 | Low-pass cutoff (Hz) — set to the max your laser resolves without melting |
| `gain_db` | -1.0 | Pre-amplification; raise as high as possible without audible clipping |

### Constants
| Param | Default | Meaning |
| --- | --- | --- |
| `SCALE_NUM` | 72 | Inch → ezdxf-unit scale factor (see gotcha below) |
| `SEC_PER_MINUTE` | 60 | — |
| `THETA_COUNT` | 5880 | Points per revolution |

When adapting to a new laser, the parameters most likely to change are `cutoff_freq`,
`dpi`, `gain_db`, `numGroovesPerFile`, and `rpm`.

## Running / environment

- Runs in **Jupyter** locally or on **Google Colab**.
- The notebooks `pip install` their non-standard deps inline: `ezdxf` (Audio to Vector) and
  `python-sound` (Audio Sample).
- Stack: `numpy`, `scipy`, `torch`, `torchaudio`, `ezdxf`, `matplotlib`.
- **NumPy caveat:** the environment must use a NumPy version compatible with SciPy. A saved
  run of `Audio Sample.ipynb` shows SciPy failing to import under NumPy 2.x — pin
  `numpy<2` (or rebuild the affected modules) if you hit `_ARRAY_API not found`.
- Set `display_plots = False` for faster runs (skips intermediate waveform plots);
  `save_audio` toggles writing the per-stage `_riaa/_lowpass/_gain/_waveform` wavs.

## Conventions & gotchas

- **Input must be stereo.** The vector script does not support single-channel wav, which is
  why `Audio Sample.ipynb` pads a second all-zeros channel.
- **Open tuning issue — pitch/scale.** `SCALE_NUM` is set to `72` with a `#25.4` comment
  (points vs mm ambiguity). The most recent commit was still "playing with the scale factor
  to try to fix the pitch calculation," so groove pitch/scaling is not yet finalized — treat
  it as a known WIP when touching geometry math.
- Outputs are split into `numGroovesPerFile`-sized polylines specifically because many lasers
  choke on a single huge path.
- The PNG export is reference-only and not usable by the laser cutter; cut from the SVG.
