# europocket — Hardware Design

Interface between the **Eurorack** and **Pocket Operator (PO)** domains. The two signal
domains differ in both timing (clock rate) and electrical levels, so the design breaks
into three functional blocks: **Power**, **Clock converter**, and **Level converter**.

> Reference designators, values and nets in this document are reconciled against
> `Netlist_europocket_2026-08-03.tel`.
>
> **Designators changed in this revision.** The Europower header moved `J3` → **`J1`**,
> reset moved `J5` → **`J7`**, the buffer and trigger test points were replaced by
> labelled PO test points (`J2`, `J4`, `J6`), and the audio outputs and clock input are
> now headers **`H1`** and **`H2`**. Older notes and captures using the previous
> numbering will not match.

---

## Power

Powered from the **Eurorack bus** via header **J1** ("Europower", 2×05 IDC). The board
takes **±12 V** from the bus and locally regulates **+3.3 V** for the CMOS logic. The
TL072 in the Level converter runs directly from **±12 V**.

**Rails**

| Rail    | Source                          | Consumers                                   |
|---------|---------------------------------|---------------------------------------------|
| +12 V   | J1 pins 9–10, via D1            | LM1117 input (U1.3), TL072 V+ (U4.8)        |
| −12 V   | J1 pins 1–2, via D2             | TL072 V− (U4.4)                             |
| +3.3 V  | LM1117 (U1) output              | U2, U3, R2/R5 pull-ups, J9 test point       |
| GND     | J1 pins 3–8                     | all blocks, plus all seven mounting holes    |

> **−12 V has exactly one consumer: the op-amp.** Nothing else on the board depends on
> it, so a dead −12 V rail leaves the clock converter working normally while the audio
> path is silent. Check it directly at U4.4 when diagnosing a dead audio path.

**Components**

- `U1` — LM1117MP-3.3 (SOT-223) LDO: **+12 V → +3.3 V** (Vin = U1.3, Vout = U1.2, GND = U1.1).
- `D1`, `D2` — 1N5819WS Schottky, reverse-polarity / mis-insertion protection on the +12 V and −12 V rails.
- Bulk: `C1` 22 µF (+12 V), `C2` 100 µF (+3.3 V), `C10` 10 µF (+12 V), `C11` 10 µF (−12 V).
- Decoupling (100 nF, one per rail/IC, adjacent to the pin): `C4`/`C5` on +3.3 V (U2/U3), `C6` on +12 V, `C7` on −12 V.

Measured on hardware (2026-08-03): **+11.2 V** at U4.8 and **−12.1 V** at U4.4 — the
asymmetry is bus tolerance plus the Schottky drop, and it means the positive half of the
audio waveform clips roughly a volt before the negative half.

---

## Clock converter

### Eurorack Trigger → Pocket Operator Sync Converter

**Purpose.** Convert a Eurorack clock/trigger into a Pocket Operator compatible sync
signal. The design assumes a **4 PPQN** Eurorack clock and produces a **2 PPQN** Pocket
Operator sync clock by dividing the clock by two.

**Requirements**

- Accept Eurorack triggers up to approximately **10–12 V**.
- Protect the logic from overvoltage and from negative excursions.
- Reject narrow glitches.
- Produce clean **3.3 V CMOS** logic.
- Divide clock by two.
- Provide deterministic reset.

#### Functional block diagram

```
Eurorack Trigger (H2.2)
        │
        ▼
Input Protection / Level Shift   (Q1, R1, R15, D3)
        │
        ▼
RC Noise Filter                  (R3 / C3)
        │
        ▼
Schmitt Trigger Inverter         (U2, 74AHC1G14)
        │
        ▼
Toggle Flip-Flop (÷2)            (U3, 74HC74)
        │
        ▼
Pocket Operator Sync Output      (R4 → J6)
```

#### Stage 1 — Input conditioning

**Connector H2 (clock in):** H2.2 = Eurorack clock, H2.1 = GND.

**Components**

- `Q1` — MMBT3904 (SOT-23) NPN
- `R1` — 10 kΩ (base series)
- `R2` — 10 kΩ collector pull-up to 3.3 V
- `R15` — 10 kΩ base–emitter pulldown
- `D3` — SOD-323 small-signal diode, base clamp

**Wiring**

```
Clock input (H2.2) → R1 → Q1 base ─┬─ R15 → GND
                                    └─ D3 cathode (anode → GND)
Q1 emitter    → GND
Q1 collector  → R2 pull-up to 3.3 V   (collector = logic signal)
```

**Behaviour**

| Input | Collector |
|-------|-----------|
| Low   | High      |
| High  | Low       |

The transistor **level shifts**, **protects** the logic, and **inverts**.

**Why R15 and D3 are present.** Without the `R15` pulldown the stage switches at a single
V_BE (~0.65 V). A Eurorack clock source measured on hardware idles **low at ~0.68 V**,
above that threshold — so Q1 never turns off and the collector sticks low. R15 divides the
base voltage and lifts the effective threshold to ~1.3 V:

| Input       | Base voltage (with R15) | Q1 |
|-------------|-------------------------|----|
| Low 0.68 V  | ÷2 → 0.34 V             | off ✓ |
| High 4.7 V  | Thevenin 2.34 V / 5 kΩ → I_B ≈ 0.33 mA | saturated ✓ |

`D3` clamps negative inputs at −0.6 V. Bipolar Eurorack signals (±5 V/±8 V LFOs, or audio
used as a clock) would otherwise reverse-bias Q1's base–emitter junction; V_EBO(max) is
only 6 V, and repeated B–E avalanche permanently degrades hFE. R1 limits the clamp
current for inputs down to at least −12 V.

#### Stage 2 — Noise filter

**Components:** `R3` = 10 kΩ, `C3` = 1 nF → **time constant ≈ 10 µs**.

```
Q1 collector
     │
     R3
     │
     +------> Schmitt Trigger input (U2.2)
     │
     C3
     │
    GND
```

**Purpose:** remove spikes, slightly slow edges before the Schmitt trigger.

#### Stage 3 — Schmitt trigger

**Device:** `U2` — 74AHC1G14SE-7 (single inverting Schmitt trigger, **SOT-353/SC-70-5** —
the `SE` suffix; 0.65 mm pitch, smaller than SOT-23-5).

- Power: VCC = 3.3 V (U2.5), GND (U2.3), 100 nF decoupling.
- Input: U2.2, from the RC network.
- Output: U2.4, clean digital clock — restores fast logic edges.

#### Stage 4 — Clock divider

**Device:** `U3` — SN74HC74DR (dual D flip-flop, SOIC-14). **First flip-flop only.**

- Power: VCC = 3.3 V (U3.14), GND (U3.7), 100 nF decoupling.
- Clock: `1CLK` (U3.3) ← Schmitt output (U2.4).
- Toggle configuration: `1D` (U3.2) ← `1Q̄` (U3.6) — converts the D flip-flop into a T flip-flop.

**Result:** every rising edge, `1Q` toggles → **output frequency = input frequency / 2**.

**Reset**

External reset via test point **J7 ("Reset")**, asynchronously clearing the first
flip-flop. On the assembled module this connects to the PO's Play button, so the divider
restarts deterministically.

- `/CLR1` (U3.1) — J7 with 10 kΩ pull-up `R5` to 3.3 V; drive low to reset.
- `/PRE1` (U3.4) — tied directly to 3.3 V (inactive).

**Unused second flip-flop.** All four inputs are tied to 3.3 V — `/PRE2` (U3.10),
`2CLK` (U3.11), `2D` (U3.12), `/2CLR` (U3.13). Outputs `2Q` (U3.9) and `2Q̄` (U3.8) are
left open. A static high on 2CLK is as valid as a static low: what matters is that no
input floats, since a floating HC clock input can self-oscillate and raise supply
current. The previous revision's discrete pull-ups R6/R7/R8 are no longer fitted.

#### Output

`1Q` (U3.5) → **1 kΩ series resistor `R4`** → test point **J6 ("PO CLK")**.

- Purpose of resistor: short-circuit protection, cable protection, output current limiting.
- Output amplitude: **0–3.3 V**.

Verified on hardware: 5.556 Hz in → 2.747 Hz out (÷2), 0–3.3 V, ~50 % duty.

#### Expected timing

```
Input (4 PPQN)   | | | | | | | |
Schmitt Output   | | | | | | | |
Flip-Flop Output __----____----____
```

Transitions occur every second input pulse — equivalent to **2 PPQN**.

---

## Level converter

Two independent buffer/amplifier channels built on a dual **TL072 (U4)** running from
**±12 V**. Each channel is an inverting stage with the non-inverting input grounded.

**Per-channel topology**

```
In (J2 / J4) → R_in ──┬── (−) op-amp ── out ── R_series ── Out (H1)
                      │                    │
                      └──── R_fb ──────────┤
                      └──── C_fb ──────────┘
             (+) op-amp → GND
```

| Element        | Channel A            | Channel B            | Value    |
|----------------|----------------------|----------------------|----------|
| Op-amp         | U4 A (in − U4.2, out U4.1) | U4 B (in − U4.6, out U4.7) | — |
| Input          | J4 "PO In R"         | J2 "PO In L"         | — |
| Input resistor | R9                   | R12                  | 49.9 kΩ  |
| Feedback       | R10                  | R13                  | 220 kΩ   |
| Feedback cap   | C8                   | C9                   | 18 pF    |
| Output series  | R11                  | R14                  | 100 Ω    |
| Output         | H1.4 "OUT-L"         | H1.2 "OUT-R"         | — |

- **Direction:** Pocket Operator output → Eurorack (boost line level up to modular level).
- **Gain:** −R_fb / R_in = −220 kΩ / 49.9 kΩ ≈ **−4.41**.
- **Level:** the PO's measured **2.2 Vpp** output (loudest volume, steady note, near-square
  chip voice) becomes **~9.7 Vpp** — standard Eurorack modular level, comfortably inside
  the TL072's ~21 Vpp swing on ±12 V.
- **Roll-off:** the feedback cap sets a pole at `1/(2π·R_fb·C_fb)` = 1/(2π·220 k·18 pF) ≈
  **40.2 kHz**. The PO voice is a near-square wave whose harmonics extend well above the
  fundamental, so the pole is deliberately kept above ~30 kHz — lower would round the
  edges and dull the tone.
- Powered from the ±12 V rails, so output can swing well beyond the 3.3 V logic domain.

**The gain assumes full PO volume.** 2.2 Vpp is the measurement at loudest; at lower
volume settings the output scales down proportionally and will not reach modular level.

**U4 orientation.** The silkscreen has no pin-1 marker. Use the decoupling caps: **C7
(−12 V) is adjacent to pin 4**, **C6 (+12 V) is adjacent to pin 8**. With the board
oriented as in `pcb-top.png`, the top row is pins 4-3-2-1 left to right and the bottom
row is 5-6-7-8, putting **pin 1 at the top-right**. Pins 4 and 8 are diagonally opposite,
so a 180° rotation swaps VCC+ and VCC− exactly — the worst mis-orientation this package
allows. Probe the C6/C7 pads rather than the gull-wing leads: same nets, far larger
targets, and a disagreement between pad and lead reveals an open joint.

---

## Connectors and test points

| Ref | Label      | Function                                        |
|-----|------------|-------------------------------------------------|
| J1  | Europower  | Eurorack ±12 V / GND bus header (IDC 2×05)      |
| H1  | —          | Audio out: H1.4 = OUT-L, H1.2 = OUT-R, H1.1/H1.3 = GND |
| H2  | —          | Clock in: H2.2 = signal, H2.1 = GND             |
| J2  | PO In L    | Pocket Operator left audio in                   |
| J4  | PO In R    | Pocket Operator right audio in                  |
| J6  | PO CLK     | Converted 2 PPQN sync out to the PO             |
| J7  | Reset      | Divider reset (drives /CLR1 low)                |
| J9  | B+         | +3.3 V out to the PO's battery + terminal       |
| J10 | GND        | Ground test point                               |
| SCREW1–4 | —     | M3 mounting holes, tied to GND                  |
| SCREW5–7 | —     | 3.5 mm socket holes, tied to GND                |

---

## Bill of materials

| Reference | Value       | Package                     | Notes                              |
|-----------|-------------|-----------------------------|------------------------------------|
| U1        | LM1117MP-3.3| SOT-223                     | +12 V → +3.3 V LDO                 |
| U2        | 74AHC1G14SE-7 | SOT-353 (SC-70-5)         | Schmitt inverter                   |
| U3        | SN74HC74DR  | SOIC-14                     | Dual D flip-flop (÷2)              |
| U4        | TL072CPS    | SOIC-8W                     | Dual op-amp (Level converter)      |
| Q1        | MMBT3904LT1G (onsemi) | SOT-23            | NPN input transistor — **specify full MPN** |
| D1, D2    | 1N5819WS    | SOD-123F                    | Schottky, reverse-polarity protect |
| D3        | BAS316Z     | SOD-323                     | Q1 base clamp, negative inputs     |
| R1        | 10 kΩ       | 0805                        | Base series resistor               |
| R2        | 10 kΩ       | 0805                        | Collector pull-up                  |
| R3        | 10 kΩ       | 0805                        | RC filter                          |
| R4        | 1 kΩ        | 0805                        | Sync output series resistor        |
| R5        | 10 kΩ       | 0805                        | /CLR1 pull-up (reset)              |
| R9, R12   | 49.9 kΩ     | 0805                        | Buffer input resistors             |
| R10, R13  | 220 kΩ      | 0805                        | Buffer feedback resistors (×4.41)  |
| R11, R14  | 100 Ω       | 0805                        | Buffer output series resistors     |
| R15       | 10 kΩ       | 0805                        | Q1 base–emitter pulldown           |
| C1        | 22 µF       | CP_Elec_5x5.3               | +12 V bulk                         |
| C2        | 100 µF      | CP_Elec_6.3x5.9             | +3.3 V bulk                        |
| C3        | 1 nF        | 0805                        | RC filter                          |
| C4, C5    | 100 nF      | 0805                        | +3.3 V decoupling (U2/U3)          |
| C6        | 100 nF      | 0805                        | +12 V decoupling                   |
| C7        | 100 nF      | 0805                        | −12 V decoupling                   |
| C8, C9    | 18 pF       | 0805                        | Buffer feedback caps (~40 kHz pole)|
| C10, C11  | 10 µF       | CP_Elec_5x5.3               | ±12 V decoupling                   |
| J1        | Europower   | IDC 2×05                    | Eurorack ±12 V / GND bus header    |
| H1        | —           | 4-pad row                   | Audio out L/R + GND                |
| H2        | —           | 2-pin header 2.54 mm        | Clock in + GND                     |
| J2, J4, J6, J7, J9, J10 | — | TestPoint 4.0×2.0 mm     | See connector table above          |

---

## Design notes

- The divider assumes the Eurorack source is 4 PPQN, matching the common default for many
  sequencers and clock generators. Pocket Operators expect 2 PPQN sync, making a
  divide-by-two stage appropriate.
- The transistor input stage tolerates a wide range of Eurorack trigger voltages without
  exposing the logic ICs to more than 3.3 V.
- The Schmitt trigger ensures reliable operation with slow edges or noisy clocks.
- The 74HC74 is configured as a T flip-flop (D = /Q), yielding a stable 50% duty-cycle
  output.
- Reference-designator note: on this board **U1 = regulator, U2 = Schmitt, U3 = flip-flop,
  U4 = op-amp** — earlier drafts of the clock-converter section numbered the Schmitt/flip-flop
  as U1/U2.
- If broader compatibility is desired, consider adding a jumper or switch to bypass the
  divider for Eurorack sources already configured to 2 PPQN. This allows selecting either
  a direct 1:1 clock or the default ÷2 output.
