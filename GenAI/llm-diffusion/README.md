# LLM Diffusion

SGLang diffusion language models are documented as a separate generation workflow in Dynamo.

## Documented example

| Backend | Example model | Endpoint |
|---------|---------------|----------|
| SGLang | `inclusionAI/LLaDA2.0-mini-preview` | `/v1/chat/completions`, `/v1/completions` |

Repo-local Kubernetes manifest:

- `sglang-agg_llada2_mini.yaml` for `inclusionAI/LLaDA2.0-mini-preview`

The worker uses `--dllm-algorithm LowConfidence`, which is the diffusion-LLM switch documented by the current SGLang runtime. Dynamo auto-detects the diffusion handler when that flag is set.

## Source

- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
- https://docs.sglang.io/advanced_features/server_arguments.html
