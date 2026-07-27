# CLAUDE.md — RuView (WiFi-DensePose)

Guidance for Claude Code when working in this repository.

---

## 1. What this project is

RuView turns ordinary WiFi radio into a contactless sensor. ESP32-S3 nodes capture
**Channel State Information** (CSI) — the per-subcarrier complex channel response the
WiFi radio already computes — and stream it over UDP to a host. Host-side DSP turns
those perturbations into presence, breathing rate, heart rate, motion, fall events,
and (aspirationally) pose.

| Path | Language | Status |
|------|----------|--------|
| `v2/` | Rust workspace, ~40 crates | **Primary.** All active work. |
| `firmware/esp32-csi-node/` | C (ESP-IDF v5.4) | Node firmware. Dual-target ESP32-S3 / ESP32-C6. |
| `archive/v1/` | Python | Archived. Kept for the deterministic proof (`data/proof/verify.py`) and reference DSP. |
| `python/` | Rust + PyO3 | `ruview` / `wifi-densepose` PyPI wheels. |

---

## 2. Repository layout

```
firmware/esp32-csi-node/   ESP-IDF C firmware — CSI capture, tiered DSP, UDP sender,
                           NVS runtime config, OTA + WASM HTTP server (port 8032)
v2/crates/                 Rust workspace (see §3)
ui/                        Vanilla-JS dashboard + Observatory served by sensing-server
dashboard/                 Vite/TS dashboard (separate, newer)
examples/                  Runnable examples — see §7
scripts/                   Host-side Python/JS tooling (relays, recorders, trainers)
docs/adr/                  Architecture Decision Records — ADR-001..ADR-282 (with gaps)
docs/user-guide.md         2500-line operator guide — the single best doc to read
docs/TROUBLESHOOTING.md    11 real, reproduced field failures + fixes. Read before debugging.
plans/                     Implementation and test plans
archive/v1/data/proof/     Deterministic pipeline proof (SHA-256 "Trust Kill Switch")
vendor/                    Git submodules (rvcsi, rufield, ruvector, ...)
```

**Never write working files, scratch notes, or tests to the repo root.**

---

## 3. Key Rust crates (`v2/crates/`)

The workspace is large. These are the ones that matter day to day:

| Crate | Binary | What it does |
|-------|--------|--------------|
| `wifi-densepose-core` | — | `CsiFrame` and core types. Everything depends on this. |
| `wifi-densepose-signal` | — | DSP + the 16 RuvSense multistatic modules (`src/ruvsense/`) |
| `wifi-densepose-sensing-server` | `sensing-server` | **The hub.** UDP CSI in → DSP → REST + WebSocket + static UI out. |
| `wifi-densepose-cli` | `wifi-densepose` | `calibrate` / `calibrate-serve` / `enroll` / `train-room` / `room-status` / `room-watch` (+ `mat` behind a feature) |
| `wifi-densepose-calibration` | — | ADR-151 per-room baseline → enrollment → specialist bank. Pure Rust, edge-deployable. |
| `wifi-densepose-vitals` | — | Breathing / heart-rate extraction (ADR-021) |
| `wifi-densepose-hardware` | — | ESP32 aggregator, TDM protocol, ADR-018 frame decode |
| `wifi-densepose-ruvector` | — | RuVector integration + cross-viewpoint fusion (`src/viewpoint/`) |
| `wifi-densepose-mat` | — | Mass Casualty Assessment Tool |
| `wifi-densepose-train` | — | Training pipeline, MAE pretraining, WiFlow-STD port |
| `ruview-unified` | — | ADR-273..282 unified RF spatial world model. Pure leaf; all numbers L0/synthetic. |

RuvSense modules live in `wifi-densepose-signal/src/ruvsense/`: `multiband`,
`phase_align`, `multistatic`, `coherence`, `coherence_gate`, `pose_tracker`,
`field_model`, `tomography`, `longitudinal`, `intention`, `cross_room`, `gesture`,
`adversarial`, `cir`, `calibration`.

Excluded from the workspace (build separately, `wasm32-unknown-unknown` targets):
`wifi-densepose-wasm-edge`, `homecore-plugin-example`, `crates/worldgraph`.

---

## 4. This machine (verified 2026-07-27)

A snapshot — re-verify before trusting it.

| Thing | State |
|-------|-------|
| OS | Windows 11 Education, 10.0.22631 |
| Shell | PowerShell 5.1 primary; Git Bash also available |
| Python | 3.11.15 at `C:\Espressif\tools\python\python.exe` — numpy 2.4.6, scipy 1.17.1, pyserial 3.5, paho-mqtt 2.1.0, websockets 16.1.1. **No esptool.** |
| ESP-IDF venv Python | `C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe` — **has esptool 5.3.1** |
| ESP-IDF | **v6.0.2** at `C:\esp\v6.0.2\esp-idf`. Firmware targets **v5.4** — see §5. |
| Docker | CLI 29.2.1 installed, **daemon not running** |
| Rust / cargo | **Not installed** |
| Node / npm | **Not installed** |
| Git submodules | **Not initialised** — see §6 |

### Hardware inventory

Two ESP32-S3 boards connected at scan time; six total available.

| Port | MAC | Chip |
|------|-----|------|
| COM3 | `f4:12:fa:9d:b6:a4` | ESP32-S3 (QFN56) rev v0.1 |
| COM4 | `f4:12:fa:9d:b6:d4` | ESP32-S3 (QFN56) rev v0.1 |

eFuse readout — authoritative, not inferred from the board silkscreen:

```
FLASH_CAP  = 8M       FLASH_VENDOR   = GD
PSRAM_CAP  = None     PSRAM_CAPACITY = 0
USB mode   = USB-Serial/JTAG   (USB VID 303A / PID 1001)
```

**8 MB in-package flash, zero PSRAM, native-USB only** — an `ESP32-S3FN8`-class
part. Three load-bearing consequences:

1. **Tier 3 WASM programmable sensing is unavailable.** It allocates 4 × 160 KB
   PSRAM arenas and there is no PSRAM. Independently, `components/wasm3/wasm3-src/`
   is absent from the tree, so WASM3 has to be fetched at build time anyway.
2. **The AMOLED display path is irrelevant** — and actively dangerous, see §6.3.
3. **The default ESP-IDF console is UART0**, which is not routed to the native USB
   port. Without `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` the serial monitor is
   silent on these boards, and every "is it alive?" check fails misleadingly.

Tier 0 (raw CSI), Tier 1 (phase unwrap + Welford + top-K) and Tier 2 (vitals,
presence, fall) are all fully available.

---

## 5. Build & test

### Rust

The workspace pins **Rust 1.89** via `v2/rust-toolchain.toml`. On Windows this needs
the MSVC linker (Visual Studio Build Tools, C++ workload).

```bash
# One-time. cargo FAILS without this — ruv-neural, ruview-swarm and worldgraph
# are workspace members whose directories are currently empty.
git submodule update --init --recursive

cd v2
cargo test --workspace --no-default-features        # ~2 min
cargo build --release -p wifi-densepose-sensing-server
cargo build --release -p wifi-densepose-cli --no-default-features
```

`--no-default-features` matters in two distinct ways:

- On the **workspace**, it drops the `ndarray-linalg` / OpenBLAS chain that would
  otherwise need vcpkg on Windows (issues #366, #415).
- On **`wifi-densepose-cli`**, it drops the `mat` feature → drops
  `wifi-densepose-nn` → drops `ort` (ONNX) → drops `openssl-sys`. The calibration
  path needs none of it.

`wifi-densepose-sensing-server` has `default = []`, so a plain `-p` build of it is
already Windows-friendly.

### Python (no Rust needed)

```bash
python archive/v1/data/proof/verify.py            # must print "VERDICT: PASS"
python archive/v1/data/proof/verify.py --generate-hash   # if numpy/scipy drifted
```

### Firmware

Both paths work; pick based on what is running.

**Path A — Docker** (what the repo documents; needs the daemon up):

```bash
MSYS_NO_PATHCONV=1 docker run --rm \
  -v "$(pwd)/firmware/esp32-csi-node:/project" -w /project \
  espressif/idf:v5.4 bash -c \
  "rm -rf build sdkconfig && idf.py set-target esp32s3 && idf.py build"
```

**Path B — native Windows, no Docker.** ESP-IDF works fine from **PowerShell**; the
repo's "Docker is the only reliable method" warning is specifically about Git Bash /
MSYS2, where `idf.py` detects `MSYSTEM` and silently skips `main()`. This machine has
ESP-IDF **v6.0.2**, but the firmware targets **v5.4** and is not known to build on
v6 — install v5.4 side-by-side (the ESP-IDF installer supports multiple versions)
and export it before building. `firmware/esp32-csi-node/build_firmware.ps1` is a
working template whose paths are hardcoded to a different machine and must be edited.

Flash offsets must match `partitions_display.csv` (8 MB):

```
bootloader       0x0
partition-table  0x8000
nvs              0x9000
otadata          0xf000
app (ota_0)      0x20000
```

Use the ESP-IDF venv Python — it is the one with esptool:

```powershell
$py = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
& $py -m esptool --chip esp32s3 --port COM3 --baud 460800 `
  write_flash --flash_mode dio --flash_size 8MB `
  0x0     build/bootloader/bootloader.bin `
  0x8000  build/partition_table/partition-table.bin `
  0xf000  build/ota_data_initial.bin `
  0x20000 build/esp32-csi-node.bin
```

Runtime config goes to NVS without reflashing:

```powershell
& $py firmware/esp32-csi-node/provision.py --port COM3 `
  --ssid "SSID" --password "PASS" --target-ip 192.168.1.20 `
  --node-id 1 --edge-tier 2
```

---

## 6. Gotchas that will cost you hours

All verified in this tree, not folklore.

1. **Submodules are not initialised.** `cargo test --workspace` fails immediately.
   Run `git submodule update --init --recursive` first.

2. **The prebuilt `release_bins/` are stale and unsafe for these boards.**
   They are v0.6.7, committed 2026-06-02. The display false-positive fix (`#1000`,
   commit `65e29ef4`) landed 2026-06-17, and `firmware/esp32-csi-node/version.txt`
   now reads **0.8.4**. Flashing the prebuilt binaries onto a display-less board
   risks exactly the 0 pps CSI starvation that fix addresses. **Build from source.**

3. **`CONFIG_DISPLAY_ENABLE` defaults to `y`.** The SH8601 QSPI panel is write-only,
   so `display_hal_init_panel()` "succeeds" on a bare board. Current source recovers
   by requiring FT3168 touch readback and bailing to headless
   (`main/display_task.c:121`), which lets `main.c:495` call
   `csi_collector_enable_data_capture()`. If that upgrade does not happen the
   promiscuous filter stays MGMT-only and **CSI yield collapses to 0 pps** while the
   node still looks healthy on the network. Build with the
   `sdkconfig.defaults.devkitc` overlay to compile display support out entirely and
   make `has_display` a compile-time false.

4. **Serial console is silent on native-USB boards.** Add
   `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` or you get no boot log on COM3/COM4.

5. **Docker Desktop on Windows drops UDP from all but one source IP.** With 6 nodes
   you lose 5. `--network host` does not help (not implemented for the Linux engine
   on Windows). Either run `sensing-server` natively, or run
   `python scripts/udp-relay.py --listen-port 5005 --forward-port 5006` and remap the
   container to `5006:5005/udp`. Full writeup: `docs/TROUBLESHOOTING.md` §9.

6. **`sensing-server` 404s on the UI** when launched from the wrong cwd — the default
   `--ui-path ../ui` is cwd-relative. Pass an absolute path.

7. **Only one process can bind UDP :5005.** The standalone aggregator, the
   calibration CLI, and `sensing-server` all want it. Run one at a time, or use
   `scripts/csi-udp-relay.py` to fan out to a loopback port.

8. **Windows Firewall blocks inbound LAN UDP** to a freshly-built `.exe`.
   `python.exe` is usually already allowed — `scripts/csi-udp-relay.py` exists
   precisely to exploit that, and needs no admin.

9. **`edge_tier` and CSI are not independent.** `calibrate` / `enroll` / `room-watch`
   need **raw CSI (`--edge-tier 0`)**. Tier 2 gives you the 32-byte vitals packet
   instead. Provision per experiment.

---

## 7. Examples worth knowing

| Path | Needs hardware? | What it gives you |
|------|-----------------|-------------------|
| `scripts/synth-csi-udp.py` | No | Emits real ADR-018 `0xC5110001` frames at 20 Hz. Exercises the whole host pipeline with no ESP32. **Best first test.** |
| `archive/v1/data/proof/verify.py` | No | Deterministic SHA-256 pipeline proof |
| `examples/research-sota/` | No | 9 folders of pure-NumPy physics derivations. `02-placement/` is the practical node-siting recipe. |
| `examples/ruview_live.py` | Yes | Unified live dashboard, CSI + optional mmWave |
| `examples/through-wall/wiflow_browser.html` | Yes | Self-contained browser capture → train → infer loop |
| `scripts/record-csi-udp.py` | Yes | Records raw CSI to JSONL for offline replay |
| `scripts/udp-relay.py` | Yes | The Docker-on-Windows multi-node workaround |

---

## 8. Wire protocols

UDP to the aggregator. First 4 bytes are the magic.

| Magic | Name | Rate | Size |
|-------|------|------|------|
| `0xC5110001` | CSI frame (ADR-018) | ~20 Hz | 20-byte header + I/Q pairs |
| `0xC5110002` | Vitals packet | 1 Hz | 32 bytes |
| `0xC5110003` | Seed feature vector | 1 Hz | 48 bytes |
| `0xC5110004` | WASM module output | event | variable |

ADR-018 header: `magic(4) · node_id(1) · n_ant(1) · n_sc(2 LE) · freq_mhz(4 LE) ·
seq(4 LE) · rssi(i8) · noise(i8) · reserved(2)`, then `n_ant × n_sc × 2` bytes of
signed I/Q.

---

## 9. Honesty rules

This repo makes a lot of claims and is unusually good about labelling which are
real. Preserve that. When you touch a claim, keep its evidence tag.

Load-bearing distinctions currently documented in-tree:

- **Firmware presence detection is a variance threshold, not a model.** It
  false-positives on fans, microwaves, and AP power swings. The firmware README says
  so explicitly — do not let a doc edit quietly upgrade it.
- **Firmware `n_persons` is a slot-capacity heuristic** over top-K subcarrier groups
  (`main/edge_processing.c:481`). It tracks subcarrier diversity, not people.
- **Without `--model`, the sensing server's on-screen skeleton is driven by signal
  heuristics**, not learned keypoint inference.
- **The Hugging Face model ships as JSONL RVF; `--model` only parses binary RVF.**
  Pointing `--model` at `model.rvf.jsonl` fails with a magic mismatch and the
  pipeline degrades to null output. Run without `--model` until a JSONL adapter lands.
- **`examples/research-sota/` numbers are synthetic-physics derivations**, not bench
  measurements. Real deployments diverge 5–30%.
- **`ruview-unified` (ADR-273..282) numbers are evidence level L0** — synthetic until
  real-data validation.

If you produce a new number, state how it was measured and on what.

---

## 10. Working rules

- Do what was asked; nothing more, nothing less.
- Read a file before editing it.
- Prefer editing an existing file to creating a new one.
- Never create `*.md` documentation proactively — only when asked.
- Never write to the repo root. Use `docs/`, `plans/`, `v2/crates/`, `scripts/`.
- Keep source files under ~500 lines.
- Never commit secrets, credentials, or `.env` files.
- Run tests after code changes; verify the build before committing.
- Commit or push only when asked. If on `main`, branch first.

### Pre-merge checklist

1. `cd v2 && cargo test --workspace --no-default-features` — 0 failed
2. `python archive/v1/data/proof/verify.py` — `VERDICT: PASS`
3. Update `README.md` / this file if scope changed
4. Add a `CHANGELOG.md` entry under `[Unreleased]`
5. Update `docs/user-guide.md` if CLI flags, data sources, or setup changed
6. Regenerate the witness bundle if tests or the proof hash changed:
   `bash scripts/generate-witness-bundle.sh`
7. Add new build artifacts to `.gitignore`

---

## 11. Pointers

- **6-node ESP32-S3 bring-up and test plan:** `plans/ruview-esp32s3-test-plan.md`
  (browsable copy: `plans/ruview-esp32s3-test-plan.html`)
- **Operator guide:** `docs/user-guide.md`
- **Field failures + fixes:** `docs/TROUBLESHOOTING.md`
- **Firmware detail:** `firmware/esp32-csi-node/README.md`
- **Per-room calibration design:** `docs/adr/ADR-151-*.md`
- **Multistatic mesh design:** `docs/adr/ADR-029-*.md`, `ADR-032-*.md`
- **Frame format:** `docs/adr/ADR-018-*.md`
