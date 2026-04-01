# Text-to-Image

Current Dynamo docs discuss two main text-to-image paths:

- vLLM-Omni
- SGLang image diffusion

## Documented examples

| Backend | Example models | Endpoint |
|---------|----------------|----------|
| vLLM-Omni | `Qwen/Qwen-Image`, `AIDC-AI/Ovis-Image-7B` | `/v1/chat/completions`, `/v1/images/generations` |
| SGLang | `black-forest-labs/FLUX.1-dev` | `/v1/images/generations` |

Repo-local Kubernetes manifests now exist for the models listed in `cmd.txt`:

- `vllm-agg_qwen_image.yaml` for `Qwen/Qwen-Image`
- `sglang-agg_flux_image.yaml` for `black-forest-labs/FLUX.1-dev`

`AIDC-AI/Ovis-Image-7B` uses the same vLLM-Omni deployment pattern as `Qwen/Qwen-Image`; change the worker `--model` argument if you want an additional manifest for that model.

For vLLM-Omni, `/v1/chat/completions` returns the generated image inline as a base64 `data:image/png` URL. If you want something easier to consume from the shell, prefer `/v1/images/generations` with `"response_format": "url"`, as shown in `cmd.txt`.

For the SGLang FLUX manifest, the runtime here accepts `--media-output-fs-url` rather than the older `--fs-url` wording shown in part of the docs. The manifest is written to match the current `1.0.1` container behavior.

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
