
---
title: "Exploration on Spare Attention"
date: 2023-04-22T15:05:55-04:00
draft: false
tags: ["transformers", "sparsity", "distributed inference"]
---

## Observations and thoughts
- **Framework overhead is large**: from Nvidia Nsight's profiling report, the kernel launch overhead in DeepSpeed is way higher (many more kernel calls and more kernel counts). Maybe we can consider using CUDA Graph context manager to reduce overhead.
  
- Compiler approach for kernel fusion to improve usability (i.e., generating or replacing fused kernel)? E.g., can we use Slapo .replace() primitive to replace kernels, say AI template or other hand-tuned CUDA kernels? Fully automatic fusion is hard, but we can start with some simple cases, or consider pattern matching.

- Can we build something upon STen to support kernel auto-generation and other sparse formats? `2:4` in Ampere/cuSPARSE, big bird and such for attention layers.

- DGL sparse. xtransformer. block sparsity. usability++ and support more sparse formats for decoder inference.

- Overhead of 

## Foundation Models

### Diffusion Model
- Stable diffusion art: https://stable-diffusion-art.com/how-stable-diffusion-work/

- Stable diffusion works on the latent space of a pre-trained VAE (variational auto-encoder). VAE maps the image from pixels to smaller latent space.

- The UNet (i.e, noise predictor works on the latent space) uses cross-attention'ed with the output from text transformer (i.e., this output is basically a query represented by the human input, and KV is the image knowledge base in UNet). 

- Hypernetwork and LoRa (and some other fine-tuning techniques) tweak the cross-attention weights to adjust the generated image styles. There are other conditioning like ControlNet to use structure data to guide the noise predictor.

- A hands-on tutorial on how to import these models in webui-stable-diffusion: https://www.youtube.com/watch?v=xkBaR5bIYqc


### Transformers and variants
- Source: https://chengh.medium.com/evolution-of-fast-and-efficient-transformers-ec0378257994

- Challenge: the computation of self-attention increases quadratically with the sequence length increasing. 

- Segment level recurrence -- **Transformer-XL** connects adjacent segments with a recurrent mechanism. It caches the representation from the previous segment, then reused it as an extended context for the current segment computing. **Compressive Transformer** has another level of activations caching on top of Transformer-XL. 

- Sparse attention: Use a sparse pattern instead of computing full attention matrix. Examples include: global, band, dilated sliding window, local, random attention and etc. **Position-based sparse pattern**: e.g., LongFormer (band + dilated + global), BigBird (band + global + random), or **Content-based (dynamic) sparsity**: e.g., routing transformer (KMeans clustering KQ to generate sparse pattern), reformer (locality-sensitive-hashing to group tokens, and then attend between chunks in parallel)

- Problems of sparse: loss of representation power, overhead of sparse format conversion, hard to write efficient kernels for sparse formats, and some kernels like softmax cannot be sparsified.

- Approximation: for QKV e.g., using low-rank SVD decomposition to calculate attention score from (i.e., projection using less hidden dimension, e.g., LinFormer). 

# Frameworks

## Meta AITemplate
- The optimization is limited to kernel fusion, and it is heavily relying on CUTLASS (but not other libraries like cuBLAS or cuPARSE).

- There is no decoder example yet. I haven't tested the inference speed yet, but one core developer told me it is up to 2x faster than TensorRT. 



