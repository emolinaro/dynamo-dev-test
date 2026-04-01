# Text-to-Video

Current Dynamo docs discuss multiple text-to-video paths:

- vLLM-Omni
- SGLang video generation
- TensorRT-LLM video diffusion

This folder keeps the general text-to-video entry points together. Dedicated notes for TensorRT-LLM also live in its own subfolder under `GenAI/`.

Repo-local Kubernetes manifest:

- `vllm-agg_wan_t2v.yaml` for `Wan-AI/Wan2.1-T2V-1.3B-Diffusers`

Concrete example used in `cmd.txt`:

- model: `Wan-AI/Wan2.1-T2V-1.3B-Diffusers`
- prompt: `"A drone flyover of a mountain landscape"`
- output: `/v1/videos` with `"response_format": "url"`

This follows the current vLLM-Omni text-to-video example from the Dynamo docs. The repo example uses the smaller Wan 2.1 1.3B model as the default concrete target.

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
- https://docs.nvidia.com/dynamo/user-guides/diffusion/trt-llm-diffusion
