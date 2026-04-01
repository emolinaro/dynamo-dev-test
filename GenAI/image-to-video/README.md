# Image-to-Video

Current Dynamo docs explicitly discuss image-to-video in the vLLM-Omni guide, and the SGLang video generation guide states that it supports both text and image prompts.

## Documented examples

| Backend | Example models | Endpoint |
|---------|----------------|----------|
| vLLM-Omni | `Wan-AI/Wan2.2-TI2V-5B-Diffusers`, `Wan-AI/Wan2.2-I2V-A14B-Diffusers` | `/v1/videos` |
| SGLang | Video generation path supports T2V and I2V | `/v1/videos` |

Repo-local Kubernetes manifest:

- `vllm-agg_wan_i2v.yaml` for `Wan-AI/Wan2.2-TI2V-5B-Diffusers`

Concrete example used in `cmd.txt`:

- model: `Wan-AI/Wan2.2-TI2V-5B-Diffusers`
- prompt: `"A bear playing with yarn, smooth motion"`
- input image: a public Unsplash URL
- output: `/v1/videos` with `"response_format": "url"`

This follows the current vLLM-Omni image-to-video example from the Dynamo docs. The repo example uses the smaller 5B Wan I2V model as the default concrete target rather than the 14B variant.

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
