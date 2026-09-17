<h1 align="center">sortbot</h1>

<p align="center">
  <b>Voice-directed VLM pick-and-place on an SO101 arm.</b><br>
  It tidies a table while you talk to it.
</p>

<p align="center">
  <img alt="1st place — Open Hardware — Berkeley Robotics Hackathon" src="https://img.shields.io/badge/1st%20place-Open%20Hardware%20%C2%B7%20Berkeley%20Robotics%20Hackathon-DAA520?style=for-the-badge">
</p>

<p align="center">
  <img alt="arm: SO101" src="https://img.shields.io/badge/arm-SO101%20%C2%B7%205%20DOF%20%2B%20gripper-0969da">
  <img alt="LeRobot 0.6.2" src="https://img.shields.io/badge/LeRobot-0.6.2%20vendored-1a7f37">
  <img alt="planner: OpenAI Responses API" src="https://img.shields.io/badge/planner-OpenAI%20Responses%20API-412991">
  <img alt="voice: ElevenLabs realtime STT" src="https://img.shields.io/badge/voice-ElevenLabs%20realtime%20STT-bc4c00">
  <img alt="HUD: localhost:8765" src="https://img.shields.io/badge/HUD-localhost%3A8765-656d76">
  <img alt="license: see lerobot/LICENSE" src="https://img.shields.io/badge/vendored%20deps-Apache%202.0-656d76">
</p>

---

## What it is

sortbot drives a 5-DOF SO101 follower arm that picks objects off a table and groups them. Two cameras —
overhead and wrist — feed a VLM planner. A cm-labelled grid is composited onto the overhead frame, and the
model reads coordinates off that grid. Each step the planner emits one tool call — `pick_at`, `place_at`,
`move_to`, `turn_to`, `turn_by`, `open_gripper`, `close_gripper`, `say`, or `done`.

You give it a task by voice or by text. With nothing specified it groups similar items together. You can
keep talking while it moves; "stop" fires through a regex pre-filter as soon as the word is recognised.

Before any close, `verify_grasp` reads both camera views and returns `{aligned, dx_cm, dy_cm, reason,
confidence}`. Low confidence counts as not aligned. After retries are exhausted the arm retreats with the
jaws open.

## Architecture

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/loop-and-grasp-gate-dark.svg">
  <img src="assets/loop-and-grasp-gate-light.svg" alt="The act loop: home, capture, composite overlay, planner call, validate, execute, record, back to home — with the inner grasp-verify gate on pick_at and close_gripper, and failures feeding back into the planner as text.">
</picture>

Every step of the run is the same seven-stage loop: `home()`, capture both cameras, composite the cm-grid
overlay onto the overhead frame, call the planner, validate its target against the safety envelope, execute
under `Session.robot_lock`, record to the decision log, repeat. The grid overlay is the coordinate
reference; the planner reads it off the image.

The planner runs on the OpenAI Responses API and gets one `function_call` per step (`tool_choice=required`,
`parallel_tool_calls=false`). Every tool is declared `strict=True` with `additionalProperties=false`.

The prompt payload is two images plus a text block: the overhead PNG and the wrist PNG, both at
`detail=high`, then a state block covering the overlay key, the end-effector pose in cm, whether the gripper
is open or holding, the reachable area, the current RULES, and the last 10 steps of history as
`tool(args) -> result`.

When validation rejects a target — outside the AABB, past the hard floor, unreachable — the rejection comes
back to the planner as a `FAILED: <reason>` tool result, folded into the same history block the next prompt
sees. A grasp that aborts after exhausting its retries reports the same way. The model re-plans from the
failure like any other observation.

**Before any close, a second model call gates it.** `verify_grasp` looks at both frames and returns a
structured `{aligned, dx_cm, dy_cm, reason, confidence}`, capped at `max_output_tokens=400`. On a no, the
arm nudges by `dx_cm`/`dy_cm` (clamped to `max_correction_cm`) and re-checks, up to `max_retries` times; if
still not aligned it retreats with the gripper open, abort reason back into history. Every verdict lands in
the decision log with a side-by-side overhead/wrist thumbnail.

> [`sortbot/config.yaml`](sortbot/config.yaml) currently ships `grasp: verify: false` (commented
> `DISABLED at user request`), so a default install runs without this gate even though the test suite pins
> verification on. Turn it on for unattended runs.

## Talking while it moves

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/threads-queues-bus-dark.svg">
  <img src="assets/threads-queues-bus-light.svg" alt="Five threads, four queues, one serial bus: thread and queue wiring, the urgent E-STOP bypass around the queues, and the robot_lock acquire timeouts per caller.">
</picture>

Motion, chat and the camera preview run on separate threads that talk through small queues.

Five threads: `voice` (mic/TTS), `luna-chat` (the ChatWorker, drains `q_heard` at 20 Hz), `Loop` (the only
thread that moves the arm), `preview` (0.4 s, ~2.5 Hz) and `hud` (FastAPI/uvicorn on `127.0.0.1:8765`).
Four queues connect them: `q_heard` (endpointed utterances), `q_directives` (rules, hints, commands, stop),
`q_say` (one TTS worker; `priority=True` drops the whole backlog) and `q_log` (a 200-entry ring).
`Loop.drain_inputs()` reads `q_directives` without blocking — an empty queue means keep moving.

**Stop bypasses the queues.** An interim transcript, still mid-sentence, hits a regex pre-filter (5 stop
patterns, 2 pause patterns) and fires `torque_off()` through `Control` events, without waiting for VAD to
endpoint. Measured firing **127 ms before the speaker finished the sentence**.

**Exactly one thread may touch the Feetech serial bus.** `Session.robot_lock` enforces it with a different
acquire timeout per caller: the HUD's `/state` poller waits 0.2 s and serves a cached pose on timeout,
ordinary robot actions wait 2.0 s and fail past that, E-STOP waits 1.0 s and jumps the queue, `Loop` holds
the lock through an entire motion. `luna-chat` and `preview` read cached JPEGs (≤512 px, quality 72) and a
cached pose. `SORTBOT_BUS_ASSERT=1` arms a proxy that raises on any unlocked bus call (`=warn` only logs).

## Frames, units, and the safety envelope

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/frames-and-z-stack-dark.svg">
  <img src="assets/frames-and-z-stack-light.svg" alt="Side elevation of the workspace showing every z plane the code enforces, the z_trim callout, the lift-translate-descend path, the base-to-table frame conversion, and the cm-to-mm unit boundary.">
</picture>

Every commanded pose passes five gates, in this order:

| # | Gate | Value |
|---|---|---|
| 1 | Hard floor (trim-independent backstop) | `z_floor_mm = -150 mm` |
| 2 | Grasp depth | `grasp_z + z_trim` |
| 3 | AABB bounds | `[120, -220, 0] .. [420, 220, 250] mm` |
| 4 | XY step limit | `max_step_mm = 600 mm` |
| 5 | IK reachability | `FK(IK)` error `<= 5.0 mm` |

A move lifts to `travel_z = 120 mm`, crosses in cartesian XY at 40 mm sub-steps, then descends — every
waypoint planned before the first tick fires. Joints interpolate at 2°/tick on a 50 Hz motion loop with a
1.5° settle tolerance. `torque_off()` clears the torque flag, and every motion call after that raises
`SafetyError` until `torque_on()` runs.

`z_trim_mm` (−150..+150, default −10, warns beyond |40|) shifts the commanded grasp plane and the envelope
floor together. `max_step_mm` bounds cartesian XY translation.

The end effector points straight down. Only `wrist_roll` varies, and `turn_to`/`turn_by` clamp it to
−90..+90°.

Units follow one rule: **the VLM-facing surface is centimetres, everything internal is millimetres** —
`robot.py`, `config.yaml`, `calib.json`, the safety envelope. The conversion happens in exactly one place.

The IK solver seeds from a coarse FK grid of ~62,000 poses (lift × elbow × wrist_flex over their joint
limits in 5° steps), evaluated once at startup. `solve()` then runs damped least squares from the 3
lowest-cost seeds, with gripper tilt cost-weighted at 3 mm/rad.

## Hardware

| Part | Spec | Notes |
|---|---|---|
| Follower arm | SO101, 6 Feetech motors: `shoulder_pan`, `shoulder_lift`, `elbow_flex`, `wrist_flex`, `wrist_roll`, `gripper` (5 arm DOF + 1 gripper) | URDF at [`SO101/so101_new_calib.urdf`](SO101/so101_new_calib.urdf). |
| Leader arm | SO101 | Teleoperates the follower during calibration. |
| Joint limits | pan ±110°, lift ±100°, elbow ±96.8°, wrist_flex ±95°, wrist_roll −157.2°/+162.8° | `turn_to`/`turn_by` clamp roll commands to ±90° regardless of the mechanical range. |
| Gripper | Motor 0–100 units; open = 60, closed = 5 | Driven directly by open/close calls. |
| Overhead camera | index 0, 640×480, 30 fps | Feeds the cm-grid overlay the planner reads. |
| Wrist camera | index 1, 640×480, 30 fps | Second view for the grasp-alignment check before every close. |
| Kinematics | IK drives 4 of the 5 arm joints (pan, lift, elbow, flex) | `wrist_roll` is commanded directly, outside the IK solve. |
| Serial ports | `robot.port` and `leader.port` in [`sortbot/config.yaml`](sortbot/config.yaml) | Per-machine. The checked-in values are one machine's macOS `/dev/tty.usbmodem*` paths — change them. |
| ArUco mat | `DICT_4X4_50`, 40 mm tags, 400×300 mm mat, ids 0–3 in TL/TR/BR/BL order | Optional; `calibration.mode` can use tags instead of or alongside ball-mode. |

## Software

| Layer | What | Version / setting |
|---|---|---|
| Arm control | LeRobot, vendored in [`lerobot/`](lerobot/) (Apache 2.0) — FK via `RobotKinematics`, plus motor I/O | 0.6.2 |
| IK | Custom damped-least-squares solver in [`sortbot/robot.py`](sortbot/robot.py) | 50 iterations, active-set joint limits |
| Tensor runtime | torch | `>=2.7,<2.12.0` |
| Vision | opencv-python-headless | `>=4.9.0,<4.14.0` |
| Numerics | numpy | `>=2.0.0,<2.3.0` |
| Env interface | gymnasium | `>=1.1.1,<2.0.0` |
| HUD | FastAPI + uvicorn, MJPEG stream per camera | bound to `127.0.0.1:8765` |
| Planner transport | OpenAI Responses API | one `function_call` per step |
| Voice STT | ElevenLabs realtime STT over WebSocket, VAD endpointing at 0.7 s of silence | `scribe_v2_realtime`; chunked `scribe_v2` fallback |
| Voice TTS | ElevenLabs TTS | `eleven_flash_v2_5` default; turbo / multilingual / v3 selectable |

The Python version ranges come straight from [`lerobot/pyproject.toml`](lerobot/pyproject.toml).

Three model roles, set in [`sortbot/config.yaml`](sortbot/config.yaml): planner (`vlm.model`), chat
(`vlm.chat_model`) and grasp check (`vlm.verify_model`), with reasoning effort held low
(`vlm.chat_effort: low`). The shipped strings — `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.4-mini` — are this
config's own names, not public OpenAI model ids. Swap them from the HUD **Tune** tab to whatever your
account can call.

## Quickstart

**Step 1, before anything else: fix the interpreter path.** [`run.sh`](run.sh) hardcodes the absolute Python
interpreter from the machine it was built on:

```bash
exec "/Users/seth/miniforge3/envs/lerobot/bin/python" "$@"
```

Point that line at your own environment.

### Dependencies

Dependencies come from the vendored [`lerobot/pyproject.toml`](lerobot/pyproject.toml) — install `lerobot`
and its extras into your environment.

### Environment variables

| Variable | Required | Effect |
|---|---|---|
| `OPENAI_API_KEY` | yes | The VLM planner. sortbot refuses to start without it. |
| `ELEVENLABS_API_KEY` | no | Realtime voice (STT + TTS). Without it, input falls back to the keyboard. |
| `SORTBOT_BUS_ASSERT` | no | `1` makes the bus-lock proxy raise on any unlocked motor call; `warn` logs instead. Debug aid. |

Both keys are read from `.env` at the repo root.

### Run it

sortbot is server-first: it boots with nothing connected. The HUD comes up, then you connect the robot,
cameras and vision model from the browser.

```bash
./run.sh -m sortbot.main
```

Open `http://127.0.0.1:8765`, go to the **Setup** tab, connect Robot / Cameras / Vision model, then hit
**Start**. `run.sh` sets `PYTHONPATH` to the repo root plus `lerobot/src` before exec'ing Python; that is the
only other thing it does.

Flags:

- `--max-steps N` — step budget for one run (default 200)
- `--hud-port P` — override the HUD port
- `--no-voice` — skip the keyboard fallback input thread
- `--rules-file PATH` — persistent rules store (default `sortbot/calib/rules.json`; survives restarts)
- `--config PATH` — alternate config file

### Tests (no hardware needed)

```bash
./run.sh -m sortbot.tests.test_e2e
./run.sh -m sortbot.tests.test_hud_actions
./run.sh -m sortbot.tests.test_bus_lock
./run.sh -m sortbot.tests.test_chat
./run.sh -m sortbot.tests.test_units
```

`MockRobot`, `MockVLM`, `SimScene` and `FakeRig` live in
[`sortbot/testing.py`](sortbot/testing.py) and the tests inject them through `Session(factories=...)`.

## Calibration

Default mode is teleop `ball`: hold a coloured target in the follower's gripper, drive the arm by hand with
the SO101 leader, and capture at each pose to pair the overhead pixel centroid with the FK xyz. Teleop runs
at 30 Hz with detection every 3rd tick (~10 Hz). A capture is refused unless the arm has settled — under
2 mm of drift over a 60 ms gap — and the new sample sits more than 15 mm from every prior one in xy.

**The sample count picks the model.** Under 8 points fits a 6-DOF affine; 8 or more fits the full 8-DOF
homography. Affine is over-determined from 4 points and cannot invent perspective, which suits a near-nadir
overhead camera. RANSAC rejects outliers past a 5 mm inlier threshold; rejects are ringed in the live
overlay and written to `calib.json`.

Finish refuses to save until seven guards pass:

| Guard | Threshold |
|---|---|
| fitted samples | ≥ 8 |
| workspace coverage | ≥ 10% |
| collinearity ratio | ≥ 0.15 |
| height spread | ≤ 25 mm |
| tilt spread | ≤ 12° |
| max residual | ≤ 8 mm |
| RANSAC rejections | 0 |

A failed attempt names the guards that are unmet. A second attempt with the same sample count and z-offset
overrides them and saves anyway. The old `calib.json` is backed up to `.bak` first.

`calibration.mode` is `ball`, `aruco` or `auto`; the shipped default is `auto` — run on the fitted
homography, let 4 visible tags override per frame.

Click-by-click walkthrough: [`sortbot/README.md`](sortbot/README.md).

## Repo layout

```
.
├── sortbot/      # the app: planner, robot control, HUD, voice, tests
├── lerobot/      # vendored LeRobot 0.6.2 (Apache 2.0) — FK, motor bus
├── SO101/        # URDF + STL meshes for the arm
├── assets/       # the diagrams on this page
└── run.sh        # sets PYTHONPATH (repo root + lerobot/src), then execs python
```

Inside `sortbot/`:

| Module | Role |
|---|---|
| `main.py` | `Session`, the action loop, the four queues, the ChatWorker |
| `robot.py` | Safety envelope (AABB, z floors, max XY step) and the damped-least-squares IK |
| `vlm.py` | Planner tool schema, the chat call, and `verify_grasp` |
| `perception.py` | The cached cm-grid overlay composited onto the overhead frame |
| `calibration.py` / `calibrate.py` | Homography fitting and the teleop capture session |
| `voice.py` | Streaming STT, the TTS worker, the urgent-word regex pre-filter |
| `hud.py` | FastAPI action registry and the `/state` endpoint |
| `testing.py` | Test doubles — imported by tests |

For everything else, go to the source:

- [`sortbot/README.md`](sortbot/README.md) — full HUD action reference, the calibration walkthrough, every config key
- [`sortbot/config.yaml`](sortbot/config.yaml) — annotated source of truth for the tunables
- [`sortbot/tests/`](sortbot/tests/) — the invariants the suite pins: units match, commands preempt the planner, stop fires under a second, alignment is checked before close

`lerobot/` is vendored under Apache 2.0; its terms are in [`lerobot/LICENSE`](lerobot/LICENSE).
