# PPP 4x KT88 – SMPS Supply Notes

Branch `smps`: replace the 50 Hz mains transformer T101 and the rectifier/filter on
`supply.kicad_sch` with a switch-mode supply. This is a concept and winding spec;
the schematic is not changed yet.
All figures are estimates from data sheets and the assumptions in `KT88.md` and
`IMPROVEMENTS.md` (≈ 438 V DC rails, 0.35 A per HV winding). Check them on the
prototype.

## Verdict

Feasible and worth doing, but it is a real power-electronics project
(≈ 400 W, offline, 440 V floating outputs, custom transformer).
You don't have to choose between "synced" and "resonant":
one fixed-frequency LLC half-bridge with **one** transformer gives both.

- **Synchronous by construction:** all outputs come from one transformer driven by one
  clock. The PFC and the heater buck are locked to that clock, so there are no
  free-running converters that can beat into the audio band.
- **Resonant:** zero-voltage switching on the primary (slow, clean edges) and
  zero-current turn-off of the output diodes (no reverse-recovery ringing).
- **Low coupling:** a 3-chamber bobbin with mirror-symmetric bank windings.
  Each bank sees only a few pF from the primary, and what is left is equal on M1 and M2,
  so it cancels in the centre-tapped output transformer.

### Why

| | Today (T101, 50 Hz) | SMPS |
|---|---|---|
| Weight | ≈ 3–4 kg custom transformer | ≈ 0.5–0.8 kg incl. PFC choke, filter, heatsinks |
| Hum | 50 Hz stray field into T2, rectifier charging pulses and diode buzz in the wiring | No 50 Hz field. Line rectifier pulses stay in a small loop on the SMPS board. B+ ripple is removed by cap multipliers |
| Interwinding C (bank ↔ rest) | a few nF | ≈ 10–30 pF |
| B+ vs. mains | follows mains ±10 % | regulated by the PFC bus |
| Heaters | AC, 12.6 V | DC, regulated, soft start |
| Sequencing / protection | none | heater first, delayed B+ ramp, overcurrent trip |
| Learning value | – | Covers PFC, resonant conversion, magnetics design, insulation coordination and EMI on a project you care about and can measure |

### Risks

- The transformer is the core of the design and is custom. Plan for 1–2 prototype iterations.
- Mains-referenced high voltage on an open board. Use an isolation transformer and a
  current-limited variac for bring-up.
- Size for music peaks (0.5 A per bank for seconds) so the current limit never
  folds back on a bass transient.
- Mains leakage through the Y-caps to PE can create a new ground-loop hum with the
  source equipment. Keep the Y-caps small.

## What the circlotron needs

From the netlist of `supply.kicad_sch` / `amp.kicad_sch`. There are three
potential domains:

| Domain | Today | Nets | Load |
|---|---|---|---|
| **M1** (swings ±150 V) | T101.11-12 → D102, T101.7-8 → D5 | A2 (+440 V), G1 screens, UG1 bias | 0.15 A idle, 0.35 A design, 0.5 A peaks; bias < 5 mA |
| **M2** (antiphase) | T101.3-4 → D101, T101.9-10 → D6 | A1, G2, UG2 | same |
| **GND** | T101.5-6 | HEATER_A/B, strings centre-grounded at the tubes | 12.6 V 3.5 A |

The driver rail (≈ 356 V zener string D1–D4) is fed from G1/G2 through R11/R12
and needs no winding of its own.

Output power: 2 × 440 V × 0.35 A + 44 W heaters + aux ≈ **360 W continuous,
≈ 500 W for a few seconds**. The design point is 400 W.

Worst-case voltage between the two bank windings is A1 against M1:
440 V + 2 × 150 V ≈ **750 V peak**. Bank to primary/chassis is ≈ 600 V peak.

## Architecture

```
mains → fuse, NTC/relay → EMI filter (X, CM choke, small Y) → bridge
      → CCM PFC boost, Vbus = 390 V (trim 380–400 V), 330 µF/450 V
           clock: fs (or fs/2), derived from the LLC clock
      → half-bridge, 600 V SJ MOSFETs, fixed fs ≈ 95 kHz, 50 %, ~200 ns dead time
      → Lr (external, ≈ 54 µH incl. leakage) + Cr 47 nF → T201 (spec below)

T201 chamber A (M1): W4 HV  → 4× SiC → 4.7 µF film → CM choke → cap multiplier → C104 → A2 / M1
                     W5 bias → D5 (UF4007, half-wave, as today) → UG1 / M1
T201 chamber C (M2): W6 HV  → same → C102 → A1 / M2
                     W7 bias → D6 → UG2 / M2
T201 chamber B:      W3 heater → Schottky bridge → sync buck 12.6 V → HEATER_A/B
                     W2 aux    → 14 V for PFC/LLC controllers and gate drivers (primary side)
```

- **Fixed frequency, slightly below resonance** (fs ≈ 0.95 fr): the gain is
  ≈ 1 and almost load-independent, and the output diodes turn off at zero current even with
  Cr tolerance. The LLC does not regulate. The PFC bus sets all outputs.
  Below 150 kHz the fundamental stays under the conducted-EMI band.
- A fixed-frequency LLC needs **no LLC controller**: a clock, a dead-time
  generator and a half-bridge driver are enough. Soft start: sweep from
  ≈ 2.5·fs down to fs over ≈ 50 ms. One option is an MCU with a high-resolution timer
  (e.g. STM32G4 HRTIM) that generates the LLC, PFC and soft-start timing from one clock.
- **The PFC bus ripple goes straight through the LLC.** At 400 W the bus carries
  ≈ 10 V pp of 100 Hz ripple on 330 µF, so ≈ 12 V pp on the raw 460 V rails.
  Each bank therefore gets a **MOSFET cap multiplier**:
  - ≈ 10 V drop: ≈ 3.5 W at 0.35 A, so it needs a heatsink. The tab sits on the bank
    potential, so use an insulated mount with equal pads on both banks for symmetry.
  - Gate RC τ ≈ 1–2 s gives ≈ 60 dB at 100 Hz and removes the switching residue.
  - A local timer per bank, powered from the raw rail, holds the gate low for
    ≈ 45 s, then ramps B+ up. This gives heater-first sequencing without any signal
    crossing the bank isolation. The bias rails come up immediately, so the tubes
    start fully biased off.
- **Keep C102/C104 large (220–470 µF, 500 V, see `IMPROVEMENTS.md` §1)** after
  the multiplier. In a circlotron the full output current flows through these caps,
  so they are in the signal path.
- **Heater buck clock taken from W3:** a comparator on the heater winding
  provides the sync. It is locked to fs automatically and needs no isolation.
- **No PFC?** Possible for a one-off build (audio amps are tested for
  EN 61000-3-2 at 1/8 power). Then Vbus ≈ 325 V ±10 % with more ripple:
  scale Np for 162 V half-bus and give the cap multipliers ≈ 25 V of headroom.
  The turns below assume the 390 V PFC bus.

## Transformer T201 – winding spec

### Core and operating point

| | |
|---|---|
| Core | ETD54 (Ae 280 mm², Ve 35.5 cm³), N97 / N87 / 3C95. ETD49 fits too but is tighter |
| Gap | centre leg, for AL ≈ 490 nH/N² (Lm ≈ 330 µH at Np = 26). Grind, or take the nearest stock AL and adjust |
| Bobbin | ETD54, **3 chambers side by side**: A (M1) │ B (primary) │ C (M2). Approx. 9 mm │ 16 mm │ 9 mm of ≈ 36 mm winding width. A 3-chamber bobbin may need extra walls glued into a 1- or 2-chamber bobbin. The walls only set spacing and capacitance; safety comes from the TIW |
| Frequency | fr ≈ 100 kHz, fs ≈ 95 kHz fixed |
| Primary voltage | square, ±195 V (Vbus/2) |
| Volts per turn | 195 V / 26 = 7.5 V |
| Flux | ΔB ≈ 0.14 T pp (Bpk ≈ 70 mT), core loss ≈ 1–1.5 W |
| Core connection | copper strap to chassis/PE (core counts as earthed, class I) |

### Windings

| # | Function | Chamber | Turns | Wire | Connects to (today's net) | Current |
|---|---|---|---|---|---|---|
| W1 | Primary | B, layers 1–3 | 26 | Litz 120 × 0.1 mm (0.94 mm²) | Lr / half-bridge, Cr | ≈ 2.4 A rms |
| W2 | Aux | B, on W1 | 2 | 0.4 mm enamelled | aux 14 V (primary side) | 0.3 A |
| S1 | Shield | B, on W2 | 1 | Cu foil 25 µm, full width, insulated overlap (**not** a shorted turn) | bus − (one end only) | – |
| W3 | Heater | B, outermost | 3 | TIW, ≈ 1 mm² (e.g. 3 × TIW 0.65 mm in parallel, or TIW litz) | bridge → buck → HEATER_A/B (T101.5-6) | ≈ 2.5 A rms |
| (S2) | optional shield | A, first layer | 1 layer | 0.2 mm TIW, one end free | M1 | – |
| W4 | HV bank M1 | A | 62 | TIW 0.5 mm (or TIW litz) | bridge → A2 (+) / M1 (−) (T101.11-12) | 0.39 A rms (0.55 peak) |
| W5 | Bias M1 | A, on W4 | 12 | TIW 0.3 mm | D5 → UG1 / M1 (T101.7-8) | < 5 mA |
| (S3) | optional shield | C, first layer | 1 layer | as S2 | M2 | – |
| W6 | HV bank M2 | C | 62 | as W4 | bridge → A1 (+) / M2 (−) (T101.3-4) | as W4 |
| W7 | Bias M2 | C, on W6 | 12 | as W5 | D6 → UG2 / M2 (T101.9-10) | < 5 mA |

Construction:

- **Chambers A and C must be mirror images.** Use the same turns, wire, layer count, tape and
  winding direction (as seen from the pin side). The start of each winding is on the inner
  layer and is marked with a dot, and both banks connect start/finish to their bridges the
  same way. That makes the switching dv/dt and the capacitance on M1 and M2 equal,
  so the residual common-mode current cancels in the centre-tapped T2 primary.
- HV windings: ≈ 11 turns per layer, 6 layers, one layer of polyester tape
  between layers (≈ 80 V per layer). Full-bridge rectification keeps the electrical
  centre of the winding quiet, so do not use a centre tap.
- All secondaries are triple-insulated wire, which gives **reinforced insulation to the
  primary** independent of bobbin creepage. Keep TIW all the way to the pins and keep the
  creepage from the earthed core to the secondary pins on the PCB.
  The primary litz to the earthed core needs basic insulation (bobbin + 2 layers of tape).
- Bank-to-bank (≈ 750 V peak working): two TIW insulations in series, plus
  the primary chamber in between.

### Resulting voltages (Vbus 390 V)

| Output | Raw | After | vs. today |
|---|---|---|---|
| HV (W4, W6) | 62 × 7.5 V − 2 × 1.5 V (SiC) ≈ 462 V no load, ≈ 450 V at 0.35 A | cap multiplier ≈ 440 V | ≈ 438 V |
| Bias (W5, W7) | 12 × 7.5 V − 1 V ≈ −89 V at C15/C16 | bias range ≈ −41 … −72 V | −85 V, −39 … −69 V |
| Heater (W3) | 3 × 7.5 V − 1 V ≈ 21.5 V | buck: 12.6 V DC | 12.6 V AC |
| Aux (W2) | ≈ 14 V | – | – |

Fine trim: Vbus 380–400 V gives ±11 V on B+. One HV turn = 7.5 V.

### Resonant tank

| | |
|---|---|
| Equivalent load | Rac = 8/π² · (195 V)² / 400 W ≈ 77 Ω, Q ≈ 0.4 at full load |
| Lr | ≈ 54 µH **total**: external inductor (≈ 30–45 µH) + measured T201 leakage. ETD34 / PQ32/20 N97 gapped, same litz, no saturation at 5 A |
| Cr | 47 nF PP pulse capacitor, ≥ 1000 V DC, ≥ 3 A rms at 100 kHz. Sees ≈ 70…320 V (195 V DC ± 125 V). Optional: split as 2 × 22 nF to bus+ and bus − |
| Ln = Lm/Lr | ≈ 6 |
| Magnetizing current | ≈ 1.55 A peak, so ZVS needs ≈ 80 ns; use 150–250 ns dead time |
| Rectifiers | HV: 650 V SiC Schottky, 2–4 A (4 per bank). Heater: 45–60 V Schottky, ≥ 10 A. Bias: D5/D6 **1N4007 → UF4007** (1N4007 is far too slow at 100 kHz) |
| Protection | primary current sense (CT or shunt), fast shutdown, then restart with soft start. OVP via PFC bus |

### Acceptance measurements (bare transformer, LCR meter at 100 kHz)

| Test | Target |
|---|---|
| Lm: W1, all others open | 330 µH ± 5 % |
| Lk: W1, all secondaries shorted | measure, then set the external Lr to 54 µH − Lk |
| C W1+W2+S1+core ↔ W4+W5 (each shorted on itself) | ≤ 15 pF |
| same ↔ W6+W7 | ≤ 15 pF, **within 10 % of the previous value** |
| C W4 ↔ W6 | ≤ 10 pF |
| Lk measured from W4 and from W6 | within 5 % (equal regulation of both banks) |
| Hipot primary ↔ secondaries; between banks; to core | reinforced for ≈ 750 V peak working (look up the value in EN 62368-1, don't assume 3 kV); banks ≥ 2 kV DC |

In the amplifier, check with a current probe on the M1 and M2 leads and a scope
on the speaker terminals at fs and its harmonics. If the 100 kHz residue is too high, fit S2/S3
and the optional Y-caps: one ≤ 470 pF Y1 from each M node to bus −, both
equal. Keep them small, because they carry the ±150 V audio swing. A 1–2.2 nF Y1 from
chassis to bus − on the SMPS board closes the loop for primary-generated
common-mode current locally.

## Changes to the schematic (to do)

- `supply.kicad_sch`: T101, D101/D102, F101 sizing and the HV filter are replaced by an SMPS
  sheet (PFC, LLC, T201, rectifiers, cap multipliers, heater buck). The labels
  A1/A2, M1/M2, G1/G2, UG1/UG2 and HEATER_A/B stay the same, so `amp.kicad_sch`
  hardly changes.
- `amp.kicad_sch`: D5/D6 1N4007 → UF4007. The heaters become DC, the string midpoints
  stay grounded and the heater domain floats otherwise.
- R101/R103 screen resistors and C101–C104 stay. C101–C104 get the 500 V rating
  from `IMPROVEMENTS.md`.

## Effect on IMPROVEMENTS.md / KT88.md

- §1 C101–C104 voltage: still applies. The no-load overshoot is gone (regulated bus,
  delayed B+ ramp), but use 500 V parts anyway.
- §1 C15/C16: ≈ 89 V, so 160 V types are still required.
- §2.3 PE bond: **mandatory** now (Y-caps, earthed core).
- §2.4 fuse/inrush: replaced by the SMPS input stage (NTC + bypass relay, 3.15 A/T).
- §3 T101 HV windings running hot: obsolete.
- `KT88.md` §2 bias range: shifts slightly to ≈ −41 … −72 V and still covers −45 … −55 V.
