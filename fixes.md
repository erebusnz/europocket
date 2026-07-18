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
