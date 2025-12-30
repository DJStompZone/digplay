You are an agentic coding model. Build a production-quality, OS-agnostic, headless-capable project named **DigPlay** (“Digit Player”) that “plays” GB/GBC/GBA games using base-10 digits of pi as controller inputs.

This version must use **Stable-Retro** (Farama) as the emulator backend (Python API).

---

## Existing dependency (DO NOT REIMPLEMENT): DigPipe

DigPipe is an external Python library that provides a digit pipeline:

`DigitSource -> DigitTape -> DigitMapper -> InputSink`

Key DigPipe facts you must respect:

- Pi digit indexing **includes the leading `3`** in the count: digit index `0` is `3`; fractional digits begin at index `1`.
- DigitTape stores digits in **packed nibbles** (two digits per byte). `chunk_size` must be **even**.
- DigPipe includes a GBA-oriented mapper that converts digits into **time-stamped press/release Actions** with configurable `hold_frames` and `release_frames` (defaults `5/5`).
- DigPipe includes a `FrameLogSink` that writes actions like `FRAME: gba.CONTROL=VALUE`. We will create a new sink/bridge to apply actions to an emulator.

Your harness must treat DigPipe as the source of truth for digits and (optionally) action scheduling.

---

## High-level goal

Given:
- a ROM
- a start digit index
- mapping configuration

Run an emulation session where each frame’s controller state is derived from pi digits (via DigPipe), and the run is:
- deterministic and resumable
- able to save/load emulator save-states
- able to record session video (segmentable)
- able to “pop the hood” later (RAM inspection hooks)

---

## Stable-Retro constraints and required approach

Use the Stable-Retro Python API (`retro.make`, `retro.RetroEnv`):

- Must step deterministically: one call to `env.step(keys)` == advance exactly one frame.
- Controller input should be provided as the correct shape for Stable-Retro’s action space (typically a list/array of button states matching `env.num_buttons` and `env.buttons`).
- Must support headless operation via `render_mode="rgb_array"` (no GUI required).
- Must support RAM inspection using `obs_type=retro.Observations.RAM` and/or `env.get_ram()` where appropriate.
- For recording:
  - Prefer Gymnasium’s `RecordVideo` wrapper OR a direct “pipe frames to ffmpeg” approach.
  - Segment output.

Important: Stable-Retro typically requires a game “integration” (metadata + .state files). DigPlay must:
- Provide clear README instructions for integrating ROMs using Stable-Retro tooling (no ROM distribution).
- Offer helpful error messages when a ROM is not integrated yet.

---

## Deliverables

### 1) DigPlay Python project

Repository layout (suggested):

- `digplay/`
  - `__init__.py`
  - `cli.py` (argparse CLI)
  - `runner.py` (orchestrates DigPipe -> backend)
  - `manifest.py` (run metadata + checkpoints)
  - `digpipe_adapter.py` (wires DigPipe into frame/action stream)
  - `backends/`
    - `base.py` (EmulatorBackend ABC)
    - `stable_retro.py` (StableRetroBackend implementation)
  - `video/`
    - `recorder.py` (ffmpeg piping/segmentation OR Gymnasium wrapper glue)
- `tests/` (pytest)

### 2) EmulatorBackend contract (Python)

Provide an `EmulatorBackend` abstract interface with methods:

- `launch(rom_path, *, config)` / `reset()`
- `step_frame(controller_state)` -> returns current frame number
- `save_state_bytes()` / `load_state_bytes(data)` (best-effort; if Stable-Retro exposes state APIs, use them)
- `read_memory(addr, size)` / `write_memory(addr, data)` (optional, using RAM access if feasible)
- `get_frame_rgb()` (required if recording frames manually)
- `close()`

For this prompt, the concrete backend is **`StableRetroBackend`**, which wraps `retro.make(...)`.

### 3) Deterministic runner + resumability

Runner requirements:

- Deterministic stepping:
  - For each frame: determine controller state from DigPipe action schedule (press/release) -> convert to Stable-Retro keys vector -> `env.step(keys)`.
- Persistent manifest (`run_manifest.json`):
  - ROM absolute path + sha256
  - backend identifier (`stable-retro`) and versions (best-effort)
  - start digit index
  - current digit index / chunk index / action cursor
  - current frame
  - checkpoint cadence
  - video segments list
  - any notable config (hold_frames, release_frames, chunk_size)
- Resume mode:
  - load latest checkpoint
  - restore emulator state (state bytes if available; otherwise document limitations and implement “best effort” checkpoints using Stable-Retro `.state` files where possible)
  - resume DigPipe cursor deterministically

### 4) Video recording (OS-agnostic + headless)

Implement a recording strategy:

Option A (preferred): Gymnasium RecordVideo wrapper
- Build env with `render_mode="rgb_array"`, wrap with `gymnasium.wrappers.RecordVideo`.
- Ensure the wrapper is triggered for long continuous runs (segmenting required; implement segmenting by closing/re-opening env/wrapper periodically).

Option B: Manual frame capture to ffmpeg
- `frame = env.render()` or use returned observation if it’s an image.
- Pipe frames to ffmpeg via stdin (`rawvideo`).
- Segment output by time or frame count.

### 5) CLI

Implement:

- `digplay run --game GAME_NAME --state STATE_NAME --start-digit-index N --outdir DIR [--max-frames N] [--checkpoint-every N] [--chunk-size N] [--hold-frames N] [--release-frames N] [--record-video]`
- `digplay resume --run-dir DIR [--max-frames N]`
- `digplay info --run-dir DIR`
- `digplay probe --run-dir DIR --dump-ram [--len N]` (optional)

Note: Stable-Retro identifies games by “integration name” (GAME_NAME), not necessarily a ROM filepath. If you accept a `--rom` path, you must map it to the integrated game name or instruct the user how to integrate it and then use `--game`.

### 6) Testing (pytest)

- Unit tests must NOT require real ROMs.
- Provide a `FakeBackend` for tests that simulates:
  - frame stepping
  - controller state application
  - state save/load (bytes)
- Tests must validate:
  - DigPipe action scheduling is applied on the correct frames
  - checkpoint/resume yields identical final state vs uninterrupted run

- Integration tests for Stable-Retro should be skipped unless env var `DIGPLAY_RUN_INTEGRATION_TESTS=1` is set.

### 7) Code quality

- Python 3.11+
- Type hints, dataclasses
- Google-style docstrings (succinct but complete)
- Use logging, avoid stdout spam
- Single-responsibility modules
- No “...” in code

---

## README requirements

Include:
- How to install DigPlay
- How to install Stable-Retro
- How to integrate ROMs (high-level; do not provide ROMs)
- Example run + resume commands
- Notes on determinism + checkpointing constraints

---

## Acceptance criteria

- A user can:
  1) Integrate a GB/GBC/GBA ROM into Stable-Retro (per README)
  2) Run: `digplay run --game SomeGame --start-digit-index 1000 --max-frames 600 --outdir ./runs/test_stable_retro --record-video`
  3) Observe creation of:
     - manifest JSON
     - checkpoint(s)
     - video segments (or RecordVideo outputs)
  4) Resume: `digplay resume --run-dir ./runs/test_stable_retro --max-frames 600`
  5) Determinism proof: the run after resume matches the uninterrupted run’s manifest end state