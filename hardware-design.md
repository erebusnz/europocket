# europocket — Hardware Design

Interface between the **Eurorack** and **Pocket Operator (PO)** domains. The two signal
domains differ in both timing (clock rate) and electrical levels, so the design breaks
into three functional blocks: **Power**, **Clock converter**, and **Level converter**.

> Reference designators and values in this document are reconciled against
> `Netlist_europocket_2026-07-18.tel`.

---

## Power

Powered from the **Eurorack bus** via header **J3** ("Europower", 2×05 IDC). The board
takes **±12 V** from the bus and locally regulates **+3.3 V** for the CMOS logic. The
TL072 in the Level converter runs directly from **±12 V**.

**Rails**

| Rail    | Source                          | Consumers                          |
|---------|---------------------------------|------------------------------------|
| +12 V   | J3 pins 9–10, via D1            | LM1117 input, TL072 V+ (U4.8)      |
| −12 V   | J3 pins 1–2, via D2             | TL072 V− (U4.4)                    |
| +3.3 V  | LM1117 (U1) output             | U2, U3 logic, pull-ups, J6 test pt |
| GND     | J3 pins 3–8                     | all blocks                         |

**Components**

- `U1` — LM1117MP-3.3 (SOT-223) LDO: **+12 V → +3.3 V** (Vin = U1.3, Vout = U1.2, GND = U1.1).
- `D1`, `D2` — 1N5819WS Schottky, reverse-polarity / mis-insertion protection on the +12 V and −12 V rails.
- Bulk: `C1` 22 µF (+12 V), `C2` 100 µF (+3.3 V), `C10` 10 µF (+12 V), `C11` 10 µF (−12 V).
- Decoupling (100 nF, one per rail/IC, adjacent to the pin): `C4`/`C5` on +3.3 V (U2/U3), `C6` on +12 V, `C7` on −12 V.

---

## Clock converter

### Eurorack Trigger → Pocket Operator Sync Converter

**Purpose.** Convert a Eurorack clock/trigger into a Pocket Operator compatible sync
signal. The design assumes a **4 PPQN** Eurorack clock and produces a **2 PPQN** Pocket
Operator sync clock by dividing the clock by two.

**Requirements**

- Accept Eurorack triggers up to approximately **10–12 V**.
- Protect the logic from overvoltage.
- Reject narrow glitches.
- Produce clean **3.3 V CMOS** logic.
- Divide clock by two.
- Provide deterministic reset.

#### Functional block diagram

```
Eurorack Trigger
        │
        ▼
Input Protection / Level Shift   (Q1)
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
Pocket Operator Sync Output
```

#### Stage 1 — Input conditioning

**Test point J4 ("Trigger")**

- J4.1: Eurorack Trigger In
- J4.2: Pocket Operator Sync Out

**Components**

- `Q1` — MMBT3904 (SOT-23) NPN
- `R1` — 10 kΩ (base)
- `R2` — 10 kΩ pull-up

**Wiring**

```
Trigger input (J4.1) → R1 → Q1 base
Q1 emitter           → GND
Q1 collector         → R2 pull-up to 3.3 V   (collector = logic signal)
```

**Behaviour**

| Input | Collector |
|-------|-----------|
| Low   | High      |
| High  | Low       |

The transistor therefore **level shifts**, **protects** the logic, and **inverts**.

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

**Device:** `U2` — 74AHC1G14 (single inverting Schmitt trigger, SOT-23-5).

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

External reset via test point **J5 ("Reset")**, asynchronously clearing the first flip-flop.

- `/CLR1` (U3.1) — J5 with 10 kΩ pull-up `R5` to 3.3 V; drive low to reset.
- `/PRE1` (U3.4) — held high via 10 kΩ pull-up `R6`.

**Unused second flip-flop**

- `/PRE2` (U3.10) — 10 kΩ pull-up `R8` to 3.3 V.
- `/CLR2` (U3.13) — 10 kΩ pull-up `R7` to 3.3 V.

> ⚠️ **Review before fab:** in the current netlist the second flip-flop's **2CLK (U3.11)**
> and **2D (U3.12)** are **not connected** — they appear in no net and would float. A
> floating HC clock input can self-oscillate and raise supply current. Recommend tying
> **2CLK → GND** and **2D → GND** (2Q/2Q̄ may be left open).

#### Output

`1Q` (U3.5) → **1 kΩ series resistor `R4`** → **J4.2**.

- Purpose of resistor: short-circuit protection, cable protection, output current limiting.
- Output amplitude: **0–3.3 V**.

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
**±12 V**, brought out to test points **J1 ("Buffer 1")** and **J2 ("Buffer 2")**. Each
channel is an inverting stage with the non-inverting input grounded.

**Per-channel topology** (values below: channel A / channel B)

```
In (J1.1 / J2.1) → R_in ──┬── (−) op-amp ── out ── R_series ── Out (J1.2 / J2.2)
                          │                    │
                          └──── R_fb ──────────┤
                          └──── C_fb ──────────┘
                 (+) op-amp → GND
```

| Element        | Channel A (Buffer 1) | Channel B (Buffer 2) | As-built | Revised (×5) |
|----------------|----------------------|----------------------|----------|--------------|
| Op-amp         | U4 A (in −U4.2, out U4.1) | U4 B (in −U4.6, out U4.7) | — | — |
| Input resistor | R9                   | R12                  | 49.9 kΩ  | 49.9 kΩ (unchanged) |
| Feedback       | R10                  | R13                  | 100 kΩ   | **249 kΩ** |
| Feedback cap   | C8                   | C9                   | 18 pF    | **6.8 pF** |
| Output series  | R11                  | R14                  | 100 Ω    | 100 Ω (unchanged) |

- **Direction:** Pocket Operator output → Eurorack (boost line level up to modular level).
- **Gain (as-built):** −R_fb / R_in = −100 k / 49.9 k ≈ **−2.0** — too low; only reaches
  ~4.4 Vpp from the measured PO output.
- **Gain (revised):** −249 k / 49.9 k ≈ **−5.0**, taking the measured **2.2 Vpp** PO output
  (loudest volume, near-square chip voice) to ~11 Vpp — standard Eurorack modular level,
  within the TL072's ~21 Vpp swing on ±12 V.
- **Feedback cap** sets a high-frequency roll-off pole at `1/(2π·R_fb·C)`. It must shrink
  with the larger R_fb (18 pF → **6.8 pF**) to keep the pole above ~30 kHz — the PO voice is
  a square wave, so its harmonics must pass or the edges round off and the tone dulls.
- Powered from the ±12 V rails, so output can swing well beyond the 3.3 V logic domain.

> The revised R10/R13/C8/C9 values are a **proposed change**, not the current board — see
> [fixes.md](fixes.md) §2. The BOM below lists the as-built (×2) values.

---

## Bill of materials

| Reference | Value       | Package                     | Notes                              |
|-----------|-------------|-----------------------------|------------------------------------|
| U1        | LM1117MP-3.3| SOT-223                     | +12 V → +3.3 V LDO                 |
| U2        | 74AHC1G14   | SOT-23-5                    | Schmitt inverter                   |
| U3        | SN74HC74DR  | SOIC-14                     | Dual D flip-flop (÷2)              |
| U4        | TL072CPS    | SOIC-8W                     | Dual op-amp (Level converter)      |
| Q1        | MMBT3904    | SOT-23                      | NPN input transistor               |
| D1, D2    | 1N5819WS    | SOD-123F                    | Schottky, reverse-polarity protect |
| R1        | 10 kΩ       | 0805                        | Base resistor                      |
| R2        | 10 kΩ       | 0805                        | Collector pull-up                  |
| R3        | 10 kΩ       | 0805                        | RC filter                          |
| R4        | 1 kΩ        | 0805                        | Output series resistor             |
| R5        | 10 kΩ       | 0805                        | /CLR1 pull-up (reset)              |
| R6        | 10 kΩ       | 0805                        | /PRE1 pull-up                      |
| R7        | 10 kΩ       | 0805                        | /CLR2 pull-up (unused FF)          |
| R8        | 10 kΩ       | 0805                        | /PRE2 pull-up (unused FF)          |
| R9, R12   | 49.9 kΩ     | 0805                        | Buffer input resistors             |
| R10, R13  | 100 kΩ      | 0805                        | Buffer feedback resistors (→ 249 kΩ, fixes.md §2) |
| R11, R14  | 100 Ω       | 0805                        | Buffer output series resistors     |
| C1        | 22 µF       | CP_Elec_5x5.3               | +12 V bulk                         |
| C2        | 100 µF      | CP_Elec_6.3x5.9             | +3.3 V bulk                        |
| C3        | 1 nF        | 0603                        | RC filter                          |
| C4, C5    | 100 nF      | 0603                        | +3.3 V decoupling (U2/U3)          |
| C6        | 100 nF      | 0603                        | +12 V decoupling                   |
| C7        | 100 nF      | 0603                        | −12 V decoupling                   |
| C8, C9    | 18 pF       | 0603                        | Buffer feedback caps (→ 6.8 pF, fixes.md §2) |
| C10, C11  | 10 µF       | CP_Elec_5x5.3               | ±12 V decoupling                   |
| J1        | Buffer 1    | TestPoint ×2                | Level-converter channel A          |
| J2        | Buffer 2    | TestPoint ×2                | Level-converter channel B          |
| J3        | Europower   | IDC 2×05                    | Eurorack ±12 V / GND bus header    |
| J4        | Trigger     | TestPoint ×2                | Trigger In (J4.1) / Sync Out (J4.2)|
| J5        | Reset       | TestPoint                   | Reset input                        |
| J6        | 3.3 V       | TestPoint                   | Rail test point                    |
| J7, J8    | GND         | TestPoint                   | Ground test points                 |

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
