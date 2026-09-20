# RoomCapture Pipeline

Phone capture (Photos / Video / LiDAR) -> dimensioned floor plan + damage
report. Runs entirely on-device/locally — no backend server, no API keys,
no external model calls at runtime. This README is a complete, literal,
step-by-step walkthrough of everything needed to go from a bare machine
to a full run, including the physical phone-capture steps.

Background docs: `CLAUDE.md` (running project log), `docs/compliance_matrix.md`
(requirement-by-requirement status).

---

## Part A — One-time machine setup (~5 minutes)

### A1. Prerequisites
- A Mac (this was built and tested on macOS, MacBook Air M3).
- An iPhone 15 or newer for Photo/Video tiers; a Pro-class iPhone (has
  LiDAR) for the LiDAR tier. Tested on iPhone 16 Pro.
- Python 3.11+ available as `python3`.
- No Xcode, no App Store developer account needed — capture uses stock
  apps only (Route 2, see Part B).

### A2. Clone the repo
```bash
git clone <this repository's URL>
cd RoomCapture-Pipeline
```

### A3. Create a virtual environment and install
```bash
python3 -m venv .venv
.venv/bin/python3 -m pip install -e ".[dev]"
```
This installs the `pipeline` package, its two runtime dependencies
(`opencv-python`, `matplotlib`), and `pytest` for verification. No other
setup — no Docker, no external services, no downloaded model weights.

### A4. Verify the install
```bash
.venv/bin/python3 -m pytest tests/ -q
```
Expect: `77 passed`. If this passes, the whole pipeline runs correctly
end to end on synthetic data — everything after this point is about
getting real phone captures in and pointing the pipeline at them.

---

## Part B — Capturing a room with your own iPhone

This project uses **Route 2**: stock, off-the-shelf apps, not a custom
iOS app. Full written protocol: `docs/capture_protocol.md`. The literal
tap-by-tap steps:

### B1. Install (one-time, on the iPhone)
1. **Camera** — already on the phone, nothing to install.
2. **3D Scanner App** by Laan Labs — App Store id `1419913995`. Free
   tier is sufficient. Open it once, sign up for the free trial if
   prompted, and grant Camera access.

### B2. Capture the LiDAR tier
1. Open **3D Scanner App**.
2. Tap **Scan Mode** (bottom-right button) -> select **LiDAR**.
3. Tap the middle red round **Record** button to start scanning.
4. Walk the room slowly, covering every red-highlighted area on screen.
   Moving too fast causes tears/blackouts in the scan — slow down if you
   see the mesh breaking up.
5. Tap **Process Scan** -> select **HD** -> tap **START**.
6. Tap **Share** -> choose **All Data** (this is the important part —
   **do not** choose "Point Cloud": that export is pre-fused by the app
   and drops the per-frame poses this pipeline needs to do its own
   reconstruction).
7. AirDrop the exported folder to your Mac.

### B3. Capture the Photo tier
1. Open **Camera**, photo mode.
2. Take 2-8 stills of the room (one per wall is a good default).
3. Select the photos -> AirDrop them to your Mac as a batch.

### B4. Capture the Video tier
1. Open **Camera**, video mode.
2. Record one continuous handheld walkthrough of the room.
3. AirDrop the video to your Mac.

### B5. Get the captures into the pipeline
On the Mac, AirDrop's save location should be set to this repo's
`media/inbox/` folder. Drop the LiDAR export folder + photos + video
for **one room** into `media/inbox/` together (all at once is fine —
the sorter waits for the whole burst to land).

From the repo root:
```bash
.venv/bin/python3 scripts/sort_media.py --watch
```
This watches `media/inbox/`, waits for the burst to settle (a 3-second
quiet window), then prompts:
```
Room name for this burst (e.g. bedroom, bedroom_take2):
```
Type a room name (lowercase, hyphenated, e.g. `living-room`). It
auto-classifies every file by type and moves everything into
`media/lidar/<room>/`, `media/photos/<room>/`, `media/video/<room>/`.
Press Ctrl+C to stop watching, or leave it running and repeat B2-B5 for
each additional room.

---

## Part C — Running the pipeline (one command per capture)

```bash
# LiDAR tier (note: the export keeps its original timestamped subfolder name)
.venv/bin/python3 -m pipeline.run --tier lidar \
  --room-dir "media/lidar/<room>/<timestamp-folder>" \
  --room-id <room>

# Photo tier
.venv/bin/python3 -m pipeline.run --tier photo \
  --room-dir "media/photos/<room>" \
  --room-id <room>

# Video tier
.venv/bin/python3 -m pipeline.run --tier video \
  --room-dir "media/video/<room>" \
  --room-id <room>
```
Each writes `<room>.json` (a schema-valid `PropertyPlan`, see
`docs/schema.md`) and `<room>.png` (a rendered top-down floor plan) to
`output/` by default (`--out-dir` to change).

**Optional — flag concealed-damage context** (building-topology facts
the rules need, since they can't be inferred from geometry alone):
```bash
.venv/bin/python3 -m pipeline.run --tier lidar --room-dir ... --room-id bedroom_1 \
  --context '{"below_bathroom": true, "load_bearing_walls": ["wall-0"]}'
```

**Multi-room property** (stitches several already-captured rooms
together via hand-declared connectors — see
`docs/multi_room_manifest.md` for the manifest format and how to find
real wall IDs):
```bash
.venv/bin/python3 -m pipeline.run --property-manifest path/to/property.json
```

---

## Part D — Running the full benchmark, device matrix, and head-to-head

Everything below is **live** — no cached model outputs, since nothing in
this pipeline calls an external model. One command reproduces every
reported number in `docs/`:

```bash
.venv/bin/python3 scripts/reproduce_all.py
```

This runs, in order (see `docs/reproduction_bundle.md` for full detail):
1. `scripts/run_benchmark.py` — reconstructs `bedroom_1` from the raw
   captures already in `media/`, scores every Part 2 gate against
   `media/ground_truth/bedroom_1/ground_truth.json`, times each
   reconstruction. Writes `docs/benchmark_report.md`,
   `docs/benchmark_results.json`, `docs/timing.md`.
2. `scripts/generate_device_matrix.py` — writes `docs/device_matrix.md`
   from the results just produced.
3. `scripts/run_head_to_head.py` — compares our LiDAR-tier output
   against the parsed magicplan export and ground truth. Writes
   `docs/head_to_head_table.md`/`.csv` and `docs/head_to_head_report.md`.

To see which gate is worst and why, run:
```bash
.venv/bin/python3 scripts/rank_gates.py
```

---

## Part E — Capturing the competitor app (magicplan) for the head-to-head

1. Install **magicplan** (free tier) from the App Store.
2. Sign up -> **New Project** -> optionally add the address -> **Select
   Floor Plans** (e.g. "Ground Floor").
3. Tap **INSERT** -> **Room** -> **Auto-Scan** -> **Confirm Scan** (if
   not reviewing) -> **Done**.
4. Tap **Configure Floor Plan** -> **Generate Floor Plan**.
5. For additional rooms, repeat from step 3 ("tap INSERT" again).
6. Tap the **Share** button (top-right) -> **Report PDF** (this
   contains every room's measurements).
7. AirDrop the PDF to your Mac.

On the Mac, move the PDF into `media/competitor_benchmark/<app_name>/`.
The measurements need to be hand-transcribed into a small JSON file
next to it (`measurements.json`) since a PDF isn't structured data —
see `media/competitor_benchmark/magicplan/measurements.json` for the
existing example and `pipeline/competitor_parser.py` for the format it
expects.

---

## Part F — The fix loop

`docs/fix_loop_report.md` is the one-page declaration (worst gate, root
cause, fix, prediction, actual result). `results/fix_loop/` holds the
regenerable before/after runs (`before/command.txt` and
`after/command.txt` are the exact commands — the difference between the
two directories is purely the code state, checked out at the commit
before vs. after the fix). `results/fix_loop/diff.md` is a plain-English
summary.

---

## Part G — Where everything is (reference table)

| Topic | File |
|---|---|
| Requirement compliance | `docs/compliance_matrix.md` |
| Capture protocol (Route 2, one-pager) | `docs/capture_protocol.md` |
| Device matrix (generated, not hand-typed) | `docs/device_matrix.md` |
| Output JSON schema | `docs/schema.md` |
| Media folder conventions | `CLAUDE.md` (media/ section) |
| Benchmark gates + repeatability + timing | `docs/benchmark_report.md`, `docs/timing.md` |
| Head-to-head vs. magicplan | `docs/head_to_head_report.md` |
| Part 4 fix loop | `docs/fix_loop_report.md`, `docs/fix_loop_diagnosis.md`, `results/fix_loop/` |
| Technical report (max 6 pages) | `docs/technical_report.md` |
| Raw benchmark data inventory | `docs/raw_data_manifest.md` |
| Reproduction bundle detail | `docs/reproduction_bundle.md` |
| Multi-room manifest format | `docs/multi_room_manifest.md` |
| Running project log (what's done, what's open, why) | `CLAUDE.md` |

## Tests
```bash
.venv/bin/python3 -m pytest tests/ -q
```

## Constraints this project honors
Handheld consumer capture only. No pretrained model/dataset/API is
currently wired in (damage detection is heuristic CV, not a model call),
so there is nothing to disclose yet beyond OpenCV/matplotlib/numpy — if
a vision model is added later (see `docs/technical_report.md`, known
failure modes), it will be disclosed here. Nothing calls external
infrastructure at runtime — everything in this repo runs as a local CLI
against local files.
