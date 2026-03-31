# FastVideo

FastVideo is documented as a dedicated text-to-video path on top of Dynamo using a custom worker rather than the built-in vLLM-Omni or SGLang generation workers.

The official docs cover:

- local Docker Compose deployment
- host-local deployment
- Kubernetes deployment

The example is optimized for B200/B300 GPUs and uses the default model `FastVideo/LTX2-Distilled-Diffusers`.

## Source

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/fastvideo
