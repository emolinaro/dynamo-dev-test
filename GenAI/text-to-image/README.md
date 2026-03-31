# Text-to-Image

Current Dynamo docs discuss two main text-to-image paths:

- vLLM-Omni
- SGLang image diffusion

## Documented examples

| Backend | Example models | Endpoint |
|---------|----------------|----------|
| vLLM-Omni | `Qwen/Qwen-Image`, `AIDC-AI/Ovis-Image-7B` | `/v1/chat/completions`, `/v1/images/generations` |
| SGLang | `black-forest-labs/FLUX.1-dev` | `/v1/images/generations` |

See `cmd.txt` for the official launch and test commands.

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
