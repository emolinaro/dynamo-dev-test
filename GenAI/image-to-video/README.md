# Image-to-Video

Current Dynamo docs explicitly discuss image-to-video in the vLLM-Omni guide, and the SGLang video generation guide states that it supports both text and image prompts.

## Documented examples

| Backend | Example models | Endpoint |
|---------|----------------|----------|
| vLLM-Omni | `Wan-AI/Wan2.2-TI2V-5B-Diffusers`, `Wan-AI/Wan2.2-I2V-A14B-Diffusers` | `/v1/videos` |
| SGLang | Video generation path supports T2V and I2V | `/v1/videos` |

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
