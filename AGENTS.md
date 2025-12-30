You are an agentic coding model. Build a production-quality, OS-agnostic, headless-capable project named **DigPlay** (“Digit Player”) that “plays” GB/GBC/GBA games using base-10 digits of pi as controller inputs.

This version must use **mGBA** as the emulator backend (via mGBA’s built-in Lua scripting API)

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
- able to “pop the hood” later (memory read/write hooks)

---

## mGBA constraints and required approach

mGBA’s scripting is **Lua-only**. Use mGBA’s documented scripting API objects and callbacks:

- `emu.runFrame()` to advance exactly one frame
- `emu.setKeys(bitmask)` / `emu.addKeys(...)` / `emu.clearKeys(...)` to control input
- `emu.saveStateBuffer()` / `emu.loadStateBuffer(...)` and/or `saveStateFile/loadStateFile`
- `emu.screenshot(...)` for snapshots
- `emu.read8/read16/read32` and `emu.write*` for memory inspection
- `callbacks.add("keysRead", fn)` or `callbacks.add("frame", fn)` to hook per-frame behavior

Also note: Lua bindings expose a basic TCP socket library (`socket.bind`, `socket.connect`). Use this to communicate with a local Python process if needed.

### Architecture requirement for DigPlay+mGBA

Implement a **two-process design**:

1) **Python orchestrator (DigPlay core)**
   - Imports and uses DigPipe to produce scheduled Actions or per-frame controller states.
   - Owns run manifests, checkpoint scheduling, segmentation, logging.
   - Talks to the Lua script over localhost TCP (single machine assumption).

2) **mGBA Lua “bridge script”**
   - Runs inside mGBA.
   - Receives commands from Python: set controller state, run frame, save/load state, read/write memory, screenshot.
   - Must support deterministic stepping: “apply input -> advance exactly one frame -> report frame index”.

### Headless requirement

DigPlay must be *usable headlessly*:
- Prefer running mGBA in a headless/virtual display mode when possible (e.g., Xvfb on Linux); otherwise support “minimized window” operation while still running the Lua bridge.
- The harness must not require interactive clicking once configured. It’s okay if first-time setup requires the user to load the Lua script in mGBA (document the steps).

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
    - `mgba_bridge.py` (Python-side TCP client for the Lua bridge)
  - `video/`
    - `recorder.py` (ffmpeg piping/segmentation)
- `scripts/`
  - `mgba_bridge.lua` (Lua server script loaded in mGBA)
- `tests/` (pytest)

### 2) EmulatorBackend contract (Python)

Provide an `EmulatorBackend` abstract interface with methods:

- `launch(rom_path, *, config)` (for mGBA, this may just verify connectivity/ROM)
- `step_frame(controller_state)` -> returns current frame number
- `save_state_bytes()` / `load_state_bytes(data)`
- `save_state_file(path)` / `load_state_file(path)` (optional convenience)
- `read_memory(addr, size)` / `write_memory(addr, data)` (optional but implement if feasible)
- `screenshot(path)` (optional)
- `close()`

For this prompt, the concrete backend is **`MgbaBackend`**, which speaks to the Lua bridge over TCP.

### 3) Deterministic runner + resumability

Runner requirements:

- Deterministic stepping:
  - For each frame: determine controller state from DigPipe action schedule (press/release) -> send to mGBA -> run exactly 1 frame.
- Persistent manifest (`run_manifest.json`):
  - ROM absolute path + sha256
  - backend identifier (`mgba + lua bridge`) and versions (best-effort)
  - start digit index
  - current digit index / chunk index / action cursor
  - current frame
  - checkpoint cadence
  - video segments list
  - any notable config (hold_frames, release_frames, chunk_size)
- Resume mode:
  - load latest checkpoint
  - restore emulator state
  - resume DigPipe cursor deterministically
  - continue seamlessly

### 4) Video recording (OS-agnostic)

Provide a recording strategy that works across OSes:

- Preferred: have Lua bridge periodically `emu.screenshot()` (PNG) to a temp folder and Python encodes frames with ffmpeg (not ideal, but reliable and portable).
- Alternative: implement an ffmpeg “screen capture” mode per-OS (documented), but keep it optional.
- Segment output (`segment_0001.mp4`, etc.) to avoid giant files.

### 5) CLI

Implement:

- `digplay run --rom PATH --start-digit-index N --outdir DIR [--max-frames N] [--checkpoint-every N] [--chunk-size N] [--hold-frames N] [--release-frames N] [--record-video]`
- `digplay resume --run-dir DIR [--max-frames N]`
- `digplay info --run-dir DIR`
- `digplay probe --run-dir DIR --read-ram ADDR --len N` (optional memory inspection)

### 6) Testing (pytest)

- Unit tests must NOT require mGBA.
- Provide a `FakeBackend` for tests that simulates:
  - frame stepping
  - controller state application
  - state save/load (bytes)
- Tests must validate:
  - DigPipe action scheduling is applied on the correct frames
  - checkpoint/resume yields identical final state vs uninterrupted run

### 7) Code quality

- Python 3.11+
- Type hints, dataclasses
- Google-style docstrings (succinct but complete)
- Use logging, avoid stdout spam
- Single-responsibility modules
- No “...” in code

---

## First-time setup instructions (must be included in DigPlay README)

- How to install DigPlay
- How to install mGBA
- How to open mGBA, load ROM, open Tools -> Scripting, load `scripts/mgba_bridge.lua`
- How to start DigPlay which connects to the Lua bridge and begins stepping
- How to resume from checkpoints

---

## Acceptance criteria

- A user can:
  1) Start mGBA, load ROM, load the Lua bridge script
  2) Run: `digplay run --rom game.gba --start-digit-index 1000 --max-frames 600 --outdir ./runs/test_mgba`
  3) Observe creation of:
     - manifest JSON
     - checkpoint(s) with emulator state
     - optional video segments
  4) Resume: `digplay resume --run-dir ./runs/test_mgba --max-frames 600`
  5) Determinism proof: the run after resume matches the uninterrupted run’s manifest end state