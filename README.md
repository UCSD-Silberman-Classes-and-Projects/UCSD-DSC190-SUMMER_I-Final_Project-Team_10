# DSC190 Final Project — Team 10

**Line following and lane following for a physical DonkeyCar, using classical computer vision.**

**UCSD DSC190, Summer Session I 2026 — Track 2: Agentic Development, New DonkeyCar Capabilities**
**Team:** Leo Okdemir (HDSI) · Talal Jeddawi (CSE)

This repository adds two autonomous driving behaviors to a DonkeyCar equipped with a
Luxonis OAK-D camera. Both are built with OpenCV only — **no neural networks, no
training data, no simulator**. The car finds yellow tape on the track by color and
shape, decides where it wants to be, and steers there. Every threshold and gain is a
named constant that can be changed in `myconfig.py` and re-tested at the track without
touching code.

The two behaviors are:

- **Line following** — follow a single strip of yellow tape.
- **Lane following** — drive a chosen lane (left or right) between two yellow boundary
  lines and a dashed yellow center divider. Switch mode and lane live from a web page
  while the car is driving.

This is also a Track 2 project, meaning the _process_ was part of the assignment: the
code was written by an AI coding agent (Claude) driven in a
`mission → agent writes code → test on the track → feedback → agent revises` loop. See
[Development process](#development-process-the-agentic-loop) below.

Deep technical documentation lives in **[docs/lane_following.md](docs/lane_following.md)** —
how each pipeline works, the camera patch, the debug overlay legend, the tuning guide,
and a failure-mode table. This README covers what the project is, how to run it, and
what we wrote.

---

## Mission progress

| Mission stage              | Status         | Notes                                                                                                                                                                                                                                                                                                  |
| -------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Line following          | ✅ Working     | Follows a single strip of yellow tape. Centroid tracking, with a "hold last heading" fallback when the line is briefly lost.                                                                                                                                                                           |
| 2. Lane following          | ✅ Working     | Drives a chosen lane between two yellow boundaries and a discontinuous center divider; mode and lane switch live from a web toggle page.                                                                                                                                                               |
| 3. Two-way road navigation | ⏳ Not reached | We ran out of runway before starting oncoming-robot detection and avoidance. An earlier "drive in your half of the road" iteration (see [Development process](#development-process-the-agentic-loop)) was exploratory groundwork toward this stage, but was superseded by the lane-following approach. |

---

## Hardware and requirements

This is what the car we built and tested on actually runs. Other configurations are
possible — everything is config-driven — but these are the values in
[donkeycar/mycar/myconfig.py](donkeycar/mycar/myconfig.py).

| Component      | What we used                                                                                      | Config                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Compute        | Raspberry Pi, Python 3.11 in a virtualenv at `~/env`                                              | —                                                                                               |
| Camera         | **Luxonis OAK-D** over USB, color only (depth costs Pi CPU and USB bandwidth and we don't use it) | `CAMERA_TYPE = "OAKD"`, `OAKD_RGB = True`, `OAKD_DEPTH = False`, `OAKD_ID = None` (auto-detect) |
| Resolution     | 426x240 at 20 fps — small enough for the Pi to keep up, wide enough to see the near ground        | `IMAGE_W = 426`, `IMAGE_H = 240`                                                                |
| Drivetrain     | VESC over USB serial                                                                              | `DRIVE_TRAIN_TYPE = "VESC"`, `VESC_MAX_SPEED_PERCENT = .35`                                     |
| Manual control | Logitech F710 gamepad on `/dev/input/js0`                                                         | `USE_JOYSTICK_AS_DEFAULT = True`, `JOYSTICK_MAX_THROTTLE = 0.35`                                |
| Track          | Yellow tape on pavement                                                                           | tuned via the HSV constants                                                                     |

Software: the [Donkeycar](https://docs.donkeycar.com) framework (this repo is a fork of
it), OpenCV, NumPy, DepthAI, and Tornado. `pytest` is needed only to run the tests.

The vehicle loop runs at **20 Hz**, so a button press on the toggle page takes effect on
the next frame — about 50 ms.

---

## Setup

### 1. Install Donkeycar

Follow the [upstream install guide](https://docs.donkeycar.com/guide/install_software/)
(the original project README is preserved here as
[README_donkeycar.md](README_donkeycar.md)). We used a Python 3.11 virtualenv at `~/env`.

### 2. Apply the OAK-D full-FOV camera patch

**This step is specific to this project and it matters.** The stock `OakD` camera part
streams the camera's `video` output, which is hard-cropped to 16:9 regardless of the
resolution you request. That crop removes the near-ground strip of image — exactly the
part the line follower needs to see on turns. Our patched part streams the full-FOV
`isp` output instead.

If Donkeycar is installed **editable** (`pip install -e`), the patched
[donkeycar/parts/oak_d.py](donkeycar/parts/oak_d.py) in this repo is already in use and
you can skip ahead. Otherwise, copy it into the installed package:

```bash
~/env/bin/python scripts/patch_oak_d.py            # apply (idempotent, keeps a .bak)
~/env/bin/python scripts/patch_oak_d.py --status   # is it applied?
~/env/bin/python scripts/patch_oak_d.py --dry-run  # show what would change
~/env/bin/python scripts/patch_oak_d.py --revert   # restore the original
```

Use the **same interpreter that runs `manage.py`**, or you will patch a package the car
never imports. **Re-apply after any environment rebuild or Donkeycar reinstall.**

Background and the exact edits: [docs/lane_following.md § Step 0](docs/lane_following.md).

### 3. Verify the camera

```bash
python scripts/oakd_color_check.py
```

This confirms the camera streams 426x240 with the recovered field of view, and settles
whether frames arrive as BGR or RGB — it writes _the same frame_ twice, as
`oakd_as_bgr.png` and `oakd_as_rgb.png`. Open both, pick the one where the yellow tape
is actually yellow, and set `CAMERA_COLOR_ORDER` accordingly. Don't guess this; a
channel swap turns yellow into blue and nothing detects.

---

## Running it

The car directory is **[donkeycar/mycar/](donkeycar/mycar/)** — it holds `manage.py` and
the tuned `myconfig.py`. Run from there:

```bash
cd donkeycar/mycar
python manage.py drive
```

Then open the web pages:

| URL                            | What it is                                                                                                                |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `http://<hostname>.local:8887` | The stock Donkeycar driving page. **This is where you engage autopilot** — set the mode selector to `local`.              |
| `http://<hostname>.local:8891` | Our toggle page: live camera feed, MODE (line/lane), LANE (left/right), DEBUG. Only served when `LANE_WEB_ENABLE = True`. |

The car starts in **user** (manual) mode and does nothing autonomous until you switch to
`local` on the 8887 page. This is enforced structurally, not by convention: the CV part
is added to the vehicle loop with `run_condition="run_pilot"`, so in manual mode it does
not execute at all. Our preflight script asserts exactly this (see
[Verifying without the car](#verifying-without-the-car)).

### Choosing which autopilot runs

The repo contains two working CV autopilots, selected by two config keys. The deployed
car currently runs the **line follower**:

```python
# donkeycar/mycar/myconfig.py — as shipped
CV_CONTROLLER_MODULE = "donkeycar.parts.line_follower"
CV_CONTROLLER_CLASS  = "LineFollower"
```

To run the **lane-following package** instead, with the toggle page:

```python
CV_CONTROLLER_MODULE = "donkeycar.parts.lane_following.controller"
CV_CONTROLLER_CLASS  = "LaneFollowingController"
START_MODE      = "lane"     # or "line"
START_LANE      = "left"     # or "right"
LANE_WEB_ENABLE = True       # serve the toggle page
LANE_WEB_PORT   = 8891       # 8887 is taken by the Donkeycar page
DEBUG_OVERLAY   = False      # True to start with the overlay on
```

Exact config blocks for each mode: [docs/lane_following.md § Running it](docs/lane_following.md).

### What to expect

On `local`, the car drives forward at `THROTTLE_FORWARD` and steers to keep its target
centered. When the line leaves view it **holds its heading**, coasts, slows, and stops —
it does not hunt around looking for something else to follow. Each transition is logged
to the console (`tracking`, `coasting`, `slowing`, `stopped`).

Turning **DEBUG** on draws the vision overlay onto the camera feed: green tint for
pixels the color threshold accepted, white for pixels that survived the shape filters
(what steering actually uses), the ROI trapezoid, frame center, and the steering target.
Comparing green against white is the fastest diagnosis available — lots of green with
little white means the color threshold is too loose. Full legend:
[docs/lane_following.md § Debug mode](docs/lane_following.md).

---

## Programs we wrote

This repository is a fork of [autorope/donkeycar](https://github.com/autorope/donkeycar).
Everything listed below is our work — either new files, or upstream files we
substantially rewrote. Everything else in the tree is upstream Donkeycar.

### The lane-following package — `donkeycar/parts/lane_following/`

A new, self-contained package. Vision knows nothing about steering, control knows
nothing about pixels, and the strategies join them.

| File                                                          | What it does                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [params.py](donkeycar/parts/lane_following/params.py)         | Every tunable constant in one labeled block, plus a `Params` class that overlays values from `myconfig.py` and validates them at startup (bad HSV ranges, impossible ROI fractions, and out-of-range gains raise immediately instead of misbehaving on the track).                                                                   |
| [state.py](donkeycar/parts/lane_following/state.py)           | Thread-safe `Mode` / `Lane` / debug state shared between the web page and the driving loop, as a lock-guarded process-wide singleton with atomic snapshot reads.                                                                                                                                                                     |
| [vision.py](donkeycar/parts/lane_following/vision.py)         | ROI cropping, HSV masking, morphological cleanup, and rotation-invariant blob shape filtering (rectangle fill, solidity, aspect ratio). Also clusters vertically-stacked blobs so a **dashed** divider reads as one line.                                                                                                            |
| [control.py](donkeycar/parts/lane_following/control.py)       | The plausibility gate (rejects target jumps too large to be real, and requires a re-acquired line to confirm over several frames), the proportional + smoothed steering controller, and the lost-line state machine `TRACKING → COASTING → SLOWING → STOPPED`.                                                                       |
| [strategies.py](donkeycar/parts/lane_following/strategies.py) | The two driving strategies. Line following aims at the area-weighted centroid. Lane following runs a `LaneModel` that tracks left boundary / divider / right boundary across frames and computes a lane center through a three-tier fallback: bracketing pair → single boundary plus a live-measured half-lane width → hold heading. |
| [controller.py](donkeycar/parts/lane_following/controller.py) | The Donkeycar Parts. `LaneFollowingController` is the autopilot; `PassThroughController` is a wiring-check part that returns zero steering and throttle, used to prove the plumbing before adding any behavior.                                                                                                                      |
| [web.py](donkeycar/parts/lane_following/web.py)               | The toggle page: a Tornado server with an inline single-page UI, an MJPEG feed at `/video`, and a small JSON API (`/api/state`, `/api/mode`, `/api/lane`, `/api/debug`). It test-binds its port on the main thread at construction, so a port clash is a loud startup error instead of a silently dead thread.                       |
| [overlay.py](donkeycar/parts/lane_following/overlay.py)       | The debug overlay — raw vs. accepted mask, ROI, frame center, steering target, identified lane lines, and a text panel. Draws on a copy of the frame, never the original.                                                                                                                                                            |

### Donkeycar parts we rewrote

| File                                                                 | What we changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [donkeycar/parts/line_follower.py](donkeycar/parts/line_follower.py) | Kept the upstream class and API, rewrote the internals: centroid tracking instead of brightest-column `argmax`; a target that defaults to image center instead of latching onto whatever frame 1 saw; a real confidence _fraction_ (upstream returned a 255-weighted sum, so the confidence threshold could never fail); side-margin masking; speck removal; smoothing; hold-then-decay when the line is lost; progressive steering for corners; and speed-scaled steering so lateral response stays constant as throttle changes. Also fixes an upstream bug where a `None` frame returned four values into a three-output part. |
| [donkeycar/parts/oak_d.py](donkeycar/parts/oak_d.py)                 | The full-FOV patch: links the camera's `isp` output instead of the 16:9-cropped `video` output; auto-probes sensor capture modes (12 MP / 13 MP) and retries rather than failing on one hard-coded guess; **validates capture by pulling a real frame and converting it**, not merely by opening the device (an earlier check only confirmed the device opened, which was masking real failures); optional auto-exposure compensation for bright outdoor track lighting; and creates output queues once instead of per frame.                                                                                                     |

### Scripts

Four scripts in [scripts/](scripts/) are ours. The other sixteen files there are
upstream Donkeycar utilities.

| Script                                                                     | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [scripts/patch_oak_d.py](scripts/patch_oak_d.py)                           | Applies this repo's patched `oak_d.py` to a non-editable installed Donkeycar. Idempotent, detects its own patch state, keeps a `.bak`. `--status`, `--dry-run`, `--revert`.                                                                                                                                                                                                                                                                |
| [scripts/oakd_color_check.py](scripts/oakd_color_check.py)                 | Camera sanity check: frame size, recovered field of view, and BGR-vs-RGB settled by writing the same frame both ways. `--out-dir`, `--warmup`. Requires the camera.                                                                                                                                                                                                                                                                        |
| [scripts/preflight_lane_following.py](scripts/preflight_lane_following.py) | **Runs the real vehicle loop with no hardware.** Builds a throwaway car directory from the real template with mock camera, drivetrain, and controller, then drives it twice — asserting the CV part does _not_ run in user mode and _does_ run in local mode. Catches import errors, wrong `run()` signatures, bad wiring, and missing config keys before you take the car outside. `--controller`, `--loops`, `--web`, `--debug-overlay`. |
| [scripts/replay_frames.py](scripts/replay_frames.py)                       | Regression harness. Replays frames through the line-following strategy and records or compares per-frame steering, throttle, state, offset, and blob counts. With no frame directory it generates a deterministic 78-frame synthetic sequence that walks acquire → track → drift → loss → coast → slow → stop → re-acquire. `--frames`, `--record`, `--check`, `--capture`.                                                                |

### Tests

| File                                                                 | Coverage                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [tests/test_lane_following.py](tests/test_lane_following.py)         | 41 tests on synthetic frames, no hardware: detection and steering direction, foliage rejection, the plausibility gate, the lost-line state machine, robustness to blank and odd-sized frames, the shared state object, lane classification (dashed divider clustering, left/right lane targeting, single-boundary fallback, stale-width hold), and controller integration including mid-drive mode switches. |
| [tests/test_lane_following_web.py](tests/test_lane_following_web.py) | 14 tests against a **real Tornado server** on a free port: port-conflict handling, the JSON API, rejection of malformed input, MJPEG streaming, frame caching, and concurrent frame writes and reads.                                                                                                                                                                                                        |
| [tests/data/line_golden.json](tests/data/line_golden.json)           | A 78-frame golden baseline recorded before the pipeline was refactored, used to prove line following came out byte-identical afterward.                                                                                                                                                                                                                                                                      |

### The car and the docs

| Path                                                                                                                               | What it is                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [donkeycar/mycar/](donkeycar/mycar/)                                                                                               | The actual deployed car directory: `manage.py` (the `cv_control` template plus a manual-mode preview hook and PID output limits) and a heavily annotated, track-tuned `myconfig.py`. |
| [donkeycar/templates/cv_control.py](donkeycar/templates/cv_control.py), [cfg_cv_control.py](donkeycar/templates/cfg_cv_control.py) | Template edits: OAK-D defaults, the lane-following controller wiring, and the optional toggle-page part.                                                                             |
| [docs/lane_following.md](docs/lane_following.md)                                                                                   | The full technical write-up — pipeline internals, the camera patch, the five foliage defenses, the tuning guide, and a failure-mode table.                                           |

---

## Configuration and tuning

Every `UPPER_CASE` constant in
[params.py](donkeycar/parts/lane_following/params.py) is read as
`getattr(cfg, NAME, default)`. That means:

- **Any** of them can be overridden in `myconfig.py` using the exact same name.
- Anything you _don't_ mention keeps its default, so an old or partial config can never
  crash the car with a missing key.
- The overrides actually in effect are logged at startup, so you can confirm your edit
  was picked up.

The tunables cover: yellow color separation (`YELLOW_HSV_LOW` / `YELLOW_HSV_HIGH`),
the region of interest (`ROI_TOP_FRAC`, `ROI_BOTTOM_FRAC`, `ROI_TOP_WIDTH_FRAC`), blob
size and shape gates, the plausibility gate, the lost-line timings, steering and
throttle (`STEERING_KP`, `STEERING_SMOOTHING`, `THROTTLE_FORWARD`), lane geometry, and
the debug/web flags. Full list with defaults and the recommended order to change them:
[docs/lane_following.md § Tuning at the track](docs/lane_following.md).

### If foliage leaks into the yellow mask

This was our most common real-track failure — dry grass and sunlit leaves read as
yellow. Tune in this order:

1. **Lower** `YELLOW_HSV_HIGH`'s hue ceiling (default `33`; foliage green starts around 40).
2. **Raise** `YELLOW_HSV_LOW`'s saturation floor (default `110`; foliage is pale).
3. **Raise** `ROI_TOP_FRAC` (default `0.55`) to crop more of the horizon away.

Change the color threshold before you tighten the shape filters. The shape filters exist
to catch what leaks through, not to compensate for a threshold that is too loose.

---

## Verifying without the car

Three checks, none of which need hardware:

```bash
# 1. Wiring: runs the real vehicle loop with mock camera and drivetrain, twice.
#    Asserts the CV part stays off in manual mode and runs in autopilot mode.
python scripts/preflight_lane_following.py --controller LaneFollowingController --web

# 2. CV logic: foliage rejection, plausibility gate, lost-line states, lane model, web API.
pytest tests/test_lane_following.py tests/test_lane_following_web.py -v

# 3. Regression: line following still produces the exact same commands as the baseline.
python scripts/replay_frames.py --check tests/data/line_golden.json
```

Run the preflight after **any** change to the parts or the config — it is the cheapest
way to find out that the car would have failed to start.

---

## Development process (the agentic loop)

Track 2 asked us to build new DonkeyCar capabilities using an AI coding agent, so the
workflow itself is part of the deliverable. We gave Claude a mission, let it write the
CV and control code, tested on the physical car, and fed back what broke. We restarted
the approach more than once. In order:

1. **First-pass line follower.** A classical-CV follower that worked in principle but
   was not robust to real track lighting.

2. **Half-lane following (superseded).** We pivoted to having the car hug one half of the
   road, aiming at eventually supporting two-way traffic. Every track test surfaced a new
   false positive that the agent then fixed: a BGR/RGB channel mix-up, shadows read as
   fake edges, gravel read as fake edges, and daylight breaking detection outright. This
   iteration made detection genuinely robust, but we concluded that boundary-hugging was
   the wrong foundation for lane following and rebuilt.

3. **Clean rebuild** (commit `2a03bab`). Line following rewritten from scratch with the
   layered architecture now in `donkeycar/parts/lane_following/`, plus the scripts, the
   tests, and the docs. The first real-track test found something the agent could never
   have found from code alone: the OAK-D part was hard-cropping away the near-ground
   strip the follower needed on turns. Fixed by streaming the full-FOV `isp` output
   (commit `75ce081`).

4. **Feedback-driven hardening** (commits `d50c77e`, `ad59785`, `65bd517`). Camera
   validation that pulls a real frame rather than just opening the device — the earlier
   check had been masking failures. Optional auto-exposure compensation for bright
   conditions. Lane-following mode and its web toggle page. And finally a steering-loop
   rewrite — centroid tracking and a centered target — after track testing showed the car
   consistently drifting off-center.

The earlier iterations were squashed onto `main` rather than kept as separate branches,
so the commit hashes above are the history; there are no other branches to check out.

**What we'd tell the next team:** the agent was good at producing working, well-factored
code quickly, and consistently wrong about the physical world until we tested. Every
significant fix in this list came from driving the car, not from reading the code. Build
the cheap verification tools (the preflight and replay scripts) early — they made each
feedback cycle minutes instead of a trip to the track.

---

## Known limitations

- **Stage 3 (two-way road navigation) was not reached.** No oncoming-vehicle detection or
  avoidance exists.
- **`CV_PREVIEW_IN_MANUAL` does not work.** `donkeycar/mycar/manage.py` imports a
  `lane_following.preview` module that is not in this repository. The flag is `False` in
  the shipped config and the car launches normally, but setting it `True` will fail on
  import. Leave it off.
- **The two autopilots are separate implementations.** `LineFollower` and the
  `lane_following` package solve stage 1 differently; the deployed car uses the former.
  Consolidating them was on the list and did not happen.
- Tuning is per-track and per-lighting. Expect to re-run the HSV tuning loop when
  conditions change.

---

## Built on Donkeycar

This project is a fork of [autorope/donkeycar](https://github.com/autorope/donkeycar), an
open-source self-driving library for Python. All of our work is implemented as Donkeycar
_parts_ and templates on top of its Vehicle/Part architecture.

- The upstream project README is preserved verbatim at [README_donkeycar.md](README_donkeycar.md).
- Upstream documentation: [docs.donkeycar.com](http://docs.donkeycar.com)
