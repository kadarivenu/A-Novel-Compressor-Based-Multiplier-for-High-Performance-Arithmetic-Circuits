# 🔢 A Novel Compressor-Based Multiplier for High-Performance Arithmetic Circuits

<div align="center">

![VLSI Design](https://img.shields.io/badge/Domain-VLSI%20Design-blue?style=for-the-badge)
![HDL](https://img.shields.io/badge/Language-Verilog%20HDL-green?style=for-the-badge)
![Tool](https://img.shields.io/badge/Simulation-Cadence%20Virtuoso-orange?style=for-the-badge)
![Tool](https://img.shields.io/badge/Synthesis-Xilinx%20Vivado-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
</div>
## 📋 Table of Contents

- [Abstract](#-abstract)
- [Project Overview](#-project-overview)
- [Background & Motivation](#-background--motivation)
- [Architecture](#-architecture)
  - [Half Adder](#half-adder)
  - [Full Adder Variants](#full-adder-variants)
  - [4:2 Compressor](#42-compressor)
  - [Wallace Tree Multiplier](#wallace-tree-multiplier)
- [Design Methodology](#-design-methodology)
- [Implementation Details](#-implementation-details)
- [Results & Performance](#-results--performance)
- [Image Processing Application](#-image-processing-application)
- [Applications](#-applications)
- [Tools Used](#-tools-used)
- [Repository Structure](#-repository-structure)
- [How to Run](#-how-to-run)
- [Future Scope](#-future-scope)
- [References](#-references)

---

## 📄 Abstract

This project presents the design and implementation of a **4:2 compressor-based 8×8 Wallace Tree Multiplier** targeting high-performance, low-power arithmetic circuits. The core contribution is a **Pass Transistor Logic (PTL) full adder** with only 15 transistors integrated as the compression element, achieving a **70% reduction in power consumption** compared to the conventional 28-transistor design. The architecture is verified using a Verilog HDL testbench in **Xilinx Vivado** and the circuit-level schematic is drawn in **Cadence Virtuoso**. The design is evaluated for its application in **image processing**, demonstrating faster pixel-level computation with reduced energy consumption.

---

## 🔍 Project Overview

Multiplication is one of the most computationally intensive operations in digital systems — used in processors, DSP units, image processing, and AI accelerators. A naive array multiplier adds partial products sequentially, resulting in **O(N) delay** that grows linearly with operand width.

This project implements a smarter approach:

```
8×8 Input Operands
       │
       ▼
Partial Product Generation (64 AND gates — Baugh-Wooley)
       │
       ▼
Wallace Tree Reduction (4:2 Compressors + Full Adders + Half Adders)
       │
       ▼
Two Final Rows (Sum Row + Carry Row)
       │
       ▼
Carry-Propagate Adder
       │
       ▼
16-bit Product Output
```

**Key Innovation:** Using **4:2 compressors** instead of simple full adders in the reduction tree. This cuts the number of reduction stages from O(log₁.₅ N) to O(log₂ N) — resulting in a significantly shorter critical path and faster multiplication.

---

## 🧠 Background & Motivation

### Why Not Array Multipliers?
- Delay grows **linearly** with bit width — O(N)
- Unsuitable for high-speed, wide-operand applications

### Why Not Booth Encoders Alone?
- Encoding/decoding logic adds overhead
- Carry propagation delay in final stages still significant

### The 4:2 Compressor Advantage

A 4:2 compressor reduces **5 inputs** (I1, I2, I3, I4, Cin) to **3 outputs** (Sum, Co1, Co2):

```
I1 + I2 + I3 + I4 + Cin = Sum + 2·Co1 + 2·Co2
```

**Critical property:** Co1 is **independent of Cin**. This breaks the horizontal carry chain across columns — the primary source of speed advantage over plain full adder trees.

---

## 🏗️ Architecture

### Half Adder

The simplest building block: two inputs A, B → Sum (XOR) and Carry (AND).

```
Sum   = A ⊕ B
Carry = A · B
```

| A | B | Sum | Carry |
|---|---|-----|-------|
| 0 | 0 |  0  |   0   |
| 0 | 1 |  1  |   0   |
| 1 | 0 |  1  |   0   |
| 1 | 1 |  0  |   1   |

---

### Full Adder Variants

Three full adder topologies were designed, simulated, and compared:

#### 1. Conventional 28T Full Adder (Baseline)
- Fully complementary static CMOS
- Independent PMOS pull-up / NMOS pull-down networks
- Full rail-to-rail swing, maximum noise margin
- **Disadvantage:** High transistor count → high capacitance, slower critical path

#### 2. 19T Full Adder (Transmission Gate Hybrid)
- Shares XOR/XNOR intermediate signals between Sum and Carry paths
- Transmission gate multiplexers reduce redundancy
- ~32% fewer transistors than 28T
- ~65.5% lower power consumption than 28T

#### 3. 15T PTL Full Adder *(Proposed — Best Performance)*
- Pass Transistor Logic (PTL) for XOR and multiplexer stages
- 6T PTL XOR network + 3T multiplexer for Carry
- Weak PMOS level restorers to counter threshold voltage drop
- **Lowest power: 0.609 µW — 70% reduction vs 28T**

```
Boolean Expressions:
  Sum  = A ⊕ B ⊕ Cin
  Cout = (A·B) + (B·Cin) + (A·Cin)
       = (A·B) + ((A ⊕ B)·Cin)    [generate-propagate form]
```

**Full Adder Truth Table:**

| A | B | Cin | Sum | Cout |
|---|---|-----|-----|------|
| 0 | 0 |  0  |  0  |  0   |
| 0 | 0 |  1  |  1  |  0   |
| 0 | 1 |  0  |  1  |  0   |
| 0 | 1 |  1  |  0  |  1   |
| 1 | 0 |  0  |  1  |  0   |
| 1 | 0 |  1  |  0  |  1   |
| 1 | 1 |  0  |  0  |  1   |
| 1 | 1 |  1  |  1  |  1   |

---

### 4:2 Compressor

Constructed by cascading **two full adders**:

```
Inputs : I1, I2, I3, I4, Cin
Outputs: Sum, Co1 (carry to next column), Co2 (carry to next column)

Stage 1 (FA1):
  S1  = I1 ⊕ I2 ⊕ I3
  Co1 = (I1·I2) + (I2·I3) + (I1·I3)   ← Independent of Cin!

Stage 2 (FA2):
  Sum = S1 ⊕ I4 ⊕ Cin
  Co2 = (S1·I4) + (I4·Cin) + (S1·Cin)

Weight preservation: I1+I2+I3+I4+Cin = Sum + 2·Co1 + 2·Co2  ✓
```

**Critical path:** Cin → FA2 → Sum (≈ 2×XOR delay). Co1 available earlier, suppressing column-to-column carry latency.

---

### Wallace Tree Multiplier

The 8×8 multiplier operates on two 8-bit signed operands using the **Baugh-Wooley algorithm** for two's complement multiplication, generating **64 partial products** across **15 bit columns**.

**Reduction hierarchy:**

```
Stage 0: 64 partial products (AND of A[i] × B[j]) in 8 rows
         ↓ 4:2 compressors applied to columns with height ≥ 5
Stage 1: Reduced partial products
         ↓ Mixed compressors, full adders, half adders
Stage 2: Further reduction
         ↓ Final compressors and adders
Stage N: Two rows remaining (Sum row K[0..14] + Carry row C[0..14])
         ↓ Carry-Propagate Adder
Output: 16-bit product
```

**Component breakdown in the 8×8 design:**

| Component | Count | Role |
|---|---|---|
| AND Gates | 64 | Partial product generation |
| Half Adders | ~14 | Columns with 2 bits |
| Full Adders | ~6 | Columns with 3 bits |
| 4:2 Compressors | ~21 | Columns with 4–8 bits |
| Inverters (for Baugh-Wooley) | 2 | Sign bit handling |

**Verification:** Simulation confirmed Y = 3074 for A = 58, B = 53 ✓

---

## 🔄 Design Methodology

The project follows a **strict bottom-up hierarchical approach**:

```
Step 1 — Define Specifications
         N=8 bit, 64 partial products, 8×15 array
         ↓
Step 2 — Design Half Adder
         XOR-AND cell, verify truth table, measure PDP
         ↓
Step 3 — Design Full Adder Variants (Parallel)
         28T / 19T / 15T — extract delay, power, area for each
         ↓
Step 4 — Build 4:2 Compressor
         Two FA cells, verify 32-row truth table, confirm Co1 is Cin-independent
         ↓
Step 5 — Reduction Stage 1
         4:2 compressors for columns h≥5; FAs for h=3–4; pass-through for h≤2
         ↓
Step 6 — Reduction Stage 2
         Re-apply compressors/FAs/HAs per column height; record heights
         ↓
Step 7 — Final Reduction Stage
         HAs and FAs column-by-column until all columns = exactly 2 rows (K0–K14)
         ↓
Step 8 — Simulate and Verify
         All 2^16 inputs (4×4) or randomized self-checking testbench (8×8); verify A×B
         ↓
Step 9 — Evaluate PDP / Area
         Measure delay, power, area per FA variant; compute PDP and PDAP for comparison
```

Each component is **individually verified** before instantiation at the next hierarchy level — errors caught early, sub-module behavior fully understood.

---

## 🔧 Implementation Details

### Gate-Level Primitives (Cadence Virtuoso — Transistor Level)

| Gate | Implementation | Notes |
|---|---|---|
| NOT | Single PMOS + NMOS inverter | Full swing |
| AND | NAND + Inverter | Rail-to-rail output |
| OR | NOR + Inverter | Rail-to-rail output |
| XOR (12T) | Complementary CMOS | Full swing, no Vt drop |
| XOR (6T) | Pass-transistor | Compact, Vt degradation possible |

### Verilog Modules (Xilinx Vivado)

```
wallace_tree_8x8/
├── half_adder.v          — HA module
├── full_adder_28t.v      — Conventional 28T FA
├── full_adder_19t.v      — Transmission gate 19T FA
├── full_adder_ptl.v      — Proposed 15T PTL FA  ← Best performance
├── compressor_4to2.v     — 4:2 compressor (2 × FA)
├── partial_product.v     — Baugh-Wooley PP generation
├── wallace_tree.v        — Reduction tree (top-level)
├── multiplier_8x8.v      — Complete 8×8 multiplier
└── tb_multiplier_8x8.v   — Self-checking testbench
```

### Testbench Strategy

```verilog
// Self-checking testbench snippet
initial begin
  for (i = 0; i < 256; i = i + 1) begin
    for (j = 0; j < 256; j = j + 1) begin
      A = i; B = j;
      #10;
      if (Y !== A * B)
        $display("FAIL: %0d x %0d = %0d (expected %0d)", A, B, Y, A*B);
    end
  end
  $display("All tests passed!");
end
```

---

## 📊 Results & Performance

### Power Consumption Comparison

| Full Adder Design | Transistor Count | Power (µW) | Reduction vs 28T |
|---|:---:|:---:|:---:|
| Conventional 28T | 28 | 2.030 | — |
| 19T (Transmission Gate) | 19 | 0.700 | **65.5%** |
| **Proposed 15T (PTL)** | **15** | **0.609** | **70.0%** |

```
Power Consumption (µW)
2.5 |
2.0 |  ████ 2.030 µW
    |  ████
1.5 |  ████
    |  ████
1.0 |  ████
    |  ████  ████ 0.700 µW
0.5 |  ████  ████              ████ 0.609 µW
0.0 +--28T---19T---------------15T-----------
```

### Simulation Verification

| Test | Input A | Input B | Expected Y | Simulated Y | Status |
|---|---|---|---|---|---|
| Spot check | 58 | 53 | 3074 | 3074 | ✅ PASS |
| Zero test | 0 | 255 | 0 | 0 | ✅ PASS |
| Max test | 255 | 255 | 65025 | 65025 | ✅ PASS |
| Random sweep | All 65536 combos | — | A×B | A×B | ✅ PASS |

### Critical Path Analysis

| Multiplier Type | Delay Complexity | Relative Speed |
|---|---|---|
| Array Multiplier | O(N) | Slow |
| Booth-Encoded | O(log N) + overhead | Moderate |
| Wallace (Full Adders) | O(log₁.₅ N) | Fast |
| **Proposed (4:2 Compressors)** | **O(log₂ N)** | **Fastest** |

---

## 🖼️ Image Processing Application

One of the demonstrated applications is **pixel-level image multiplication/blending** using the 8×8 Wallace Tree multiplier.

### How It Works

```
Image A (pixel values → 8-bit integers)
         +
Image B / Mask (pixel values → 8-bit integers)
         ↓
  8×8 Wallace Tree Multiplier (per pixel pair)
         ↓
Output pixel = (A_pixel × Mask_pixel) >> 8
         ↓
  Output Image (combined/blended result)
```

### Tested Images

- **Cameraman** standard test image (256×256 grayscale)
- **Lena** standard test image (512×512 grayscale)
- Gaussian radial mask applied via pixel multiplication

### Results

Blending variants tested with different inexact/exact compressor configurations (IAM00-VC8 through IAM11-VC8):

| Configuration | PSNR (dB) | SSIM |
|---|---|---|
| Exactly blended | ∞ (reference) | 1.0 |
| IAM00-VC8 | 36.484 | 0.76173 |
| IAM01-VC8 | 41.2401 | 0.77089 |
| IAM10-VC8 | 31.5684 | 0.7461 |
| IAM11-VC8 | 29.3684 | 0.73649 |

The exact compressor configuration preserves image quality while enabling real-time processing throughput.

---

## 💡 Applications

| Domain | Use Case |
|---|---|
| 🖼️ **Image Processing** | Convolution, filtering, blending, edge detection |
| 🎬 **Video Encoding** | H.264/H.265 DCT, motion estimation at high frame rates |
| 🧠 **AI / Neural Networks** | MAC units in CNN accelerators, edge inference |
| 📡 **DSP** | FIR/IIR filters, radar, audio processing |
| 💻 **Microprocessors** | ALU integer/floating-point execution units |
| 🤖 **Robotics** | Real-time kinematics, sensor fusion, path planning |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| **Cadence Virtuoso** | Transistor-level schematic design & SPICE simulation |
| **Xilinx Vivado** | Verilog HDL synthesis, behavioral simulation |
| **Verilog HDL** | RTL description of all modules |
| **MATLAB / Python** | Image processing testbench and output visualization |

---

## 📁 Repository Structure

```
compressor-based-multiplier/
│
├── 📂 verilog/
│   ├── half_adder.v
│   ├── full_adder_28t.v
│   ├── full_adder_19t.v
│   ├── full_adder_ptl.v          ← Proposed 15T design
│   ├── compressor_4to2.v
│   ├── partial_product_gen.v
│   ├── wallace_tree.v
│   ├── multiplier_8x8.v
│   └── tb_multiplier_8x8.v       ← Self-checking testbench
│
├── 📂 cadence/
│   ├── schematics/               ← Virtuoso schematic exports
│   │   ├── not_gate.png
│   │   ├── and_gate.png
│   │   ├── or_gate.png
│   │   ├── xor_gate_12t.png
│   │   ├── xor_gate_6t.png
│   │   ├── half_adder.png
│   │   ├── full_adder_28t.png
│   │   ├── full_adder_19t.png
│   │   ├── full_adder_ptl.png
│   │   ├── compressor_4to2.png
│   │   └── multiplier_8x8.png
│   └── simulations/              ← Cadence waveform screenshots
│
├── 📂 images/
│   ├── cameraman.txt             ← Pixel data for testbench
│   ├── lena.txt
│   └── results/
│       ├── blended_output.png
│       └── psnr_ssim_comparison.png
│
├── 📂 docs/
│   ├── project_report.pdf        ← Full IEEE-style report
│   ├── wallace_tree_diagram.png
│   ├── architecture_diagram.png
│   └── power_comparison_chart.png
│
└── README.md
```

---

## ▶️ How to Run

### Prerequisites

- Xilinx Vivado (2020.x or later) for Verilog simulation
- Cadence Virtuoso (IC617 or later) for transistor-level simulation
- Python 3.x with Pillow/NumPy (optional, for image testbench)

### Verilog Simulation (Xilinx Vivado)

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/compressor-based-multiplier.git
cd compressor-based-multiplier

# 2. Open Xilinx Vivado and create a new project
#    Add all files from /verilog/ as sources
#    Set tb_multiplier_8x8.v as the simulation top

# 3. Run behavioral simulation
#    Vivado → Flow → Run Simulation → Run Behavioral Simulation

# 4. Check the console for PASS/FAIL messages
```

### Running Only the Testbench (iverilog, open-source)

```bash
cd verilog/

# Compile all modules
iverilog -o sim_out \
  half_adder.v full_adder_ptl.v compressor_4to2.v \
  partial_product_gen.v wallace_tree.v multiplier_8x8.v \
  tb_multiplier_8x8.v

# Run simulation
vvp sim_out

# View waveforms (optional)
gtkwave dump.vcd
```

### Image Processing Demo (Python)

```bash
cd images/

# Install dependencies
pip install numpy pillow matplotlib

# Run the blending demo
python image_blend_demo.py \
  --imageA cameraman.png \
  --mask gaussian_mask.png \
  --output blended_result.png
```

---

## 🔮 Future Scope

- **Wider operands:** Scale to 16-bit, 32-bit, 64-bit for modern processor compatibility
- **Modified Booth Encoding (MBE):** Integrate into partial product generation to halve the number of partial products and further reduce delay
- **Higher-order compressors:** Explore 5:2 and 7:2 compressor architectures for even shallower trees
- **Physical layout:** Full-custom layout in Cadence, followed by DRC, LVS, and post-layout parasitic extraction for real-world performance characterization
- **FPGA evaluation:** Timing analysis and resource utilization benchmarking on Xilinx/Intel FPGAs
- **Low-power techniques:** Clock gating, operand isolation, and multi-threshold CMOS for further power reduction
- **AI accelerators:** Integration as the MAC unit in systolic array architectures for CNN inference at the edge
- **Approximate computing variant:** Explore controlled inexactness for error-tolerant applications (image filtering, ML inference) with greater power savings

---

## 📚 References

1. S. Swetha and A. Pranay Bhargav, "Design of low power 1-bit full adder for biomedical applications," *International Journal of Electronics Letters*, 2025. doi: 10.1080/21681724.2025.2559243
2. C. S. Wallace, "A suggestion for a fast multiplier," *IEEE Transactions on Electronic Computers*, vol. EC-13, no. 1, pp. 14–17, 1964.
3. Baugh and Wooley, "A two's complement parallel array multiplication algorithm," *IEEE Transactions on Computers*, 1973.
4. M. Shams and M. Bayoumi, "A novel high-performance CMOS 1-bit full-adder cell," *IEEE Transactions on Circuits and Systems II*, vol. 47, no. 5, 2000.
5. R. Zimmermann and W. Fichtner, "Low-power logic styles: CMOS versus pass-transistor logic," *IEEE Journal of Solid-State Circuits*, vol. 32, no. 7, 1997.
6. P. Bhattacharyya et al., "Performance analysis of a low-power high-speed hybrid 1-bit full adder circuit," *IEEE Transactions on VLSI Systems*, vol. 23, no. 10, 2015.
7. K. Navi et al., "A novel low-power full-adder cell for low voltage," *VLSI Design*, 2009.
</div>
