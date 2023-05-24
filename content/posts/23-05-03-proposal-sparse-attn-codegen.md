---
title: "Proposal: User-Level Spare Attention Abstraction and GPU CodeGen"
date: 2023-05-07T15:05:55-04:00
draft: false
tags: ["transformers", "sparsity", "distributed inference"]
---

## Workloads
- Sparse attention: attention structure is sparse by design. In sparse transformer, it is typically using blocked SpMM and SDDMM to compute sparse QKV (by leveraging Ampere's sparse TC). The sparse pattern can be static (position-based) or dynamic (content-based).

- Weight pruning: structured pruning (e.g., block or channel pruning) over transformer weights. SpMM, e.g., with BSR/DBSR/Blocked-Ellpack for block sparse, or some tiling+group solution for unstructured sparse (e.g., Magicube).

- Overhead of sparse format conversion (i.e., sparsifier that converts dense to sparse format) can incur significant overhead. This can be a problem for replacement dense kernels with sparse kernels, or for dynamic sparsity.


## How is sparsity expressed in other frameworks?
### Sparse library
- `xFormers`: it supports sparse and block sparse (by triton for GPU with SM > 70 Ampere sparse TC). 

```python
# fine-grained sparse: attention mask
# https://github.com/facebookresearch/xformers/blob/main/xformers/components/attention/scaled_dot_product.py#L132
y = scaled_dot_product_attention(
    q=q, k=k, v=v, att_mask=att_mask, dropout=self.attn_drop
)

# block sparse: triton operators
# xFromers: https://github.com/facebookresearch/xformers/blob/main/xformers/components/attention/blocksparse.py
# Triton blocksparse: https://github.com/openai/triton/issues/243
from triton.ops.blocksparse import matmul as blocksparse_matmul
from triton.ops.blocksparse import softmax as blocksparse_softmax
q = q / math.sqrt(q.size(-1))
sparse_att_mat = self.sparse_dot_sdd(q, k)
```

- Third-party extensions to support block sparsity in Torch. Most of them provides a drop-in replacement for `nn.Linear` with minimal code changes.

```python
# https://www.reddit.com/r/MachineLearning/comments/iq55ig/p_pytorch_extension_for_gpuaccelerated_block/

# https://github.com/huggingface/pytorch_block_sparse
# self.fc = nn.Linear(1024, 256)
self.fc = BlockSparseLinear(1024, 256, density=0.1)
```

- cuSPARSELt and Apex: fine-grained sparse (i.e., 2:4 sparsity by [Apex's automatic sparsity](https://github.com/NVIDIA/apex/tree/master/apex/contrib/sparsity) to do inference in pytorch with Ampere sparse TC), or coarse-grained sparse (e.g., block sparse. Block-SpMM with blocked-ELL format). 

```c++
// Block-SpMM: A is sparse MxN matrix with blocked-ELL format
cusparseCreateBlockedEll(&matA, A_num_rows, A_num_cols,
                         ell_blocksize, ell_cols,
                         d_ell_colidx, d_ell_values, 
                         CUSPARSE_INDEX_32I, CUSPARSE_INDEX_BASE_ZERO,
                         AB_type);
cusparseSpMM(handle, opA, opB, alpha, matA, matB,
             beta, matC, compute_type,
             CUSPARSE_SPMM_ALG_DEFAULT, d_buffer);
```

- DeepSpeed. How users can [customize the sparse pattern (e.g., attending locally, globally or random)](https://www.deepspeed.ai/2020/09/08/sparse-attention.html) for both forward and backward passes. It is developed as an extension library based on PyTorch and Triton (CUDA not added yet). 

```python
# Config sparse attention pattern in DeepSpeed
# https://www.deepspeed.ai/tutorials/sparse-attention/#how-to-config-sparsity-structures
class BigBirdSparsityConfig(SparsityConfig):
    num_random_blocks = ...
    num_sliding_window_blocks = ...

# The configuration is passed to the sparse attention module
sparse_self_attention = SparseSelfAttention(..., sparsity_config=BigBirdSparsityConfig)
```

### Compiler approach
- [Sten](https://arxiv.org/pdf/2304.07613.pdf): customizable sparse tensor layout in PyTorch (idea is similar to TACO or TVM SpareTIR) for inference + training.  PyTorch natively only supports [COO and CSR formats](https://discourse.llvm.org/t/rfc-sparse-tensor-support-in-torch-mlir/63627) with a lot of limitations.

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
   # implementation
```

- TVM: block-sparsity example of PruneBERT model with 95% sparsity. The kernel is completely hidden from programmers. This may become part of Torch 2.0 ecosystem with the [PyTorch 2.0/TVM integration plan](https://octoml.ai/blog/pytorch-2-0-apache-tvm-better-together/) 

```python
# https://octoml.ai/blog/leveraging-block-sparsity-with-apache-tvm-to-halve-your-cloud-bill-for-nlp/
# https://tvm.apache.org/docs/how_to/deploy_models/deploy_sparse.html#run-the-sparse-graph

# check if weight can be converted into Block Compressed Row Format (BSR).
# if so, relay.transform.DenseToSparse is applied to replace ops
mod, params = ddo.bsr_dense.convert(mod, params, (bs_r, 1), sparsity_threshold=0.8)
```

- SparseTIR: tensor-level, composable sparse abstraction and kernel generation in TVM. it is 1.5x to 3x faster than Triton blocked SpMM & SDDMM, and 1-2x faster than DGL/TACO on SDDMM workloads. https://dl.acm.org/doi/pdf/10.1145/3582016.3582047

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


### Conclusion
- SparseTIR looks pretty strong, but (1) it cannot be used as drop-in replacement for sparsity in PyTorch model, (2) limited to inference, no autograd, (3) no support for dynamic sparsity, (4) not account for sparsifier overhead

- It may be useful to extend PyTorch with user-level sparse abstraction (e.g., [Torch-Sparse](https://github.com/rusty1s/pytorch_sparse/)) and automatic CodeGen capabilities, like [MLIR-SparseTensor+TACO](https://mlir.llvm.org/docs/Dialects/SparseTensorOps/) which only takes sparse as property at high level. [TACO integration in PyTorch](https://github.com/tensor-compiler/taco/issues/464)

- A simple demo of defining sparse tensor in PyTorch and generate CUDA kernel using TACO. Say if we want to create a position-based sparse attention mask, we can define the sparse format and sparsifier (with auograd support) 

```python
# define a random spare pattern (position-based) in transformer





```