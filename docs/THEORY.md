# RuView — Theory of Operation

**A ground-up explanation of WiFi sensing, and of the system this repository builds on top of it.**

Written 2026-07-27 against repo state `94914bf2`, firmware `v0.8.4`.
Companion browsable copy: [`docs/THEORY.html`](THEORY.html).
Practical counterpart: [`plans/ruview-esp32s3-test-plan.md`](../plans/ruview-esp32s3-test-plan.md).

---

## How to read this document

This assumes **no prior knowledge** — not of radio, not of signal processing, not
of machine learning, not of this codebase. It builds each idea from the one
before it.

| Part | What it answers | Read it if |
|------|-----------------|------------|
| [0 · Orientation](#part-0--orientation) | What is this thing, in 90 seconds? | Always. Start here. |
| [1 · Motivation](#part-1--motivation-why-build-this-at-all) | Why does this need to exist? Who is it for? | You want the *why* before the *how* |
| [2 · Physics](#part-2--physics-from-first-principles) | How can radio see a person? | You want to understand rather than operate |
| [3 · Signal → inference](#part-3--from-signal-to-inference) | How do wiggly numbers become "someone is breathing"? | You will touch DSP or models |
| [4 · The system](#part-4--the-ruview-system-component-by-component) | What is every piece of this repo, and why? | You will touch the code |
| [5 · Life of a frame](#part-5--life-of-a-frame) | How does one measurement travel end to end? | You are debugging |
| [6 · Honesty ledger](#part-6--the-honesty-ledger) | What is actually proven vs. aspirational? | Before you cite a number |
| [7 · Comparisons](#part-7--how-this-design-compares-to-the-alternatives) | Why this design and not radar / cameras / the other WiFi tools? | You are evaluating |
| [8 · Theory → test](#part-8--from-theory-to-test-mapping-onto-plans) | How do I *verify* each idea on real hardware? | You are about to run the test plan |
| [9 · Resources](#part-9--resource-library) | Where do I read more? | Continuously |
| [10 · Glossary](#part-10--glossary) | What does that word mean? | Continuously |

### Conventions used here

- **Physics blocks** derive a number from first principles. The arithmetic is
  shown so you can check it.
- **Evidence tags** mark how strongly a claim is supported. This repository is
  unusually disciplined about this and this document preserves the discipline:

| Tag | Meaning |
|-----|---------|
| **[PHYS]** | Follows from physics/mathematics. Independent of this codebase. |
| **[CODE]** | Verified by reading this repository at the cited path. |
| **[SYNTH]** | A number from a synthetic-physics simulation, not a bench measurement. Real deployments diverge 5–30%. |
| **[BENCH]** | Measured on a published dataset or benchmark, with a stated protocol. |
| **[FIELD]** | Observed on real hardware in this project, reproduced at least once. |
| **[OPEN]** | Designed and documented, not yet validated. Treat as a hypothesis. |

- File references are clickable, e.g.
  [`v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs).

---

# Part 0 — Orientation

## 0.1 The ninety-second version

Every WiFi device in your home is already flooding your rooms with 2.4 GHz radio
waves. Those waves bounce off walls, furniture, and — crucially — off *you*. The
receiver has to measure exactly how the room distorted the signal, otherwise it
could not decode the data. That measurement is called **Channel State
Information (CSI)**, and it is computed by essentially every WiFi chip on Earth,
several hundred times a second, and then thrown away.

RuView does not throw it away.

```
   ┌──────────┐        radio waves        ┌──────────┐
   │  WiFi TX │ ~~~~~~~~~~~~~~~~~~~~~~~~► │  ESP32-S3│
   │  (any AP)│    ╲                  ╱   │  receiver│
   └──────────┘     ╲   ┌────────┐   ╱    └────┬─────┘
                     ╲─►│ person │─►╱          │ CSI: 52–64 complex
                        └────────┘             │ numbers, ~20×/second
                     reflection changes        ▼
                     with every breath    ┌─────────────┐
                                          │  DSP + tiny │
                                          │   models    │
                                          └──────┬──────┘
                                                 ▼
                       presence · breathing rate · heart rate ·
                       motion · falls · posture · (aspirationally) pose
```

The whole project is the engineering required to go from "the radio field is
perturbed" to "a person is lying down in the bedroom, breathing at 14 breaths per
minute, and has not moved for six hours" — without a camera, without a wearable,
in the dark, and through a drywall wall.

## 0.2 What is physically true, and what is engineering

Two claims are worth separating immediately, because the field is full of hype.

**True, and not controversial:** a human body is a large, wet, strongly
reflecting object at 2.4 GHz. Moving it — even by the few millimetres your chest
moves when you breathe — measurably changes the multipath channel. This is not
speculative; it is the same physics that makes radar work, and it has been
demonstrated in peer-reviewed work since roughly 2013. **[PHYS]**

**Hard, and where all the real work is:** the channel is *also* perturbed by the
fan, the microwave, the neighbour's access point changing power, the temperature
of the room, and your own hardware's clock drift. Extracting "person" from that
soup, in a room the system has never seen before, is the actual problem. Most of
this repository is about that problem, not about the physics.

## 0.3 The shape of the repository

| Path | What it is | Status |
|------|-----------|--------|
| [`firmware/esp32-csi-node/`](../firmware/esp32-csi-node/) | ESP-IDF C firmware for the sensor node | Active, `v0.8.4` |
| [`v2/`](../v2/) | Rust workspace, ~40 crates. All host-side logic. | **Primary.** All active work. |
| [`ui/`](../ui/) · [`dashboard/`](../dashboard/) | Browser front-ends | Active |
| [`docs/adr/`](adr/) | 207 Architecture Decision Records | The design record |
| [`examples/research-sota/`](../examples/research-sota/) | Pure-NumPy physics derivations | **[SYNTH]** |
| [`plans/`](../plans/) | Implementation and validation plans | See [Part 8](#part-8--from-theory-to-test-mapping-onto-plans) |
| [`archive/v1/`](../archive/v1/) | The original Python implementation | Archived; kept for the deterministic proof |

A structural note worth internalising early: **the ADRs are the primary design
documents, and the code cites them by number.** When a Rust module opens with
`//! ADR-134 ...`, that ADR is the specification the module implements. Reading
`docs/adr/ADR-134-*.md` before reading `ruvsense/cir.rs` will save you an hour.

---

# Part 1 — Motivation: why build this at all

## 1.1 The problem: rooms are unobserved

There is a large and growing class of questions that are trivially answerable if
you are physically present, and nearly impossible to answer otherwise:

- Did my father get out of bed this morning?
- Is the person in the bathroom still conscious?
- How many people are in this conference room right now?
- Did the resident's breathing pattern change over the last three weeks?
- Is anyone still alive under this collapsed building?

Every existing answer has a serious cost.

| Approach | What it costs |
|----------|---------------|
| **Cameras** | Total privacy loss. Socially unacceptable in bedrooms and bathrooms — which is exactly where falls happen. Fails in darkness and behind furniture. Legally fraught (GDPR, HIPAA, state biometric laws). |
| **Wearables** | Only work if worn. Compliance among elderly users is famously poor. Must be charged. The person most likely to fall is the least likely to be wearing the pendant. |
| **PIR motion sensors** | Detect gross motion only. A person who has collapsed is, to a PIR sensor, indistinguishable from an empty room. No vitals, no posture, no count. |
| **Pressure mats / bed sensors** | Single-point, easily defeated by walking around them. |
| **mmWave radar** | Genuinely good (see [Part 7](#part-7--how-this-design-compares-to-the-alternatives)) — but a dedicated 60 GHz sensor costs $30–200 per room, does not penetrate walls, and requires new infrastructure in every room you want covered. |

## 1.2 The opportunity: the infrastructure is already installed

The argument for WiFi sensing is not that it is the most sensitive modality. It
is not. The argument is **marginal cost**:

- Radio infrastructure already exists in essentially every occupied building.
- The measurement (CSI) is already computed as a mandatory part of decoding a
  packet. Sensing is a *free by-product* of a link that is running anyway.
- 2.4 GHz penetrates interior walls. One sensor can cover more than one room.
- It works in complete darkness, through smoke, and under blankets.
- A capable node costs **$6–10** (ESP32-C6 / ESP32-S3), not $200.

That last number is what makes the design space different. At $9/node you can
afford to put six nodes in a room and solve with geometric diversity what a
single expensive sensor would have to solve with raw sensitivity. This is the
central architectural bet of this project, and it is what "multistatic" means in
practice (see [§2.11](#211-multistatic-geometry-why-more-nodes-beats-a-better-node)).

## 1.3 Who this is actually for

| Domain | Concrete need | What RuView targets |
|--------|---------------|---------------------|
| **Aging in place** | 3 in 10 adults over 65 fall each year; the outcome depends heavily on time-to-discovery | Fall events, bed-exit, inactivity anomaly, overnight breathing trend |
| **Sleep & respiratory health** | Sleep apnea is massively underdiagnosed; polysomnography is expensive and unrepresentative of home sleep | Contactless respiration rate and regularity, night after night, in the person's own bed |
| **Post-acute / hospital-at-home** | Continuous monitoring without tethering the patient | Presence, posture, respiration, restlessness |
| **Building operations** | HVAC and lighting driven by real occupancy, not schedules | Room-level occupancy and coarse count |
| **Security** | Detect presence where a camera is not permitted or not possible | Through-wall presence, structural change detection |
| **Disaster response** | Locate survivors in rubble where nothing else can see | [`wifi-densepose-mat`](../v2/crates/wifi-densepose-mat/) — Mass Casualty Assessment Tool |

## 1.4 The privacy argument — and its limits

The honest framing is that WiFi sensing is **privacy-*preferable*, not
privacy-free**.

**What it genuinely improves:** there is no image. A leaked CSI recording does
not show a person's face, body, or what they were reading. The failure mode of a
compromised camera and the failure mode of a compromised CSI stream are not
comparable.

**What it does not solve:** CSI is still a biometric-adjacent signal. Published
research has shown gait can be used to identify individuals from RF, and this
repository takes that seriously rather than waving it away — see the BFLD privacy
class system ([ADR-120](adr/ADR-120-bfld-privacy-class-and-hash-rotation.md)),
where privacy classes P0–P5 gate what may leave the device at all, and raw and
identity-bearing tiers are structurally forbidden from egress
([`v2/crates/wifi-densepose-bfld/`](../v2/crates/wifi-densepose-bfld/)). The
architecture's answer to "this could be abused" is **edge-first computation**:
the raw signal never leaves the box, only the derived semantic state does.

That is a design stance, not a proof. It should be evaluated as such.

## 1.5 Why now

Three things changed roughly simultaneously:

1. **Cheap CSI-capable silicon.** Espressif exposed the CSI callback on the
   ESP32 family. Before that, CSI extraction required an Intel 5300 NIC with a
   patched driver on a specific kernel, or a jailbroken Broadcom chip.
2. **The IEEE standardised sensing.** [IEEE 802.11bf](https://www.ieee802.org/11/Reports/tgbf_update.htm)
   defines WLAN sensing as a first-class service — sensing sessions, measurement
   negotiation, and reporting. This turns "clever hack" into "protocol feature",
   and every major chipset vendor now has a sensing story.
3. **Small models became viable on microcontrollers.** A 8 KB int4 embedding
   model that runs in microseconds changes what "edge" means.

---

# Part 2 — Physics from first principles

This part builds the physical model with no assumed background. If you already
know OFDM and multipath, skip to [§2.6](#26-why-a-human-body-shows-up-at-all).

## 2.1 Radio waves, in the only terms you need

An electromagnetic wave is an oscillating electric and magnetic field that
propagates through space at $c \approx 3 \times 10^8$ m/s. Two properties matter
here:

- **Frequency** $f$ — oscillations per second. WiFi's 2.4 GHz band means
  $2.4 \times 10^9$ oscillations per second.
- **Wavelength** $\lambda$ — the physical distance between successive wave
  crests, $\lambda = c/f$.

> **[PHYS] The single most important number in this document**
>
> $$\lambda = \frac{c}{f} = \frac{3\times10^8\ \text{m/s}}{2.4\times10^9\ \text{Hz}} = 0.125\ \text{m} = 125\ \text{mm}$$
>
> At 5 GHz, $\lambda = 60$ mm.
>
> Every sensitivity claim in this project traces back to this. A radio wave at
> 2.4 GHz completes a full cycle every 12.5 cm of travel. Therefore a path-length
> change of a *few millimetres* is a large fraction of a cycle — and phase is
> measurable to a fraction of a degree.

## 2.2 What happens to a wave in a room

A transmitted wave does not reach the receiver by one path. It:

- **Attenuates** with distance (free-space path loss),
- **Reflects** off conductive and high-permittivity surfaces (walls, metal, wet
  tissue),
- **Diffracts** around edges,
- **Scatters** off objects comparable in size to $\lambda$,
- **Penetrates** materials with loss.

> **[PHYS] Free-space path loss**
>
> $$\text{FSPL(dB)} = 20\log_{10}(d) + 20\log_{10}(f) - 147.55$$
>
> At $d = 1$ m and $f = 2.4$ GHz: $0 + 187.6 - 147.55 \approx 40$ dB.
> Doubling the distance costs another 6 dB. This is why node placement dominates
> results — the difference between 1 m and 4 m from a subject is 12 dB of link
> budget, which is most of your breathing SNR.

Building-material penetration at 2.4 GHz, one-way, approximately:

| Material | Loss | Consequence for this project |
|----------|------|------------------------------|
| Drywall / plasterboard | 3–5 dB | Through-wall sensing works well |
| Wooden door | 3–6 dB | Works |
| Brick | 8–15 dB | Marginal; needs close nodes |
| Concrete | 15–25 dB | Difficult |
| Reinforced concrete / metal mesh | 25 dB+ | Effectively opaque |

This table is why the repo bounds its through-wall claim at "**up to ~5 m,
signal-dependent**" rather than a flat number. It is also why the test plan's
through-wall step tells you to start with drywall.

## 2.3 Multipath: the room as a filter

Because there are many paths, the receiver sees the **sum** of many copies of the
transmitted signal, each with its own delay and attenuation. Mathematically, the
room is a linear filter with impulse response

$$h(\tau) = \sum_{p} a_p\, \delta(\tau - \tau_p)$$

where path $p$ arrives with complex gain $a_p$ after delay $\tau_p$. This
$h(\tau)$ is the **Channel Impulse Response (CIR)** — literally "what the room
does to an instantaneous pulse".

Its Fourier transform, $H(f)$, is the **Channel Frequency Response (CFR)** — what
the room does to each frequency. These are two views of the same object:

```
   CIR  h(τ)   — time domain            CFR  H(f)  — frequency domain
   ┌──────────────────────┐             ┌──────────────────────┐
   │ █                    │             │  ▁▃▅▂▇▁▄▆▂▁▅▃▁▂▄     │
   │ █  █                 │   ◄─ FFT ─► │                      │
   │ █  █    █     █      │             │                      │
   └──────────────────────┘             └──────────────────────┘
    direct  wall  person  far            per-subcarrier gain/phase
    path    bounce        bounce
```

**This equivalence is load-bearing in this codebase.** WiFi hands you $H(f)$
(that is what CSI is). Physical intuition lives in $h(\tau)$ — "how far away is
the thing that reflected". Converting one to the other is exactly what
[`ruvsense/cir.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs)
does, implementing [ADR-134](adr/ADR-134-csi-to-cir-time-domain-multipath.md).

## 2.4 OFDM: why WiFi gives you many measurements, not one

WiFi does not transmit one carrier. **Orthogonal Frequency-Division Multiplexing
(OFDM)** splits the channel into many narrow, mutually orthogonal subcarriers
transmitted in parallel.

For 802.11n on a 20 MHz channel:

| Quantity | Value | Derivation |
|----------|-------|-----------|
| FFT size | 64 | Standard |
| Subcarrier spacing | 312.5 kHz | 20 MHz / 64 |
| Non-null subcarriers | 56 | 52 data + 4 pilot; the rest are guard/DC nulls |
| Symbol duration | 3.2 µs + 0.8 µs guard | 1 / 312.5 kHz |

Why OFDM matters for *sensing* rather than communication: to equalise each
subcarrier, the receiver **must estimate the channel at each subcarrier
separately**. Instead of one number describing the room, you get 56 — a sampled
version of $H(f)$ across 20 MHz of bandwidth. That is a spectrum, and spectra
carry structure.

> **Note on subcarrier counts you will see in this repo.** The ESP32 CSI callback
> returns raw buffers whose length depends on the PHY mode: HT20 yields 64
> entries (of which the guard bands are null), HT40 yields 128. Different
> chipsets report differently — Intel 5300 gives 30 *grouped* subcarriers,
> Atheros gives 56, Broadcom via Nexmon gives the full FFT. The repo normalises
> all of these onto a canonical 56-bin grid in
> [`hardware_norm.rs`](../v2/crates/wifi-densepose-signal/src/hardware_norm.rs)
> (ADR-027), using Catmull-Rom cubic interpolation, precisely so a model trained
> on one chipset is not useless on another. **[CODE]**

## 2.5 CSI: the actual measurement

For each subcarrier $k$, the receiver estimates a complex number:

$$H_k = |H_k| \, e^{j\angle H_k}$$

- $|H_k|$ — **amplitude**: how much that frequency was attenuated.
- $\angle H_k$ — **phase**: how much that frequency was delayed, modulo $2\pi$.

Stack those across $k$ and you have one **CSI frame**. Stack frames over time and
you have a 2-D signal: subcarrier × time.

```
             subcarrier index k →
          ┌──────────────────────────────────┐
    t=0   │ 3.1  2.8  4.0  3.9  2.2  ...     │  amplitudes
    t=1   │ 3.1  2.7  4.1  3.8  2.3  ...     │
    t=2   │ 3.0  2.9  3.9  4.0  2.2  ...     │   ← a person breathing
     ↓    │  ⋮                               │     shows up as a slow
   time   └──────────────────────────────────┘     ripple down a column
```

**Everything downstream in this project is a function of this matrix.**

In this repo, one frame is [`CsiFrame`](../v2/crates/wifi-densepose-core/src/types.rs)
in `wifi-densepose-core`, and it is the type every other crate depends on.

## 2.6 Why a human body shows up at all

At 2.4 GHz, biological tissue is mostly water, and water has a very high relative
permittivity. Approximate values at 2.45 GHz:

| Medium | Relative permittivity $\varepsilon_r$ | Conductivity $\sigma$ (S/m) |
|--------|--------------------------------------|------------------------------|
| Air | 1.0 | 0 |
| Dry skin | ~38 | ~1.5 |
| Muscle | ~53 | ~1.7 |
| Pure water | ~78 | — |

> **[PHYS] Reflection at the air–body boundary**
>
> For normal incidence, the reflection coefficient is
> $$\Gamma = \frac{\sqrt{\varepsilon_1}-\sqrt{\varepsilon_2}}{\sqrt{\varepsilon_1}+\sqrt{\varepsilon_2}}$$
> With $\varepsilon_1 = 1$ (air) and $\varepsilon_2 \approx 53$ (muscle):
> $$|\Gamma| = \frac{|1 - 7.28|}{1 + 7.28} \approx 0.76 \quad\Rightarrow\quad |\Gamma|^2 \approx 0.58$$
>
> **Roughly 58% of incident power is reflected at the body surface.** A human is
> not a weak scatterer — a human is close to a mirror. Additionally, the
> penetration depth in muscle at 2.45 GHz is only ~2 cm, so the body is
> effectively an opaque, strongly-reflecting obstacle, not a transparent one.

This is the physical basis of the entire field. It is also why the signal is
*obstructed* by the body — a node behind you sees a shadow, and a node in front
of you sees a reflection. Multistatic geometry exploits exactly this asymmetry.

## 2.7 Breathing: from millimetres to degrees

Chest-wall displacement during normal quiet breathing is **4–12 mm**. Here is why
that is enormous.

> **[PHYS] Phase sensitivity to displacement**
>
> If a reflecting surface moves by $\Delta d$ along the propagation direction,
> the reflected path length changes. For a monostatic geometry (transmitter and
> receiver co-located) the path changes by $2\Delta d$ and
> $$\Delta\phi = \frac{2\pi \cdot 2\Delta d}{\lambda}$$
> For a bistatic geometry the factor is between 1 and 2 depending on the angle.
> Taking the conservative one-way form used in the repo's test plan:
> $$\Delta\phi = \frac{2\pi \Delta d}{\lambda} = \frac{2\pi \times 0.012}{0.125} = 0.60\ \text{rad} \approx 35°$$
>
> **A 12 mm chest movement produces roughly 35° of phase shift** (and up to ~70°
> in the monostatic limit). Phase noise on a clean link is a small number of
> degrees. The breathing signal is therefore *tens of dB above the noise floor* —
> in principle.

The "in principle" carries all the weight. It assumes the chest motion is aligned
with the propagation path, the person is in a favourable Fresnel zone, nothing
else in the room is moving, and hardware phase artefacts have been removed.
[§2.9](#29-the-hardware-lies-cfo-sfo-and-why-raw-phase-is-garbage) explains why
that last assumption is the difficult one.

Heartbeat, for comparison, produces surface displacement on the order of
**0.2–0.5 mm** — 20–50× smaller, hence ~26–34 dB weaker, and it sits underneath
the harmonics of breathing. This is the honest reason heart rate is the weakest
capability in this system, and why the test plan tells you to hold your breath
when measuring it.

## 2.8 Frequency bands: where breathing and heartbeat live

Physiological rhythms occupy distinct, narrow frequency bands. This is what makes
bandpass filtering such an effective first-line separator.

| Signal | Frequency band | Rate | Where implemented |
|--------|---------------|------|-------------------|
| Breathing | 0.1–0.5 Hz | 6–30 breaths/min | [`vitals/breathing.rs`](../v2/crates/wifi-densepose-vitals/src/breathing.rs) |
| Heartbeat | 0.8–2.0 Hz | 48–120 beats/min | [`vitals/heartrate.rs`](../v2/crates/wifi-densepose-vitals/src/heartrate.rs) |
| Gross motion / walking | ~1–20 Hz Doppler | — | [`motion.rs`](../v2/crates/wifi-densepose-signal/src/motion.rs) |
| Environmental drift | < 0.01 Hz | hours | [`ruvsense/longitudinal.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/longitudinal.rs) |

> **A small arithmetic note, stated plainly.** The firmware documents the heart
> band as "0.8–2.0 Hz → 40–120 BPM". $0.8\ \text{Hz} \times 60 = 48$ BPM, not 40.
> The band as implemented bottoms out at 48 BPM; the quoted 40 comes from the
> zero-crossing estimator's reporting range, not the filter. If you are
> validating bradycardia detection, this matters. **[CODE]**

Two sampling constraints follow directly:

> **[PHYS] Nyquist and window length**
>
> 1. **Nyquist:** to observe a 2 Hz heartbeat you must sample above 4 Hz. At the
>    firmware's ~20 Hz CSI rate you have 10 Hz of usable bandwidth — comfortable.
> 2. **Frequency resolution:** $\Delta f = 1/T$ where $T$ is the observation
>    window. To distinguish 12 BPM (0.20 Hz) from 14 BPM (0.233 Hz) you need
>    $\Delta f < 0.033$ Hz, so $T > 30$ s.
>
> **This is why the enrollment anchors for breathing are 30 seconds and the
> others are 15.** It is not an arbitrary UX choice; it is the resolution
> requirement. See [`plans/ruview-esp32s3-test-plan.md` Step 7.3](../plans/ruview-esp32s3-test-plan.md).

## 2.9 The hardware lies: CFO, SFO, and why raw phase is garbage

Here is the fact that surprises everyone new to CSI: **the phase you read out of
a commodity WiFi chip is not the physical channel phase.** It is the physical
phase plus a pile of hardware artefacts that are often larger than the signal.

| Artefact | Cause | Effect on measured phase |
|----------|-------|--------------------------|
| **Carrier Frequency Offset (CFO)** | TX and RX local oscillators are not identical | Adds a term that drifts packet to packet |
| **Sampling Frequency Offset (SFO)** | ADC clocks differ | Adds a term *linear in subcarrier index* |
| **Packet Detection Delay (PDD)** | Symbol timing estimate jitters | Also linear in subcarrier index, random per packet |
| **Phase-Locked Loop noise** | Oscillator instability | Random walk |
| **Automatic Gain Control (AGC)** | RX gain changes per packet | Amplitude scale jumps discontinuously |

Consequence: **raw per-packet phase is effectively random.** Any pipeline that
uses it naively produces noise and calls it a signal.

There are three standard remedies, and this repo uses all three where applicable:

1. **Linear detrending / phase sanitization.** SFO and PDD are both linear in
   subcarrier index, so unwrap the phase across subcarriers and subtract the
   best-fit line. This removes the linear artefacts at the cost of destroying
   absolute time-of-flight. Implemented in
   [`phase_sanitizer.rs`](../v2/crates/wifi-densepose-signal/src/phase_sanitizer.rs). **[CODE]**

2. **Conjugate multiplication (the CSI-ratio model).** If you have two antennas
   sharing one RF chain, they share the same CFO/SFO/PDD. Computing
   $H_i[k] \cdot \overline{H_j[k]}$ cancels all common-mode terms and leaves only
   the environmental difference. This is the technique from SpotFi (SIGCOMM 2015)
   and IndoTrack (MobiCom 2017), implemented in
   [`csi_ratio.rs`](../v2/crates/wifi-densepose-signal/src/csi_ratio.rs). **[CODE]**

   > **Important limitation for this hardware.** The ESP32-S3 is **single
   > antenna** — the ADR-018 frame header carries `n_ant = 1`. Conjugate
   > multiplication across antennas is therefore *unavailable* on the primary
   > node hardware. That path exists for multi-antenna sources (Intel 5300,
   > Atheros, MediaTek MIMO). On ESP32 the system compensates with amplitude-based
   > features, sanitized phase, and **multiple nodes instead of multiple
   > antennas**. This is a real and under-appreciated architectural consequence.

3. **Differential / baseline-referenced measurement.** Instead of asking "what is
   the phase", ask "how has the phase changed relative to the empty-room
   baseline". Artefacts that are stationary over the measurement window cancel.
   This is the entire basis of
   [ADR-135](adr/ADR-135-empty-room-baseline-calibration.md) and the `calibrate`
   command.

## 2.10 Fresnel zones: why placement dominates everything

Between any transmitter and receiver there is a family of nested ellipsoids —
**Fresnel zones** — defined by path-length difference. A reflector inside the
first Fresnel zone contributes to the received signal roughly in phase with the
direct path; one in the second contributes out of phase, and so on.

> **[PHYS] First Fresnel zone radius**
>
> $$r_n = \sqrt{\frac{n \lambda d_1 d_2}{d_1 + d_2}}$$
>
> For a 5 m link with the subject at the midpoint ($d_1 = d_2 = 2.5$ m), $n=1$,
> $\lambda = 0.125$ m:
> $$r_1 = \sqrt{\frac{1 \times 0.125 \times 2.5 \times 2.5}{5}} = \sqrt{0.156} \approx 0.40\ \text{m}$$

The practical consequence is severe and counter-intuitive: **as a chest moves, it
crosses Fresnel boundaries, and the received amplitude oscillates.** At some
positions breathing produces a strong sinusoid; move the person 3 cm and the same
breathing produces almost nothing (a "blind spot"), or produces a *doubled*
apparent rate as the chest crosses a boundary twice per breath.

This is modelled in [`fresnel.rs`](../v2/crates/wifi-densepose-signal/src/fresnel.rs)
and studied in [`examples/research-sota/01-physics-floor/`](../examples/research-sota/01-physics-floor/).

It is *the* reason the repo's placement study reports a **93× sensing-coverage
difference between naive and physics-aware placement** **[SYNTH]**, and why the
test plan puts node placement (Step 2.4) before any hardware is powered on.
Placement is not a detail. Placement is the dominant term.

## 2.11 Multistatic geometry: why more nodes beats a better node

A single TX–RX pair gives you one projection of the scene. It cannot distinguish
"a small motion nearby" from "a large motion far away", and it has blind zones
where the geometry is unfavourable.

With $N$ nodes you get up to $N(N-1)$ ordered links (or $N$ receive viewpoints of
a common illuminator). Each link is a different projection.

```
       N=1                        N=4
   TX ────► RX          N1 ◄──────────► N2
                         ▲ ╲          ╱ ▲
   one projection        │  ╲   ▣    ╱  │      12 ordered links
   many blind spots      │   ╲      ╱   │      → geometric diversity
                         ▼    ╲    ╱    ▼      → blind spots of one link
                        N4 ◄───────────► N3      are covered by another
```

Three distinct benefits, and it is worth keeping them separate:

1. **Coverage.** A person invisible to node 1's geometry is visible to node 3's.
2. **Disambiguation.** Intersecting constraints from several links localise a
   perturbation that a single link can only detect, not place.
3. **Spoof resistance.** A real body perturbs *every* link that passes through
   its location, in a geometrically consistent way. An injected fake signal on one
   link cannot satisfy the constraints of the others. This is precisely the check
   in [`ruvsense/adversarial.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/adversarial.rs). **[CODE]**

The repo's placement study finds **100% coverage of a 1–4 occupant household at
N = 5 anchors** **[SYNTH]** — which is the stated reason the reference deployment
is six nodes: you are at the plateau, with one spare.

## 2.12 Bandwidth is resolution — and why channel hopping exists

Radar resolution in range is set by bandwidth, not by frequency:

> **[PHYS] Range resolution**
>
> $$\Delta R = \frac{c}{2B}$$
>
> - 20 MHz (one WiFi channel): $\Delta R = \dfrac{3\times10^8}{2 \times 2\times10^7} = 7.5\ \text{m}$
> - 60 MHz (channels 1 + 6 + 11 stitched): $\Delta R = 2.5\ \text{m}$
> - 160 MHz (802.11ax wide channel): $\Delta R = 0.94\ \text{m}$
> - 4 GHz (a 60 GHz mmWave radar): $\Delta R = 3.75\ \text{cm}$

**Read that first line again.** With a single 20 MHz WiFi channel you cannot
resolve two reflectors closer together than 7.5 metres. You are not building an
imaging radar; you are building a *perturbation detector* with essentially no
native range resolution.

This one number explains an enormous amount of this project's architecture:

- **Why channel hopping exists.** Hopping 1/6/11 and stitching the bands (after
  correcting the per-channel oscillator phase offset) synthesises 60 MHz of
  aperture — 3× the range resolution. That is
  [`ruvsense/multiband.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/multiband.rs)
  + [`ruvsense/phase_align.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/phase_align.rs),
  specified in [ADR-029](adr/ADR-029-ruvsense-multistatic-sensing-mode.md) §2.3. **[CODE]**
- **Why sparse recovery is used for CIR.** With only 52–64 frequency samples you
  cannot simply FFT your way to a clean impulse response. You must exploit the
  fact that the true CIR is *sparse* (few dominant reflectors) and solve an
  underdetermined inverse problem with an $\ell_1$ penalty. That is ISTA in
  [`ruvsense/cir.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs),
  per [ADR-134](adr/ADR-134-csi-to-cir-time-domain-multipath.md). **[CODE]**
- **Why the system claims "presence and vitals", not "imaging".** Anyone claiming
  centimetre-accurate skeletal imaging from a single 20 MHz WiFi link is claiming
  something the bandwidth does not support. This repository is careful about
  that; see [Part 6](#part-6--the-honesty-ledger).

## 2.13 Doppler and the Body Velocity Profile

A moving reflector shifts the frequency of what it reflects:

> **[PHYS] Doppler shift**
>
> $$f_d = \frac{2v}{\lambda}$$
>
> | Motion | $v$ | $f_d$ at 2.4 GHz |
> |--------|-----|------------------|
> | Walking | 1.0 m/s | 16 Hz |
> | Slow arm gesture | 0.2 m/s | 3.2 Hz |
> | Chest, breathing | ~5 mm/s | 0.08 Hz |
> | Chest, heartbeat | ~1 mm/s | 0.016 Hz |

Since different body parts move at different speeds simultaneously, the Doppler
*spectrum* at any instant is a signature of the whole-body motion pattern. Plot
that spectrum against time and you get the **Body Velocity Profile (BVP)** —
implemented in [`bvp.rs`](../v2/crates/wifi-densepose-signal/src/bvp.rs).

BVP's importance is not sensitivity, it is **generalisation**. Raw CSI values are
hopelessly environment-specific: the same gesture in two rooms produces
completely different CSI. But *velocity distributions* are much more nearly
environment-invariant, because velocity is a property of the body, not the room.
This is the core insight of the Widar3.0 line of work, and it is why BVP is a
standard intermediate representation in the field.

## 2.14 Where the physics says "no"

An honest theory document must include the negative results. This repository has
them, in [`examples/research-sota/07-negative-results/`](../examples/research-sota/07-negative-results/).

| Claim you will see elsewhere | Physical verdict |
|---|---|
| **Contactless blood pressure from CSI** | The required pulse-transit-time discrimination sits about **5 dB below** what the link budget can deliver. Physically blocked at these frequencies and bandwidths, not merely unsolved. **[SYNTH]** |
| **Centimetre-resolution imaging from one 20 MHz link** | $\Delta R = 7.5$ m. Not possible without synthesising bandwidth or aperture. **[PHYS]** |
| **Identity recognition that is "anonymous"** | Gait is identifying. If a system can track individuals across sessions, it is handling biometric data regardless of the marketing. |
| **Reliable through reinforced concrete** | 25 dB+ one-way; 50 dB+ round-trip. Effectively opaque. **[PHYS]** |

Knowing where the wall is makes the space in front of it much easier to work in.

---

# Part 3 — From signal to inference

Physics gives you a matrix of complex numbers. Applications want a sentence:
*"someone is in the bedroom, breathing normally."* This part is the bridge.

## 3.1 The central difficulty: domain shift

Before any technique, understand the problem that defines this field.

> **A model trained on CSI from one room does not work in another room.**

Not "works a bit worse" — often near chance. The reasons are structural:

- The multipath structure is a fingerprint of the *room*, not the person. Move
  the furniture and the fingerprint changes.
- The transmitter and receiver positions set the whole geometry. A model can
  trivially learn "subcarrier 23 goes up when someone sits" — a fact true only in
  the room it was trained in. This failure has a name in the literature:
  **coordinate overfitting**, diagnosed by PerceptAlign and cited directly in
  [`calibration/geometry.rs`](../v2/crates/wifi-densepose-calibration/src/geometry.rs). **[CODE]**
- Chipsets differ in subcarrier count, scaling, and phase artefacts.
- Even the *same* room drifts: temperature, humidity, an open door, a new router.

There are four broad strategies, and it is worth knowing which one any given
system has picked, because it determines everything else about its architecture:

| Strategy | Idea | Cost | Who uses it |
|---|---|---|---|
| **Domain-invariant features** | Transform to a representation the room cannot see — BVP, Doppler spectra, velocity | Loses fine detail | Widar3.0 line; [`bvp.rs`](../v2/crates/wifi-densepose-signal/src/bvp.rs) here |
| **Massive diverse training** | Train on enough rooms that the model generalises | Needs a huge, expensive dataset | Large-model WiFi-pose research |
| **Per-room calibration** | Accept the room-specificity; *learn each room quickly and cheaply* | Requires an enrollment ritual per install | **RuView's primary bet** — [ADR-151](adr/ADR-151-room-calibration-specialist-training.md) |
| **Adaptation / transfer** | Pretrain a general encoder, fine-tune per room with a small adapter | Best of both, most machinery | RuView's stated upgrade path (LoRA over a frozen RF encoder) |

**RuView's architectural thesis, stated plainly:** *specialisation over scale.*
Rather than one large model that must generalise across all rooms, fit a bank of
tiny models to *your* room in a few minutes, and be honest when the room drifts
far enough that the bank is stale. This is why the calibration crate is pure
Rust with no neural framework at all — the specialists are thresholds, prototype
vectors, and band-limited periodicity reads. **[CODE]**

## 3.2 Layer 1 — Cleaning: making the measurement trustworthy

| Stage | Technique | Why | Implementation |
|---|---|---|---|
| Outlier removal | **Hampel filter** — running median + MAD | Mean/σ methods are corrupted by the very outliers they should catch; median/MAD tolerate up to 50% contamination | [`hampel.rs`](../v2/crates/wifi-densepose-signal/src/hampel.rs) |
| Phase repair | Unwrap + linear detrend | Removes SFO/PDD (both linear in subcarrier index) — see [§2.9](#29-the-hardware-lies-cfo-sfo-and-why-raw-phase-is-garbage) | [`phase_sanitizer.rs`](../v2/crates/wifi-densepose-signal/src/phase_sanitizer.rs) |
| Hardware normalisation | Resample to canonical 56 bins, z-score amplitude | Lets one model span Intel/Atheros/ESP32/Broadcom | [`hardware_norm.rs`](../v2/crates/wifi-densepose-signal/src/hardware_norm.rs) (ADR-027) |
| Static suppression | EMA subtraction of the slowly-varying component | The room's static reflections dwarf the human perturbation; subtract the mean to see the residual | [`vitals/preprocessor.rs`](../v2/crates/wifi-densepose-vitals/src/preprocessor.rs) |
| Baseline referencing | Welford mean/variance + von Mises phase statistics of the **empty** room | Converts "what is the value" into "how surprising is this value" | [`ruvsense/calibration.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/calibration.rs) (ADR-135) |

Two statistical tools appear repeatedly and are worth understanding on their own:

**Welford's online algorithm** computes running mean and variance in one pass
with no accumulated round-off, without storing the samples. That property —
constant memory, numerically stable — is what makes it viable *on the
microcontroller* as well as on the host. It is used for the empty-room baseline,
for longitudinal drift tracking, and inside the firmware's Tier 1 DSP.

**Von Mises statistics** are the circular analogue of the Gaussian. Phase lives on
a circle: the mean of 359° and 1° is 0°, not 180°. Any code that averages phase
linearly is wrong, subtly and intermittently. The baseline calibration uses
circular statistics precisely to avoid this.

## 3.3 Layer 2 — Representation: choosing what to look at

| Representation | What it captures | Best for | Implementation |
|---|---|---|---|
| **Per-subcarrier amplitude/phase stats** | Variance, mean, median, range | Presence, motion — cheap and robust | [`features.rs`](../v2/crates/wifi-densepose-signal/src/features.rs) |
| **Top-K sensitive subcarriers** | Which subcarriers actually respond to motion (variance ratio) | +6–10 dB SNR vs. using all of them | [`subcarrier_selection.rs`](../v2/crates/wifi-densepose-signal/src/subcarrier_selection.rs) |
| **Spectrogram (STFT)** | Time–frequency energy | Activity recognition; the standard CNN input since ~2018 | [`spectrogram.rs`](../v2/crates/wifi-densepose-signal/src/spectrogram.rs) |
| **Body Velocity Profile** | Velocity distribution over time | Cross-environment gesture/activity | [`bvp.rs`](../v2/crates/wifi-densepose-signal/src/bvp.rs) |
| **CIR (sparse, via ISTA)** | Reflector delays → distances | Multipath separation, RF-SLAM | [`ruvsense/cir.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs) (ADR-134) |
| **Learned embedding (128-dim)** | Whatever the contrastive objective found useful | Re-ID, similarity search, downstream heads | [`wifi-densepose-aether`](../v2/crates/wifi-densepose-aether/) (ADR-024) |

**Why top-K subcarrier selection matters more than it sounds.** Of ~56
subcarriers, only a handful are well-positioned in the Fresnel geometry to
respond to a given subject at a given place. The rest contribute noise. Ranking
by (motion variance / static variance) and keeping the top few is one of the
highest-return-per-line-of-code operations in the entire pipeline — and it is
also what the firmware does at Tier 1 to cut bandwidth.

## 3.4 Layer 3 — Estimation: rate extraction

For breathing and heart rate the task is: given a noisy 1-D series, find the
dominant periodicity in a known band.

Three methods, in increasing robustness:

1. **Zero-crossing counting** after bandpass filtering. Cheap enough for a
   microcontroller (the firmware uses a biquad IIR bandpass plus zero crossings).
   Fragile: a single noise-induced crossing inflates the rate.
2. **Autocorrelation peak.** Correlate the signal with itself at increasing lags;
   the first strong peak inside the band is the period. Far more robust to
   additive noise, because noise decorrelates and the periodic component does
   not. This is what fixed the heart-rate estimator in this project
   (issue #987), and it is what
   [`calibration/extract.rs`](../v2/crates/wifi-densepose-calibration/src/extract.rs)
   uses. **[CODE]**
3. **Spectral peak with inter-subcarrier coherence weighting.** Compute the
   spectrum on many subcarriers and weight each by how consistent its phase is
   with the others. A real physiological rhythm is coherent across subcarriers;
   noise is not. Used in
   [`vitals/heartrate.rs`](../v2/crates/wifi-densepose-vitals/src/heartrate.rs). **[CODE]**

> **A trap worth naming.** Breathing harmonics land inside the heart band. A
> 20 BPM respiration (0.33 Hz) has a third harmonic at 1.0 Hz = 60 BPM — squarely
> in the middle of a plausible heart rate. A naive heart-rate estimator will
> confidently report your breathing's third harmonic as your pulse. This is why
> the test plan's heart-rate protocol has you **hold your breath**: it removes
> the interferer so you can tell whether you are measuring a heart or an
> artefact.

## 3.5 Layer 4 — Decision: from numbers to states

| Decision | Method here | Honest limitation |
|---|---|---|
| **Presence (firmware)** | Phase-variance vs. an adaptively-learned threshold, 60 s ambient calibration at boot | **A variance threshold, not a model.** False-positives on fans, microwaves, AP power swings. Documented as such in the firmware README. **[FIELD]** |
| **Presence (calibrated)** | `PresenceSpecialist` fit against the labelled `empty` anchor | Room-specific; degrades honestly to `STALE` when the baseline drifts |
| **Posture** | Nearest-prototype over enrollment anchor features | Only distinguishes postures that were *distinguishable at enrollment* |
| **Fall** | Phase-acceleration threshold + 3-frame debounce + 5 s cooldown | Threshold detector. Tunable via NVS `fall_thresh` without reflashing |
| **Person count (firmware)** | Slot-capacity heuristic over top-K subcarrier groups | **Tracks subcarrier diversity, not people.** Reads 6–8 for 5 bodies. **[FIELD]** |
| **Person count (host)** | Stoer-Wagner min-cut on a cross-link correlation graph | Better-founded; still characterisation, not validated accuracy |
| **Anomaly / veto** | `AnomalySpecialist` holds a veto over physically implausible windows | Prevents confident hallucination; can suppress true positives |

**The min-cut idea is worth a paragraph** because it is genuinely elegant. Build
a graph whose nodes are (link, subcarrier-group) observations and whose edge
weights are correlations. Two people moving independently produce two weakly
connected clusters; the minimum cut of the graph separates them. The Stoer-Wagner
algorithm finds the global minimum cut without needing a source and sink. It also
provides an adversarial check: a spoofed signal injected on one link produces a
graph that cannot be cut consistently
([`ruvector/`](../v2/crates/wifi-densepose-ruvector/) + `ruvector-mincut`).

## 3.6 Layer 5 — Fusion: combining evidence honestly

With six nodes you have six opinions. Naive averaging is wrong — a node with no
line of sight should not dilute a node that has one.

The rules actually implemented in
[`calibration/multistatic.rs`](../v2/crates/wifi-densepose-calibration/src/multistatic.rs)
are per-quantity, and each rule encodes a physical argument: **[CODE]**

| Quantity | Fusion rule | The physical argument |
|---|---|---|
| Presence | **OR** across nodes | Any node that sees a person is evidence of a person; a node that sees nothing may simply be blocked |
| Posture / breathing / heart | **Highest-confidence node wins** | These need a *good viewpoint*, and averaging a good view with a bad one is worse than taking the good one |
| Restlessness | **Max** | Motion anywhere is motion |
| Anomaly | **Any** (veto propagates) | One node detecting something physically impossible invalidates the room |
| Stale | **Any** | If any bank's baseline has drifted, say so |

Alongside this sits a *quality* layer — [ADR-137](adr/ADR-137-fusion-engine-quality-scoring-evidence.md),
[`ruvsense/fusion_quality.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/fusion_quality.rs) —
in which every fusion stage emits not just a value but a `QualityScore` naming
the **evidence** that justified it and the **contradictions** it tolerated. A
contradiction lowers the downstream privacy class. This is the "trust
throughline": the system can always answer *why should I believe this?*

## 3.7 Layer 6 — Tracking: persistence over time

Instantaneous detections flicker. Tracking imposes temporal consistency:

- **Kalman filtering** with a constant-velocity model. Each tracked keypoint
  carries a 6-D state $(x,y,z,v_x,v_y,v_z)$; the filter predicts forward and
  corrects on measurement. [`ruvsense/pose_tracker.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/pose_tracker.rs)
- **Track lifecycle** — `Tentative → Active → Lost → Terminated` — so a
  one-frame blip never becomes a reported person.
- **Data association** by a joint cost: 60% Mahalanobis distance (is this
  measurement geometrically plausible for this track?) + 40% embedding cosine
  similarity (does it *look* like the same person?). The embedding term is what
  prevents identity swaps when two people cross.
- **Coherence gating** — [`ruvsense/coherence_gate.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/coherence_gate.rs) —
  which decides per frame whether to `Accept` the measurement, run
  `PredictOnly` with inflated noise, `Reject` it, or trigger `Recalibrate` after
  10 s of sustained low coherence. This is how the system avoids slowly absorbing
  environmental drift into its model of a person. **[CODE]**

## 3.8 Layer 7 — Learning: where the neural networks actually are

| Model | What it is | Where it lives | Status |
|---|---|---|---|
| **Contrastive CSI encoder** | 128-dim embedding, trained self-supervised on 60K frames / 610K triplets | HF [`ruvnet/wifi-densepose-pretrained`](https://huggingface.co/ruvnet/wifi-densepose-pretrained) | 82.3% held-out temporal-triplet accuracy **[BENCH]** |
| **Presence head** | Small classifier over the embedding | Same bundle | Ships |
| **MM-Fi pose model** | 17-keypoint pose from CSI | HF [`ruvnet/wifi-densepose-mmfi-pose`](https://huggingface.co/ruvnet/wifi-densepose-mmfi-pose) | 82.69% torso-PCK@20 on MM-Fi `random_split` **[BENCH]** |
| **On-device `pose_v1`** | The pose model actually committed for edge use | [`cog-pose-estimation`](../v2/crates/cog-pose-estimation/) | PCK@20 = 3.0%, below the ≥35% target; runtime path is a `confidence=0` stub **[CODE]** |
| **Specialist bank** | Presence/posture/breathing/heartbeat/restlessness/anomaly | [`wifi-densepose-calibration`](../v2/crates/wifi-densepose-calibration/) | **Not neural** — statistical, per-room, edge-deployable |

Two things to internalise:

1. **The impressive benchmark numbers and the live pipeline are different
   things.** 82.69% torso-PCK on MM-Fi is a real result on a research dataset
   with a stated protocol. The model running on your ESP32 is not that model.
   The repo states this explicitly and so does this document.
2. **Self-supervised pretraining is the strategy that makes this tractable.**
   Labelling WiFi data requires synchronised ground truth (a camera, a chest
   belt, a pulse oximeter) which is expensive and privacy-invasive to collect.
   Contrastive learning over *temporal* structure — frames close in time should
   embed close together — needs no labels at all. That is why the encoder is the
   asset and the heads are cheap.

## 3.9 The trust chain: proof, witness, evidence

An unusual and, frankly, admirable feature of this project is that it treats
"can this claim be checked" as a first-class engineering requirement.

- **The deterministic proof** ("Trust Kill Switch"):
  [`archive/v1/data/proof/verify.py`](../archive/v1/data/proof/verify.py) pushes
  1,000 synthetic CSI frames (seed 42) through the production pipeline and
  SHA-256s the feature output against a committed hash. If the DSP silently
  changes, this fails. One command, no hardware.
- **Witness chains** ([ADR-010](adr/ADR-010-witness-chains-audit-trail-integrity.md)):
  Ed25519-signed attestation of measurements, so a recorded result can be shown
  to be the one actually produced.
- **The capability matrix**: [`docs/WITNESS-LOG-028.md`](WITNESS-LOG-028.md) — a
  33-row table of claims with per-claim evidence, distinguishing *empirical* from
  *claimed*.
- **Evidence levels**: `ruview-unified` (ADR-273..282) explicitly labels its
  numbers **L0** — synthetic until real-data validation.

If you take one practice from this repository into your own work, take this one.

---

# Part 4 — The RuView system, component by component

## 4.0 The layer map

```
┌─────────────────────────────────────────────────────────────────────────┐
│ L6  CONSUMERS      Browser UI · Home Assistant · Apple Home / HAP ·      │
│                    Matter · MQTT · Python wheels · MCP · Desktop (Tauri) │
├─────────────────────────────────────────────────────────────────────────┤
│ L5  SEMANTICS      HomeCore state machine · semantic states · automation │
│                    ("someone-sleeping", "possible-distress", "bed-exit")  │
├─────────────────────────────────────────────────────────────────────────┤
│ L4  WORLD MODEL    WorldGraph · RF-SLAM · tomography · evolution tracker │
│                    ruview-unified (ADR-273..282) · OccWorld prediction   │
├─────────────────────────────────────────────────────────────────────────┤
│ L3  INFERENCE      Specialist banks (per room) · vitals · pose tracker · │
│                    embeddings + HNSW · adaptive classifier · MAT         │
├─────────────────────────────────────────────────────────────────────────┤
│ L2  FUSION         Multistatic fusion · coherence gating · quality       │
│                    scoring · cross-viewpoint attention · array coord.    │
├─────────────────────────────────────────────────────────────────────────┤
│ L1  DSP            Hampel · phase sanitize · hardware normalise ·        │
│                    subcarrier select · CIR/ISTA · spectrogram · BVP      │
├─────────────────────────────────────────────────────────────────────────┤
│ L0  TRANSPORT      UDP :5005 · ADR-018 frames · TDM · aggregator         │
├─────────────────────────────────────────────────────────────────────────┤
│ L-1 EDGE           ESP32-S3/C6 firmware · tiers 0–3 · NVS · OTA · WASM   │
├─────────────────────────────────────────────────────────────────────────┤
│ L-2 PHYSICS        WiFi radio · CSI · the room                           │
└─────────────────────────────────────────────────────────────────────────┘
```

Every subsection below answers three questions: **what it is**, **why it exists**
(what breaks without it), and **its role in the whole**.

---

## 4.1 The sensor node — ESP32-S3

### What it is

A dual-core Xtensa LX7 microcontroller at 240 MHz with an integrated 2.4 GHz
802.11b/g/n radio. The boards used in this project are `ESP32-S3FN8`-class:
**8 MB in-package flash, no PSRAM, native USB-Serial/JTAG**, ~$6–10 each.

### Why this chip

| Requirement | Why the ESP32-S3 satisfies it |
|---|---|
| **Must expose CSI** | Espressif's SDK provides a CSI callback (`esp_wifi_set_csi_rx_cb`) giving per-subcarrier I/Q. Almost no other consumer radio does this without patched drivers. |
| **Must be cheap enough to deploy 6 per room** | ~$9. At $200/node the multistatic architecture is economically dead. |
| **Must run real DSP locally** | Two cores at 240 MHz: one for the radio, one for signal processing. Enough for biquad filtering, Welford statistics, and threshold detection at 20 Hz. |
| **Must be programmable in the field** | OTA app update + NVS runtime config + (with PSRAM) a WASM sandbox. |
| **Must be well documented and buyable** | ESP-IDF is mature, open, and the parts are in stock everywhere. |

### What it costs you

| Limitation | Consequence |
|---|---|
| **Single antenna** | No conjugate-multiplication CFO/SFO cancellation ([§2.9](#29-the-hardware-lies-cfo-sfo-and-why-raw-phase-is-garbage)). Compensated by multiple nodes. |
| **20 MHz / 2.4 GHz only** (S3) | Range resolution 7.5 m ([§2.12](#212-bandwidth-is-resolution--and-why-channel-hopping-exists)). Compensated by channel hopping. Will not associate with a 5 GHz-only SSID. |
| **No PSRAM on these boards** | Tier 3 WASM programmable sensing is unavailable — it needs 4 × 160 KB arenas. |
| **`WIFI_PS_NONE` required** | Modem sleep must be off for continuous CSI → sustained current draw → **thermal risk on coin-sized clones**. Real field reports of regulator failure. |
| **Native-USB console** | Without `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` the serial monitor is silent and you will misdiagnose a healthy board as dead. |

### The alternatives, and why they are supported anyway

| Hardware | Role in this project |
|---|---|
| **ESP32-C6** ([ADR-110](adr/ADR-110-esp32-c6-firmware-extension.md)) | WiFi 6 (HE-LTF subcarriers), 802.15.4 mesh, TWT power saving, LP-core wake-on-motion. Research target; same firmware, different overlay. |
| **Intel 5300 / Atheros AR9580** | 3×3 MIMO, real multi-antenna CSI. The reference research NICs. |
| **MediaTek Filogic** ([ADR-266](adr/ADR-266-mediatek-filogic-csi-platform.md)) | MIMO CSI wire protocol; adapter in [`hardware/mediatek_csi.rs`](../v2/crates/wifi-densepose-hardware/src/mediatek_csi.rs) |
| **Qualcomm/Atheros** ([ADR-268](adr/ADR-268-qualcomm-atheros-csi-platform.md)) | `QCS1` codec + deterministic replay; hardware adapter gated (modern ath11k/ath12k do not publicly export CSI) |
| **RTL8720F** ([ADR-263](adr/ADR-263-rtl8720f-2-4ghz-fmcw-radar-platform.md)) | 2.4 GHz FMCW radar mode — a genuinely different modality on adjacent silicon |
| **Any laptop** | RSSI-only. Coarse presence and motion, no vitals. [`wifi-densepose-wifiscan`](../v2/crates/wifi-densepose-wifiscan/) |

The vendor-adapter program ([ADR-270](adr/ADR-270-vendor-rf-sensing-integration-program.md))
has an explicit rule worth noting: adapters whose capability is
telemetry-only or network-only **cannot masquerade as CSI**. The type system
refuses to let a weaker source be silently substituted for a stronger one.

---

## 4.2 The firmware — [`firmware/esp32-csi-node/`](../firmware/esp32-csi-node/)

ESP-IDF v5.4 C. Version `0.8.4`. Dual-target S3 / C6.

### Task structure

```
Core 0 — radio                        Core 1 — DSP
────────────────                      ─────────────
WiFi STA + CSI callback   ──ring──►   Tier 0: passthrough
Channel hopping (ADR-029)  buffer     Tier 1: unwrap, Welford, top-K
NDP injection             (SPSC)      Tier 2: vitals, presence, fall
TDM slot management                   Tier 3: WASM dispatch
────────────────────────────────────────────────────────────────
NVS config │ OTA + HTTP :8032 │ UDP sender │ power management
```

The **single-producer/single-consumer ring buffer** between cores is the critical
piece of engineering: the CSI callback fires in radio context hundreds of times
per second and must never block. It writes and returns; Core 1 drains at its own
pace and drops when it falls behind. Blocking here is what causes the classic
lwIP pbuf exhaustion crash, addressed in ADR-018 with a 50 Hz rate limiter and an
`ENOMEM` backoff. **[CODE]**

### The four tiers — the central design idea

| Tier | Runs on node | On the wire | Bandwidth | Use it for |
|---|---|---|---|---|
| **0** | Nothing — raw passthrough | Full CSI frames @ ~20 Hz | ~5 KB/s | **Calibration, enrollment, training, all host DSP** |
| **1** | Phase unwrap, Welford, top-K | Compressed CSI | Reduced | Bandwidth-constrained links |
| **2** | Full DSP: vitals, presence, fall | 32-byte vitals packet @ 1 Hz (+ CSI) | Tiny | **Standalone node, no host needed** |
| **3** | WASM modules | Custom events | Variable | Programmable sensing — **needs PSRAM** |

Why tiering is the right abstraction: it makes the **compute/bandwidth trade-off
a runtime decision**, not an architectural one. The same firmware image is a dumb
sensor feeding a powerful host, or an autonomous device that reports semantic
events over a 32-byte packet. Switching is one NVS write and a power cycle — **no
reflash**.

> **The coupling that catches everyone:** `edge_tier` and CSI availability are not
> independent. `calibrate` / `enroll` / `room-watch` need **raw CSI (tier 0)**.
> Tier 2 substitutes the compact vitals packet. Provision per experiment.

### Runtime configuration — NVS

Settings live in the NVS partition at flash offset `0x9000` and override
compiled-in Kconfig defaults.
[`provision.py`](../firmware/esp32-csi-node/provision.py) writes them over serial.

| Key | Flag | Meaning |
|---|---|---|
| `ssid` / `password` | `--ssid` / `--password` | WiFi credentials |
| `target_ip` / `target_port` | `--target-ip` / `--target-port` | Where CSI is sent |
| `node_id` | `--node-id` | 0–255, **must be unique** — duplicates silently merge two physical nodes into one logical one |
| `edge_tier` | `--edge-tier` | 0 / 1 / 2 |
| `tdm_slot` / `tdm_nodes` | `--tdm-slot` / `--tdm-total` | Mesh transmit scheduling |
| `chan_list` / `dwell_ms` | `--hop-channels` / `--hop-dwell` | Channel hopping |
| `fall_thresh` | `--fall-thresh` | Fall sensitivity (default 2.0 rad/s²) |
| `power_duty` | `--power-duty` | Duty cycle — the thermal mitigation |

`provision.py --state` prints the merged state **without flashing**, which is the
correct first move when a node behaves unexpectedly.

### Why TDM

Six nodes transmitting continuously on one channel interfere with each other
*and* with the WiFi you are trying to measure. Time-division multiplexing gives
each node a slot in a repeating cycle. `--tdm-total` tells each node how many
slots exist so it can compute its cycle offset. Without it, six nodes is often
worse than two.

### The failure mode you must understand before touching hardware

> **The display false-positive → 0 pps CSI starvation.** **[FIELD]**
>
> The firmware supports an optional SH8601 QSPI AMOLED panel. That panel is
> **write-only** — there is no readback path — so `display_hal_init_panel()`
> "succeeds" on a board with no panel attached. When the firmware believes it has
> a display, it keeps the promiscuous filter at MGMT-only and never calls
> `csi_collector_enable_data_capture()`.
>
> **The result: CSI yield collapses to 0 packets/second while the node
> associates, pings, and looks perfectly healthy on the network.** The current
> source recovers by additionally requiring FT3168 touch readback
> ([`main/display_task.c:121`](../firmware/esp32-csi-node/main/display_task.c)),
> but the reliable fix is to compile display support out entirely
> (`# CONFIG_DISPLAY_ENABLE is not set`), making `has_display` a **compile-time**
> false.
>
> This is the single nastiest failure in the project, it wastes hours, and it is
> why the test plan builds firmware from source rather than using the prebuilt
> binaries (which predate the fix).

---

## 4.3 The wire protocols

Four UDP message types, discriminated by a 4-byte magic. All little-endian.

| Magic | Name | Rate | Size |
|---|---|---|---|
| `0xC5110001` | CSI frame (ADR-018) | ~20 Hz | 20-byte header + I/Q |
| `0xC5110002` | Vitals packet | 1 Hz | 32 bytes |
| `0xC5110003` | Seed feature vector | 1 Hz | 48 bytes |
| `0xC5110004` | WASM module output | event | variable |

### ADR-018 CSI frame layout

```
Offset  Size  Field
  0      4    Magic 0xC5110001 (LE)
  4      1    Node ID (0–255)
  5      1    Number of antennas
  6      2    Number of subcarriers (LE u16)
  8      4    Frequency MHz (LE u32)
 12      4    Sequence number (LE u32)
 16      1    RSSI (i8, dBm)
 17      1    Noise floor (i8, dBm)
 18      2    Reserved  ← C6 uses these for HE-LTF tagging
 20    N*2    I/Q pairs (i8, i8) per subcarrier, repeated per antenna
```

Total = `20 + n_ant × n_sc × 2` bytes. For 1 antenna × 52 subcarriers: 124 bytes.

**Design decisions worth noting:**

- **8-bit I/Q, not floats.** Quantisation is acceptable because the useful signal
  is a *change*, not an absolute level, and it keeps a node's stream at ~5 KB/s.
- **Sequence numbers.** Let the host detect loss without a reliability layer.
- **UDP, not TCP.** A late CSI frame is worthless; retransmission would add
  latency and jitter for no benefit. Loss is preferable to delay.
- **Per-frame node ID.** Makes the aggregator stateless with respect to source
  addresses — which is exactly what makes the Docker-on-Windows relay workaround
  possible.
- **RSSI and noise floor in-band.** Lets the host compute SNR per frame and gate
  on link quality without a side channel.

The parser is [`hardware/esp32_parser.rs`](../v2/crates/wifi-densepose-hardware/src/esp32_parser.rs).
It validates magic, bounds-checks `n_subcarriers ≤ 512`, and can resync a stream
by searching for the magic — because UDP gives you datagram boundaries but a
replayed file does not. Security review of this deserialiser is
[ADR-172](adr/ADR-172-cli-core-csi-deserialiser-security-review.md). **[CODE]**

---

## 4.4 Host DSP — [`wifi-densepose-signal`](../v2/crates/wifi-densepose-signal/)

The largest and most important crate. Two halves.

### Half 1 — classical DSP (crate root)

| Module | Role | Reference |
|---|---|---|
| [`csi_processor.rs`](../v2/crates/wifi-densepose-signal/src/csi_processor.rs) | Core CSI preprocessing and buffering | — |
| [`hampel.rs`](../v2/crates/wifi-densepose-signal/src/hampel.rs) | Robust outlier removal (median + MAD) | Hampel 1974; WiGest, WiDance |
| [`phase_sanitizer.rs`](../v2/crates/wifi-densepose-signal/src/phase_sanitizer.rs) | Unwrap, detrend, smooth | — |
| [`hardware_norm.rs`](../v2/crates/wifi-densepose-signal/src/hardware_norm.rs) | Cross-chipset canonicalisation to 56 bins | ADR-027 MERIDIAN |
| [`csi_ratio.rs`](../v2/crates/wifi-densepose-signal/src/csi_ratio.rs) | Conjugate multiplication, CFO/SFO cancel | SpotFi 2015; IndoTrack 2017 |
| [`subcarrier_selection.rs`](../v2/crates/wifi-densepose-signal/src/subcarrier_selection.rs) | Top-K by variance ratio (+6–10 dB SNR) | WiDance, WiGest |
| [`fresnel.rs`](../v2/crates/wifi-densepose-signal/src/fresnel.rs) | Fresnel-zone breathing model | FarSense 2019; Wi-Sleep 2021 |
| [`bvp.rs`](../v2/crates/wifi-densepose-signal/src/bvp.rs) | Body Velocity Profile (domain-independent) | Widar3.0 line |
| [`spectrogram.rs`](../v2/crates/wifi-densepose-signal/src/spectrogram.rs) | STFT time–frequency matrices | Standard CNN input |
| [`features.rs`](../v2/crates/wifi-densepose-signal/src/features.rs) | Amplitude/phase/correlation/Doppler/PSD features | — |
| [`motion.rs`](../v2/crates/wifi-densepose-signal/src/motion.rs) | Motion + presence from features | ADR-154 |

### Half 2 — RuvSense multistatic modules ([`src/ruvsense/`](../v2/crates/wifi-densepose-signal/src/ruvsense/))

Specified primarily by [ADR-029](adr/ADR-029-ruvsense-multistatic-sensing-mode.md)
and [ADR-030](adr/ADR-030-ruvsense-persistent-field-model.md). This is the
"sensing-first RF mode" — the argument that you do not need a new WiFi standard,
you need to use the existing one differently.

**The pipeline, in order:**

| # | Module | What it does | Why it must exist |
|---|---|---|---|
| 1 | [`multiband`](../v2/crates/wifi-densepose-signal/src/ruvsense/multiband.rs) | Fuse per-channel frames from hopping into one wideband snapshot | 20 MHz → 60 MHz aperture; 3× range resolution |
| 2 | [`phase_align`](../v2/crates/wifi-densepose-signal/src/ruvsense/phase_align.rs) | Remove the LO phase rotation between channels | Without it the stitched band is incoherent and the extra bandwidth is worthless |
| 3 | [`multistatic`](../v2/crates/wifi-densepose-signal/src/ruvsense/multistatic.rs) | Attention-weighted cross-node fusion; min-cut person separation | Amplifies cells with correlated motion across nodes (a body), suppresses single-node energy (multipath artefact) |
| 4 | [`coherence`](../v2/crates/wifi-densepose-signal/src/ruvsense/coherence.rs) | Score consistency of the observation against a running template | Gives a scalar "do I trust this frame" |
| 5 | [`coherence_gate`](../v2/crates/wifi-densepose-signal/src/ruvsense/coherence_gate.rs) | Accept / PredictOnly / Reject / Recalibrate | Stops the model absorbing environmental drift as if it were a person |
| 6 | [`pose_tracker`](../v2/crates/wifi-densepose-signal/src/ruvsense/pose_tracker.rs) | 17-keypoint Kalman tracking + re-ID | Temporal persistence, no identity swaps |

**Supporting modules** (the "exotic tiers" of ADR-030):

| Module | What it does | Role |
|---|---|---|
| [`field_model`](../v2/crates/wifi-densepose-signal/src/ruvsense/field_model.rs) | SVD of empty-room covariance → eigenmodes; runtime residual = observation − projection | The persistent EM world model. Separates *room drift* from *body perturbation* |
| [`calibration`](../v2/crates/wifi-densepose-signal/src/ruvsense/calibration.rs) | Welford + circular phase baseline; deviation scoring (ADR-135) | The empty-room fingerprint everything else references |
| [`cir`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs) | ISTA/L1 sparse CIR recovery (ADR-134) | Frequency → time domain; "how far away is the reflector" |
| [`tomography`](../v2/crates/wifi-densepose-signal/src/ruvsense/tomography.rs) | Invert per-link attenuations into a coarse 3-D occupancy volume | Turns N links into a spatial map |
| [`rf_slam`](../v2/crates/wifi-densepose-signal/src/ruvsense/rf_slam.rs) | Persistent reflector discovery, static-anchor learning (ADR-143) | Learn the room's furniture; ships fixed-map first, discovery gated |
| [`evolution`](../v2/crates/wifi-densepose-signal/src/ruvsense/evolution.rs) | Cross-link change-point detection + temporal voxel map (ADR-142) | Detect *the room changed*, not *someone moved* |
| [`longitudinal`](../v2/crates/wifi-densepose-signal/src/ruvsense/longitudinal.rs) | Per-person biophysical baselines over days/weeks | Gait symmetry, breathing regularity, micro-tremor drift. **Requires ≥7 days before activating; alerts need >2σ sustained ≥3 days; outputs metrics, never diagnoses** |
| [`attractor_drift`](../v2/crates/wifi-densepose-signal/src/ruvsense/attractor_drift.rs) | Phase-space embedding, Lyapunov exponents | Catches regime changes a z-score cannot see |
| [`gesture`](../v2/crates/wifi-densepose-signal/src/ruvsense/gesture.rs) / [`temporal_gesture`](../v2/crates/wifi-densepose-signal/src/ruvsense/temporal_gesture.rs) | DTW against a gesture template library | Through-wall gesture input |
| [`intention`](../v2/crates/wifi-densepose-signal/src/ruvsense/intention.rs) | Anticipatory postural adjustment detection, 200–500 ms pre-movement | Fall *prediction* rather than fall detection |
| [`cross_room`](../v2/crates/wifi-densepose-signal/src/ruvsense/cross_room.rs) | Identity continuity across rooms via room fingerprints + embedding matching | Track a person through a house without optics |
| [`adversarial`](../v2/crates/wifi-densepose-signal/src/ruvsense/adversarial.rs) | Multi-link consistency, field-model constraints, temporal continuity | Reject spoofed/injected signals |
| [`fusion_quality`](../v2/crates/wifi-densepose-signal/src/ruvsense/fusion_quality.rs) | `QualityScore` + `ContradictionFlag` on every stage (ADR-137) | The trust throughline; feeds privacy classing |
| [`array_coordinator`](../v2/crates/wifi-densepose-signal/src/ruvsense/array_coordinator.rs) | Gate array nodes on geometry and clock quality (ADR-138) | Emits *directional evidence*, deliberately not pose decisions |

---

## 4.5 Per-room calibration — [`wifi-densepose-calibration`](../v2/crates/wifi-densepose-calibration/)

**This is the most important crate to understand**, because it embodies the
project's answer to domain shift ([§3.1](#31-the-central-difficulty-domain-shift)),
and because — unlike the pose path — it is complete and runnable today.

Specified by [ADR-151](adr/ADR-151-room-calibration-specialist-training.md).
Motto in the source: *"Teach the room before you teach the model."*
Pure Rust, no torch, no candle, edge-deployable.

### The four stages

```
  ①  baseline          ②  enroll              ③  extract        ④  train
  ─────────────        ──────────────         ───────────       ─────────
  30 s empty room  →   8 labelled anchors  →  feature vectors → specialist bank
  Welford μ/σ per      empty / stand / sit     variance,         presence
  subcarrier +         lie / breathe_slow      motion rate,      posture
  von Mises phase      breathe_normal          periodicity,      breathing
  (ADR-135)            small_move / sleep      dominant rhythm   heartbeat
                       + quality gate                            restlessness
                       + geometry record                         anomaly
```

### Stage 1 — baseline ([ADR-135](adr/ADR-135-empty-room-baseline-calibration.md))

Per-subcarrier amplitude mean/variance (Welford) and circular phase statistics of
the **empty** room. Everything downstream is expressed as a deviation from this,
which is what makes stationary hardware artefacts cancel.

Guard rail: an **abort on running z-score > threshold** (default 2.0). If
something moves during calibration it refuses to bake a person into the "empty"
model rather than silently producing a poisoned baseline. **[CODE]**

### Stage 2 — enrollment

Eight guided anchors, each a short labelled capture:

| # | Anchor | Duration | Teaches |
|---|---|---|---|
| 1 | `empty` | 15 s | The negative class |
| 2 | `stand_still` | 15 s | Presence without motion |
| 3 | `sit` | 15 s | Posture: sitting |
| 4 | `lie_down` | 15 s | Posture: lying |
| 5 | `breathe_slow` | **30 s** | Low-rate respiration (needs the window — see [§2.8](#28-frequency-bands-where-breathing-and-heartbeat-live)) |
| 6 | `breathe_normal` | **30 s** | Normal respiration |
| 7 | `small_move` | 15 s | Restlessness |
| 8 | `sleep_posture` | **30 s** | The overnight-monitoring case |

**The adaptive quality gate** is where the real engineering is. Bad anchors
poison small models far more than large ones, so an anchor is only accepted if
its statistics match what it is supposed to teach.

> **A subtle bug this crate documents and avoids** (ADR-152, "z-band squeeze"):
> motion is **not** taken from a threshold on the deviation z-score. That flag
> fires on `amplitude_z_median > 2.0` — deviation from the *empty* baseline —
> which conflates *presence strength* with *motion*. A strongly-reflecting person
> standing perfectly still would score z > 2 on every frame and be rejected as
> "too much motion". Instead, motion is derived from the frame-to-frame **change**
> in the deviation series (|Δz|, |Δφ|), which is presence-independent. **[CODE]**
>
> This is a good example of the kind of error that is invisible in a demo and
> fatal in deployment.

The session is **event-sourced**, so re-enrollment is incremental and auditable.

### Stage 2b — geometry ([ADR-152](adr/ADR-152-wifi-pose-sota-2026-intake.md) §2.1.1–2.1.2)

Optional records of where each node physically is: position (m), orientation
(rad), and a free-form `method` provenance string (`"tape-measure"`,
`"floor-plan"`, `"unknown"`).

Why bother: PerceptAlign diagnosed **coordinate overfitting** — heads trained
without an explicit layout model memorise the deployment geometry and break
elsewhere. The cheap half of the fix is simply to *record* the geometry so every
bank knows what layout it was trained under.

[`geometry_embedding.rs`](../v2/crates/wifi-densepose-calibration/src/geometry_embedding.rs)
turns that record into a fixed 32-dim conditioning vector with three deliberate
properties: **fixed dimension** for any node count, **permutation-invariant**
(sorted by node ID, order-free aggregates), and **total** — an all-unknown layout
yields a well-defined vector, never `NaN`. Adversarial inputs (non-finite
coordinates, absurd magnitudes) are treated as unmeasured rather than crashing.
It is explicitly *not* a trained network; the learned part would live in whatever
head consumes it. **[CODE]**

### Stage 4 — the specialist bank

Six small models, one per biological signal. Each is a threshold, a handful of
nearest-prototype vectors, or a band-limited periodicity read:

| Specialist | Output |
|---|---|
| `PresenceSpecialist` | present / absent |
| `PostureSpecialist` | standing / sitting / lying |
| `BreathingSpecialist` | BPM + confidence |
| `HeartbeatSpecialist` | BPM + confidence |
| `RestlessnessSpecialist` | scalar 0..1 |
| `AnomalySpecialist` | veto flag |

**The runtime fusion rules keep it honest** ([`runtime.rs`](../v2/crates/wifi-densepose-calibration/src/runtime.rs)):

- The **anomaly specialist holds a veto** — a physically implausible window
  suppresses positive vitals/posture rather than propagating a hallucination.
- **`presence = absent` short-circuits** breathing/heartbeat/posture to `None`.
  You cannot have a respiration rate in an empty room. Systems that report one
  are reporting noise.
- A **`STALE` bank** (baseline drifted) flags every reading rather than emitting
  confident-but-wrong values — the calibration analogue of the firmware's
  `DEGRADED` flag.

Those three rules are the difference between a system that fails loudly and one
that fails silently. Prefer the former.

---

## 4.6 Vital signs — [`wifi-densepose-vitals`](../v2/crates/wifi-densepose-vitals/)

[ADR-021](adr/ADR-021-vital-sign-detection-rvdna-pipeline.md). Four stages:

1. **[`preprocessor.rs`](../v2/crates/wifi-densepose-vitals/src/preprocessor.rs)** —
   EMA-based static-component suppression → per-subcarrier residuals.
2. **[`breathing.rs`](../v2/crates/wifi-densepose-vitals/src/breathing.rs)** —
   0.1–0.5 Hz bandpass + zero-crossing.
3. **[`heartrate.rs`](../v2/crates/wifi-densepose-vitals/src/heartrate.rs)** —
   0.8–2.0 Hz bandpass + autocorrelation peak + inter-subcarrier phase-coherence
   weighting.
4. **[`anomaly.rs`](../v2/crates/wifi-densepose-vitals/src/anomaly.rs)** —
   z-score against Welford running statistics for apnea / tachycardia /
   bradycardia flags.

Results land in a [`VitalSignStore`](../v2/crates/wifi-densepose-vitals/src/store.rs)
with configurable retention.

Note there are **two** vitals paths: this host-side one and the firmware's Tier 2
on-device one. `/api/v1/vital-signs` returns the host estimate;
`/api/v1/edge-vitals` returns the node's. Comparing them is a genuinely useful
cross-check — agreement is evidence, disagreement localises the fault.

---

## 4.7 The hub — [`wifi-densepose-sensing-server`](../v2/crates/wifi-densepose-sensing-server/)

Binary: `sensing-server`. The composition root of the live system.

```
UDP :5005 ──► parse ──► DSP ──► inference ──► ┬─► REST  :3000
(CSI in)                                      ├─► WebSocket :3001
                                              ├─► static UI
                                              ├─► MQTT (Home Assistant)
                                              └─► Matter / HAP bridges
```

### Notable internal modules

| Module | Role |
|---|---|
| [`csi.rs`](../v2/crates/wifi-densepose-sensing-server/src/csi.rs) | UDP ingest + frame decode |
| [`multistatic_bridge.rs`](../v2/crates/wifi-densepose-sensing-server/src/multistatic_bridge.rs) | Wire RuvSense fusion into the live loop |
| [`vital_signs.rs`](../v2/crates/wifi-densepose-sensing-server/src/vital_signs.rs) | Host vitals pipeline |
| [`adaptive_classifier.rs`](../v2/crates/wifi-densepose-sensing-server/src/adaptive_classifier.rs) | Train a classifier from recordings at runtime (ADR-048) |
| [`recording.rs`](../v2/crates/wifi-densepose-sensing-server/src/recording.rs) | Capture CSI to JSONL for offline replay |
| [`model_manager.rs`](../v2/crates/wifi-densepose-sensing-server/src/model_manager.rs) · [`rvf_container.rs`](../v2/crates/wifi-densepose-sensing-server/src/rvf_container.rs) | RVF model load/unload; **binary RVF only** |
| [`rufield_surface.rs`](../v2/crates/wifi-densepose-sensing-server/src/rufield_surface.rs) | Signed `FieldEvent` egress (ADR-262) |
| [`mqtt/`](../v2/crates/wifi-densepose-sensing-server/src/mqtt/) · [`matter/`](../v2/crates/wifi-densepose-sensing-server/src/matter/) | Home Assistant / Matter integration |
| [`semantic/`](../v2/crates/wifi-densepose-sensing-server/src/semantic/) | Raw signals → semantic states |
| [`ws_ticket.rs`](../v2/crates/wifi-densepose-sensing-server/src/ws_ticket.rs) · [`bearer_auth.rs`](../v2/crates/wifi-densepose-sensing-server/src/bearer_auth.rs) | WebSocket ticket auth (ADR-272), bearer tokens |
| [`path_safety.rs`](../v2/crates/wifi-densepose-sensing-server/src/path_safety.rs) · [`host_validation.rs`](../v2/crates/wifi-densepose-sensing-server/src/host_validation.rs) | Traversal and host-header hardening |
| Vendor adapters | [`mediatek_csi.rs`](../v2/crates/wifi-densepose-sensing-server/src/mediatek_csi.rs), [`qualcomm_csi.rs`](../v2/crates/wifi-densepose-sensing-server/src/qualcomm_csi.rs), [`realtek_radar.rs`](../v2/crates/wifi-densepose-sensing-server/src/realtek_radar.rs), `vendor_*.rs` |

### The API surface you will actually use

| Endpoint | Returns |
|---|---|
| `GET /health` | `{"status":"ok","source":"esp32",...}` — first thing to check |
| `GET /api/v1/nodes` | Which node IDs have been seen, and when |
| `GET /api/v1/sensing/latest` | Latest frame: amplitude and phase arrays, motion |
| `GET /api/v1/vital-signs` | Host-computed breathing + heart + confidence |
| `GET /api/v1/edge-vitals` | Node-computed (Tier 2) vitals |
| `GET /api/v1/mesh` · `/api/v1/mesh/metrics` | Fleet sync state; Prometheus exposition |
| `GET /api/v1/pose/current` | 17 COCO keypoints — **heuristic-driven without `--model`** |
| `POST /api/v1/recording/{start,stop}` | Record CSI to JSONL |
| `POST /api/v1/config/dedup-factor` | Runtime person-count tuning |
| `GET /api/field` · `WS /ws/field` | Signed RuField `FieldEvent`s, privacy-gated to P1/P2 |

Full table: [`docs/user-guide.md`](user-guide.md#rest-api-reference).

### Three operational gotchas, all real

1. **`--ui-path` is resolved against the current working directory**, not the
   binary location. The default `../ui` 404s from anywhere else. Pass an absolute
   path.
2. **Only one process can bind UDP :5005.** The standalone aggregator, the
   calibration CLI, and `sensing-server` all want it. Run one, or fan out with
   [`scripts/csi-udp-relay.py`](../scripts/csi-udp-relay.py).
3. **Docker Desktop on Windows drops UDP from all but one source IP.** With six
   nodes you see one. `--network host` does not help (not implemented for the
   Linux engine on Windows). Run natively, or relay through
   [`scripts/udp-relay.py`](../scripts/udp-relay.py) so every datagram arrives
   from the same loopback source. [`docs/TROUBLESHOOTING.md`](TROUBLESHOOTING.md) §9.

---

## 4.8 The operator CLI — [`wifi-densepose-cli`](../v2/crates/wifi-densepose-cli/)

Binary: `wifi-densepose`. This is the calibration pipeline's user interface.

| Command | What it does |
|---|---|
| `calibrate` | Capture the empty-room baseline → `.bin` |
| `calibrate-serve` | Serve the calibration flow over HTTP |
| `enroll` | Guided 8-anchor capture with quality gate → `.json` |
| `train-room` | Fit the specialist bank → `.json` |
| `room-status` | Inspect a trained bank |
| `room-watch` | Live inference; `--bank` for one node, repeated `--node-bank N:PATH` for multistatic |
| `mat` | Mass Casualty Assessment Tool (behind the `mat` feature) |

Build it with `--no-default-features`: that drops the `mat` feature →
`wifi-densepose-nn` → `ort` (ONNX Runtime) → `openssl-sys`, none of which the
calibration path needs and the last of which is a notorious Windows build
failure.

---

## 4.9 Vector infrastructure — [`wifi-densepose-ruvector`](../v2/crates/wifi-densepose-ruvector/)

Why a vector database belongs in a sensing system: once CSI is embedded into a
128-dim vector, most questions become **nearest-neighbour queries**.

- *Who is this?* → nearest neighbour among enrolled person embeddings
- *Which room is this?* → nearest neighbour among room fingerprints
- *Have I seen this pattern before?* → k-NN over history
- *Is this novel?* → distance to the nearest neighbour exceeds a threshold

| Module | Role |
|---|---|
| [`hnsw.rs`](../v2/crates/wifi-densepose-ruvector/src/hnsw.rs) · [`hnsw_quantized.rs`](../v2/crates/wifi-densepose-ruvector/src/hnsw_quantized.rs) | Hierarchical Navigable Small World approximate k-NN — sub-linear search, quantized for edge memory budgets |
| [`viewpoint/`](../v2/crates/wifi-densepose-ruvector/src/viewpoint/) | Cross-viewpoint fusion, clock-quality gating |
| [`sketch.rs`](../v2/crates/wifi-densepose-ruvector/src/sketch.rs) | Streaming sketches (bounded-memory summaries) |
| [`coverage.rs`](../v2/crates/wifi-densepose-ruvector/src/coverage.rs) | Sensing-coverage computation for placement |
| [`event_log.rs`](../v2/crates/wifi-densepose-ruvector/src/event_log.rs) | Append-only event log |
| [`crv/`](../v2/crates/wifi-densepose-ruvector/src/crv/) | CRV signal-line protocol (ADR-033) |

Related external algorithms in the RuVector family used here: `ruvector-mincut`
(Stoer-Wagner, dynamic min-cut, person matching), `ruvector-attn-mincut`
(attention-weighted cross-node fusion), `ruvector-solver` (Neumann series solver,
used as the ISTA warm start).

---

## 4.10 Learning — [`wifi-densepose-nn`](../v2/crates/wifi-densepose-nn/), [`-train`](../v2/crates/wifi-densepose-train/), [`-aether`](../v2/crates/wifi-densepose-aether/)

| Crate | Role |
|---|---|
| `wifi-densepose-nn` | Inference across ONNX Runtime / tch-rs / Candle backends |
| `wifi-densepose-train` | Training pipeline: config, dataset loading with subcarrier interpolation, losses, metrics, MAE pretraining, WiFlow-STD port |
| `wifi-densepose-aether` | The dependency-free contrastive-embedding leaf, hoisted out of the server so the Python wheel can bind it without dragging in Axum/tokio (ADR-024, ADR-185) |
| `wifi-densepose-occworld-candle` | OccWorld TransVQVAE future-occupancy prediction (ADR-147) |

The `aether` split is a small thing that teaches a general lesson: **the shape of
your crate graph is a product decision.** The Python wheel had a ≤5 MB budget;
that budget forced the pure-compute surface to be a leaf. Dependency hygiene is
not aesthetics.

---

## 4.11 The rest of the workspace, briefly

| Crate / group | Role | Why it exists |
|---|---|---|
| [`wifi-densepose-core`](../v2/crates/wifi-densepose-core/) | `CsiFrame`, `PoseEstimate`, `Keypoint`, traits, errors | The shared vocabulary. Everything depends on it. |
| [`wifi-densepose-hardware`](../v2/crates/wifi-densepose-hardware/) | ESP32 parser, UDP aggregator, TDM, sync packets, vendor codecs, [`ieee80211bf/`](../v2/crates/wifi-densepose-hardware/src/ieee80211bf/) | The boundary between wire bytes and typed frames |
| [`wifi-densepose-engine`](../v2/crates/wifi-densepose-engine/) | Composition root wiring ADR-135..146 into one trust-traceable cycle | Proves the pieces compose, end to end |
| [`wifi-densepose-bfld`](../v2/crates/wifi-densepose-bfld/) | Beamforming Feedback (802.11ac/ax BFI) sensing, privacy-classed P0–P5 | A *second* sensing modality from the same radios — and the privacy control plane |
| [`wifi-densepose-rufield`](../v2/crates/wifi-densepose-rufield/) | Anti-corruption bridge to the RuField MFS spec; signed `FieldEvent`s | Single, auditable coupling point to an external spec |
| [`wifi-densepose-mat`](../v2/crates/wifi-densepose-mat/) | Mass Casualty Assessment Tool — survivor detection in rubble | The disaster-response vertical (ADR-001, ADR-026) |
| [`wifi-densepose-pointcloud`](../v2/crates/wifi-densepose-pointcloud/) | Camera depth (MiDaS) + CSI + mmWave → unified 3-D | Multimodal fusion |
| [`wifi-densepose-wifiscan`](../v2/crates/wifi-densepose-wifiscan/) | Cross-platform BSSID scanning, RSSI-only sensing (ADR-022) | The zero-hardware entry point |
| [`wifi-densepose-desktop`](../v2/crates/wifi-densepose-desktop/) | Tauri desktop app (ADR-052/054/056) | Non-technical operators |
| [`wifi-densepose-wasm`](../v2/crates/wifi-densepose-wasm/), [`-wasm-edge`](../v2/crates/wifi-densepose-wasm-edge/) | Browser and embedded WASM builds | In-browser demos; on-node modules |
| `homecore-*` (11 crates) | A Rust port of the Home Assistant core model: state machine, event bus, plugins, automation, REST/WS API, recorder, HAP bridge, migration | Semantic layer + smart-home integration without depending on a Python HA install (ADR-127..133) |
| `cog-*` | Packaged edge modules ("Cogs"): pose estimation, person count, HA/Matter | The distribution format for edge intelligence (ADR-100/101/102) |
| [`nvsim`](../v2/crates/nvsim/), [`nvsim-server`](../v2/crates/nvsim-server/) | NV-diamond magnetometer simulator, deterministic and WASM-ready | Research into a quantum-sensing tier that could recover what RF cannot (ADR-089/090/092) |
| [`ruview-auth`](../v2/crates/ruview-auth/) | OAuth **resource server**; offline JWKS verification | Runs on Pi-class hardware with no outbound calls (ADR-271) |
| [`ruview-unified`](../v2/crates/ruview-unified/) | Unified RF spatial world model — WiFi CSI, cellular SRS, FMCW radar, UWB CIR into one persistent scene memory | The forward-looking architecture (ADR-273..282). **All numbers L0 / synthetic.** |
| `worldgraph`, `ruv-neural`, `ruview-swarm` | Submodules (empty until `git submodule update --init --recursive`) | **`cargo` refuses to parse the workspace without these.** |

---

## 4.12 The Python surface — [`python/`](../python/)

Rust + PyO3 → `ruview` / `wifi-densepose` wheels on PyPI (ADR-117, ADR-185).
Same compiled core, ~250 KB, abi3-py310, Linux/macOS/Windows.

```python
from ruview import BreathingExtractor, HeartRateExtractor
from ruview.client import SensingClient, RuViewMqttClient   # pip install "ruview[client]"
```

The reason this exists is adoption: the research and data-science ecosystem for
this field is Python, and requiring a Rust toolchain to try the library would
have cut off most of its potential users. The reason it is a *compiled* wheel
rather than a reimplementation is that a second implementation is a second set of
bugs — the deterministic-proof discipline would be meaningless if two pipelines
could drift.

---

## 4.13 Governance: ADRs, evidence, and the honesty rules

207 ADRs in [`docs/adr/`](adr/). This is not incidental documentation — it is the
project's control system.

- Code cites ADRs by number in module doc comments. Deviations from an ADR are
  documented *in the code*, with the reason. (Example: `array_coordinator.rs`
  explains at length why it lives in `signal` rather than where ADR-138 placed
  it — a dependency cycle — rather than silently moving it.)
- Claims carry evidence tags, and the repo distinguishes *empirical* from
  *claimed* in [`docs/WITNESS-LOG-028.md`](WITNESS-LOG-028.md) and
  [`docs/WITNESS-LOG-110.md`](WITNESS-LOG-110.md).
- [`CLAUDE.md`](../CLAUDE.md) §9 lists the load-bearing honesty distinctions and
  instructs contributors not to let a documentation edit quietly upgrade a
  heuristic into a model.

**Recommended ADR reading order for a newcomer:**

| Order | ADR | Why |
|---|---|---|
| 1 | [ADR-012](adr/ADR-012-esp32-csi-sensor-mesh.md) | Why an ESP32 mesh at all |
| 2 | [ADR-018](adr/ADR-018-esp32-dev-implementation.md) | The frame format and the build sequence |
| 3 | [ADR-021](adr/ADR-021-vital-sign-detection-rvdna-pipeline.md) | Vital signs |
| 4 | [ADR-029](adr/ADR-029-ruvsense-multistatic-sensing-mode.md) | Multistatic sensing — the fidelity argument |
| 5 | [ADR-039](adr/ADR-039-esp32-edge-intelligence.md) | The tier model |
| 6 | [ADR-135](adr/ADR-135-empty-room-baseline-calibration.md) | Empty-room baseline |
| 7 | [ADR-151](adr/ADR-151-room-calibration-specialist-training.md) | Per-room specialist training — the central bet |
| 8 | [ADR-134](adr/ADR-134-csi-to-cir-time-domain-multipath.md) | CSI → CIR |
| 9 | [ADR-137](adr/ADR-137-fusion-engine-quality-scoring-evidence.md) | Quality scoring and the trust throughline |
| 10 | [ADR-120](adr/ADR-120-bfld-privacy-class-and-hash-rotation.md) | Privacy classes |

---

# Part 5 — Life of a frame

Follow one measurement from photons to a sentence. This is the mental model to
hold while debugging, because **every failure in this system is a break in this
chain**, and knowing the chain tells you where to look.

```
 ① PHYSICS         An 802.11n packet crosses the room. It reflects off walls,
                   furniture, and a person's chest, which has moved 8 mm since
                   the last packet.

 ② RADIO           The ESP32-S3 receives it. To equalise it, the PHY estimates
                   the channel at each of 64 subcarrier bins. That estimate is
                   handed to firmware via the CSI callback, in radio context.
                        ↓  ~200–500 callbacks/s
 ③ RATE LIMIT      csi_collector rate-limits to ~50 Hz and writes into an SPSC
                   ring buffer. Never blocks. Overflow drops, silently.
                        ↓
 ④ TIER            Core 1 drains the ring. Tier 0 → passthrough. Tier 1 →
                   unwrap/Welford/top-K. Tier 2 → also runs vitals, presence,
                   fall on-device and emits a 32-byte packet at 1 Hz.
                        ↓
 ⑤ SERIALISE       20-byte ADR-018 header + i8 I/Q pairs.  magic 0xC5110001
                        ↓
 ⑥ TRANSPORT       UDP → target_ip:5005. No retransmit. ENOMEM backoff 100 ms
                   if lwIP runs out of packet buffers.
                        ↓   ~124 bytes × 20 Hz × N nodes
 ⑦ INGEST          sensing-server binds :5005. Esp32CsiParser validates magic,
                   bounds-checks n_subcarriers, decodes I/Q → complex.
                        ↓
 ⑧ NORMALISE       Hampel → phase sanitize (unwrap + detrend) → hardware_norm
                   resample to canonical 56 bins → z-score.
                        ↓
 ⑨ REFERENCE       Subtract the ADR-135 empty-room baseline. Output is now a
                   *deviation*, so stationary hardware artefacts cancel.
                        ↓
 ⑩ REPRESENT       Depending on the consumer: per-subcarrier statistics, a
                   top-K selection, an STFT spectrogram, a BVP, a sparse CIR,
                   or a 128-dim embedding.
                        ↓
 ⑪ FUSE            Multiple nodes/channels: multiband stitch → phase_align →
                   attention-weighted multistatic fusion → coherence gate
                   decides Accept / PredictOnly / Reject / Recalibrate.
                        ↓
 ⑫ INFER           The room's specialist bank (or vitals pipeline, or tracker)
                   emits {value, confidence} per quantity. Anomaly may veto.
                   Absent presence short-circuits vitals to None.
                        ↓
 ⑬ TRACK           Kalman update; track lifecycle; re-ID by embedding.
                        ↓
 ⑭ QUALITY         QualityScore attaches evidence refs and contradiction flags.
                   Contradictions lower the downstream privacy class.
                        ↓
 ⑮ SEMANTICS       "someone-sleeping", "bed-exit", "possible-distress".
                        ↓
 ⑯ EGRESS          REST / WebSocket / MQTT / Matter / HAP — subject to the
                   privacy gate: P0 raw and P3–P5 identity never leave the box.
```

### Where the chain breaks, and the symptom

| Break | Symptom you observe | First thing to check |
|---|---|---|
| ② no CSI enabled | Node online, **0 pps** | Display false-positive → rebuild with `CONFIG_DISPLAY_ENABLE` unset |
| ② no traffic to measure | Low but non-zero pps | CSI only exists when packets exist. Generate traffic. |
| ③ ring overflow | Gaps in sequence numbers | Node is CPU-starved; lower tier or subcarrier count |
| ⑥ firewall | Zero packets at host, node claims it is sending | Allow inbound UDP on the Private profile |
| ⑥ Docker on Windows | Exactly one node visible out of six | Run natively or use `udp-relay.py` |
| ⑦ port contention | Server sees nothing, recorder sees packets | Only one process may bind :5005 |
| ⑨ poisoned baseline | Everything reads "present", always | Someone was in the room during `calibrate` |
| ⑫ bank stale | `STALE` flag on every reading | Room changed; re-baseline |
| ⑫ indistinct anchors | Posture never changes | Re-enroll with more separation between postures |

**Debugging heuristic:** walk the chain *forward* from the physics, not backward
from the symptom. The two most common failures (0 pps CSI, and firewall-blocked
UDP) both occur before any interesting code runs, and both present as "the
sophisticated part is broken."

---

# Part 6 — The honesty ledger

This repository takes an unusual position: it publishes what does *not* work
alongside what does. That is a feature, and this document would be worthless if
it flattened it. Here is the consolidated state.

## 6.1 Solid — expect these to work

| Capability | Basis |
|---|---|
| Raw CSI streaming at ~20 Hz per node | **[FIELD]** |
| Breathing rate on a still subject at 1–1.5 m | **[FIELD]** — physics is strongly in your favour ([§2.7](#27-breathing-from-millimetres-to-degrees)) |
| Motion / activity detection | **[FIELD]** |
| The ADR-151 baseline → enrollment → bank → live inference pipeline | **[CODE]** complete and runnable; accuracy is yours to measure |
| Multistatic fusion of several nodes | **[CODE]** |
| Deterministic pipeline proof | **[CODE]** one command, no hardware |

## 6.2 Weak — expect noise

| Capability | Why it is weak |
|---|---|
| **Heart rate** | 20–50× smaller displacement than breathing, and sits under breathing harmonics. Needs a still subject within ~1 m, ideally breath-holding. |
| **Presence (firmware)** | A **variance threshold, not a model.** False-positives on fans, microwaves, AP power swings. The firmware README says so; do not let a doc edit upgrade it. |
| **Person counting (firmware)** | `n_persons` is a **slot-capacity heuristic** over top-K subcarrier groups ([`main/edge_processing.c:481`](../firmware/esp32-csi-node/main/edge_processing.c)). It tracks subcarrier diversity, not people. Documented reading 6–8 for 5 bodies. Expect **monotonic**, not accurate. |
| **Through-wall** | Works on drywall; degrades fast through brick; blocked by reinforced concrete. Also very easy to fool yourself — **run a negative control.** |

## 6.3 Not real yet — do not build on these

| Claim | Actual state |
|---|---|
| **17-keypoint pose from CSI alone, on-device** | The committed `pose_v1` scores **PCK@20 = 3.0%** against a ≥35% target, and its runtime path is a `confidence=0` stub. Without `--model`, the on-screen skeleton is driven by **amplitude-variance heuristics**, not learned inference. |
| **The 82.69% MM-Fi pose number applies to your ESP32** | It does not. That is a separate published benchmark on a research dataset with a stated protocol (`random_split`), not the live single-node path. Both numbers are real; they describe different things. |
| **Loading the Hugging Face weights into `sensing-server`** | The HF bundle ships as **JSONL RVF**; `--model` parses **binary RVF** only. It fails with a magic mismatch and the pipeline degrades to null output. **Run without `--model`** until a JSONL adapter lands. |
| **Tier 3 WASM programmable sensing** | Needs 4 × 160 KB PSRAM arenas; these boards report `PSRAM_CAP = None`. Also `components/wasm3/wasm3-src/` is absent from the tree. |
| **`examples/research-sota/` numbers** | **[SYNTH]** — synthetic-physics derivations. The folder's own README states real deployments diverge **5–30%**. |
| **`ruview-unified` (ADR-273..282) numbers** | Evidence level **L0** — synthetic until real-data validation. |

## 6.4 The rule to carry forward

> **If you produce a new number, state how it was measured and on what.**

A well-characterised 70% is worth more than an unreproducible 95%. That is the
repository's standard, stated in [`CLAUDE.md`](../CLAUDE.md) §9, and it is the
reason its claims are worth reading at all.

---

# Part 7 — How this design compares to the alternatives

Comparison is only useful against a stated axis. Three axes matter here:
**what modality**, **what architecture**, and **what honesty regime**.

## 7.1 Against other sensing modalities

| | **RuView (WiFi CSI)** | **mmWave radar (60 GHz)** | **UWB** | **Camera + pose** | **PIR** | **Wearable** |
|---|---|---|---|---|---|---|
| Bandwidth | 20 MHz (60 MHz hopped) | 1–7 GHz | ~500 MHz | — | — | — |
| Range resolution | 7.5 m → 2.5 m | **3–15 cm** | ~30 cm | N/A (angular) | none | N/A |
| Through walls | **Yes** (drywall, ~5 m) | No | Partial | No | No | N/A |
| Works in dark | Yes | Yes | Yes | Only with IR | Yes | Yes |
| Vitals | Breathing yes, HR weak | **Breathing + HR, strong** | Breathing | HR via rPPG, fragile | No | **Gold standard** |
| Pose quality | Poor (research-grade at best) | Coarse point cloud | Coarse | **Excellent** | None | Joint angles only |
| Privacy | Good — no image | Good | Good | **Poor** | Excellent | Good but personal |
| Cost per room | **$9–54** | $30–200 | $20–100 | $20–200 + compute | $5 | $50–300/person |
| Infrastructure needed | **None new** — reuses WiFi | New sensor per room | New anchors | New camera per room | New sensor | Person must wear it |
| Compliance burden | None | None | None | None | None | **High** |
| Deployment maturity | Research → early product | **Shipping products** | Shipping (positioning) | Mature | Mature | Mature |

**The honest summary.** mmWave radar is *technically superior* for in-room
sensing on almost every axis except wall penetration and cost. It has 100–300×
the bandwidth, which buys it real range resolution and a genuinely reliable heart
rate. If you need one excellent room sensor and can install hardware, buy a
mmWave sensor.

WiFi sensing wins on exactly three things, and they are strategic rather than
technical:

1. **Marginal cost approaching zero at scale** — the radios exist already.
2. **Wall penetration** — one node can cover more than one room.
3. **Ubiquity path** — if 802.11bf lands in mainstream APs, the sensor becomes
   *the router you already own*, and no other modality can match that
   distribution.

RuView's design reflects a bet on (1) and (3): rather than compete with radar on
sensitivity, spend the cost advantage on **more viewpoints** and on **learning
each room cheaply**.

## 7.2 Against other WiFi-CSI toolchains

| | **RuView** | **Linux 802.11n CSI Tool** (Intel 5300) | **Atheros CSI Tool** | **Nexmon CSI** (Broadcom) | **ESP32-CSI-Tool** | **PicoScenes** |
|---|---|---|---|---|---|---|
| Hardware | ESP32-S3/C6 (+ vendor adapters) | Intel 5300 | AR9xxx | BCM43xx (incl. Raspberry Pi) | ESP32 | Intel AX200/210, QCA, USRP |
| Subcarriers | 52–64 (HT20) | 30 (grouped) | 56 | Full FFT, up to 80 MHz | 64/128 | Up to 160 MHz, 802.11ax |
| MIMO | No (1 antenna) | **3×3** | 3×3 | Varies | No | **Yes** |
| Cost | **$9** | ~$50 used, scarce | ~$30, scarce | Pi + dongle | $9 | $$$ |
| Kernel/driver hacking | **None** | Patched driver, old kernel | Patched driver | Firmware patching | None | Custom stack |
| Ships an application? | **Yes — end-to-end** | No — CSI dump only | No | No | Minimal | Research platform |
| Multi-node mesh | **Yes, TDM + fusion** | Manual | Manual | Manual | Manual | Yes |
| Calibration pipeline | **Yes (ADR-151)** | No | No | No | No | No |
| Edge processing | **Yes (tiers 0–3)** | No | No | No | No | No |
| Smart-home integration | **Yes (HA/Matter/HAP/MQTT)** | No | No | No | No | No |

**Where RuView is genuinely differentiated:** it is one of very few WiFi-sensing
projects that is a *system* rather than a *capture tool*. The academic tools give
you a CSI dump and leave the other 95% to you. RuView ships the firmware, the
transport, the DSP, the calibration ritual, the fusion, the semantic layer, the
smart-home bridges, and the honesty documentation.

**Where it is behind:** the research NICs give you 3×3 MIMO and, with PicoScenes,
802.11ax at 160 MHz — 8× RuView's bandwidth and real spatial diversity in a
single box. For pure signal quality per node, RuView's hardware is the weakest
option on this table. It is compensating with node count and with software.

## 7.3 Against commercial WiFi sensing

| | **RuView** | **Origin Wireless** | **Cognitive Systems (WiFi Motion)** | **Router-vendor motion features** |
|---|---|---|---|---|
| Deployment | Your own hardware, your own server | Licensed into carrier/OEM devices | Licensed into carrier/OEM devices | Built into consumer mesh systems |
| Sensing basis | CSI from dedicated ESP32 nodes | CSI/time-reversal from existing APs + clients | CSI from existing mesh nodes | Typically RSSI/CSI from mesh links |
| Vitals | Yes (breathing solid, HR weak) | Yes, claimed | Motion/presence focus | Motion only |
| Source available | **Yes, MIT** | No | No | No |
| Runs offline | **Yes, fully** | Vendor-dependent | Vendor-dependent | Usually cloud-assisted |
| Verifiable claims | **Yes** — proof script, witness logs, evidence tags | Marketing + patents | Marketing + patents | Marketing |
| Scale proven | Lab / small deployments | Millions of homes | Millions of homes | Millions of homes |

**The honest read:** the commercial systems have vastly more real-world
validation — they run in millions of homes and have had to survive the long tail
of weird apartments. RuView has essentially none of that. What RuView has that
they do not is **inspectability**: you can read exactly how a number is produced,
re-derive it, and check the hash. For research, for regulated deployments, and
for anyone who has been burned by an unfalsifiable vendor claim, that is not a
small thing.

## 7.4 Against the academic state of the art

| System | Contribution | How RuView relates |
|---|---|---|
| **WiTrack / Vital-Radio / RF-Pose** (MIT) | FMCW radio at 5.4–7 GHz with GHz of bandwidth; through-wall pose and vitals | Different hardware class entirely. Their bandwidth is what RuView lacks. Sets the ceiling. |
| **DensePose from WiFi** (CMU, 2023) | Dense body-surface estimation from 3×3 MIMO CSI | The direct ancestor of this project's name and ambition. Requires MIMO NICs and paired camera supervision. RuView's on-device pose path does **not** reach it. |
| **Person-in-WiFi** (ICCV 2019) | Body segmentation/keypoints from CSI | Established the supervised-with-camera paradigm RuView's ADR-079 follows |
| **Widar 2.0 / 3.0** (Tsinghua) | BVP as a domain-independent representation; cross-domain gesture recognition | Directly implemented here: [`bvp.rs`](../v2/crates/wifi-densepose-signal/src/bvp.rs) |
| **SpotFi** (SIGCOMM 2015) | Decimetre localisation; the conjugate-multiplication sanitisation trick | Implemented: [`csi_ratio.rs`](../v2/crates/wifi-densepose-signal/src/csi_ratio.rs) |
| **FarSense** (2019) | CSI ratio to extend breathing-sensing range | Implemented in spirit: [`fresnel.rs`](../v2/crates/wifi-densepose-signal/src/fresnel.rs) |
| **SenseFi** (2023) | A benchmark + library for deep-learning WiFi sensing | The right yardstick for anyone claiming a learned model here |
| **MM-Fi** (NeurIPS D&B 2023) | Multi-modal non-intrusive human sensing dataset | The benchmark RuView's published pose model is measured on |

**Positioning:** RuView is not competing with MIT's FMCW work on capability — it
cannot, at 20 MHz. It is competing on **deployability**: taking techniques the
literature has established and packaging them into something that runs on $9
hardware, in a real house, with no kernel patches, and reports honestly.

## 7.5 Against the standards path — IEEE 802.11bf

[IEEE 802.11bf](https://www.ieee802.org/11/Reports/tgbf_update.htm) makes WLAN
sensing a first-class service: sensing sessions, measurement setup and
negotiation, and standardised reporting, at sub-7 GHz and in the 60 GHz DMG band.

| | **RuView today** | **802.11bf future** |
|---|---|---|
| How CSI is obtained | Promiscuous capture on dedicated nodes | Negotiated sensing session with the AP |
| Hardware | Dedicated $9 nodes | Any compliant AP/STA |
| Coordination | Custom TDM over NVS config | Standardised measurement scheduling |
| Interop | Single-vendor | Multi-vendor by construction |
| Availability | Now | As silicon ships |

RuView already has an [`ieee80211bf/`](../v2/crates/wifi-densepose-hardware/src/ieee80211bf/)
module ([ADR-153](adr/ADR-153-ieee-802-11bf-sensing-protocol-layer.md)),
positioning the CSI acquisition layer as swappable. That is the right structural
bet: **the physics and the DSP survive a standards transition; only the
acquisition layer changes.**

## 7.6 Architectural comparisons

These are the design choices worth arguing about, independent of modality.

### Edge-first vs. cloud-first

| | Edge-first (**RuView**) | Cloud-first (typical IoT) |
|---|---|---|
| Latency | Sub-millisecond on-node decisions | Round-trip to a datacentre |
| Privacy | Raw signal never leaves the room | Raw signal leaves the room |
| Offline | Fully functional | Dead without internet |
| Cost at scale | Zero marginal | Per-device, forever |
| Model updates | OTA / WASM hot-swap | Trivial |
| Compute available | 240 MHz, 512 KB RAM | Unlimited |

The tier system is what makes the edge-first choice tractable: you are not forced
to pick once. Tier 0 is cloud-ish (dumb sensor, smart host), Tier 2 is fully
autonomous, and it is a runtime switch.

### Calibrate-then-infer vs. train-one-big-model

| | Calibrate-then-infer (**RuView / ADR-151**) | One large general model |
|---|---|---|
| Data needed | ~5 minutes per room, from the user | Thousands of hours across many rooms |
| Handles domain shift | By construction — it *is* the domain | Only if the training set covered it |
| Model size | Kilobytes; runs on an MCU | Megabytes; needs an accelerator |
| Cold start | Requires an enrollment ritual | Works out of the box, badly |
| Degradation | Explicit `STALE` flag | Silent accuracy loss |
| Ceiling | Bounded by what a few anchors can teach | Higher, in principle |

Neither is strictly right. RuView picks calibration-first because it is *runnable
today on the actual hardware* and because the failure mode is honest. The stated
upgrade path — LoRA heads over a frozen RF foundation encoder — is the hybrid.

### Rust vs. Python for the core

| | Rust (**v2**) | Python (**archive/v1**) |
|---|---|---|
| Real-time determinism | Yes, no GC pauses | Difficult |
| Cross-compiles to MCU/WASM | Yes | No |
| Memory safety on untrusted input | Compile-time | Runtime |
| Ecosystem for ML research | Growing (Candle, ort, tch) | Dominant |
| Contributor pool | Smaller | Enormous |

The repo's resolution is a compiled Rust core with PyO3 bindings: one
implementation, two audiences. The v1 Python tree is kept for the deterministic
proof and as reference DSP, and is explicitly labelled archived
([ADR-187](adr/ADR-187-archive-v1-deprecation-honest-labeling.md)).

## 7.7 A fair scorecard

| Dimension | RuView | Best-in-class | Gap |
|---|---|---|---|
| Cost per covered room | **Excellent** | RuView | — |
| Through-wall | **Good** | MIT FMCW | Bandwidth |
| Breathing accuracy | Good | mmWave | ~1 order of magnitude in SNR |
| Heart-rate accuracy | Weak | mmWave / contact | Large |
| Pose from RF | Weak on-device; SOTA on a benchmark | MIT RF-Pose, CMU DensePose-from-WiFi | Hardware + data |
| Person counting | Heuristic | mmWave | Large |
| System completeness | **Excellent** | RuView | — |
| Documentation & honesty | **Excellent** | RuView | — |
| Real-world validation at scale | Poor | Commercial vendors | Very large |
| Privacy architecture | **Strong** (P0–P5 classes, edge-first) | RuView | — |
| Standards alignment | Good (802.11bf layer present) | Vendor consortia | — |

---

# Part 8 — From theory to test: mapping onto `plans/`

Theory you cannot check is a story. This part connects every idea above to a
concrete, falsifiable experiment in [`plans/`](../plans/).

## 8.1 What is in `plans/`, and which parts are current

| File | Scope | Currency |
|---|---|---|
| [`plans/ruview-esp32s3-test-plan.md`](../plans/ruview-esp32s3-test-plan.md) | **10-phase, 6-node ESP32-S3 bring-up and validation** | ✅ **Current.** Written 2026-07-27 against repo `2cc378c1`, firmware `v0.8.4`. This is the plan to run. |
| [`plans/ruview-esp32s3-test-plan.html`](../plans/ruview-esp32s3-test-plan.html) | Browsable copy with progress checkboxes | ✅ Current |
| [`plans/overview.md`](../plans/overview.md) | 9-week programme plan: FastAPI, PostgreSQL, React, Kubernetes | ⚠️ **Historical.** Describes the **v1 Python** architecture now in [`archive/v1/`](../archive/v1/). Read for intent, not for instructions. |
| [`plans/phase1-specification/`](../plans/phase1-specification/) | Functional, technical, API specs and system requirements | ⚠️ Historical (2025-06, "InvisPose" era) |
| [`plans/phase2-architecture/`](../plans/phase2-architecture/) | System, API, hardware, neural-network architecture | ⚠️ Historical. Still valuable as a statement of *requirements* — the v2 Rust workspace is an answer to these documents, not a repudiation of them. |
| [`plans/ui-pose-detection-rebuild.md`](../plans/ui-pose-detection-rebuild.md) | Pose-UI WebSocket rebuild (port 8000, v1 backend) | ⚠️ Historical |

> **If you read only one:** the ESP32-S3 test plan. Everything below maps onto it.

## 8.2 Concept → experiment map

Each row: an idea from this document, the phase that tests it, and what a
successful test actually demonstrates.

| Theory | Test-plan step | What passing proves |
|---|---|---|
| [§2.5](#25-csi-the-actual-measurement) CSI is a real, decodable measurement | **2.3** — `synth-csi-udp.py` → server → `/api/v1/sensing/latest` | The host parses genuine ADR-018 frames and exposes amplitude/phase arrays. Establishes what *healthy* looks like before hardware exists. |
| [§3.9](#39-the-trust-chain-proof-witness-evidence) The DSP is deterministic | **2.1** — `archive/v1/data/proof/verify.py` | The pipeline produces bit-identical output to the published hash. If this fails, nothing downstream is trustworthy. |
| [§2.10](#210-fresnel-zones-why-placement-dominates-everything) Placement dominates | **2.4** — run the four `examples/research-sota/` scripts | You derive **your own** layout from physics: chest height ~1.0–1.2 m, surround the subject volume, N=5 is the coverage plateau. |
| [§4.2](#42-the-firmware--firmwareesp32-csi-node) Display false-positive → 0 pps | **3.2 → 3.3** — build with `# CONFIG_DISPLAY_ENABLE is not set` | You make the nastiest failure mode *structurally impossible* rather than hoping the runtime recovery fires. |
| [§4.1](#41-the-sensor-node--esp32-s3) Native-USB console | **3.2, 4.2** — `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`, then read the boot log | You can see the node think. Without this, every later diagnosis is guesswork. |
| [§4.3](#43-the-wire-protocols) UDP transport is lossy and unacknowledged | **5.1** — `record-csi-udp.py`, expect ≥ 500 packets / 30 s | Splits "the node isn't sending" from "the server can't parse" — two failures that look identical from the UI. |
| [§2.6](#26-why-a-human-body-shows-up-at-all) A body is a strong reflector | **5.3** — empty room vs. occupied, by eye, in the Observatory UI | The core physical claim of the whole project, verified with your own body. If this is not *obvious* on screen, fix placement before proceeding. |
| [§2.7](#27-breathing-from-millimetres-to-degrees) mm chest motion → tens of degrees of phase | **6.2** — three 2-minute runs: normal / slow / fast, vs. a stopwatch count | Tests the physics *and* the estimator. Passing = tracks the direction of change and lands within ±3 BPM in ≥2 of 3 runs. |
| [§2.7](#27-breathing-from-millimetres-to-degrees), [§3.4](#34-layer-3--estimation-rate-extraction) HR is 20–50× weaker and hides under breathing harmonics | **6.3** — breath-hold protocol vs. an oximeter | Characterises a known-weak capability. ±10 BPM during breath-hold is a pass. **Do not sink hours here.** |
| [§3.5](#35-layer-4--decision-from-numbers-to-states) Threshold detectors need tuning | **6.4** — 5 deliberate drops onto a mattress; tune `--fall-thresh` | Demonstrates the sensitivity/false-positive trade-off is a runtime knob, not a rebuild. |
| [§3.1](#31-the-central-difficulty-domain-shift) Domain shift is the central problem | **Phase 7 entire** — baseline → enroll → train-room → room-watch | **The most important phase.** You watch a system *learn your specific room* in ~5 minutes and then classify presence and posture in it. This is RuView's central thesis, executed. |
| [§4.5](#45-per-room-calibration--wifi-densepose-calibration) Baseline abort guard | **7.2** — leave the room; a z-score excursion aborts the capture | Proves the system refuses to bake a person into an "empty" baseline. |
| [§2.8](#28-frequency-bands-where-breathing-and-heartbeat-live) Δf = 1/T ⇒ 30 s windows | **7.3** — note that breathing anchors are 30 s and others 15 s | The anchor durations are a frequency-resolution requirement, not UX. |
| [§4.5](#45-per-room-calibration--wifi-densepose-calibration) Specialisation over scale | **7.4** — `train-room`, then `room-status` | A bank of six small specialists fitted from 8 anchors. Inspect what actually trained. |
| [§3.6](#36-layer-5--fusion-combining-evidence-honestly) Fusion rules and honest degradation | **7.5** — `room-watch`; watch for `VETO` and `STALE` | Confirms the anomaly veto and staleness flags are live, not decorative. |
| [§2.11](#211-multistatic-geometry-why-more-nodes-beats-a-better-node) More viewpoints beat a better sensor | **9.3** — run the *same* validation protocol single-node then 6-node fused | **The experiment that justifies six nodes.** Any measurable improvement is a real result. Record the actual numbers. |
| [§4.5](#45-per-room-calibration--wifi-densepose-calibration) Geometry recording (ADR-152) | **9.1–9.2** — write `geometry.json`, pass `--geometry` to `train-room` | `[train-room] geometry: 6 node(s) snapshotted into the bank` — the anti-coordinate-overfitting record. |
| [§2.2](#22-what-happens-to-a-wave-in-a-room) Wall attenuation table | **9.4** — through-wall, 5 present/absent cycles **plus a negative control** | The negative control is the part that matters. Without it you are measuring footsteps and door movement. |
| [§3.5](#35-layer-4--decision-from-numbers-to-states) Person counting is a heuristic | **9.5** — step occupancy 0→1→2→3 | Pass = **monotonic**, not accurate. Characterisation, not validation. |
| [§4.1](#41-the-sensor-node--esp32-s3) `WIFI_PS_NONE` thermal cost | **8.3** — touch every board at 2, 5, 10 minutes | Warm is normal; uncomfortably hot is a failure with a documented field precedent. |
| [§4.2](#42-the-firmware--firmwareesp32-csi-node) TDM avoids self-interference | **8.2** — unique `node_id` + `tdm_slot` per board; verify with `--state` | Duplicate node IDs merge two physical nodes into one logical one and produce baffling symptoms. |
| [§2.12](#212-bandwidth-is-resolution--and-why-channel-hopping-exists) Bandwidth is resolution | **8.5** — optional `--hop-channels 1,6,11` | ~3× sensing bandwidth at ~1/3 the per-channel frame rate. **Never during calibration or enrollment** — those assume a stable channel. |
| [§6](#part-6--the-honesty-ledger) Report what you measured | **10.3** — fill in the results table | The point of the whole exercise. A characterised 70% beats an unreproducible 95%. |

## 8.3 Suggested reading and doing order

```
   READ                              DO
   ────                              ──
1. THEORY Part 0–1  ─────────────►  (nothing yet — build the why)
2. THEORY Part 2    ─────────────►  Test plan Phase 0  (concepts)
3. THEORY Part 4.1–4.3 ──────────►  Test plan Phase 1  (toolchain)
4. THEORY Part 3, 5 ─────────────►  Test plan Phase 2  (no-hardware validation)
                                     ↑ highest value per minute in the whole plan
5. THEORY Part 4.2  ─────────────►  Test plan Phase 3–4 (firmware, first node)
6. THEORY Part 2.6–2.8 ──────────►  Test plan Phase 5–6 (real CSI, vitals)
7. THEORY Part 3.1, 4.5 ─────────►  Test plan Phase 7  (calibration — the core)
8. THEORY Part 2.10–2.12, 3.6 ───►  Test plan Phase 8–9 (six nodes, multistatic)
9. THEORY Part 6    ─────────────►  Test plan Phase 10 (report honestly)
```

## 8.4 Extending the plan: experiments the theory suggests but the plan does not cover

If you finish the plan and want to keep going, these are well-posed and grounded
in the theory above:

| Experiment | Theory it tests | Method sketch |
|---|---|---|
| **Fresnel blind-spot map** | [§2.10](#210-fresnel-zones-why-placement-dominates-everything) | Fix the node; move a seated subject in 5 cm steps along the link axis; plot breathing SNR. You should see periodic nulls at roughly λ/2 = 6.25 cm spacing. A direct, visual confirmation of Fresnel structure. |
| **Bandwidth vs. resolution** | [§2.12](#212-bandwidth-is-resolution--and-why-channel-hopping-exists) | Record the same scene with a fixed channel and with 1/6/11 hopping; compare CIR tap separability from [`ruvsense/cir.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/cir.rs). |
| **Phase-artefact demonstration** | [§2.9](#29-the-hardware-lies-cfo-sfo-and-why-raw-phase-is-garbage) | Plot raw per-packet phase vs. sanitized phase on a *static* scene. The raw trace should look like noise; the sanitized one flat. The single most convincing demo of why sanitization exists. |
| **Cross-room transfer failure** | [§3.1](#31-the-central-difficulty-domain-shift) | Train a bank in room A, run `room-watch` in room B. Expect degradation. Quantify it — this is the number that justifies per-room calibration. |
| **Baseline drift half-life** | [§4.5](#45-per-room-calibration--wifi-densepose-calibration) | Capture a baseline, then re-measure deviation hourly for 48 h with the room empty. When does the bank go `STALE`? That interval is your recalibration cadence. |
| **Harmonic confusion** | [§3.4](#34-layer-3--estimation-rate-extraction) | Breathe deliberately at 20 BPM (0.33 Hz) and watch whether reported heart rate locks onto 60 BPM. If it does, you have reproduced the harmonic trap. |
| **Adversarial injection** | [§2.11](#211-multistatic-geometry-why-more-nodes-beats-a-better-node) | Replay recorded CSI into the aggregator as if from a seventh node; confirm [`adversarial.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/adversarial.rs) flags the geometric inconsistency. |
| **Edge vs. host vitals agreement** | [§4.6](#46-vital-signs--wifi-densepose-vitals) | Log `/api/v1/vital-signs` and `/api/v1/edge-vitals` together for an hour. Bland-Altman plot. Disagreement localises the fault. |

---

# Part 9 — Resource library

Grouped by what you are trying to learn. Where a canonical URL exists it is
given; otherwise the citation is enough to find the work.

## 9.1 Start here — if you know nothing about radio

| Resource | Why |
|---|---|
| **Tse & Viswanath, *Fundamentals of Wireless Communication*** — [free PDF](https://web.stanford.edu/~dntse/wireless_book.html) | The standard graduate text. Chapters 2 (the wireless channel) and 3 (point-to-point) are exactly the material in [Part 2](#part-2--physics-from-first-principles). |
| **Goldsmith, *Wireless Communications*** (CUP, 2005) | More engineering-oriented; excellent on path loss, shadowing, and fading. |
| **Molisch, *Wireless Communications*** (Wiley) | Best single treatment of OFDM and multipath for a newcomer. |
| **Richards, *Fundamentals of Radar Signal Processing*** | RuView is a bistatic radar wearing a WiFi costume. This is the radar half. |
| **Skolnik, *Introduction to Radar Systems*** | The classic. Read the chapters on resolution and the radar equation. |
| **Pozar, *Microwave Engineering*** | For reflection coefficients, permittivity, and transmission-line intuition. |

## 9.2 WiFi sensing — surveys and entry points

| Resource | Why |
|---|---|
| **Ma, Zhou & Wang, "WiFi Sensing with Channel State Information: A Survey"**, *ACM Computing Surveys* 52(3), 2019 | The canonical survey. Read this before any paper. |
| **Yousefi et al., "A Survey on Behavior Recognition Using WiFi CSI"**, *IEEE Communications Magazine*, 2017 | Shorter, activity-recognition focused. |
| **Yang et al., "SenseFi: A Library and Benchmark on Deep-Learning-Empowered WiFi Human Sensing"**, *Patterns* (Cell Press), 2023 — [code](https://github.com/xyanchen/WiFi-CSI-Sensing-Benchmark) | The benchmark to measure a learned model against. Also the fastest way to get a baseline running. |
| **Liu et al., "Wireless Sensing for Human Activity: A Survey"**, *IEEE Comm. Surveys & Tutorials*, 2020 | Broader — covers radar, RFID, and WiFi together. |

## 9.3 Landmark papers by topic

**Localisation and phase sanitisation**
- Kotaru et al., **"SpotFi: Decimeter Level Localization Using WiFi"**, SIGCOMM 2015 — the conjugate-multiplication / MUSIC approach implemented in [`csi_ratio.rs`](../v2/crates/wifi-densepose-signal/src/csi_ratio.rs).
- Xie, Li & Li, **"Precise Power Delay Profiling with Commodity WiFi"** (Splicer), MobiCom 2015 — channel splicing; the theoretical basis for the multiband stitching in [`multiband.rs`](../v2/crates/wifi-densepose-signal/src/ruvsense/multiband.rs).
- Li et al., **"IndoTrack: Device-Free Indoor Human Tracking with Commodity Wi-Fi"**, UbiComp/IMWUT 2017.

**Vital signs**
- Adib et al., **"Smart Homes that Monitor Breathing and Heart Rate"** (Vital-Radio), CHI 2015 — [project](https://witrack.csail.mit.edu/vitalradio/). FMCW, not WiFi CSI, but it defines the ceiling.
- Liu et al., **"Tracking Vital Signs During Sleep Leveraging Off-the-shelf WiFi"**, MobiHoc 2015 — the paper that made CSI vitals credible.
- Zeng et al., **"FarSense: Pushing the Range Limit of WiFi-based Respiration Sensing with CSI Ratio"**, IMWUT 2019 — the CSI-ratio trick for range extension.
- Wang et al., **"Fresnel Zone Based Theory for Human Respiration Detection with Commodity WiFi"** — the Fresnel model behind [`fresnel.rs`](../v2/crates/wifi-densepose-signal/src/fresnel.rs).

**Pose and body sensing**
- Wang et al., **"Person-in-WiFi: Fine-grained Person Perception using WiFi"**, ICCV 2019 — [arXiv:1904.00276](https://arxiv.org/abs/1904.00276).
- Geng, Huang & De la Torre, **"DensePose From WiFi"**, 2023 — [arXiv:2301.00250](https://arxiv.org/abs/2301.00250). The direct namesake of this project.
- Zhao et al., **"Through-Wall Human Pose Estimation Using Radio Signals"** (RF-Pose), CVPR 2018 — [project](http://rfpose.csail.mit.edu/).
- Zhao et al., **"RF-based 3D Skeletons"** (RF-Pose3D), SIGCOMM 2018.
- Jiang et al., **"Towards 3D Human Pose Construction Using WiFi"** (WiPose), MobiCom 2020.

**Domain generalisation**
- Zheng et al., **"Zero-Effort Cross-Domain Gesture Recognition with Wi-Fi"** (Widar3.0), MobiSys 2019 — [dataset](http://tns.thss.tsinghua.edu.cn/widar3.0/). The BVP representation.
- Qian et al., **"Widar2.0: Passive Human Tracking with a Single Wi-Fi Link"**, MobiSys 2018.
- Jiang et al., **"Towards Environment Independent Device Free Human Activity Recognition"** (EI), MobiCom 2018 — adversarial domain adaptation.

**Through-wall and imaging**
- Adib & Katabi, **"See Through Walls with WiFi!"**, SIGCOMM 2013 — the paper that started the public conversation.
- Adib et al., **"3D Tracking via Body Radio Reflections"** (WiTrack), NSDI 2014 — [project](https://witrack.csail.mit.edu/).
- Huang et al., **"Feasibility and Limits of Wi-Fi Imaging"**, SenSys 2014 — an honest look at the ceiling, worth reading alongside the hype.

**Security and privacy**
- Zhu et al., **"Et Tu Alexa? When Commodity WiFi Devices Turn into Adversarial Motion Sensors"**, NDSS 2020 — the attack case for why privacy classing matters.
- Work on gait-based identification from CSI (e.g. WiWho, WifiU) — the reason "anonymous RF sensing" needs scare quotes.

## 9.4 Datasets

| Dataset | Contents | Link |
|---|---|---|
| **MM-Fi** | Multi-modal (WiFi CSI, mmWave, LiDAR, RGB-D) human sensing, 40 activities | [github.com/ybhbingo/MMFi_dataset](https://github.com/ybhbingo/MMFi_dataset) — the benchmark RuView's published pose model uses |
| **Widar 3.0** | Cross-domain gesture, 75 domains | [tns.thss.tsinghua.edu.cn/widar3.0](http://tns.thss.tsinghua.edu.cn/widar3.0/) |
| **SignFi** | 276 sign-language gestures via CSI | [github.com/yongsen/SignFi](https://github.com/yongsen/SignFi) |
| **UT-HAR** | Early activity-recognition CSI benchmark | Bundled in SenseFi |
| **NTU-Fi** | HAR + human ID, used by SenseFi | Bundled in SenseFi |
| **OPERAnet** | Multi-modal (WiFi CSI, PWR, UWB, Kinect) | Published via IEEE DataPort / Zenodo |
| **XRF55** | Large multi-modal RF action-recognition set | Search by name |

## 9.5 CSI extraction toolchains

| Tool | Hardware | Link |
|---|---|---|
| **ESP32-CSI-Tool** | ESP32 family | [stevenmhernandez.github.io/ESP32-CSI-Tool](https://stevenmhernandez.github.io/ESP32-CSI-Tool/) |
| **esp-csi** (Espressif official) | ESP32 family | [github.com/espressif/esp-csi](https://github.com/espressif/esp-csi) |
| **Linux 802.11n CSI Tool** | Intel 5300 | [dhalperi.github.io/linux-80211n-csitool](https://dhalperi.github.io/linux-80211n-csitool/) |
| **Atheros CSI Tool** | AR9xxx | [wands.sg/research/wifi/AtherosCSI](https://wands.sg/research/wifi/AtherosCSI/) |
| **Nexmon CSI** | Broadcom / Raspberry Pi | [github.com/seemoo-lab/nexmon_csi](https://github.com/seemoo-lab/nexmon_csi) |
| **PicoScenes** | Intel AX200/210, QCA, USRP; 802.11ax | [ps.zpj.io](https://ps.zpj.io/) |

## 9.6 Standards and specifications

| Item | Link |
|---|---|
| **IEEE 802.11bf — WLAN Sensing** task group | [ieee802.org/11/Reports/tgbf_update.htm](https://www.ieee802.org/11/Reports/tgbf_update.htm) |
| **"IEEE 802.11bf: Toward Ubiquitous Wi-Fi Sensing"** (survey) | [arXiv:2207.04859](https://arxiv.org/abs/2207.04859) |
| **IEEE 802.11-2020** base standard | IEEE Xplore |
| **Matter specification** | [csa-iot.org/all-solutions/matter](https://csa-iot.org/all-solutions/matter/) |
| **HomeKit Accessory Protocol (HAP)** | [developer.apple.com/homekit](https://developer.apple.com/homekit/) |

## 9.7 Hardware and firmware documentation

| Item | Link |
|---|---|
| ESP32-S3 datasheet | [espressif.com](https://www.espressif.com/en/support/documents/technical-documents) |
| ESP-IDF v5.4 programming guide | [docs.espressif.com/projects/esp-idf/en/v5.4/](https://docs.espressif.com/projects/esp-idf/en/v5.4/) |
| ESP-IDF Wi-Fi CSI API | ESP-IDF guide → Wi-Fi → Channel State Information |
| ESP32-C6 (Wi-Fi 6, 802.15.4) | [espressif.com/en/products/socs/esp32-c6](https://www.espressif.com/en/products/socs/esp32-c6) |

## 9.8 Algorithms referenced in this codebase

| Algorithm | Canonical reference |
|---|---|
| **Welford's online variance** | Welford (1962); Knuth, *TAOCP* Vol. 2 §4.2.2 |
| **Hampel filter** | Hampel (1974), "The Influence Curve and its Role in Robust Estimation" |
| **ISTA / FISTA** ($\ell_1$ sparse recovery) | Beck & Teboulle (2009), *SIAM J. Imaging Sciences* |
| **HNSW** (approximate k-NN) | Malkov & Yashunin, [arXiv:1603.09320](https://arxiv.org/abs/1603.09320) |
| **Stoer-Wagner min-cut** | Stoer & Wagner (1997), *JACM* |
| **Dynamic Time Warping** | Sakoe & Chiba (1978) |
| **Kalman filter** | Kalman (1960); Bar-Shalom, *Estimation with Applications to Tracking* |
| **von Mises / circular statistics** | Mardia & Jupp, *Directional Statistics* |
| **Contrastive learning** | Chen et al., SimCLR, [arXiv:2002.05709](https://arxiv.org/abs/2002.05709) |
| **Masked autoencoder pretraining** | He et al., MAE, [arXiv:2111.06377](https://arxiv.org/abs/2111.06377) |
| **LoRA** (low-rank adaptation) | Hu et al., [arXiv:2106.09685](https://arxiv.org/abs/2106.09685) |
| **Tissue dielectric properties** | Gabriel et al. (1996); [IT'IS tissue-property database](https://itis.swiss/virtual-population/tissue-properties/database/dielectric-properties/) |

## 9.9 Tooling used by this project

| Tool | Link |
|---|---|
| Rust | [rust-lang.org](https://www.rust-lang.org/) · [The Book](https://doc.rust-lang.org/book/) |
| Candle (Rust ML) | [github.com/huggingface/candle](https://github.com/huggingface/candle) |
| ONNX Runtime (`ort`) | [onnxruntime.ai](https://onnxruntime.ai/) |
| safetensors | [github.com/huggingface/safetensors](https://github.com/huggingface/safetensors) |
| PyO3 (Rust ↔ Python) | [pyo3.rs](https://pyo3.rs/) |
| MediaPipe (camera ground truth) | [ai.google.dev/edge/mediapipe](https://ai.google.dev/edge/mediapipe) |
| Home Assistant | [home-assistant.io](https://www.home-assistant.io/) |

## 9.10 This project's own artefacts

| Artefact | Link |
|---|---|
| Pretrained CSI encoder | [huggingface.co/ruvnet/wifi-densepose-pretrained](https://huggingface.co/ruvnet/wifi-densepose-pretrained) |
| MM-Fi pose model | [huggingface.co/ruvnet/wifi-densepose-mmfi-pose](https://huggingface.co/ruvnet/wifi-densepose-mmfi-pose) |
| AetherArena benchmark Space | [huggingface.co/spaces/ruvnet/aether-arena](https://huggingface.co/spaces/ruvnet/aether-arena) |
| PyPI | [`ruview`](https://pypi.org/project/ruview/) · [`wifi-densepose`](https://pypi.org/project/wifi-densepose/) |
| Operator guide (2,500 lines) | [`docs/user-guide.md`](user-guide.md) |
| Field failures + fixes | [`docs/TROUBLESHOOTING.md`](TROUBLESHOOTING.md) |
| MM-Fi study (the honest picture) | [`docs/benchmarks/mmfi-wifi-sensing-study.md`](benchmarks/mmfi-wifi-sensing-study.md) |
| Efficiency frontier | [`docs/benchmarks/wifi-pose-efficiency-frontier.md`](benchmarks/wifi-pose-efficiency-frontier.md) |
| Capability attestation matrix | [`docs/WITNESS-LOG-028.md`](WITNESS-LOG-028.md) |
| Physics derivations | [`examples/research-sota/`](../examples/research-sota/) |

---

# Part 10 — Glossary

| Term | Meaning |
|---|---|
| **ADR** | Architecture Decision Record. A numbered document capturing one design decision, its context, and its consequences. This repo has 207 of them and the code cites them by number. |
| **AGC** | Automatic Gain Control. The receiver's per-packet gain adjustment — a source of amplitude discontinuities in CSI. |
| **Anchor** | A short, labelled capture during enrollment (`empty`, `sit`, `breathe_slow`, …) used to fit the room's specialists. |
| **Bank** (specialist bank) | The versioned collection of small models fitted to one room, tied to the baseline it was trained against. |
| **Baseline** | The empty-room statistical fingerprint (Welford amplitude + circular phase). Everything downstream is a deviation from it. |
| **BFI / BFLD** | Beamforming Feedback Information — compressed channel state that 802.11ac/ax stations report to the AP. BFLD is this project's privacy-gated sensing layer built on it. |
| **Bistatic / Multistatic** | Bistatic: transmitter and receiver in different places. Multistatic: many such pairs, giving geometric diversity. |
| **BVP** | Body Velocity Profile — a velocity × time representation that generalises across environments better than raw CSI. |
| **CFO / SFO / PDD** | Carrier Frequency Offset / Sampling Frequency Offset / Packet Detection Delay. The three hardware artefacts that make raw CSI phase unusable. |
| **CIR** | Channel Impulse Response — the time-domain view: what the room does to an instantaneous pulse. Inverse-FFT dual of the CFR. |
| **Cog** | This project's packaging format for an edge module — a signed, versioned binary that runs on a node or gateway. |
| **Coherence gate** | The rule that decides per frame whether to Accept / PredictOnly / Reject / Recalibrate, based on how consistent the observation is with a running template. |
| **COCO keypoints** | The 17-joint skeleton convention (nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles). |
| **CSI** | Channel State Information — the per-subcarrier complex channel estimate. **The measurement this whole system is built on.** |
| **DensePose** | Originally a computer-vision task: dense body-surface correspondence from an image. Here, the aspiration of doing something similar from RF. |
| **Edge tier** | The firmware's 0–3 setting controlling how much DSP happens on the node vs. the host. Runtime-switchable via NVS. |
| **Enrollment** | The guided capture of labelled anchors that teaches the system your room. |
| **Fresnel zone** | Nested ellipsoids between TX and RX defined by path-length difference. Determines whether a given chest position produces a strong or null breathing signal. |
| **HNSW** | Hierarchical Navigable Small World — the graph structure enabling sub-linear approximate nearest-neighbour search over embeddings. |
| **HT20 / HT40 / HE20** | 802.11n 20 / 40 MHz and 802.11ax 20 MHz PHY modes. Determines subcarrier count; must match the `--tier` flag in the CLI. |
| **I/Q** | In-phase and Quadrature — the real and imaginary parts of a complex sample. ADR-018 ships them as signed 8-bit pairs. |
| **MAT** | Mass Casualty Assessment Tool — the disaster-response vertical for finding survivors in rubble. |
| **Min-cut** | The graph partition of minimum total edge weight. Used here to separate multiple people and to detect adversarial injection. |
| **Multipath** | The many reflected paths a signal takes. The reason CSI is informative and the reason it is hard. |
| **NDP** | Null Data Packet — a frame with no payload, transmitted purely so the receiver can measure the channel. |
| **NVS** | Non-Volatile Storage — the ESP32 key-value partition holding runtime config. Changing it never requires reflashing the app. |
| **OFDM** | Orthogonal Frequency-Division Multiplexing — parallel narrow subcarriers. The reason CSI is a vector, not a scalar. |
| **PCK@20** | Percentage of Correct Keypoints within a 20-pixel (or normalised) threshold. The pose accuracy metric used here. |
| **Privacy class (P0–P5)** | BFLD's egress classification. P0 raw and P3–P5 identity/biometric are structurally forbidden from leaving the device. |
| **RSSI** | Received Signal Strength Indicator — one scalar per packet. Far less informative than CSI; the fallback for laptops. |
| **RuvSense** | This project's multistatic sensing bounded context (ADR-029/030) inside `wifi-densepose-signal`. |
| **RVF** | The RuVector model container format. **Binary** RVF is what `--model` parses; the Hugging Face bundle currently ships JSONL RVF. |
| **Specialist** | One small room-calibrated model for one biological signal (presence, posture, breathing, heartbeat, restlessness, anomaly). |
| **STALE** | The bank's honest degradation flag: the baseline has drifted far enough that readings should not be trusted. |
| **Subcarrier** | One of the parallel narrow frequency slots in an OFDM symbol. 312.5 kHz apart in 802.11n. |
| **TDM** | Time-Division Multiplexing — giving each node its own transmit slot so six nodes do not jam each other. |
| **Tomography** | Reconstructing a spatial occupancy map by inverting many path-integral (per-link attenuation) measurements. |
| **Welford's algorithm** | Numerically stable, constant-memory online mean and variance. Used on both the MCU and the host. |
| **Witness chain** | An Ed25519-signed audit trail proving a reported measurement is the one actually produced. |

---

## Closing note

The best thing about this repository is not any single capability. It is that
**it tells you which of its capabilities are real.** That is rare in this field,
where the gap between a demo and a deployment is enormous and rarely disclosed.

If you take three things from this document:

1. **Placement and calibration dominate.** More than the model, more than the
   node count, more than anything in the software. [§2.10](#210-fresnel-zones-why-placement-dominates-everything), [§3.1](#31-the-central-difficulty-domain-shift).
2. **Bandwidth is resolution.** 20 MHz gives you 7.5 m. Every architectural
   choice in this project is downstream of that number. [§2.12](#212-bandwidth-is-resolution--and-why-channel-hopping-exists).
3. **State how you measured it.** A characterised 70% is worth more than an
   unreproducible 95%. [Part 6](#part-6--the-honesty-ledger).

Then go run [`plans/ruview-esp32s3-test-plan.md`](../plans/ruview-esp32s3-test-plan.md).




