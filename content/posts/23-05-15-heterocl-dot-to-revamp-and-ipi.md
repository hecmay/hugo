---
title: "Revamp intra-kernel data placement in HeteroCL"
date: 2023-05-07T15:05:55-04:00
draft: false
tags: ["systolic array", "mlir", "FPGA"]
---

## Revamp intra-kernel data placement in HeteroCL

TL;DR
- The dataflow dialect is no longer maintained. Instead, we will put dataflow optimizations and conditional data placement in HeteroCL dialect directly. 

- The host-accelerator and inter-kernel data placement is already supported in HeteroCL dialect. We need to revamp `.to()` for intra-kernel data placement, and extend it to support IP integration.

- For intra-kernel part, I am looking at the MLIR/Polygiest dialect; Polygiest can lower from MLIR-SCF to polyhedral representation in MLIR. We can then pass the polyhedral MLIR IR to Poly tools (e.g., `isl`, `OpenScop`) for polyhedral transformation and later use the IR for code generation. 

- For IP integration, we use `.to()` to connect IPs together. This can also be used to build a systolic array with a bottom-up by  connecting different IPs together.


### Systolic Array Generation MLIR/Polygiest dialect
- It's basically rewriting AutoSA using [MLIR/Polygiest dialect](https://acohen.gitlabpages.inria.fr/impact/impact2021/papers/IMPACT_2021_paper_1.pdf), but we are starting from something simple. More specifically, we are only generating the PEs for now, and do not touch the I/O network generation. 

- AutoSA's I/O network generation part is very complex, and it is particularly designed to interface with off-chip memory. We can probably rely on the IP integration feature to plug in manually-written custom IPs to handle the I/O network.

- Proposed workflow: if any kernel inside the input program is customized with `.to()` for intra-kernel data placement, then the compiler will extract that kernel and process it with Polygiest dialect along with `isl` (that performs polyheradrl transformation). The transformed IR is passed back as `mlir-scf` dialect, which is then hooked with rest of the program for HLS code generation.

```cpp
// hcl-dialect [ => mlir-polygiest (isl) ] => scf => HLS CodeGen
```

### IP integration using .to()
TBA. Niansong and I had a discussion on how to combine hierarchical scheduling with IP integration, and use the customized for System C code generation. Will write more about this later after we have a concrete plan.