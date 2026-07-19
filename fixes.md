# europocket — Fixes

Outstanding fixes to apply before fabrication. Cross-referenced against
`Netlist_europocket_2026-07-18.tel` and [hardware-design.md](hardware-design.md).

---

## 1. Floating inputs on the unused SN74HC74DR half (U3)

**Problem.** Only the first flip-flop of the dual `U3` (SN74HC74DR) is used. Its second
half has inputs left floating in the current netlist:

| Pin      | Function | Netlist status                    |
|----------|----------|-----------------------------------|
| U3.11    | 2CLK     | **floating** — in no net          |
| U3.12    | 2D       | **floating** — in no net          |
| U3.10    | /PRE2    | OK — 10 kΩ pull-up `R8` to 3.3 V  |
| U3.13    | /CLR2    | OK — 10 kΩ pull-up `R7` to 3.3 V  |

A floating HC clock input can self-oscillate and draw excess supply current; floating
CMOS inputs violate the datasheet and can latch at indeterminate levels.

**Pinout** (SN74HC74DR, SOIC-14 — pin 1 at the dot/notch, count down the left, up the right):

```
                    ┌───────\/───────┐
     /1CLR   1 ─────┤ ●            14 ├───── VCC
        1D   2 ─────┤              13 ├───── /2CLR   ← R7 pull-up ✓
      1CLK   3 ─────┤   SN74HC74   12 ├───── 2D      ◄══ TIE TO GND
     /1PRE   4 ─────┤     (U3)     11 ├───── 2CLK    ◄══ TIE TO GND
        1Q   5 ─────┤              10 ├───── /2PRE   ← R8 pull-up ✓
       1QN   6 ─────┤               9 ├───── 2Q      (leave open)
       GND   7 ─────┤               8 ├───── 2QN     (leave open)
                    └────────────────┘
        │                            │
   chip GND (✓)              1QN = 1Q̄ (inverted output)
```

**Fix.** Tie both unused inputs to a defined level — connect them to the `power_GND` net
(same net as pin 7):

- **2CLK — pin 11 → GND**
- **2D   — pin 12 → GND**

Outputs 2Q (pin 9) and 2Q̄ (pin 8) may be left open. No new components required — just add
the two net connections to GND (`power_GND`).

---

## 2. Gain for Pocket Operator → Eurorack line level

**Problem.** The Level-converter buffers (`U4` TL072, channels A/B) are currently set for
an inverting gain of only **×2**:

```
Gain = −R_fb / R_in = −100 kΩ / 49.9 kΩ ≈ −2.0
```

That lifts the PO output to only ~4.4 Vpp — well short of Eurorack modular level
(~10 Vpp, ±5 V).

**Measured PO output** (DS1104Z, 10× probe, AC coupled, PO on battery + board off the
rack to break the ground loop; steady note at **loudest volume**):

| Metric | Value |
|--------|-------|
| Waveform | trapezoidal / slew-limited square (chip voice) |
| VPP | **2.2 V** (Max +1.08 V / Min −1.12 V) |
| VRMS | 0.90 V |
| Frequency | ~880 Hz (steady) |

**Target gain** — to reach ~10 Vpp modular level from the measured 2.2 Vpp max:

| Basis | Gain | Result |
|-------|------|--------|
| ~10 Vpp exactly | **×4.5** | 9.9 Vpp |
| headroom for below-max volume | **×5** | 11 Vpp |

**Recommended: ×5.** 2.2 Vpp is at *loudest*; ×5 lands near modular level at normal
volume and still clears the TL072's swing limit. (Old estimate of ×5 confirmed by
measurement.)

**Fix — feedback resistors** (keep `R_in` = 49.9 kΩ = R9 / R12):

| Reference | Now    | ×4.5 (~10 Vpp) | ×5 (~11 Vpp) |
|-----------|--------|----------------|--------------|
| R10 (Buffer 1 fb) | 100 kΩ | 226 kΩ | **249 kΩ** |
| R13 (Buffer 2 fb) | 100 kΩ | 226 kΩ | **249 kΩ** |

(Nearest E96 values; 226 kΩ and 249 kΩ both exist.)

**Fix — feedback caps (required alongside).** `C8 / C9` = 18 pF set the roll-off pole at
`1/(2π·R_fb·C)`. Raising R_fb without shrinking the cap pushes the pole down toward the
audio band. Keep `R_fb × C_fb` roughly constant:

| Gain | R_fb   | C8 / C9 (≈) |
|------|--------|-------------|
| ×4.5 | 226 kΩ | ~8.2 pF     |
| ×5   | 249 kΩ | ~6.8 pF     |

**Headroom check.** TL072 on ±12 V swings ~±10.5 V (~21 Vpp), so an 11 Vpp output at ×5
has adequate margin.

**Note.** The PO voice is a near-square wave, so its harmonics extend well above the
fundamental — keep the feedback-cap roll-off above ~30 kHz (the values above do) to avoid
rounding the edges and dulling the tone.

---

## 3. Wrong part fitted at Q1 — entire assembled batch (PCBWay claim)

**Problem.** All assembled boards (PCBWay order **T-Y6W679717A**, 5 sets, 2023-11-06 BOM)
have a non-functional part at Q1. The BOM line was quoted as generic "MMBT3904"; the part
placed is marked **"1E"** (large) + **"A3"** (rotated) — not an MMBT3904 marking (onsemi
MMBT3904 = "1A"/"1AM"). No substitution was noted in the PCBWay quote columns.

**Evidence** (DS1104Z + DMM, two boards tested, identical failure on both):

| Test | Result | A real 3904 would give |
|------|--------|------------------------|
| In-circuit: 0.39 mA base drive (4.6 V clock via R1) | base clamps at 0.62–0.68 V ✓ | same |
| In-circuit: collector (10 kΩ pull-up, needs 0.33 mA) | **never pulls low — stuck at 3.3 V** | saturates to ~0.1 V (forced β ≈ 0.85 vs hFE ≥ 100) |
| Diode test on legs: 1→2 | 0.68 V | ~0.65 V ✓ |
| Diode test on legs: 1→3 | 0.67 V | ~0.65 V ✓ |
| Diode test on legs: 2↔3 | open (2.15 V reading = in-circuit sneak path via U2 ESD diode + R3) | open ✓ |

Junctions present, **zero current gain** → the device behaves as a common-anode dual
diode (BAW56-class), not a transistor. Diode-mode testing cannot distinguish the two —
only transistor action (which the circuit itself tests) can.

**Verification.** Replacing Q1 with a genuine onsemi MMBT3904 (marked "1AM") restored
correct switching immediately (together with fix §5 below). Divider confirmed working:
5.556 Hz in → 2.747 Hz out (÷2), 0–3.3 V, ~50 % duty.

**Fix.**
- Rework Q1 on all remaining boards with a genuine MMBT3904.
- File a claim with PCBWay: wrong component placed vs the quoted BOM line, batch-wide.
  Evidence: scope captures, diode-test table above, part-marking photos ("1E"/"A3").
- See §4 to prevent recurrence.

---

## 4. BOM: fully-specified orderable MPN for Q1

**Problem.** The BOM's Q1 MPN was the generic string "MMBT3904", which allowed the
assembler's supply chain to substitute silently (see §3).

**Fix.** Specify a full orderable part number and manufacturer in the BOM:

| Reference | Was | Change to |
|-----------|-----|-----------|
| Q1 | `MMBT3904` (generic) | **`MMBT3904LT1G`** (onsemi) — or `PMBT3904,215` (Nexperia) |

---

## 5. Input-stage hardening (Q1 bias + protection)

**Problem A — input threshold too low (verified on hardware).** The trigger input
switches at a single V_BE (~0.65 V). The measured Eurorack clock source idles **low at
~0.68 V** (not 0 V; grounds verified bonded, offset 0.001 V) — above the threshold — so a
working Q1 never turns off and the collector sticks low. Masked previously by the §3 dud
part.

**Fix A (proven by bodge on board 1): add a 10 kΩ base–emitter pulldown** (`R15`,
base → GND, fits across SOT-23 pins 1–2):

| Input | Base voltage (with 10 k pulldown) | Q1 |
|-------|------------------------------------|----|
| Low 0.68 V | ÷2 → 0.34 V | off ✓ |
| High 4.7 V | Thevenin 2.34 V / 5 k → I_B ≈ 0.33 mA | saturated ✓ |

Effective threshold rises from ~0.65 V to ~1.3 V. Bonus: the base no longer floats
through R1 into an unpatched jack.

**Problem B — no protection for negative inputs.** Bipolar Eurorack signals (±5 V/±8 V
LFO or audio used as clock) reverse-bias Q1's B–E junction; V_EBO(max) = 6 V, and repeated
B–E avalanche permanently degrades hFE.

**Fix B: add a 1N4148/BAS316 clamp diode** (`D3`), anode → GND, cathode → Q1 base:
clamps negative inputs at −0.6 V. R1 (10 k) limits clamp current to safe levels for
inputs down to at least −12 V.

---

## 6. BOM: wrong MPN for the 100 nF decoupling caps C4–C7

**Problem.** The source BOM lists `C0603C103K4REC7867` for C4/C5/C6/C7 — the **103** code
is **10 nF**, not the intended 100 nF. PCBWay caught this at assembly (note dated
2023-11-20: "C4,C5,C6,C7 补料 C0603C104K4REC7867") and substituted the correct part, but
the error is still latent in the project BOM and will recur on any re-order.

**Fix.**

| Reference | Was (10 nF!) | Change to (100 nF) |
|-----------|--------------|---------------------|
| C4, C5, C6, C7 | `C0603C103K4REC7867` | **`C0603C104K4REC7867`** |
