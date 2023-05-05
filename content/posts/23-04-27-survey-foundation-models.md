

## Foundation Models

### Diffusion Model
- Stable diffusion art: https://stable-diffusion-art.com/how-stable-diffusion-work/

- Stable diffusion works on the latent space of a pre-trained VAE (variational auto-encoder). VAE maps the image from pixels to smaller latent space.

- The UNet (i.e, noise predictor works on the latent space) uses cross-attention'ed with the output from text transformer (i.e., this output is basically a query represented by the human input, and KV is the image knowledge base in UNet). 

- Hypernetwork and LoRa (and some other fine-tuning techniques) tweak the cross-attention weights to adjust the generated image styles. There are other conditioning like ControlNet to use structure data to guide the noise predictor.

- A hands-on tutorial on how to import these models in webui-stable-diffusion: https://www.youtube.com/watch?v=xkBaR5bIYqc

## Frameworks
I will mostly focus on the frameworks for deployment and inference.

### Meta AITemplate
- The optimization is limited to kernel fusion, and it is heavily relying on CUTLASS (but not other libraries like cuBLAS or cuPARSE).

- There is no decoder example yet. I haven't tested the inference speed yet, but one core developer told me it is up to 2x faster than TensorRT. 

### TensorRT


