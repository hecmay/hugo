---
title: "Proposal: A scheduling language that decouples sparsity in PyTorch"
date: 2023-05-07T15:05:55-04:00
draft: false
tags: ["transformers", "sparsity", "sparse attention"]
---


## TL;DR
The main idea is to create a scheduling language that separates the specification of sparsity from the PyTorch model definition. Here is simple example to demonstrate how it looks like:

```python
import sparse

# Create schedule from existing Torch model
model = BertModel.from_pretrained('bert-base-uncased')
s = sparse.create_schedule(model, ...)

# Schedule to sparsify the linear layer weight
# Users can use predefined formats or define their own formats
s['linear1.weight']
    .set_format( sparse.CscTensor )
```

### Proposal
- **User-defined sparse format**: users can define their own sparse tensor format. Here is a simple example of defining Ellpack format by compressing & packing the column in a given tensor. If only `.compress()` is used, the tensor will be compressed in CSR format. 

```python
class EllpackTensor(sparse.SparseTensor):
    def __init__(self, ...):
        with sparse.iter(dim=2) as I, J:
            J.compress(coordinate=True).pack()
            self.indices = [I, J]
```

- **Hybrid sparse format**: Users can combine multiple sparse formats to represent their tensors. Here's a quick example of formatting a tensor into a hybrid format that combines blocked Ellpack and COO formats.

```python
# hybrid format: blocked ELL and COO
s['linear1.weight']
    .block(16, 16)
    .set_format( sparse.EllpackTensor, axis = [0, 1] )
    .set_format( sparse.CooTensor, axis = [2, 3] )
```

- **Customize sparse pattern**: users can impose additional constraints on the sparse pattern for a given tensor. The sparse pattern can be static or dynamic. For example, *routing transformer* requires dynamic sparsity, where its sparse attention is generated based on similarity between Q and K at runtime. 

![IMAGE](https://raw.githubusercontent.com/lucidrains/routing-transformer/master/routing_attention.png)

We can use a `sparsify()` primitive to specify a custom sparsifier to describe this additional constraint.

```python
# Custom sparsifier that computes mask dynamically
class SimilaritySparsifier(sparse.Sparsifier):
    def __init__(self, ...):
        ...

s['linear1.attn']
    .set_format( ... )
    .sparsify( SimilaritySparsifier )
```

- **Compiler optimization**:  We can probably also provide some primitives to guide (e.g., like TACO scheduling language) to guide the compiler to generate efficient kernels.

## Prior work and comparison
### Sparse library
- **`xFormers`**: it supports block sparse; the implementation is based on Triton IR targeting GPU with Ampere sparse TC (SM > 70). 

```python
# sparse pattern is specified with a mask
y = scaled_dot_product_attention(
    q=q, k=k, v=v, att_mask=att_mask, dropout=self.attn_drop
)

# block sparse is provided as a library
from triton.ops.blocksparse import matmul as blocksparse_matmul
from triton.ops.blocksparse import softmax as blocksparse_softmax

sparse_att_mat = self.sparse_dot_sdd(q, k)
```

- **PyTorch extension library** to support block sparsity, such as [`pytorch_block_sparse`](https://github.com/huggingface/pytorch_block_sparse) or [`torch-blocksparse`](https://github.com/ptillet/torch-blocksparse). They can provide a drop-in replacement for `nn.Linear` with minimal code changes.

```python
# example of usage
self.fc = BlockSparseLinear(1024, 256, density=0.1)
```

- **cuSPARSELt and Apex**: they can support both fine-grained sparsity (i.e., 2:4 sparsity by [Apex's automatic sparsity](https://github.com/NVIDIA/apex/tree/master/apex/contrib/sparsity)), and coarse-grained sparsity (e.g., Block-SpMM with blocked-ELL format). 

```c++
// cuSPARSELt: SpMM with Blocked-ELL format
cusparseCreateBlockedEll(&matA, A_num_rows, A_num_cols,
                         ell_blocksize, ell_cols,
                         d_ell_colidx, d_ell_values, 
                         CUSPARSE_INDEX_32I, CUSPARSE_INDEX_BASE_ZERO,
                         AB_type);
cusparseSpMM(handle, opA, opB, alpha, matA, matB,
             beta, matC, compute_type,
             CUSPARSE_SPMM_ALG_DEFAULT, d_buffer);
```

- **DeepSpeed**. users can [customize the sparse pattern (e.g., local, global or random attention)](https://www.deepspeed.ai/2020/09/08/sparse-attention.html) for both forward and backward passes. It is developed as an extension library based on Triton IR, which can be used in PyTorch model directly. 

```python
class BigBirdSparsityConfig(SparsityConfig):
    num_random_blocks = ...
    num_sliding_window_blocks = ...

# The configuration is passed to the sparse attention module
sparse_self_attention = SparseSelfAttention(
    sparsity_config = BigBirdSparsityConfig, ...)
```

### Compiler approach
- **[STen](https://arxiv.org/pdf/2304.07613.pdf)**: let users define their own sparse tensor format, sparsifier, and spare kernel operator. STen does not generate operators -- users need to provide manual written kernels. No support to compose sparse formats into a new format.

```python
# Add new sparse operator
# sparse tensor (self.weight) also needs to be wrapped into a custom class
def forward(self, input):
    sparse_op = sten.sparsified_op(
        orig_op=torch.nn.functional.linear,
        # sparse strategy
        out_fmt=tuple(
            [(sten.KeepAll(), torch.Tensor,
              sten.KeepAll(), torch.Tensor)]
        ),
        grad_out_fmt=tuple(
            [(sten.KeepAll(), torch.Tensor,
              sten.KeepAll(), torch.Tensor)]
        ),
    )
    return sparse_op(input, self.weight, self.bias)

# The sparsification process is specified using decorators
# forward and backward operator implementations are specified separately
class GroupedNMSparsifier:
    def __init__(self, n, m, g):
        self.n, self.m, self.g = n, m, g
@sten.register_sparsifier_implementation(
    sparsifier=GroupedNMSparsifier,
    inp=torch.Tensor, out=FixedMaskTensor)
def dense_to_grouped_n_m(sparsifier, tensor, grad_fmt=None):
   # implementation ...
```

- **TVM**: can support BSR, but it is more like a library style -- users cannot really customize the sparse pattern or introduce a new sparse format. 

```python
# TVM compiler checks if weight can be converted into Block Compressed Row Format (BSR).
# if so, relay.transform.DenseToSparse is applied to replace ops
mod, params = ddo.bsr_dense.convert(mod, params, (bs_r, 1), sparsity_threshold=0.8)
```

- [**SparseTIR**](https://dl.acm.org/doi/pdf/10.1145/3582016.3582047): tensor-level, composable sparse abstraction and kernel generation in TVM. it is 1.5x to 3x faster than Triton blocked SpMM & SDDMM, and 1-2x faster than DGL/TACO on SDDMM workloads. 

```python
# dense or sparse iterations
I = dense_fixed(m, "int32")
J = sparse_variable(I, (n, nnz),(j_indptr, j_indices), "int32")

# buffer binding
A = match_sparse_buffer(a, (I, J), "float32")

# sparse iterations (operator)
with sp_iter([IO, II, JO, JI, K], "SSSSR", "spmm_bsr_2") as [io, ii, jo, ji, k]:
    with init():
        C[io * 2 + ii, k] = 0.0
    C[io * 2 + ii, k] = C[io * 2 + ii, k] +\
        A_bsr[io, jo, ii, ji] * B[jo * 2 + ji, k]
```