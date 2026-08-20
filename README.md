# Rail-Optimized Topology Generator

Generate a **rail-optimized GPU cluster network topology** — the fabric design used in NVIDIA DGX SuperPOD-style and Meta RoCE-style AI clusters — from three numbers: total GPU count, GPUs per node, and switch radix.

The tool outputs a Graphviz **topology diagram**, a full **port-mapping CSV** (every physical link, switch, and port), and a **JSON summary** of switch/link counts — everything a network engineer needs to rack, cable, and configure the fabric.

![Concept: traditional ToR vs rail-optimized](docs/concept_diagram.png)

---

## Table of Contents

- [Why rail-optimized?](#why-rail-optimized)
- [How it works](#how-it-works)
- [Installation](#installation)
- [Quick start](#quick-start)
- [CLI reference](#cli-reference)
- [Outputs](#outputs)
- [Examples](#examples)
- [Design assumptions & guardrails](#design-assumptions--guardrails)
- [Code architecture](#code-architecture)
- [Roadmap](#roadmap)
- [License](#license)

---

## Why rail-optimized?

In a **traditional** design, all of a node's GPU NICs plug into the same top-of-rack switch, and traffic from different GPUs mixes together on shared uplinks.

In a **rail-optimized** design, GPU *i* on every node is wired to a dedicated "rail *i*" switch. GPU 0 on node 5 and GPU 0 on node 200 sit on the same isolated, non-blocking fabric — while GPU 1 traffic never touches GPU 0's switches. Cross-GPU traffic within a single node is handled by NVLink/NVSwitch, not this network. This is exactly how NVIDIA DGX SuperPOD and large-scale RoCE/InfiniBand AI clusters (e.g. Meta's) are built, because it:

- gives every GPU pair a **fully non-blocking, congestion-isolated path** for collective communication (all-reduce, all-to-all),
- keeps the **blast radius of a switch failure contained to one rail**,
- and makes cabling and port-mapping **regular and predictable** at scale.

The diagram above shows a small 2-node / 4-GPU example: on the left, both ToR switches see a mix of all 4 GPUs' traffic; on the right, each color (GPU index) has its own dedicated switch end-to-end.

## How it works

```
gpu_count, gpus_per_node, switch_radix, oversubscription
                    │
                    ▼
      TopologyConfig  (validates inputs)
                    │
                    ▼
      build_topology()
        • split switch_radix into down-ports (node-facing) / up-ports (spine-facing)
        • leaves_per_rail = ceil(nodes / down_ports_per_leaf)
        • spines_per_rail = up_ports_per_leaf   (0 if a single leaf covers all nodes)
        • assign every GPU NIC → (leaf switch, port)
        • assign every leaf uplink → (spine switch, port)
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   *_port_mapping.csv   *_summary.json   *_diagram.(png|svg|pdf)
```

![Pipeline flow diagram](docs/flow_diagram.png)

Each of the `gpus_per_node` rails is built as its **own independent 2-tier fat tree** (leaf switches ↔ spine switches), sized from the switch radix and a configurable down:up port split (oversubscription ratio). If a rail's node count fits on a single leaf switch, the spine layer for that rail is skipped entirely. If the fabric would need a 3rd (super-spine) tier to fit within the given radix, the tool stops and tells you exactly why, instead of silently generating an under-provisioned design.

## Installation

Requires Python 3.8+, the [`graphviz`](https://pypi.org/project/graphviz/) Python package, and the Graphviz system binary (`dot`).

```bash
# system Graphviz binary
sudo apt-get install graphviz        # Debian/Ubuntu
brew install graphviz                # macOS

# python package
pip install graphviz
```

Clone or download `rail_topology.py` — it's a single, dependency-light script with no other requirements.

## Quick start

```bash
python3 rail_topology.py \
  --gpus 1024 \
  --gpus-per-node 8 \
  --radix 64 \
  --output-dir out \
  --basename cluster1
```

This produces `out/cluster1_diagram.png`, `out/cluster1_port_mapping.csv`, and `out/cluster1_summary.json` for a 1024-GPU cluster (128 nodes × 8 GPUs) built from 64-port switches.

## CLI reference

| Flag | Required | Default | Description |
|---|---|---|---|
| `--gpus` | ✅ | — | Total GPU count in the cluster |
| `--gpus-per-node` | ✅ | — | GPUs per compute node (this also sets the number of rails) |
| `--radix` | ✅ | — | Switch port count (radix) for leaf & spine switches |
| `--oversubscription` | | `1.0` | Down:up port ratio per leaf switch. `1.0` = non-blocking; `2.0` = 2:1 oversubscribed (fewer uplinks, more nodes per leaf) |
| `--output-dir` | | `.` | Directory to write all output files to |
| `--basename` | | `rail_topology` | Filename prefix for all outputs |
| `--format` | | `png` | Diagram format: `png`, `svg`, or `pdf` |
| `--no-diagram` | | off | Skip diagram rendering and only produce the CSV + JSON |

`--gpus` must be evenly divisible by `--gpus-per-node`.

## Outputs

**1. Topology diagram** (`*_diagram.png/svg/pdf`)
A Graphviz rendering of the fabric, color-coded per rail. For large clusters, repeated nodes/leaves/spines/rails are truncated with an ellipsis label (e.g. `... +120 more nodes`) so the image stays legible — the CSV always contains the complete, untruncated mapping.

**2. Port-mapping CSV** (`*_port_mapping.csv`)
Every physical link in the fabric, one row each:

| link_type | endpoint_a | endpoint_a_port | endpoint_b | endpoint_b_port |
|---|---|---|---|---|
| node-leaf | Node0-GPU0-NIC | - | R0-L0 | 0 |
| node-leaf | Node1-GPU0-NIC | - | R0-L0 | 1 |
| leaf-spine | R0-L0 | 32 | R0-S0 | 0 |

**3. Summary JSON** (`*_summary.json`)
Derived quantities and totals — node/rail counts, leaves/spines per rail, total switch and link counts, and any design warnings (e.g. the effective oversubscription ratio).

## Examples

### 1024 GPUs · 8 GPUs/node · 64-port switches (non-blocking)

128 nodes, 8 rails, 4 leaves + 32 spines *per rail* → 288 switches, 2048 links total.

![1024 GPU example diagram](examples/example_1024gpu_diagram.png)

```bash
python3 rail_topology.py --gpus 1024 --gpus-per-node 8 --radix 64 --output-dir examples --basename example_1024gpu
```

### 64 GPUs · 8 GPUs/node · 32-port switches (single leaf per rail, no spine needed)

8 nodes fit entirely on one leaf switch per rail, so no spine layer is generated for this size.

![64 GPU example diagram](examples/example_64gpu_single_leaf_diagram.png)

```bash
python3 rail_topology.py --gpus 64 --gpus-per-node 8 --radix 32 --output-dir examples --basename example_64gpu_single_leaf
```

### Guardrail: fabric too large for a 2-tier design

```bash
python3 rail_topology.py --gpus 100000 --gpus-per-node 8 --radix 32
# Error: leaves_per_rail (782) exceeds spine switch radix (32): a 3-tier
# (super-spine) fabric would be required. Increase --radix or reduce
# --gpus / increase --gpus-per-node.
```

## Design assumptions & guardrails

- **Rail isolation**: GPU *i* on every node connects only to rail-*i* switches; rails never cross-connect within this fabric.
- **Per-rail 2-tier fat tree**: leaf switches face nodes (down-ports) and spines (up-ports); each spine connects to every leaf in its rail exactly once (classic non-blocking fat-tree wiring).
- **Port split**: controlled by `--oversubscription` (down:up ratio), default 1:1.
- **Single-leaf shortcut**: if a rail's nodes fit on one leaf switch, no spine layer is built for that rail — all ports go to nodes.
- **3-tier detection**: if `leaves_per_rail` would exceed the spine radix, the tool raises a clear error rather than generating a fabric that can't actually be wired — this design intentionally stops at 2 tiers per rail.

## Code architecture

![Code architecture diagram](docs/architecture_diagram.png)

- `TopologyConfig` — validates and holds the four input parameters.
- `build_topology(cfg)` — the core algorithm: computes port splits, leaf/spine counts, the full node→leaf and leaf→spine port assignment, and enforces the 2-tier guardrail.
- `Topology` — the resulting data object (per-link mappings + derived counts), consumed by all three exporters.
- `write_port_mapping_csv`, `write_summary_json`, `render_diagram` — independent exporters, so you can script against `build_topology()` directly and skip any output you don't need.
- `main()` — thin argparse CLI wrapper around the above.

## Roadmap

- [ ] Optional 3-tier (super-spine) support for clusters beyond a single spine radix
- [ ] Cable/BOM export (cable lengths by rack position)
- [ ] Shared (non-dedicated) spine mode across rails, for switch-count-constrained designs
- [ ] JSON/YAML topology import for round-tripping existing designs

## License

MIT — use, modify, and adapt freely for your own cluster designs.
