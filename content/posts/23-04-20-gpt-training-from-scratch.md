---
title: "Mini-GPT training from scratch"
date: 2023-04-21T15:05:55-04:00
draft: true
tags: ["mlir", "compiler", "dataflow"]
---

[Training and serve a GPT model from scratch (in PyTorch)](https://www.youtube.com/watch?v=kCc8FmEb1nY)

Just a quick summary of this great talk:

- **Masked attention**: use lower-triangle GeMM to compute masked attention scores (so that only preceding tokens are attended to)

- **Sparsity**: attention is basically a directed graph of nodes representing tokens and edges representing attention scores. 

- Cross attention: attention where the query comes from a different sequence (i.e., source) than the key and value.

- Scaled dot-product attention: 1/sqrt(head_dimension_size) to avoid skewing of output of softmax by ensuring the variance of the input scores is 1. MHA is group of scaled dot-product attention (similar to group convolution).

- Pre-training and fine-tuning: pre-training learns the completion rules (unaligned). and fine-tuning is about alignment (supervised learning, reward model from human feedback, RL to optimize sampling policy to please RM)
