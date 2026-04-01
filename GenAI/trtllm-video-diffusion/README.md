# TRT-LLM Video Diffusion

TensorRT-LLM video diffusion is documented as an experimental text-to-video path in Dynamo.

## Documented example

| Backend | Example model | Endpoint |
|---------|---------------|----------|
| TensorRT-LLM | `Wan-AI/Wan2.1-T2V-1.3B-Diffusers` | `/v1/videos` |

Repo-local Kubernetes manifest:

- `trtllm-agg_wan_t2v.yaml` for `Wan-AI/Wan2.1-T2V-1.3B-Diffusers`

Concrete example used in `cmd.txt`:

- backend: TensorRT-LLM with `--modality video_diffusion`
- model path: `Wan-AI/Wan2.1-T2V-1.3B-Diffusers`
- request model name: `wan_t2v`
- output: `/v1/videos` with the generated MP4 stored under `file:///tmp/dynamo_media`

This follows the current official quick start for experimental TRT-LLM video diffusion and adapts it into a DynamoGraphDeployment pattern consistent with the rest of this repo.

## Source

- https://docs.nvidia.com/dynamo/user-guides/diffusion/trt-llm-diffusion
- https://docs.nvidia.com/dynamo/dev/backends/tensor-rt-llm
