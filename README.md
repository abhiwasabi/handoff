# handoff — Phase 1

A local, simulation-first control studio for a six-servo smart-glasses hand glove. The React/TypeScript/Tailwind interface has Dashboard, Manual Control, Macro Studio, AI Vision, Calibration, and Devices sections. FastAPI provides persistent SQLite macros and calibration, six-channel simulation, stop handling, and WebSocket telemetry.

## Servo map

| ID | Movement |
|---|---|
| 1 | Index finger |
| 2 | Middle finger |
| 3 | Thumb flexion |
| 4 | Thumb opposition |
| 5 | Ring finger |
| 6 | Pinky finger |

The attached hardware PDF contains older wrist assignments as well as this newer individual-finger layout. handoff follows the newer six-servo layout. Its suggested servo counts also vary between sections, so all displayed calibration counts start **unverified**. They are placeholders, not commands or recommended physical limits.

## Run locally

Use Python 3.11+ and Node.js 20+ in two terminals.

**Terminal 1 — API**

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

**Terminal 2 — UI**

```bash
cd frontend
npm install
npm run dev -- --port 5173
```

Open <http://127.0.0.1:5173/>. The API docs are at <http://127.0.0.1:8000/docs>. The Vite server proxies `/api` and `/ws` to the local API. Keep both terminals running.

On first launch, `backend/handos.sqlite3` is created with five editable sample poses and six unverified calibration records. To use another database path, set `HANDOS_DB` before starting the API. Macro export downloads a versioned JSON file; importing adds copies of its macros. Sequences have up to 30 steps with 100–10,000 ms duration per step.

## Try the first milestone

1. Open **Macro Studio** and choose **New macro**.
2. Set a name and six finger percentages, then **Save macro**.
3. Choose **Play**. The Dashboard channel meters and WebSocket telemetry update as the six simulated channels move.
4. Use **Stop** to latch the software stop; **Reset simulation** allows movement again.

Manual Control changes a draft until **Apply to simulation** is pressed. The AI Vision page can show a browser camera preview. It does not analyze frames. Detection events never trigger a macro.

## Safety boundary

Phase 1 has **no physical servo driver**, serial port access, or route that can switch to hardware mode. `backend/app/hardware_boundary.py` defines the future transport interface and a gate that requires:

- Valid, measured and verified calibration for every servo in the pose.
- A tested physical E-stop recorded for that session.
- Explicit approval for that physical actuation.
- No latched software stop; every mapped target inside measured limits.

The physical E-stop must interrupt servo power independently of handoff. Test it with the glove off the body before any wearable trial. A software stop is useful for the app but cannot substitute for a power cut. Attach the glove's proven Feetech/ST3215 control code to `ServoTransport` only after reviewing its protocol, limits, torque behavior, and stop behavior. None was present in the supplied workspace, so no control code was copied or rewritten.

## Checks

```bash
cd backend && .venv/bin/python -m pytest -q
cd ../frontend && npm run build
```

The first build passed with five backend tests. The UI was also opened in a browser, where saving and playing a new pose updated the simulated hand. The frontend's current Tailwind 3 build dependencies have seven npm audit advisories (five high, two moderate), all in build tooling. Keep this development server bound to localhost and upgrade the build toolchain before any public deployment.

## Integration next steps

1. Obtain the existing Feetech controller repository and identify its known-good read/write and torque-off functions. Keep it behind `ServoTransport`.
2. Measure each servo's open, closed, and hard limits unloaded; verify direction and speed per channel. Store those values in Calibration, then test the E-stop.
3. Build a deliberate physical-arm flow that checks the gate before every pose or sequence step and requires a fresh confirmation for each action.
4. Connect the glasses camera or object detector. Send real events to `POST /api/vision/detection` with `source: "external"`; add a reviewable object-to-macro mapping and user confirmation before allowing any resulting motion.

The API is intended for local development and has no user authentication. Do not expose it to an untrusted network.
