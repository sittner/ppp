# PPP 6x EL34 – Design Review / Improvements

Review of the KiCad schematic (`amp.kicad_sch`, `supply.kicad_sch`), Rev 1, 2026-10-20.
Nothing in the schematic has been changed yet; this file lists the proposed changes.

## Assumptions

All operating points below are calculated estimates and should be verified by
measurement on the built amplifier.

- HV windings 310 V~ / 0.25 A → ≈ 438 V DC under load, ≈ 470–510 V at no load
  (cold tubes, silicon bridge) and +10 % mains.
- EL34 idle current ≈ 40 mA per tube.
- Full power ≈ 140 W into Rkk 330 Ω → ≈ 215 V rms across the primary,
  M1/M2 each swing ≈ ±150 V peak against GND.
- Driver: V1B cathode follower ≈ 1 mA, V2 grid ≈ 49 V, V2 cathodes ≈ 52 V
  (estimated from ECC83/ECC81 data sheet curves).

## 1. Must fix (ratings exceeded)

| Part | Problem | Proposal |
|---|---|---|
| C101–C104 470µ/450V | Rails ≈ 440 V nominal; up to 470–510 V at no load / high mains (B+ appears before the tubes conduct). | 500/550 V types, or two caps in series with balancing resistors. |
| R102, R104 150k (DIN0414) | 440²/150k ≈ 1.3 W in a 1 W body; 440–480 V exceeds the usual 0414 working voltage. | 2× 68k / 3 W in series (≈ 0.7 W each, voltage split, τ ≈ 130 s). |
| C15, C16 47µ/100V | Half-wave peak of the ≈ 60 V~ bias winding ≈ 85 V, close to 100 V at high mains. | 160 V or 200 V. (C17/C18 see ≈ 63 V, 100 V is fine.) |
| R13 2.7k (LTP tail, DIN0414) | ≈ 52 V across it → ≈ 19 mA, ≈ 1.0 W in a 1 W body. | 3 W type. |
| R15, R16 10k (DIN0414) | ≈ 9.5 mA each → ≈ 0.9 W. | 2–3 W type. |
| V2 ECC81 | ≈ 9.5 mA × 280 V ≈ 2.7 W per triode (max 2.5 W). | Verify by measurement; if confirmed, R13 → 3.3k (≈ 7.7 mA, ≈ 2.3 W) or R15/R16 → 12k. |

## 2. Safety / wiring

1. **EL34 heater–cathode voltage.** In a circlotron the cathode buses M1/M2
   swing ≈ ±150 V peak at full power while the heaters are referenced to GND.
   EL34 Vhk max is 100 V. Use a separate 6.3 V winding per output bank,
   referenced to M1 and M2 respectively. With one NTR-P/1 per monoblock the
   second 6.3 V winding is free; this also relieves the current single
   winding, which carries 9.6 A of its 10 A rating.
2. **Speaker secondary floats.** T2 pin 7 ("0") only connects to J3. Tie it to
   chassis/GND so a primary–secondary insulation fault cannot put HV on the
   speaker terminals.
3. **PE is unconnected.** `Earth_Protective` only goes to J101.3. Add a
   PE–chassis bond and a GND link (direct, or via a ground-lift 10 Ω ∥ 100 nF).
4. **Mains fuse too large.** Total load ≈ 230 VA (≈ 1 A primary). F101 5 A/T →
   2–2.5 A/T (3.15 A/T if T101 is a toroid). Add an NTC or soft-start:
   4× 470 µF at 440 V are charged without any current limiting.
5. **Bias pot fail-safe.** If a wiper of P1–P6 loses contact, that EL34 grid
   floats and the tube runs away. Connect each wiper (pin 2) to pin 3 (the
   negative end) so a failure gives maximum negative bias.
   The trimmers sit on M1/M2 potential – adjust only with an insulated tool.

## 3. Smaller improvements

- **R101, R103 470R (DIN0617, 2 W):** ≈ 0.3 W at idle, but ≈ 1.5–3.5 W under
  heavy drive (screen current) → 5 W type. Screen voltage is ≈ 418 V at idle
  and above the EL34 Vg2 max of 425 V at high mains; consider 1–1.5k.
- **R49, R50 47R:** ≈ 0.21 W each in a 0207 → 100R or 0.5–1 W types.
- **R37–R42 10R:** power is fine; use 1 % / 1 W for accurate bias
  measurement and add test points.
- **C1 2.2µ/40V bipolar:** no DC on either side → 2.2 µ/63 V MKT film.
- **C7 0.22µ/630V:** sees only ≈ 150 V → 400 V type, much smaller footprint.
  C8–C14 really need 630 V (≈ 370 V DC plus the M1/M2 swing).
- **Feedback pickup:** R10/C6 connect at the V3 cathode (above R37), not at
  the M1 bus, so the drop across R37 (V3 current only) enters the loop.
  Take the feedback from M1 instead.
- **T101 HV windings:** each 0.25 A winding supplies ≈ 150 mA DC at idle into a
  capacitor-input bridge – already at its rating, exceeded under sustained
  full power. Same as the original design; expect the windings to run hot.

## 4. Checked and OK

- Zener string D1–D4: ≤ 0.55 W per 1 W diode (worst case at power-on).
- C4/C5 47µ/450V on the ≈ 356 V zener-regulated driver rail.
- C10 22µ/100V at ≈ 49 V.
- Bias filter R17/R18 + C17/C18: ≈ 14 mV ripple.
- B500C1500 bridges: voltage and current rating sufficient.
- Screen stoppers R43–R48 150R, grid stoppers R31–R36 10k.
- Grid leaks R19–R24 220k (EL34 fixed-bias max 500k).
- Bias range ≈ −36 V … −63 V, which fails safe toward minimum −36 V.
