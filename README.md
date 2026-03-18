# CNN-FPGA

A Convolutional Neural Network inference accelerator implemented in SystemVerilog, extending the fixed-point fully-connected framework from [NeuralNet-for-FPGA](https://github.com/archieelth/FPGA_CNN). Trained on MNIST and simulated with Verilator.

---

## Overview

This project adds convolutional and pooling layers to the existing parameterisable FPGA neural network framework. The fully-connected layers (`layer.sv`, `neuron.sv`) are reused unchanged; the new modules handle spatial feature extraction before the FC stages.

```
28×28 input → [Conv Layer: N filters, 3×3] → [2×2 Max Pool] → [FC Layer 1] → [FC Layer 2] → argmax → digit
```

---

## Architecture

### Convolutional Layer

A 3×3 sliding window traverses the 28×28 input image with stride 1, producing a 26×26 feature map per filter (no padding). The window is implemented as a **line buffer**:

- Two full rows are buffered (2 × 28 registers) plus the incoming pixel
- Each new pixel completes a valid 3×3 window once two rows are loaded
- All filters receive the same window pixels in parallel each clock cycle
- A position counter tracks `(row, col)` and suppresses output at row boundaries

Each filter is a 9-weight neuron (dot product + ReLU), using the same MAC accumulator as the existing `neuron.sv`.

### Max Pooling

A 2×2 max-pool reduces each 26×26 feature map to 13×13, cutting data volume by 4× before the FC stages. Reuses the pixel streaming infrastructure with a modified stride counter.

### Fully-Connected Layers

The pooled, flattened feature maps feed directly into the existing `layer.sv` / `neuron.sv` modules, which are already parameterisable and require no structural changes.

---

## Fixed-Point Representation

Inherits the **Q2.13 signed 16-bit** scheme from the parent project:

| Field | Bits | Notes |
|---|---|---|
| Sign | 1 | |
| Integer | 2 | Range ≈ ±4 |
| Fractional | 13 | Precision ≈ 0.000122 |

MAC products accumulate in a 48-bit Q4.26 accumulator, right-shifted by 13 at the end to return to Q2.13. Convolutional kernel weights are exported in the same `.hex` format as FC weights.

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `NUM_FILTERS` | `8` | Number of 3×3 convolutional filters |
| `HIDDEN_NODES` | `64` | FC hidden layer neuron count |
| `DATAWIDTH` | `16` | Bit width of all data and weights |
| `NEURONNODES` | `10` | Output classes (digits 0–9) |

---

## Project Structure

```
rtl/              SystemVerilog source + Verilator C++ driver
models/           Trained weights — one subfolder per configuration
data/             MNIST CSV dataset
test_images/      Pre-generated test images
scripts/          Utility scripts (quantization, inference, etc.)
fixedtest.py      Train a model and export weights to models/
testbench.py      Run hardware simulation + Python comparison + plot
Makefile          Build system
COMMANDS.txt      Quick reference for all commands
```

---

## Usage

**Train a model**
```bash
python3 fixedtest.py
```

**Build the simulation**
```bash
make build
```

**Run the testbench**
```bash
python3 testbench.py
```

**Run a single image manually**
```bash
./obj_dir/Vnetwork +IMAGEFILE=test_images/image_0.hex +WEIGHTSDIR=models/cnn_8filter
```

**View waveforms**
```bash
gtkwave wave.vcd
```

See `COMMANDS.txt` for the full reference.

---

## Prerequisites

- [Verilator](https://verilator.org/) ≥ 4.0
- Python 3 with `numpy`, `matplotlib`
- (Optional) GTKWave for waveform viewing
