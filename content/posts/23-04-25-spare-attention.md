---
title: "Exploration on Spare Attention"
date: 2023-04-25T15:05:55-04:00
draft: false
tags: ["transformers", "sparsity", "distributed inference"]
---

## Misc
- Extending Slapo to distributed inference might not be a good idea. `Slapo`'s key feature is "trace-by-need" progressive optimization, which is not useful in inference.

- `Slapo` only has DeepSpeed and Megatron-LM backends for now. DeepSpeed is less optimized than FT on inference speed.

- **Framework overhead is large**: from Nvidia Nsight's profiling report, the kernel launch overhead in DeepSpeed is way higher (many more kernel calls and more kernel counts). Maybe we can consider using CUDA Graph context manager to reduce overhead.

- **Kernel fusion**: can we use Slapo .replace() primitive to replace kernels, say AI template or other hand-tuned CUDA kernels?
  
- Compiler approach for kernel fusion to improve usability (i.e., generating or replacing fused kernel)?

## Sparse in PyTorch
- [STen: Productive and Efficient Sparsity in PyTorch](https://arxiv.org/pdf/2304.07613.pdf): customizable sparse tensor layout in PyTorch (idea is similar to TACO or TVM SpareTIR) for inference + training. [Source code](https://github.com/spcl/sten)

- Example of specifying custom sparse layout: CSC. Kernel is not auto-generated. May fallback to dense kernel.

```python
# Define custom sparse layout
class CscTensor: 
    def __init__(self, data):
        self.data = scipy.sparse.csc_matrix(data)
    def to_dense(self):
        return torch.from_numpy(self.data.todense())


# Use PM to dispatch CSR*dense to target implementation
a = sten.torch_tensor_to_csr(sparsifier, torch.randn(4, 4))
b = torch.randn(4, 4) 
c = torch.mm(a, b)
```

- Case study of `n:m` sparse format. 

- Can we build something upon STen to support kernel auto-generation and other sparse formats? `2:4` in Ampere/cuSPARSE, big bird and such for attention layers.

## TACO: auto-generate sparse kernels with scheduling language. 
- Source: https://github.com/tensor-compiler/taco
- Limited types of sparse formats are supported

```c++
Format csr({Dense,Sparse});
Tensor<double> A({2,3},   csr);
```

- DGL sparse. xtransformer. block sparsity. usability++ and support more sparse formats for decoder inference.

