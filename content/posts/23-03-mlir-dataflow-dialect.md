---
title: "MLIR Dataflow Dialect: Design, Simulation, and Optimization"
date: 2023-04-21T15:05:55-04:00
draft: true
tags: ["mlir", "compiler", "dataflow"]
---

## Dataflow in TAPA/HLS/TaskFlow

- Example in TAPA. A simple example to define a vector adder with two back-to-back kernels in HLS. TAPA's interface is similar to HLS's, but with a much++ simple interface for host-accelerator communication.

```cpp
#include <tapa.h>

// Kernel specification
void Add(tapa::istream<float>& in, tapa::ostream<float>& out, uint64_t n) {
  for (uint64_t i = 0; i < n; ++i) {
    out << (in.read() + 1);
  }
}

// DFG specification (top-level function)
void Top(tapa::mmap<const float> in, tapa::mmap<float> out, uint64_t n) {
  tapa::stream<float> fifo("a");

  tapa::task()
      .invoke(Add, in, fifo)
      .invoke(Add, fifo, out)
}

// Host code: loading bitstream and prepare OCL buffers
tapa::invoke(VecAdd, FLAGS_bitstream,
    tapa::read_only_mmap<const float>(in),
    tapa::write_only_mmap<float>(out), n);
```

- A table to summarize the differences between TAPA, HLS, and TaskFlow.

|               | TAPA              | HLS         | TaskFlow                  |
| ------------- | ----------------- | ----------- | ------------------------- |
| Dataflow      | yes               | yes         | no (task dependency only) |
| Scheduling    | data driven       | data driven | work stealing             |
| Memory access | async/sync mmap\* | sync mmap   | sync mmap                 |
| Targets       | FPGA              | FPGA        | CPU+GPU                   |

> \*async mmap: TAPA inserts an extra AXI adapter that can dynamically batch memory requests into a burst transfer.

### An MLIR dialect for TAPA and more

The dataflow MLIR dialect supports new features from TAPA, including:

- Non-destructive APIs for FIFOs: `eot()` and `peek()`
- Async mmap and sync mmap

It also includes new features that are not supported by TAPA, including:

- Input dependent dataflow
- Explicit task dependency and scheduling
- Explicit task placement to target devices 

```c++
func @Add(%in: !dataflow.stream<f32>, %out: !dataflow.stream<f32>) {
  %0 = dataflow.read %in : stream<f32>
  %1 = addf %0, 1.0 : f32
  dataflow.write %1, %out : stream<f32>
  return
}

func @Top(%in: memref<f32>, %out: memref<f32>) {
  %0 = dataflow.mmap_read %in : !dataflow.mmap<f32>
  %1 = dataflow.mmap_read %out : !dataflow.mmap<f32>
  %2 = dataflow.stream_create "fifo0" : !dataflow.stream<f32> { depth = 32}

  // Explicit device placement
  dataflow.task @Add(%0, %2) { device = "FPGA[0]" }

  // Explicit input dependent task dependency
  dataflow.task @Add(%2, %1) { device = "FPGA[1]" }  {
    ^when (%2 : dataflow.stream<f32>):
        %3 = dataflow.peek %2 : stream<f32>
        %3 % 2 == 1
    }
  return
}
```

### Connection to HCL dialect
- HCL dialect is used for operator-level optimization. All the graph-level optimizations should be done in the dataflow dialect.

- The HeteroCL python API is untouched. But under the hood, HeteroCL compiler can use dataflow dialect (along with `scf`, `affine`, etc) to build the DFG and perform graph-level optimizations (e.g., DFG partition, device placement, operator fusion). 

- The output DFG from dataflow dialect is then passed to HCL dialect for operator-level optimizations, e.g., loop tiling, memory partitioning, and data layout optimizations.

### Simulations
- We have added support of using multiple MLIR JIT execution engine to simulate task-parallel dataflow programs, but this is only for testing correctness, not performance on hardware.

- dataflow dialect IR can be used to generate TAPA code, and thus we can use the [co-sim support in TAPA](https://tapa.readthedocs.io/en/release/tutorial/fast_cosim.html)

- Or alternatively, we can use third-party dataflow simulators to estimate performance of input dataflow programs. For example, [DF-Sim](https://github.com/tbennun/dfsim) or [EventQueue](https://github.com/cucapra/EventQueue)


## Some considerations
- do we need a `device` or `platform` dialect to define and manage devices? 

## Progress
- [x] Dataflow dialect core operations (i.e., stream_create, read, write).
- [x] A simple translation function that translate dataflow dialect into TAPA code.
