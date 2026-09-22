# EC2201 Digital Systems: Adder-Based Resource Allocation Engine & Virtual Logic Analyzer

**Department of Electrical & Computer Engineering**  
**Laboratory Project Specification & Software Verification Suite**  
**Author / Lab Team**: ECE Digital Systems Design Group  
**Target Environment**: 100% Client-Side Single Page Application (`index.html`)

---

## 1. Engineering Motivation & System Architecture

In modern multicore processors and embedded system-on-chip (SoC) micro-architectures, resource allocation schedulers continuously accumulate discrete load quanta (such as compute slice reservations, memory buffer allocations, and DMA descriptor queues).

Rather than handling binary accumulation at an abstract software level (e.g., standard ALU addition instructions), this laboratory software suite models the exact **gate-level combinational arithmetic pipeline** constructed from silicon-verified macrocells:

```
                  ┌─────────────────────────────────────────────────────────┐
                  │                 Operand Register File                   │
                  │  A[N-1:0] (Task Load 1)      B[N-1:0] (Task Load 2)    │
                  └───────────────┬─────────────────────────┬───────────────┘
                                  │                         │
                                  ▼                         ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                           N-Bit Ripple Carry Adder Pipeline                               │
│                                                                                           │
│   C_in=0 ──►┌─────────────┐──► C[1] ──►┌─────────────┐──► ... ──►┌─────────────┐──► C_out  │
│             │ Full Adder 0│            │ Full Adder 1│           │Full Adder N-1│ (Overflow)│
│             └──────┬──────┘            └──────┬──────┘           └──────┬──────┘          │
│                    │                          │                         │                 │
│                    ▼                          ▼                         ▼                 │
│                   S[0]                       S[1]                     S[N-1]              │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                  ┌────────────────────────────────────────┐
                  │ Accumulator Result Bus & Logic Analyzer│
                  │ S[N-1:0] + Unsigned Carry Flag (CF)   │
                  └────────────────────────────────────────┘
```

---

## 2. Theoretical Background & Logic Synthesis

### 2.1 Half Adder (HA) Macrocell
Computes the binary sum of two single-bit inputs $A$ and $B$:
$$\text{Sum} = A \oplus B = A\bar{B} + \bar{A}B$$
$$\text{Carry} = A \cdot B$$
- **IC Implementation**: $\frac{1}{4}$ of **74HC86** (Quad 2-Input XOR) + $\frac{1}{4}$ of **74HC08** (Quad 2-Input AND).
- **Silicon Budget**: 14 MOSFET transistors (8T transmission-gate XOR + 6T CMOS NAND/Inverter).

### 2.2 Full Adder (FA) Macrocell
Accommodates incoming carry $C_{in}$ from adjacent low-order bit stages:
$$\text{Sum} = A \oplus B \oplus C_{in}$$
$$C_{out} = AB + BC_{in} + AC_{in} = (A \cdot B) + (C_{in} \cdot (A \oplus B))$$
Constructed by cascading two Half Adders with an OR gate for carry aggregation.

### 2.3 Karnaugh Map (K-Map) SOP Derivation
For $C_{out}(A, B, C_{in})$, minterm evaluation yields $\sum m(3, 5, 6, 7)$:
- **Minterm Set**:
  - $m_3 = \bar{A}B C_{in}$
  - $m_5 = A\bar{B} C_{in}$
  - $m_6 = AB \bar{C}_{in}$
  - $m_7 = ABC_{in}$
- **Adjacent Grouping on Gray-Code Grid**:
  1. Quad/Pair $\{m_3, m_7\} \implies B \cdot C_{in}$ (literal $A$ eliminates via $A + \bar{A} = 1$)
  2. Pair $\{m_5, m_7\} \implies A \cdot C_{in}$ (literal $B$ eliminates via $B + \bar{B} = 1$)
  3. Pair $\{m_6, m_7\} \implies A \cdot B$ (literal $C_{in}$ eliminates via $C_{in} + \bar{C}_{in} = 1$)
- **Minimal Sum-of-Products Form**:
  $$C_{out} = AB + BC_{in} + AC_{in}$$

---

## 3. Propagation Delay & Silicon Timing Model

The critical path through an $N$-bit Ripple Carry Adder is determined by the carry propagation delay:
$$t_{\text{total}} = (N - 1) \cdot t_{\text{carry\_stage}} + \max(t_{\text{carry\_stage}}, t_{\text{sum\_stage}})$$

### Technology Family Library Parameters
| Parameter | 74HC CMOS (5.0V) | 74LVC (3.3V) | Normalized Model ($\tau$) |
|---|---|---|---|
| $t_{pd}(\text{XOR})$ | $11.0\text{ ns}$ | $4.2\text{ ns}$ | $3\tau$ |
| $t_{pd}(\text{AND})$ | $9.0\text{ ns}$ | $3.6\text{ ns}$ | $2\tau$ |
| $t_{pd}(\text{OR})$ | $8.0\text{ ns}$ | $3.2\text{ ns}$ | $2\tau$ |
| **Carry Stage** ($t_{\text{AND}} + t_{\text{OR}}$) | **$17.0\text{ ns}$** | **$6.8\text{ ns}$** | **$4\tau$** |
| **4-Bit Total Delay** | **$68.0\text{ ns}$** | **$27.2\text{ ns}$** | **$16\tau$** |
| **8-Bit Total Delay** | **$136.0\text{ ns}$** | **$54.4\text{ ns}$** | **$32\tau$** |

---

## 4. Software Features & Instrumentation

1. **Virtual Logic Analyzer (Timing Oscilloscope)**:
   - Live multi-channel waveform visualization capturing: `CLK`, `A[Bus]`, `B[Bus]`, `C_IN`, `C_RIPPLE`, `SUM[Bus]`, and `C_OUT`.
   - Real-time representation of stage-by-stage delay and steady-state settling.
2. **Interactive Schematic Diagrams**:
   - ANSI / IEEE Std 91-1984 compliant logic gate outlines for XOR, AND, and OR gates.
   - Interactive logic toggle switches with high/low signal flow feedback.
3. **Karnaugh Map Synthesis Engine**:
   - 2-Variable (Half Adder) and 3-Variable (Full Adder) Gray-code matrix maps.
   - Dynamic grouping overlay loops and bidirectional crosshair highlighting with truth table minterms.
4. **Automated Verilog-Style Testbench**:
   - 16 test vectors (11 normal operational loads + 5 boundary/overflow assertions).
   - Real-time Verilog `$display` simulation stdout logging with timing timestamps.
   - CSV export and clipboard copy utilities.

---

## 5. Verification Testbench Vector Table

| Vector ID | Category | Operands ($A + B$) | Bit-Width | Expected Sum | Expected CF | Assertion Target |
|---|---|---|---|---|---|---|
| `VEC_01` | Normal | $0 + 0$ | 4-bit | $0$ | $0$ | Zero additive identity |
| `VEC_02` | Normal | $1 + 0$ | 4-bit | $1$ | $0$ | LSB single-bit assertion |
| `VEC_03` | Normal | $5 + 3$ | 4-bit | $8$ | $0$ | Nominal workload load |
| `VEC_04` | Normal | $7 + 8$ | 4-bit | $15$ | $0$ | Maximum 4-bit unsigned sum ($2^4 - 1$) |
| `VEC_05` | Normal | $10 + 5$ | 4-bit | $15$ | $0$ | Alternating binary pattern ($1010 + 0101$) |
| `VEC_06` | Normal | $9 + 6$ | 4-bit | $15$ | $0$ | Complementary dual carry generation |
| `VEC_07` | Normal | $2 + 2$ | 4-bit | $4$ | $0$ | Single-stage carry propagation |
| `VEC_08` | Normal | $14 + 1$ | 4-bit | $15$ | $0$ | Boundary minus one |
| `VEC_09` | Normal | $50 + 75$ | 8-bit | $125$ | $0$ | Standard 8-bit process scheduler sum |
| `VEC_10` | Normal | $128 + 64$ | 8-bit | $192$ | $0$ | MSB power-of-two accumulation |
| `VEC_11` | Normal | $100 + 155$ | 8-bit | $255$ | $0$ | Maximum 8-bit unsigned sum ($2^8 - 1$) |
| `VEC_12` | Fault/Overflow | $15 + 1$ | 4-bit | $0$ | $1$ | 4-bit ripple cascade with carry-out set |
| `VEC_13` | Fault/Overflow | $15 + 15$ | 4-bit | $14$ | $1$ | 4-bit all-1s saturation overflow |
| `VEC_14` | Fault/Overflow | $255 + 1$ | 8-bit | $0$ | $1$ | Full 8-stage ripple carry propagation |
| `VEC_15` | Fault/Overflow | $255 + 255$ | 8-bit | $254$ | $1$ | 8-bit all-1s maximum saturation |
| `VEC_16` | Fault/Edge | $0 + 255$ | 8-bit | $255$ | $0$ | Zero + maximum boundary |

---

## 6. Execution Instructions
Open [index.html](file:///c:/Users/Asus/Downloads/new%20dev/adder-resource-calculator/index.html) directly in any modern web browser:
```powershell
Start-Process "c:\Users\Asus\Downloads\new dev\adder-resource-calculator\index.html"
```
No package manager, compiler, or network connectivity is required.
