# RuView — 6-Node ESP32-S3 Test Plan

**Target:** 6 × ESP32-S3, 8 MB flash, no PSRAM, native USB · Windows 11 host
**Written:** 2026-07-27 · against repo state `2cc378c1`, firmware `v0.8.4`
**Scope chosen:** Core CSI + vitals · Per-room calibration & training · Multistatic mesh & through-wall

---

## How to use this document

The plan runs in **10 phases**. Phases 0–2 need **no hardware at all** — do them
first, because they teach you what "correct" looks like before an antenna is
involved. Phase 3 onward is hardware.

Every step has the same shape:

> **Goal** — what you are trying to establish
> **Why** — what breaks later if you skip it
> **Do** — exact commands
> **Expect** — what a healthy result looks like
> **✅ Pass / ❌ Fail** — the decision rule
> **If it fails** — the specific fix

**Do not skip a Pass gate.** Almost every hard-to-debug WiFi-sensing failure is a
skipped gate three phases earlier. The single most common one in this project is
"the node is on the network and looks alive, but CSI yield is 0 pps."

### Notation

- `PS>` = PowerShell prompt. This is the primary shell on your machine.
- `$py` = the ESP-IDF venv Python, which is the one that has `esptool`:
  ```powershell
  $py = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
  ```
- `$repo` = `C:\Users\otaia0a\Desktop\RuView`
- `<HOST_IP>` = your laptop's LAN IPv4 address. Find it with `ipconfig`. It must
  be on the **same subnet** as the WiFi network the ESP32s join.

### Time budget

| Phase | What | Time |
|-------|------|------|
| 0 | Concepts | 30 min reading |
| 1 | Host toolchain | 45–90 min (mostly waiting on installs) |
| 2 | No-hardware validation | 45 min |
| 3 | Build firmware | 30–60 min |
| 4 | First node alive | 45 min |
| 5 | First real CSI | 30 min |
| 6 | Vitals | 60 min |
| 7 | Per-room calibration & training | 90 min |
| 8 | Scale to 6 nodes | 2–3 h |
| 9 | Multistatic & through-wall | 3–4 h |
| 10 | Record, analyse, report | 2 h |

---

# Phase 0 — Concepts

Read this once. It is the vocabulary the rest of the plan uses. No commands.

## 0.1 What CSI actually is

Your WiFi radio does not receive one signal. An 802.11n 20 MHz channel is split
into **56 subcarriers** — 56 narrow frequency slots transmitted in parallel. To
decode a packet the receiver must estimate, for every subcarrier, how the channel
distorted it. That estimate is a complex number: an **amplitude** (how much the
signal was attenuated) and a **phase** (how much it was delayed).

That per-subcarrier complex vector is **Channel State Information**. Every WiFi
chip computes it. Almost none of them let you read it. The ESP32 does.

```
        transmitted                     received
   ┌─────────────────┐             ┌─────────────────┐
   │ ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁ │  ──────►    │ ▁▃▅▂▇▁▄▆▂▁▅▃▁▂▄ │   56 subcarriers
   └─────────────────┘   the room   └─────────────────┘
                          changes
                            it
```

**Why a human shows up in it.** Radio waves reflect off you. Your body is mostly
water, which is a strong reflector at 2.4 GHz. The signal reaching the receiver is
the sum of the direct path plus every reflected path. Move — or just *breathe* —
and the reflected path lengths change by millimetres, which changes the phase of
the sum. Chest wall motion during breathing is 4–12 mm. The wavelength at
2.4 GHz is 125 mm. A 12 mm path change is ~35° of phase. That is enormous
compared to the noise floor.

**This is why the system works through walls and in the dark.** It is not seeing
you. It is measuring how you perturb a radio field that already fills the room.

## 0.2 What each thing in this repo does

```
┌────────────┐   UDP :5005   ┌──────────────────┐   HTTP :3000  ┌─────────┐
│  ESP32-S3  │ ────────────► │  sensing-server  │ ────────────► │ Browser │
│   node     │  ADR-018      │  (Rust, host)    │  WebSocket    │   UI    │
└────────────┘  CSI frames   └──────────────────┘               └─────────┘
      │                              │
      │ on-device DSP                │ host DSP
      │ (tiers 0/1/2)                │ (RuvSense, vitals, fusion)
      ▼                              ▼
  32-byte vitals               presence / breathing / heart rate
  packet @ 1 Hz                motion / falls / multi-person
```

The **firmware tiers** are a compile-time and runtime choice on the node:

| Tier | What runs on the ESP32 | What goes over the wire | Use it for |
|------|------------------------|-------------------------|------------|
| **0** | Nothing — raw passthrough | Full CSI frames, ~20 Hz, ~5 KB/s | **Calibration, enrollment, training, host-side DSP** |
| **1** | Phase unwrap, Welford stats, top-K subcarriers | Compressed CSI | Bandwidth-constrained links |
| **2** | Full DSP: vitals, presence, fall detect | 32-byte vitals packet @ 1 Hz + CSI | **Standalone node, no host DSP** |
| **3** | WASM modules | Custom events | ❌ **Not available on your boards** — needs PSRAM |

You will switch between tier 0 and tier 2 during this plan. That switch is a
runtime NVS setting; **it does not require reflashing.**

## 0.3 What is real vs. what is a demo

This repo is unusually honest about this, and you should hold it to that standard.
Going in with correct expectations:

**Solid — expect these to work:**
- Raw CSI streaming at ~20 Hz
- Breathing rate (0.1–0.5 Hz band → 6–30 BPM) on a still subject
- Motion / activity detection
- The ADR-151 per-room calibration → enrollment → specialist bank pipeline
- Multistatic fusion of several nodes

**Weaker — expect these to be noisy:**
- Heart rate (0.8–2.0 Hz band). Much lower SNR than breathing; needs a still
  subject close to the node.
- Presence. The firmware's is a **variance threshold, not a model** — it
  false-positives on fans, microwaves, and neighbouring-AP power swings.
- Person counting. The firmware's `n_persons` is a **slot-capacity heuristic**
  over top-K subcarrier groups (`main/edge_processing.c:481`). It tracks
  subcarrier diversity, not people. `docs/TROUBLESHOOTING.md` §2 documents it
  reading 6–8 for 5 bodies.

**Not real yet — do not spend time here:**
- **17-keypoint pose from CSI alone.** The committed on-device model scores
  PCK@20 = 3.0% against a ≥35% target and its runtime path is a `confidence=0`
  stub. Without `--model`, the on-screen skeleton is driven by amplitude-variance
  heuristics, not learned inference. The impressive 82.69% MM-Fi number is a
  *separate published benchmark* on a research dataset, not this live path.
- **Loading the Hugging Face weights into `sensing-server`.** The HF bundle ships
  as JSONL RVF; `--model` only parses binary RVF. It errors with a magic mismatch
  and degrades to null output. Run **without** `--model`.
- **Tier 3 WASM.** No PSRAM on your boards.

## 0.4 Your specific hardware

Read from the chips' eFuses, not from a datasheet guess:

```
Chip           ESP32-S3 (QFN56) rev v0.1
Flash          8 MB, in-package, GigaDevice
PSRAM          None    (PSRAM_CAP = None, PSRAM_CAPACITY = 0)
USB            USB-Serial/JTAG, native      (VID 303A / PID 1001)
MACs seen      f4:12:fa:9d:b6:a4 (COM3)
               f4:12:fa:9d:b6:d4 (COM4)
```

Three things follow, and all three will bite you if ignored:

1. **No PSRAM** → Tier 3 WASM is out, and the AMOLED display code has no
   framebuffer memory. You will compile both out.
2. **Native USB only** → the default ESP-IDF console goes to UART0, which is not
   wired to this USB port. Without a config change your serial monitor is
   **silent**, and you will misdiagnose a working board as dead.
3. **Display probe false-positive** → the SH8601 panel driver is write-only, so
   its init "succeeds" on a board with no panel. When the firmware thinks it has a
   display it keeps the promiscuous filter at MGMT-only, and **CSI yield collapses
   to 0 pps** while the node still associates, still pings, and still looks
   perfectly healthy. This is the single nastiest failure mode in this project.
   You will compile display support out to make it impossible.

---

# Phase 1 — Host toolchain

**Goal:** a machine that can build the firmware, build the Rust server, and flash
a board.

## Step 1.1 — Inventory what you already have

**Do:**
```powershell
PS> $py = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
PS> & $py --version
PS> & $py -m esptool version
PS> python --version
PS> git --version
PS> cargo --version
PS> docker version --format '{{.Server.Version}}'
```

**Expect (as of the scan of your machine):**

| Tool | State | Action |
|------|-------|--------|
| ESP-IDF venv Python 3.11 + esptool 5.3.1 | ✅ present | none |
| Base Python 3.11.15 + numpy/scipy/pyserial | ✅ present | none |
| Git | ✅ present | none |
| **Rust / cargo** | ❌ absent | **Step 1.3** |
| **Docker daemon** | ❌ not running | optional, Step 1.5 |
| ESP-IDF **v6.0.2** at `C:\esp\v6.0.2\esp-idf` | ⚠️ wrong version | firmware needs **v5.4** — Step 1.4 |

## Step 1.2 — Initialise git submodules

**Goal:** make the Rust workspace buildable.
**Why:** `v2/crates/ruv-neural`, `ruview-swarm` and `worldgraph` are declared as
workspace members but their directories are **empty**. Cargo will refuse to parse
the workspace at all. This is a 100%-reproducible failure, not a maybe.

**Do:**
```powershell
PS> cd $repo
PS> git submodule update --init --recursive
```

**Expect:** several minutes of cloning; `vendor/rvcsi`, `vendor/rufield`,
`vendor/ruvector`, `v2/crates/worldgraph`, `v2/crates/ruv-neural`,
`v2/crates/ruview-swarm` all become non-empty.

**✅ Pass:** `git submodule status` shows no leading `-` on any line.
**❌ Fail:** leading `-` remains → that submodule did not clone. Retry it
individually: `git submodule update --init v2/crates/worldgraph`.

## Step 1.3 — Install Rust

**Goal:** `cargo` available, MSVC linker present.
**Why:** the sensing server, the calibration CLI, and the whole multistatic path
are Rust. The workspace pins **Rust 1.89** in `v2/rust-toolchain.toml`; rustup will
fetch that automatically once installed.

**Do — two parts, in order.**

**(a) MSVC build tools** (the linker). Skip only if you already have Visual Studio
with the C++ workload.
```powershell
PS> winget install --id Microsoft.VisualStudio.2022.BuildTools `
      --override "--quiet --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

**(b) Rust:**
```powershell
PS> winget install --id Rustlang.Rustup
# close and reopen PowerShell so PATH updates
PS> rustup default stable
PS> cargo --version
```

**Expect:** `cargo 1.x.y`. When you later `cd v2`, rustup reads
`rust-toolchain.toml` and silently installs 1.89 on first build.

**✅ Pass:** `cargo --version` prints a version.
**❌ Fail — `link.exe not found`:** the C++ workload did not install. Open
"Visual Studio Installer" → Modify → check **Desktop development with C++**.

## Step 1.4 — ESP-IDF v5.4 (only if building firmware natively)

**Goal:** an ESP-IDF v5.4 environment.
**Why:** the firmware targets v5.4. You have v6.0.2. ESP-IDF major versions carry
breaking API changes and this firmware is not known to build on v6.

You have two routes; **either is fine**, pick one:

| Route | Needs | Notes |
|-------|-------|-------|
| **A — Docker** | Docker daemon running | What the repo documents. Zero IDF setup. Go to Step 1.5. |
| **B — native v5.4** | ~2 GB disk, 30 min | No Docker needed. Works today. |

**Route B — install v5.4 side by side:**

Run the ESP-IDF Windows installer (`esp-idf-tools-setup-*.exe` from Espressif) and
choose version **v5.4**, install path `C:\esp\v5.4`. It coexists with v6.0.2 —
they are separate directories with separate tool sets.

Then, to build, open PowerShell and export v5.4 explicitly:
```powershell
PS> $env:IDF_PATH        = 'C:\esp\v5.4\esp-idf'
PS> $env:IDF_TOOLS_PATH  = 'C:\Espressif\tools'
PS> & 'C:\esp\v5.4\esp-idf\export.ps1'
PS> idf.py --version
```

**Expect:** `ESP-IDF v5.4` or `v5.4.x`.

> **Note on the repo's "Docker is the only reliable method" warning.** That warning
> is specifically about **Git Bash / MSYS2**, where `idf.py` detects the `MSYSTEM`
> environment variable and silently skips `main()` — the build appears to do
> nothing. **PowerShell is unaffected.** Build from PowerShell, not Git Bash.

**✅ Pass:** `idf.py --version` reports 5.4.
**❌ Fail — command not found:** `export.ps1` did not run in this shell. It only
affects the current session; re-run it in every new terminal.

## Step 1.5 — Docker (optional, for tomorrow)

If you start Docker Desktop later, you gain the containerised firmware build and
the prebuilt `ruvnet/wifi-densepose` server image.

**Do:** start Docker Desktop, then `docker ps` to confirm the daemon answers.

> ⚠️ **When you do use Docker with more than one node, read this.** Docker Desktop
> on Windows runs the engine inside a WSL2/Hyper-V VM. Inbound UDP from multiple
> source IPs gets demultiplexed onto a single virtual socket — **the first source
> IP wins and every other node is silently dropped.** With 6 nodes you would see 1.
> `--network host` does not help (not implemented for the Linux engine on Windows).
>
> Fix: run the bundled relay on the host so every datagram arrives from the same
> loopback source, and remap the container port:
> ```powershell
> PS> python scripts\udp-relay.py --listen-port 5005 --forward-port 5006
> # then in docker/docker-compose.yml change  "5005:5005/udp"  →  "5006:5005/udp"
> ```
> Full detail: `docs/TROUBLESHOOTING.md` §9. **Running `sensing-server` natively
> avoids this entirely**, which is why this plan builds it natively.

## Step 1.6 — Build the Rust binaries

**Do:**
```powershell
PS> cd $repo\v2
PS> cargo build --release -p wifi-densepose-sensing-server
PS> cargo build --release -p wifi-densepose-cli --no-default-features
```

**Why `--no-default-features` on the CLI:** it drops the `mat` feature, which
otherwise pulls `wifi-densepose-nn` → `ort` (ONNX Runtime) → `openssl-sys`. None
of that is needed for calibration, and `openssl-sys` is a notorious Windows build
failure. You want the calibration path, so drop it.

**Expect:** first build takes 10–20 minutes (it compiles ~400 crates). Then:
```
target\release\sensing-server.exe
target\release\wifi-densepose.exe
```

**✅ Pass:**
```powershell
PS> .\target\release\wifi-densepose.exe version
PS> .\target\release\sensing-server.exe --help
```
both print output.

**❌ Fail — errors about `ndarray-linalg` / OpenBLAS / vcpkg:** add
`--no-default-features` to the sensing-server build too. That drops the optional
BLAS chain (issues #366, #415).

**❌ Fail — "failed to load manifest for workspace member":** Step 1.2 did not
complete. Go back and initialise submodules.

---

# Phase 2 — Validate with no hardware

**Goal:** prove the host pipeline is correct *before* introducing an antenna.
**Why:** when Phase 5 gives you a weird number, you need to already know whether
the host math is trustworthy. This phase establishes that.

## Step 2.1 — The deterministic proof ("Trust Kill Switch")

**Goal:** confirm the signal-processing pipeline produces bit-identical output to
the published reference.

**Do:**
```powershell
PS> cd $repo
PS> python archive\v1\data\proof\verify.py
```

This feeds 1,000 synthetic CSI frames (seed=42) through the production pipeline
and SHA-256s the feature output, comparing against a committed hash.

**Expect:** `VERDICT: PASS`

**✅ Pass:** `VERDICT: PASS`.
**❌ Fail — hash mismatch:** usually a numpy/scipy version difference, not a bug.
Regenerate and re-verify:
```powershell
PS> python archive\v1\data\proof\verify.py --generate-hash
PS> python archive\v1\data\proof\verify.py
```
If it fails a second time with the freshly generated hash, something is genuinely
wrong — stop and investigate.

## Step 2.2 — Rust test suite

**Do:**
```powershell
PS> cd $repo\v2
PS> cargo test --workspace --no-default-features
```

**Expect:** ~2 minutes, 1,400+ tests, **0 failed**.

**✅ Pass:** `test result: ok` with 0 failures.
**❌ Fail:** note which crate. If it is a submodule crate (`ruview-swarm`,
`worldgraph`), it does not block anything in this plan — proceed, but write it
down. If it is `wifi-densepose-signal`, `-calibration`, or `-sensing-server`,
**stop** — those are on your critical path.

## Step 2.3 — Fake an ESP32 and run the whole pipeline

**Goal:** watch real CSI frames flow into the real server, with no hardware.
**Why:** this is the single highest-value 10 minutes in the plan. It teaches you
what a healthy `/api/v1/sensing/latest` looks like, so that in Phase 5 you can
tell "the ESP32 isn't sending" from "the server isn't parsing".

`scripts/synth-csi-udp.py` emits **genuine ADR-018 `0xC5110001` frames** — same
header layout, same I/Q encoding, 20 Hz, 52 subcarriers, `node_id=42`. The server
cannot tell it from a real node.

**Do — three terminals.**

*Terminal 1 — the server:*
```powershell
PS> cd $repo\v2
PS> .\target\release\sensing-server.exe `
      --source esp32 --udp-port 5005 --http-port 3000 --ws-port 3001 `
      --ui-path C:\Users\otaia0a\Desktop\RuView\ui
```

> **Always pass an absolute `--ui-path`.** The default `../ui` is resolved against
> your *current working directory*, not the binary location, so the UI 404s from
> anywhere else (`docs/TROUBLESHOOTING.md` §10).

*Terminal 2 — the fake node:*
```powershell
PS> cd $repo
PS> python scripts\synth-csi-udp.py --host 127.0.0.1 --port 5005 `
      --duration-s 120 --rate-hz 20 --motion-after-s 30
```

`--motion-after-s 30` injects amplitude jitter after 30 s — a synthetic "person
walks in" event. Watch the server react to it.

*Terminal 3 — poke the API:*
```powershell
PS> curl.exe http://localhost:3000/health
PS> curl.exe http://localhost:3000/api/v1/sensing/latest
PS> curl.exe http://localhost:3000/api/v1/nodes
PS> curl.exe http://localhost:3000/api/v1/vital-signs
```

**Expect:**
- `/health` → `{"status":"ok","source":"esp32",...}`
- `/api/v1/nodes` → node `42` present
- `/api/v1/sensing/latest` → arrays of amplitudes and phases, ~52 entries
- After t+30 s, motion-related fields rise noticeably

Open `http://localhost:3000/ui/index.html` and `http://localhost:3000/ui/observatory.html`.

**✅ Pass:** node 42 appears in `/api/v1/nodes`, amplitude arrays are non-empty
and *changing between polls*, and the motion injection is visible.
**❌ Fail — node list empty:** Windows Firewall. Loopback is normally exempt; if
prompted, allow `sensing-server.exe` on **Private** networks.
**❌ Fail — 404 on the UI:** absolute `--ui-path`, see above.

> 📌 **Write down** what a healthy `/api/v1/sensing/latest` looks like. Save the
> JSON. You will diff against it in Phase 5.

## Step 2.4 — Physics groundwork for node placement

**Goal:** decide *where* to put 6 nodes, from physics rather than vibes.
**Why:** placement dominates results. The repo's own study measured a **93×
sensing-coverage difference** between naive and physics-aware placement. Six nodes
badly placed will underperform two nodes well placed.

**Do:**
```powershell
PS> cd $repo
PS> python examples\research-sota\01-physics-floor\r6_fresnel_zone.py
PS> python examples\research-sota\02-placement\r6_2_2_multistatic_placement.py
PS> python examples\research-sota\02-placement\r6_2_5_multi_subject.py
PS> python examples\research-sota\02-placement\r6_2_3_chest_centric.py
```

Each is pure NumPy, runs in seconds, prints headline numbers, and writes a
`*_results.json` beside itself.

**Key results to internalise:**

| Finding | Meaning for your layout |
|---|---|
| 93× coverage lift from physics-aware placement | Placement > node count |
| 100% coverage of a 1–4 occupant household at **N=5 anchors** | **6 nodes is the right number.** You are at the plateau. |
| Chest-centric geometry matters | Mount near **chest height (~1.0–1.2 m)**, not floor or ceiling |
| ~50% breathing-band SNR cost from body multi-scatterer | Expect real breathing SNR ≈ half the idealised model |

> ⚠️ **Honest framing:** these are synthetic-physics derivations, not bench
> measurements. `examples/research-sota/README.md` says real deployments diverge
> **5–30%**. Use them to choose a layout, not to predict your accuracy.

**Deliverable:** sketch your test room. Mark 6 node positions and the intended
"subject" position. Keep it — Phase 8 turns it into a geometry JSON file.

---

# Phase 3 — Build the firmware

**Goal:** an `esp32-csi-node.bin` correct for a display-less, PSRAM-less,
native-USB ESP32-S3.

## Step 3.1 — Understand why you are not using the prebuilt binaries

`firmware/esp32-csi-node/release_bins/` contains ready-to-flash binaries. **Do not
use them.** The evidence:

| Fact | Source |
|---|---|
| `release_bins` are v0.6.7, committed **2026-06-02** | `git log -- firmware/esp32-csi-node/release_bins/` |
| The display false-positive fix landed **2026-06-17** | commit `65e29ef4`, issue #1000 |
| Current firmware source is **v0.8.4** | `firmware/esp32-csi-node/version.txt` |

The prebuilt binaries **predate the fix for exactly the failure mode your boards
are vulnerable to.** Building from source costs 20 minutes and removes the risk.

## Step 3.2 — Create the board overlay

**Goal:** a config that compiles out display and WASM, and routes the console to
native USB.

**Why each line:**

| Setting | Reason |
|---|---|
| `# CONFIG_DISPLAY_ENABLE is not set` | Makes `has_display` a **compile-time** false, so `main.c:495` always calls `csi_collector_enable_data_capture()`. This is the 0 pps insurance. |
| `# CONFIG_WASM_ENABLE is not set` | No PSRAM for the 4 × 160 KB arenas; `components/wasm3/wasm3-src/` is also absent. Saves ~120 KB flash. |
| `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` | Your only USB port is native USB-JTAG. Without this the boot log goes to UART0 pins and you see **nothing**. |

**Do:**
```powershell
PS> cd $repo\firmware\esp32-csi-node
PS> @'
# Overlay for ESP32-S3FN8 boards: 8 MB flash, NO PSRAM, native USB only.
# Layer on top of sdkconfig.defaults.

# No AMOLED panel. The SH8601 driver is write-only so its probe false-positives
# on a bare board; when the firmware believes it has a display it keeps the
# promiscuous filter at MGMT-only and CSI yield collapses to 0 pps (#893/#1000).
# Compiling it out makes has_display a compile-time false.
# CONFIG_DISPLAY_ENABLE is not set

# Tier 3 WASM needs 4 x 160 KB PSRAM arenas. These chips report PSRAM_CAP=None.
# CONFIG_WASM_ENABLE is not set

# Console over the native USB-Serial/JTAG peripheral, not UART0 pins.
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
'@ | Set-Content -Encoding utf8 sdkconfig.defaults.s3fn8
```

**✅ Pass:** `Get-Content sdkconfig.defaults.s3fn8` shows the four settings.

## Step 3.3 — Build

**Route A — native PowerShell (no Docker):**
```powershell
PS> cd $repo\firmware\esp32-csi-node
PS> $env:IDF_PATH = 'C:\esp\v5.4\esp-idf'
PS> & 'C:\esp\v5.4\esp-idf\export.ps1'
PS> Remove-Item -Recurse -Force build,sdkconfig -ErrorAction SilentlyContinue
PS> idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.s3fn8" set-target esp32s3
PS> idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.s3fn8" build
```

**Route B — Docker (when the daemon is up):**
```bash
$ MSYS_NO_PATHCONV=1 docker run --rm \
    -v "$(pwd)/firmware/esp32-csi-node:/project" -w /project \
    espressif/idf:v5.4 bash -c \
    "rm -rf build sdkconfig && \
     idf.py -DSDKCONFIG_DEFAULTS='sdkconfig.defaults;sdkconfig.defaults.s3fn8' set-target esp32s3 && \
     idf.py -DSDKCONFIG_DEFAULTS='sdkconfig.defaults;sdkconfig.defaults.s3fn8' build"
```

**Expect:** 3–10 minutes, ending with a binary-size summary. Artifacts:
```
build\bootloader\bootloader.bin
build\partition_table\partition-table.bin
build\ota_data_initial.bin
build\esp32-csi-node.bin          # ~820 KB with display+WASM removed
```

**✅ Pass:** all four files exist; app size well under the 2 MB `ota_0` partition.
**❌ Fail — `idf.py` does nothing, no output:** you are in Git Bash. `idf.py`
detects `MSYSTEM` and skips `main()`. Use PowerShell.
**❌ Fail — CMake errors about `wasm3-src`:** WASM did not get disabled. Confirm
your overlay is actually being read — the `-DSDKCONFIG_DEFAULTS` string must be
passed to **both** `set-target` and `build`.
**❌ Fail — IDF version errors:** you exported v6.0.2. Re-check `$env:IDF_PATH`.

## Step 3.4 — Archive this build

**Why:** you need six identical nodes. Rebuilding per node risks drift.

```powershell
PS> mkdir $repo\..\ruview-testbuild -Force
PS> Copy-Item build\bootloader\bootloader.bin,`
              build\partition_table\partition-table.bin,`
              build\ota_data_initial.bin,`
              build\esp32-csi-node.bin `
              $repo\..\ruview-testbuild\
PS> Get-FileHash $repo\..\ruview-testbuild\*.bin | Format-Table Hash,Path
```

Record those hashes in your log. Every node gets these exact bytes.

---

# Phase 4 — First node alive

**Goal:** one board flashed, on WiFi, and talking.

Do **one** node completely before touching the other five.

## Step 4.1 — Flash

**Do:**
```powershell
PS> $py = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
PS> cd $repo\firmware\esp32-csi-node
PS> & $py -m esptool --chip esp32s3 --port COM3 --baud 460800 `
      write_flash --flash_mode dio --flash_size 8MB `
      0x0     build\bootloader\bootloader.bin `
      0x8000  build\partition_table\partition-table.bin `
      0xf000  build\ota_data_initial.bin `
      0x20000 build\esp32-csi-node.bin
```

**Offsets are not negotiable** — they must match `partitions_display.csv`:

```
0x0       bootloader
0x8000    partition table
0x9000    nvs          ← provision.py writes here
0xf000    otadata
0x11000   phy_init
0x20000   ota_0  (app, 2 MB)
0x220000  ota_1  (app, 2 MB)
0x420000  spiffs
```

**Expect:** `Hash of data verified.` then `Hard resetting via RTS pin...`

**✅ Pass:** all four writes verify.
**❌ Fail — port busy:** close every serial monitor. Only one process can hold a
COM port.
**❌ Fail — `Failed to connect`:** hold **BOOT**, tap **RESET**, release BOOT, then
retry. Native-USB boards sometimes need manual bootloader entry.

## Step 4.2 — Confirm it boots

**Do:**
```powershell
PS> & $py -m serial.tools.miniterm COM3 115200
```
Press RESET on the board.

**Expect** (roughly — version strings will differ):
```
I (396) csi_collector: Early capture node_id=1 (before WiFi init)
I (406) main: ESP32-S3 CSI Node (ADR-018) — v0.8.4 — Node ID: 1
I (566) main: WiFi STA initialized, connecting to SSID: wifi-densepose
```

It will fail to connect — you have not provisioned credentials yet. That is fine.
**What you are testing here is that the console speaks at all.**

**✅ Pass:** you see boot text.
**❌ Fail — completely silent:** this is the native-USB console problem. Confirm
`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` made it into the build:
```powershell
PS> Select-String -Path build\config\sdkconfig.h -Pattern "USB_SERIAL_JTAG"
```
If absent, the overlay was not applied — return to Step 3.3.

Exit miniterm with `Ctrl+]`.

## Step 4.3 — Provision

**Goal:** give the node WiFi credentials, an identity, and a destination.

Settings go into the **NVS partition at 0x9000** and override the compiled-in
Kconfig defaults. **Changing them never requires reflashing the app.**

First get your host IP:
```powershell
PS> ipconfig | Select-String -Pattern "IPv4"
```

**Do** (substitute your values):
```powershell
PS> cd $repo
PS> & $py firmware\esp32-csi-node\provision.py --port COM3 `
      --ssid "YourWiFi" --password "YourPassword" `
      --target-ip 192.168.1.20 --target-port 5005 `
      --node-id 1 --edge-tier 0
```

**Why `--edge-tier 0`:** Phase 5 and Phase 7 need **raw CSI**. Tier 2 replaces
some of that with the compact 32-byte vitals packet. You will switch to tier 2 in
Phase 6 and back again — it is a one-command change.

**Key NVS keys** (full table in `firmware/esp32-csi-node/README.md`):

| Key | Flag | Meaning |
|-----|------|---------|
| `ssid` / `password` | `--ssid` / `--password` | WiFi credentials |
| `target_ip` / `target_port` | `--target-ip` / `--target-port` | Where CSI goes |
| `node_id` | `--node-id` | 0–255, **must be unique per node** |
| `edge_tier` | `--edge-tier` | 0 = raw, 1 = DSP, 2 = vitals |
| `tdm_slot` / `tdm_nodes` | `--tdm-slot` / `--tdm-total` | Mesh scheduling (Phase 8) |
| `chan_list` / `dwell_ms` | `--hop-channels` / `--hop-dwell` | Channel hopping |

**Expect:** `Flashing NVS partition (N bytes) to COM3...` then
`NVS provisioning complete!`

**✅ Pass:** completes without error.
**❌ Fail — "NVS partition generator not available":** the ESP-IDF venv Python
does not have it. Either export ESP-IDF first (so `IDF_PATH` is set and the
bundled `nvs_partition_gen.py` is found) or `& $py -m pip install esp-idf-nvs-partition-gen`.
**❌ Fail — WiFi trio warning:** you must supply `--ssid`, `--password` and
`--target-ip` together on a first provision.

## Step 4.4 — Confirm it joins the network

**Do:** reopen the serial monitor and press RESET.

**Expect:**
```
I (1023) main: Connected to WiFi
I (1025) main: CSI streaming active → 192.168.1.20:5005 (edge_tier=0, OTA=ready, WASM=off, ...)
```

`WASM=off` is correct and expected — you compiled it out.

**✅ Pass:** "Connected to WiFi" and a "CSI streaming active" line naming your host IP.
**❌ Fail — association loops:** 5 GHz-only SSID. ESP32-S3 is **2.4 GHz only**.
Use a 2.4 GHz SSID or a dual-band SSID with 2.4 GHz enabled.
**❌ Fail — connects, then nothing:** known post-flash limp state
(`docs/TROUBLESHOOTING.md` §1). Power-cycle: unplug USB, wait 2 s, replug.

---

# Phase 5 — First real CSI

**Goal:** real frames from real hardware, at a healthy rate.

## Step 5.1 — Watch the raw wire

**Goal:** confirm packets physically arrive, before any parsing is involved.
**Why:** this splits "the node isn't sending" from "the server can't parse" —
the two failures look identical from the UI.

**Do:**
```powershell
PS> cd $repo
PS> python scripts\record-csi-udp.py --duration 30 --output data\recordings
```

**Expect:** a running count of received packets, ~20/s, and a JSONL file written.

**✅ Pass:** ≥ 500 packets in 30 s (≈17 pps or better).
**❌ Fail — zero packets:**
1. **Firewall.** Most likely cause. Allow `python.exe` inbound on Private
   networks, or temporarily test with the firewall profile off.
2. **Wrong `target_ip`.** Re-check with:
   ```powershell
   PS> & $py firmware\esp32-csi-node\provision.py --port COM3 --state
   ```
   That prints the merged NVS state without flashing anything.
3. **Client isolation / AP isolation** on your router — blocks station-to-station
   traffic. Disable it, or use a phone hotspot for testing.

**❌ Fail — packets arrive but at ~0–2 pps:** ⚠️ **This is the display-probe
starvation.** Verify `CONFIG_DISPLAY_ENABLE` really is unset:
```powershell
PS> Select-String -Path build\config\sdkconfig.h -Pattern "DISPLAY_ENABLE"
```
Expect **no** `#define CONFIG_DISPLAY_ENABLE 1`. If it is there, rebuild with the
overlay (Step 3.3).

> **Also worth knowing:** CSI frames only generate when there is WiFi *traffic* to
> measure. A totally idle network yields few frames. If the rate is low but
> non-zero, generate traffic — start a large download, or ping the AP in a loop.

## Step 5.2 — Feed it to the server

**Do — Terminal 1:**
```powershell
PS> cd $repo\v2
PS> .\target\release\sensing-server.exe `
      --source esp32 --udp-port 5005 --http-port 3000 --ws-port 3001 `
      --bind-addr 0.0.0.0 `
      --ui-path C:\Users\otaia0a\Desktop\RuView\ui
```

`--bind-addr 0.0.0.0` lets the server accept UDP from the LAN, not just loopback.

*Terminal 2:*
```powershell
PS> curl.exe http://localhost:3000/api/v1/nodes
PS> curl.exe http://localhost:3000/api/v1/sensing/latest
PS> curl.exe http://localhost:3000/health
```

**Expect:** node `1` in `/api/v1/nodes`; `/health` reports `"source":"esp32"`.

**✅ Pass:** node 1 present, amplitude arrays populated and changing between polls.
**❌ Fail — node absent but Step 5.1 saw packets:** something else holds UDP 5005.
Only one process can bind it — make sure `record-csi-udp.py` is stopped. Check:
```powershell
PS> Get-NetUDPEndpoint -LocalPort 5005 -ErrorAction SilentlyContinue
```

## Step 5.3 — See yourself in the data

**Goal:** the first genuinely convincing moment.

**Do:** open `http://localhost:3000/ui/observatory.html`. Then, in order:

1. Leave the room. Watch for **60 s**. Note the baseline amplitude variance.
2. Walk back in. Watch the amplitude heatmap.
3. Stand still 2 m from the node for 60 s.
4. Wave one arm slowly.
5. Leave again.

**Expect:** a clear, obvious step change between empty and occupied. Arm motion
produces visible transients.

**✅ Pass:** empty vs occupied is distinguishable **by eye**, without squinting.
**❌ Fail — no visible difference:** move the node. Line of sight to where you
stand, roughly chest height, ≥ 1 m from large metal, not tucked behind a monitor.
Placement is the usual culprit — that is why you ran Step 2.4.

> 📌 **Log this.** Empty-room variance and occupied variance for this node and
> this position. It is your reference for every later comparison.

---

# Phase 6 — Vitals

**Goal:** breathing rate, then heart rate.

## Step 6.1 — Switch the node to Tier 2

**Do:**
```powershell
PS> & $py firmware\esp32-csi-node\provision.py --port COM3 --edge-tier 2
```

Only the changed key is written; SSID and target survive. Power-cycle the node.

**What Tier 2 adds on-device:**
- Breathing: biquad IIR bandpass **0.1–0.5 Hz**, zero-crossing → 6–30 BPM
- Heart rate: bandpass **0.8–2.0 Hz** → 40–120 BPM
- Presence: phase variance vs an adaptively calibrated threshold (60 s ambient
  learning **at boot**)
- Fall: phase acceleration over threshold
- A 32-byte vitals packet (`0xC5110002`) at 1 Hz

> ⚠️ **The 60-second ambient calibration happens at boot.** Power-cycle the node
> **with the room empty**, and stay out for the first minute. Calibrate with a
> person present and the threshold is poisoned for the whole session.

## Step 6.2 — Measure breathing

**Setup that actually works** — breathing is a millimetre-scale chest motion, so
geometry matters more than anything else:

- Node **1.0–1.5 m** from your chest
- Node at **chest height** when you are seated
- Chest facing the node (chest-centric geometry, per Step 2.4)
- **Sit still.** Any gross motion swamps the breathing band completely.
- Quiet RF: no microwave running, no fan directly in the path

**Do:**
```powershell
PS> curl.exe http://localhost:3000/api/v1/vital-signs
PS> curl.exe http://localhost:3000/api/v1/edge-vitals
```
`/api/v1/vital-signs` is the **host-computed** estimate.
`/api/v1/edge-vitals` is the **node-computed** one from the Tier 2 pipeline.
Comparing them is a genuinely useful cross-check.

**Protocol — 3 × 2-minute runs:**

| Run | You do | Expect |
|-----|--------|--------|
| A | Breathe normally, still | 12–18 BPM |
| B | Breathe deliberately slowly, ~6 s/cycle | ~10 BPM |
| C | Breathe fast, ~2 s/cycle | ~30 BPM |

Count your actual breaths with a stopwatch. That is your ground truth.

**✅ Pass:** reported BPM **tracks the direction** of A → B → C and lands within
±3 BPM of your count in at least 2 of 3 runs.
**❌ Fail — flat or nonsense:** almost always geometry. Move closer, get to chest
height, hold still. Also confirm the node is still streaming (Step 5.1) — a
silent node reports stale vitals rather than erroring.

## Step 6.3 — Heart rate (expect this to be hard)

Heart rate is a **much** weaker signal than breathing: smaller displacement,
higher frequency, and it sits inside breathing harmonics. Set expectations low.

**Setup:** everything from 6.2, plus node within **~1 m**, and hold your breath
for 15–20 s bursts to remove the dominant interferer.

**Do:** poll `/api/v1/vital-signs` and compare against a pulse oximeter, a fitness
watch, or a manual radial-pulse count.

**✅ Pass:** within ±10 BPM during breath-hold, on at least some runs.
**❌ Fail:** this is a known-weak capability. Record what you saw and move on —
do not sink hours here.

## Step 6.4 — Fall detection

⚠️ **Use a mattress or thick cushion. Do not actually fall.**

Fall detection is a phase-acceleration threshold with a 3-frame debounce and a
5 s cooldown. Default threshold: 2.0 rad/s² (`fall_thresh=2000`).

**Do:** from standing, drop rapidly onto a mattress in view of the node. Watch
`/api/v1/edge-vitals` for the fall flag (bit 1 of the flags byte).

Tune sensitivity without reflashing:
```powershell
PS> & $py firmware\esp32-csi-node\provision.py --port COM3 --fall-thresh 1500
```
(Lower = more sensitive = more false positives.)

**✅ Pass:** flag fires on 3 of 5 deliberate drops, without firing during normal
walking.

---

# Phase 7 — Per-room calibration & training

**Goal:** the ADR-151 pipeline — baseline → enrollment → trained specialist bank →
live inference.

**Why this phase matters most.** Everything so far has been fixed thresholds. This
is the part of the repo that actually **learns your room**: it builds a
per-subcarrier statistical baseline of your empty space, records you performing a
labelled set of postures, and fits a bank of small specialist models
(presence / posture / breathing / heartbeat / restlessness / anomaly). It is pure
Rust, runs on the edge, and — unlike the pose path — is real and complete.

## Step 7.1 — Back to raw CSI

Calibration needs full CSI frames, not the compact vitals packet.

```powershell
PS> & $py firmware\esp32-csi-node\provision.py --port COM3 --edge-tier 0
```
Power-cycle. Stop `sensing-server` — **it and the CLI both want UDP 5005**, and
only one process can bind it.

## Step 7.2 — Empty-room baseline

**Goal:** a per-subcarrier statistical model of your room with nobody in it
(Welford amplitude mean/variance + von Mises phase statistics).

**Do:**
```powershell
PS> cd $repo\v2
PS> mkdir ..\ruview-data -Force
PS> .\target\release\wifi-densepose.exe calibrate `
      --udp-port 5005 --bind 0.0.0.0 `
      --duration-s 30 --tier ht20 `
      --output ..\ruview-data\baseline-node1.bin
```

**Then leave the room immediately and stay out for the full 30 seconds.**

`--tier ht20` must match your PHY. Valid: `ht20` / `ht40` / `he20` / `he40`.
HT20 at 20 Hz needs 600 frames = 30 s. There is an abort guard: if the running
z-score exceeds `--abort-z-threshold` (default 2.0), it aborts rather than bake a
person into the baseline.

**Expect:**
```
[calibrate] capturing 600 frames (~30 s, tier=ht20) — ensure room is empty
[calibrate] ...
[calibrate] summary: frames=600 tier=Ht20 subcarriers=52
```

**✅ Pass:** completes, writes `baseline-node1.bin`.
**❌ Fail — aborts on z-threshold:** something moved. A person, a pet, a fan, a
door. Re-run with the room genuinely still.
**❌ Fail — too few frames:** the node is not streaming at 20 Hz. Back to Step 5.1.

## Step 7.3 — Guided enrollment

**Goal:** labelled feature vectors for 8 anchor postures.

The CLI walks you through a fixed sequence, prompting on screen and re-prompting
any capture that fails its quality gate:

| # | Anchor | Prompt | Duration |
|---|--------|--------|----------|
| 1 | `empty` | Leave the room empty and still | ~15 s |
| 2 | `stand_still` | Stand still, in view of the sensor | ~15 s |
| 3 | `sit` | Sit down and stay still | ~15 s |
| 4 | `lie_down` | Lie down and stay still | ~15 s |
| 5 | `breathe_slow` | Lie or sit still and breathe slowly | **30 s** |
| 6 | `breathe_normal` | Stay still and breathe normally | **30 s** |
| 7 | `small_move` | Make small movements (wave a hand, shift) | ~15 s |
| 8 | `sleep_posture` | Lie in your sleep posture and relax | **30 s** |

**Do:**
```powershell
PS> .\target\release\wifi-densepose.exe enroll `
      --udp-port 5005 --bind 0.0.0.0 `
      --baseline ..\ruview-data\baseline-node1.bin `
      --room-id living-room --tier ht20 --fs-hz 15 `
      --attempts 2 `
      --output ..\ruview-data\enrollment-node1.json
```

**Read the prompts and follow them precisely.** Each accepted anchor prints its
presence z-score, motion rate, and frame count. Rejected anchors are re-prompted
up to `--attempts` times.

**Expect:**
```
[enroll] empty — Leave the room empty and still…
[enroll]   accepted  z=0.4  motion=2.1%  frames=225
[enroll] stand_still — Stand still, in view of the sensor…
...
[enroll] done: 8/8 anchors accepted → ...\enrollment-node1.json (next: `train-room`)
```

**✅ Pass:** ≥ 6 of 8 anchors accepted.
**❌ Fail — `empty` rejected:** something in the room is moving. Same fix as 7.2.
**❌ Fail — postures rejected:** you are too far from the node, or too still to be
distinguishable. Move to 1–2 m and make the postures more distinct.

> **Practical tip:** enrollment takes ~4 minutes of you moving between postures
> while the laptop is in another room. Put the terminal where you can see it, or
> have someone read the prompts to you.

## Step 7.4 — Train the specialist bank

**Do:**
```powershell
PS> .\target\release\wifi-densepose.exe train-room `
      --enrollment ..\ruview-data\enrollment-node1.json `
      --output ..\ruview-data\bank-node1.json
```

**Expect:**
```
[train-room] no transceiver geometry recorded — bank will not support geometry conditioning
[train-room] room='living-room' trained N specialists from 8 anchors → ...\bank-node1.json
[train-room]   • Presence
[train-room]   • Posture
[train-room]   • Breathing
...
```

The geometry warning is expected for a single node. You will supply geometry in
Phase 8.

**Inspect it:**
```powershell
PS> .\target\release\wifi-densepose.exe room-status --bank ..\ruview-data\bank-node1.json
```

**✅ Pass:** at least Presence, Posture and Breathing specialists trained.

## Step 7.5 — Live inference

**Do:**
```powershell
PS> .\target\release\wifi-densepose.exe room-watch `
      --bank ..\ruview-data\bank-node1.json `
      --udp-port 5005 --bind 0.0.0.0 `
      --tier ht20 --fs-hz 15 --window 200 --seconds 0
```

`--window 200` at 15 Hz ≈ a 13 s rolling window. `--seconds 0` = until Ctrl-C.

**Expect** one line per second:
```
presence=present posture=sitting  breathing=15.3bpm heart=68bpm  restless=0.12
```
Flags `VETO` (anomaly specialist rejected the inference) and `STALE` (window too
old) may appear.

**Validation protocol:**

| Ground truth | Expect |
|---|---|
| Leave the room, 60 s | `presence=empty` |
| Walk in, stand, 60 s | `presence=present posture=standing` |
| Sit, 60 s | `posture=sitting` |
| Lie down, 60 s | `posture=lying` |
| Sit still and breathe slowly | breathing drops toward ~10 BPM |
| Fidget continuously | `restless` rises |

**✅ Pass:** presence is correct ≥ 80% of seconds; posture transitions in the
right direction within ~15 s (one window) of the real change.
**❌ Fail — everything `-`:** the window has not filled. Needs ≥ 32 frames. Give
it 5 s and confirm CSI is still flowing.
**❌ Fail — posture always the same:** anchors were not distinguishable at
enrollment. Re-enroll with more separation between postures.

> 📌 **This is your headline single-node result.** Record the confusion matrix:
> for each of the 4 ground-truth states, what fraction of seconds got each label.

---

# Phase 8 — Scale to 6 nodes

**Goal:** all six nodes streaming, uniquely identified, non-interfering.

## Step 8.1 — Layout

Use your Step 2.4 sketch. Principles that came out of those scripts:

- **Chest height, ~1.0–1.2 m.** Not floor, not ceiling.
- **Surround the subject volume.** Diversity of viewing angle is what multistatic
  buys you; six nodes along one wall is barely better than one.
- **≥ 1 m from large metal** (radiators, filing cabinets, fridges).
- **Line of sight to the target zone** for at least 3 nodes.
- **Measure and write down** each node's (x, y, z) in a room frame with a fixed
  origin and axis convention. You need real numbers for the geometry file.

Suggested layout for one room, origin at a floor corner, +X and +Y along the walls:

```
        ┌──────────────────────────────────┐
        │  N2                          N3  │   z = 1.1 m for all
        │                                  │
        │            [subject]             │
        │  N1                          N4  │
        │                                  │
        │       N6              N5         │
        └──────────────────────────────────┘
```

## Step 8.2 — Flash and provision all six

**Flash** each board with the identical binaries from Step 3.4.

**Provision** with a unique `node_id` and TDM slot. Plug boards in one at a time
and note the COM port each enumerates as.

```powershell
PS> $py   = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
PS> $prov = "$repo\firmware\esp32-csi-node\provision.py"
PS> $ssid = 'YourWiFi'; $pass = 'YourPassword'; $host_ip = '192.168.1.20'

# Repeat per board, changing -port / -node-id / -tdm-slot:
PS> & $py $prov --port COM3 --ssid $ssid --password $pass `
      --target-ip $host_ip --target-port 5005 `
      --node-id 1 --tdm-slot 0 --tdm-total 6 --edge-tier 0
```

| Board | `--node-id` | `--tdm-slot` |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 1 |
| 3 | 3 | 2 |
| 4 | 4 | 3 |
| 5 | 5 | 4 |
| 6 | 6 | 5 |

**Why TDM.** Six nodes all transmitting continuously on the same channel
interfere with each other and with your actual WiFi. Time-division multiplexing
gives each node its own transmit slot in a repeating cycle. `--tdm-total 6` tells
each node how many slots exist so it can compute the cycle.

**Verify each** before moving on:
```powershell
PS> & $py $prov --port COM3 --state
```
This prints the merged NVS state for that port **without flashing**. Confirm
`node_id` and `tdm_slot` are what you intended. **Duplicate node IDs are the most
common multi-node mistake** and produce very confusing symptoms — frames from two
physical nodes merge into one logical node.

## Step 8.3 — Thermal check

⚠️ Your boards run the WiFi radio with modem sleep **disabled** (`WIFI_PS_NONE`,
required for continuous CSI) plus a DSP pipeline. That is sustained high current
with no duty cycling.

**Do:** power all six, then **touch each board after 2, 5 and 10 minutes.**

Warm is normal. **Uncomfortably hot is not** — the repo documents at least one
field report of a board that ran hot and then failed to power on again
(regulator damage suspected).

**If a board is too hot to hold:** power it down, give it airflow, do not stack or
enclose them, and consider reducing duty cycle:
```powershell
PS> & $py $prov --port COM3 --power-duty 60
```

## Step 8.4 — Confirm all six arrive

**Do:**
```powershell
PS> cd $repo\v2
PS> .\target\release\sensing-server.exe `
      --source esp32 --udp-port 5005 --http-port 3000 --ws-port 3001 `
      --bind-addr 0.0.0.0 `
      --ui-path C:\Users\otaia0a\Desktop\RuView\ui

# other terminal
PS> curl.exe http://localhost:3000/api/v1/nodes
PS> curl.exe http://localhost:3000/api/v1/mesh
PS> curl.exe http://localhost:3000/api/v1/mesh/metrics
```

**Expect:** all six node IDs in `/api/v1/nodes`, each with a recent timestamp.

**✅ Pass:** 6 distinct nodes, each ≥ 15 pps.
**❌ Fail — only one node appears, and you are using Docker:** this is the Windows
Docker multi-source UDP bug. Run natively (as above), or start the relay:
```powershell
PS> python scripts\udp-relay.py --listen-port 5005 --forward-port 5006 --verbose
```
and remap the container to `5006:5005/udp`. `--verbose` prints one line per
packet — you should see all six source IPs.
**❌ Fail — fewer than six, running natively:** check for duplicate `node_id`
(Step 8.2), then power-cycle the missing nodes (post-flash limp state, §1).
**❌ Fail — all six appear but throughput drops:** six nodes at 20 Hz is
~30 KB/s of UDP plus your normal WiFi. Reduce rate with a smaller
`--subk-count`, or move some nodes to a different channel:
```powershell
PS> & $py $prov --port COM5 --channel 11
```

## Step 8.5 — Optional: channel hopping

Multi-band capture increases sensing bandwidth ~3× by using more of the spectrum
(and your neighbours' APs as free illuminators).

```powershell
PS> & $py $prov --port COM3 --hop-channels "1,6,11" --hop-dwell 200
```

Trade-off: per-channel frame rate drops proportionally (three channels at 200 ms
dwell → each channel sampled ~1/3 of the time). **Do not enable this during
calibration or enrollment** — those assume a stable channel.

---

# Phase 9 — Multistatic & through-wall

**Goal:** results that need more than one node.

## Step 9.1 — Node geometry file

**Goal:** tell the fusion layer where the nodes physically are.
**Why:** attention-weighted multistatic fusion uses geometric diversity to weight
each node's contribution. Without positions, every node is weighted the same,
which throws away most of the benefit of having six.

Format: a JSON array of `NodeGeometry` records
(`v2/crates/wifi-densepose-calibration/src/geometry.rs`). Positions in **metres**,
angles in **radians**, in whatever room frame you defined in Step 8.1.

**Do** — measure with a tape and write `..\ruview-data\geometry.json`:
```json
[
  { "node_id": 1, "position": {"x_m": 0.0, "y_m": 0.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": 0.785, "elevation_rad": 0.0},
    "method": "tape-measure" },
  { "node_id": 2, "position": {"x_m": 0.0, "y_m": 4.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": -0.785, "elevation_rad": 0.0},
    "method": "tape-measure" },
  { "node_id": 3, "position": {"x_m": 5.0, "y_m": 4.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": 3.927, "elevation_rad": 0.0},
    "method": "tape-measure" },
  { "node_id": 4, "position": {"x_m": 5.0, "y_m": 0.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": 2.356, "elevation_rad": 0.0},
    "method": "tape-measure" },
  { "node_id": 5, "position": {"x_m": 3.5, "y_m": 0.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": 1.571, "elevation_rad": 0.0},
    "method": "tape-measure" },
  { "node_id": 6, "position": {"x_m": 1.5, "y_m": 0.0, "z_m": 1.1},
    "orientation": {"azimuth_rad": 1.571, "elevation_rad": 0.0},
    "method": "tape-measure" }
]
```

`method` is free-form provenance — `"tape-measure"`, `"floor-plan"`, `"unknown"`.
Be honest; it is recorded into the trained bank.

Also give the sensing server the same positions:
```powershell
PS> .\target\release\sensing-server.exe --source esp32 --udp-port 5005 `
      --http-port 3000 --ws-port 3001 --bind-addr 0.0.0.0 `
      --node-positions "0,0,1.1;0,4,1.1;5,4,1.1;5,0,1.1;3.5,0,1.1;1.5,0,1.1" `
      --ui-path C:\Users\otaia0a\Desktop\RuView\ui
```

## Step 9.2 — Per-node banks

Each node sees the room from a different angle, so **each needs its own baseline,
enrollment and bank**. There is no shortcut here.

For each node in turn: power **only that node**, then run Phase 7 Steps 7.2–7.4
with per-node filenames.

```powershell
# Repeat with N = 1..6
PS> .\target\release\wifi-densepose.exe calibrate `
      --udp-port 5005 --duration-s 30 --tier ht20 `
      --output ..\ruview-data\baseline-node1.bin

PS> .\target\release\wifi-densepose.exe enroll `
      --baseline ..\ruview-data\baseline-node1.bin `
      --room-id living-room --tier ht20 --fs-hz 15 `
      --output ..\ruview-data\enrollment-node1.json

PS> .\target\release\wifi-densepose.exe train-room `
      --enrollment ..\ruview-data\enrollment-node1.json `
      --geometry ..\ruview-data\geometry.json `
      --output ..\ruview-data\bank-node1.json
```

**Now `--geometry` is supplied**, so `train-room` snapshots the layout into each
bank (ADR-152 §2.1.1) and you should see:
```
[train-room] geometry: 6 node(s) snapshotted into the bank
```

> **Budget realistically:** ~10 minutes per node × 6 = **an hour of enrollment**.
> This is the most tedious part of the plan. It is also what makes the multistatic
> result meaningful, so do not rush the anchors.

## Step 9.3 — Multistatic room-watch

**Goal:** all six banks fused into one live readout.

**Do** — power all six nodes, then:
```powershell
PS> .\target\release\wifi-densepose.exe room-watch `
      --node-bank 1:..\ruview-data\bank-node1.json `
      --node-bank 2:..\ruview-data\bank-node2.json `
      --node-bank 3:..\ruview-data\bank-node3.json `
      --node-bank 4:..\ruview-data\bank-node4.json `
      --node-bank 5:..\ruview-data\bank-node5.json `
      --node-bank 6:..\ruview-data\bank-node6.json `
      --udp-port 5005 --bind 0.0.0.0 `
      --tier ht20 --fs-hz 15 --window 200
```

`--node-bank N:PATH` is repeatable. When any are supplied, frames are grouped by
node ID and fused (ADR-029/151) instead of running the single-node path.

**Expect:**
```
[room-watch] multistatic over nodes [1, 2, 3, 4, 5, 6]
[room-watch] fusing on udp://0.0.0.0:5005 (window=200 frames)
presence=present posture=sitting  breathing=15.1bpm heart=71bpm  restless=0.09
```

**The experiment that justifies six nodes.** Run the *same* validation protocol as
Step 7.5, twice:

1. Single node: `room-watch --bank ..\ruview-data\bank-node1.json`
2. All six: the multistatic command above

Compare accuracy on identical ground truth.

**✅ Pass:** multistatic beats single-node on presence accuracy and/or posture
accuracy. Any measurable improvement is a real result — **write down the actual
numbers**, not an impression.

## Step 9.4 — Through-wall

**Goal:** detect a person the nodes cannot see.

**Physics, so you know what to expect.** Attenuation through building materials at
2.4 GHz, one-way, approximately:

| Material | Loss |
|---|---|
| Drywall / plasterboard | 3–5 dB |
| Wooden door | 3–6 dB |
| Brick | 8–15 dB |
| Concrete | 15–25 dB |
| Reinforced concrete / metal mesh | 25 dB+ — effectively opaque |

The repo claims through-wall to **~5 m, signal-dependent**. Drywall works well.
Reinforced concrete does not. Start with the easiest wall you have.

**Do:**
1. Put 2–3 nodes on one side of an interior drywall wall.
2. Record an empty-room baseline **with nobody on either side**:
   ```powershell
   PS> .\target\release\wifi-densepose.exe calibrate --duration-s 60 `
         --output ..\ruview-data\baseline-throughwall.bin
   ```
3. Have someone walk on the far side while you watch amplitude variance and
   presence.
4. Alternate 60 s present / 60 s absent, **five cycles**, logging each.

**✅ Pass:** presence tracks the far-side person on ≥ 3 of 5 cycles.
**❌ Fail:** thicker wall than expected, or too far. Move nodes closer to the
wall, try a doorway (huge difference), and confirm same-side detection still works
so you know the nodes are healthy.

> ⚠️ **Guard against fooling yourself.** Through-wall demos are easy to fake
> accidentally — footsteps vibrating the floor, a door moving, the person's phone
> associating with the AP. Run a **negative control**: the same five cycles with
> the far-side person replaced by nothing at all. If your detector fires anyway,
> you are measuring something other than the person.

## Step 9.5 — Multi-person

**Set expectations first.** `docs/TROUBLESHOOTING.md` §2 documents this honestly:
the current dedup factor is "a guess", and `estimated_persons` read **6–8 for 5
bodies**. Treat this as characterisation, not validation.

**Do:** with all six nodes fused, step the room occupancy 0 → 1 → 2 → 3 people,
60 s each, and log `estimated_persons` from `/api/v1/sensing/latest`.

There is a runtime tuning knob:
```powershell
PS> curl.exe -X POST http://localhost:3000/api/v1/config/dedup-factor `
      -H "Content-Type: application/json" -d "{\"factor\": 2.5}"
```

**✅ Pass:** the reported count **increases monotonically** with true occupancy.
Absolute accuracy is not expected.

---

# Phase 10 — Record, analyse, report

## Step 10.1 — Capture datasets

Recording raw CSI lets you re-run analysis without re-running the experiment —
which matters, because physical experiments are expensive and irreproducible.

```powershell
PS> python scripts\record-csi-udp.py --duration 600 --output data\recordings\empty-room
PS> python scripts\record-csi-udp.py --duration 600 --output data\recordings\one-person-still
PS> python scripts\record-csi-udp.py --duration 600 --output data\recordings\one-person-moving
PS> python scripts\record-csi-udp.py --duration 600 --output data\recordings\through-wall
```

The server also has a recording API:
```powershell
PS> curl.exe -X POST http://localhost:3000/api/v1/recording/start
PS> curl.exe http://localhost:3000/api/v1/recording/list
PS> curl.exe -X POST http://localhost:3000/api/v1/recording/stop
```

## Step 10.2 — Overnight run

**Goal:** stability, drift, and thermal behaviour over hours — the things a
20-minute test cannot show.

```powershell
PS> python scripts\overnight-empty-capture.py
```

**Check the next morning:**
- Are all six nodes still streaming? (`/api/v1/nodes`)
- Did presence false-positive in an empty room, and how often?
- Did breathing/heart rate drift?
- Are any boards hot?

This is where you find out whether the baseline holds as the room's temperature,
furniture, and RF environment change.

## Step 10.3 — Write it up

A results table that is actually useful:

| Test | Ground truth | Measured | Error | Verdict |
|------|--------------|----------|-------|---------|
| CSI rate, node 1 | — | ___ pps | — | |
| Breathing, normal | ___ BPM (counted) | ___ BPM | ___ | |
| Breathing, slow | ___ BPM | ___ BPM | ___ | |
| Breathing, fast | ___ BPM | ___ BPM | ___ | |
| Heart rate, breath-hold | ___ BPM (oximeter) | ___ BPM | ___ | |
| Presence, single node | occupied/empty | ___% correct | — | |
| Presence, 6-node fused | occupied/empty | ___% correct | — | |
| Posture, single node | 4 classes | ___% correct | — | |
| Posture, 6-node fused | 4 classes | ___% correct | — | |
| Fall detection | 5 drops | ___/5 | — | |
| Through-wall | 5 cycles | ___/5 | — | |
| Through-wall, negative control | 0 cycles | ___ false fires | — | |
| Person count | 0,1,2,3 | ___ | — | monotonic? |
| Overnight stability | 8 h | ___ nodes alive | — | |

**Report honestly.** State how each number was measured, what the ground truth
was, and how many trials. A well-characterised 70% is far more valuable than an
unreproducible 95%. That standard is the repo's own, and it is worth keeping.

---

# Appendix A — Troubleshooting index

| Symptom | Likely cause | Fix |
|---|---|---|
| Node on WiFi, **0 pps CSI** | Display probe false-positive keeping MGMT-only filter | Rebuild with `# CONFIG_DISPLAY_ENABLE is not set` (Step 3.2) |
| Serial monitor completely silent | Console on UART0, board is native-USB only | `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` (Step 3.2) |
| Only 1 of 6 nodes visible | Docker Desktop Windows multi-source UDP drop | Run natively, or `scripts/udp-relay.py` (§9 of TROUBLESHOOTING) |
| `cargo` fails: workspace member manifest | Submodules not initialised | `git submodule update --init --recursive` |
| Zero UDP packets received | Windows Firewall | Allow `python.exe` / `sensing-server.exe` inbound, Private profile |
| UI returns 404 | cwd-relative `--ui-path` | Pass an absolute path |
| `Address already in use` on 5005 | Two processes want the port | Run one; or `scripts/csi-udp-relay.py` to a loopback port |
| Node associates then goes quiet | Post-flash limp state | Power-cycle: unplug, 2 s, replug |
| Boot loop on `--edge-tier 1`/`2` | Old firmware WDT bug | You are on v0.8.4; if it recurs, capture the backtrace (#438) |
| Person count stuck / overcounts | Known heuristic limitation | Tune `dedup-factor`; expect monotonic, not accurate |
| Vitals jitter between nodes | Best-node selection is authoritative | Expected; per-node smoothing already applied |
| WiFi won't associate | 5 GHz-only SSID | ESP32-S3 is 2.4 GHz only |
| Board very hot | `WIFI_PS_NONE` + DSP, no duty cycling | Airflow; `--power-duty 60` |
| `--model` errors on magic | HF weights are JSONL RVF; loader wants binary | Run without `--model` |

Fuller writeups: `docs/TROUBLESHOOTING.md` (11 reproduced field failures).

---

# Appendix B — Command reference

```powershell
# ── Setup ────────────────────────────────────────────────────────────────
$py   = 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe'
$repo = 'C:\Users\otaia0a\Desktop\RuView'
$prov = "$repo\firmware\esp32-csi-node\provision.py"

# ── Hardware identification ──────────────────────────────────────────────
& $py -m esptool --port COM3 --no-stub flash-id     # chip, flash, MAC
& $py -m espefuse --port COM3 summary               # PSRAM, package, USB
Get-CimInstance Win32_SerialPort | Select DeviceID, Description

# ── Flash ────────────────────────────────────────────────────────────────
& $py -m esptool --chip esp32s3 --port COM3 --baud 460800 `
  write_flash --flash_mode dio --flash_size 8MB `
  0x0 build\bootloader\bootloader.bin `
  0x8000 build\partition_table\partition-table.bin `
  0xf000 build\ota_data_initial.bin `
  0x20000 build\esp32-csi-node.bin

# ── Provision (NVS, no reflash) ──────────────────────────────────────────
& $py $prov --port COM3 --ssid S --password P --target-ip IP --node-id 1 --edge-tier 0
& $py $prov --port COM3 --edge-tier 2               # switch tier only
& $py $prov --port COM3 --state                     # dry-run: show merged state
& $py $prov --port COM3 --reset                     # wipe this port's state file

# ── Serial ───────────────────────────────────────────────────────────────
& $py -m serial.tools.miniterm COM3 115200          # exit: Ctrl+]

# ── Server ───────────────────────────────────────────────────────────────
cd $repo\v2
.\target\release\sensing-server.exe --source esp32 --udp-port 5005 `
  --http-port 3000 --ws-port 3001 --bind-addr 0.0.0.0 `
  --ui-path C:\Users\otaia0a\Desktop\RuView\ui

# ── API ──────────────────────────────────────────────────────────────────
curl.exe http://localhost:3000/health
curl.exe http://localhost:3000/api/v1/nodes
curl.exe http://localhost:3000/api/v1/sensing/latest
curl.exe http://localhost:3000/api/v1/vital-signs
curl.exe http://localhost:3000/api/v1/edge-vitals
curl.exe http://localhost:3000/api/v1/mesh
curl.exe http://localhost:3000/api/v1/mesh/metrics

# ── Calibration pipeline ─────────────────────────────────────────────────
.\target\release\wifi-densepose.exe calibrate   --duration-s 30 --output baseline.bin
.\target\release\wifi-densepose.exe enroll      --baseline baseline.bin --room-id R --output enroll.json
.\target\release\wifi-densepose.exe train-room  --enrollment enroll.json --geometry geo.json --output bank.json
.\target\release\wifi-densepose.exe room-status --bank bank.json
.\target\release\wifi-densepose.exe room-watch  --bank bank.json
.\target\release\wifi-densepose.exe room-watch  --node-bank 1:b1.json --node-bank 2:b2.json

# ── No-hardware testing ──────────────────────────────────────────────────
python scripts\synth-csi-udp.py --port 5005 --duration-s 120 --motion-after-s 30
python archive\v1\data\proof\verify.py
python scripts\record-csi-udp.py --duration 300 --output data\recordings
python scripts\udp-relay.py --listen-port 5005 --forward-port 5006 --verbose

# ── Tests ────────────────────────────────────────────────────────────────
cd $repo\v2; cargo test --workspace --no-default-features
```

---

# Appendix C — Node log sheet

Copy per node. The tedium pays for itself the first time two nodes disagree.

```
NODE ___

  COM port at provisioning     ______
  MAC                          ______________________
  node_id                      ______
  tdm_slot / tdm_total         ______ / 6
  edge_tier                    ______
  channel                      ______
  Position  x / y / z  (m)     ______ / ______ / ______
  Orientation azimuth (rad)    ______
  Firmware SHA-256 (app)       __________________________________

  Flashed                 ☐    date ____________
  Boots, console readable ☐
  Joins WiFi              ☐
  CSI rate at host        ____ pps        ☐ ≥15 pps
  Empty-room variance     ______
  Occupied variance       ______
  Baseline captured       ☐    file ____________________
  Enrollment  __/8 anchors     file ____________________
  Bank trained            ☐    specialists: ______________
  Thermal @10 min              ☐ cool  ☐ warm  ☐ HOT — investigate

  Notes:
  ______________________________________________________________
```

---

# Appendix D — Where to read more

| Topic | File |
|---|---|
| Operator guide (2,500 lines — the best single doc) | `docs/user-guide.md` |
| Firmware internals, NVS keys, wire formats | `firmware/esp32-csi-node/README.md` |
| 11 reproduced field failures + fixes | `docs/TROUBLESHOOTING.md` |
| Per-room calibration design | `docs/adr/ADR-151-*.md` |
| Multistatic mesh | `docs/adr/ADR-029-*.md` |
| Mesh security hardening | `docs/adr/ADR-032-*.md` |
| CSI frame format | `docs/adr/ADR-018-*.md` |
| Edge intelligence tiers | `docs/adr/ADR-039-*.md` |
| Vital signs | `docs/adr/ADR-021-*.md` |
| Physics + placement studies | `examples/research-sota/` |
| Repo conventions and gotchas | `CLAUDE.md` |
