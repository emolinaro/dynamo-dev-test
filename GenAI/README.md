# GenAI

This directory collects generation-focused examples discussed in the current Dynamo documentation and adapts them into repo-local Kubernetes workflows where practical.

Unlike the `Text/`, `Image/`, `Audio/`, and `Video/` folders in this repo, many of the official GenAI examples are documented as local launch scripts in the upstream Dynamo repository rather than as Kubernetes manifests. The files here either:

- provide repo-local `DynamoGraphDeployment` manifests adapted from those official examples, or
- keep a placeholder/reference folder where the public docs do not yet provide a stable runnable path

## Folders

| Folder | Scope | Status |
|--------|-------|--------|
| `text-to-image/` | vLLM-Omni and SGLang image generation | Runnable repo-local manifests for `Qwen/Qwen-Image` and `black-forest-labs/FLUX.1-dev` |
| `text-to-video/` | vLLM-Omni text-to-video plus SGLang notes | Runnable repo-local manifest for `Wan-AI/Wan2.1-T2V-1.3B-Diffusers` |
| `text-to-audio/` | Placeholder for text-to-audio generation | No first-class Dynamo guide found in current public docs |
| `image-to-video/` | vLLM-Omni image-to-video plus SGLang notes | Runnable repo-local manifest for `Wan-AI/Wan2.2-TI2V-5B-Diffusers` |
| `llm-diffusion/` | SGLang diffusion language models | Runnable repo-local manifest for `inclusionAI/LLaDA2.0-mini-preview` |
| `trtllm-video-diffusion/` | TensorRT-LLM video diffusion | Repo-local manifest added, but this path is experimental and more sensitive to host driver / GPU compatibility |

## Current Coverage

The concrete manifests currently in this repo are:

- `text-to-image/vllm-agg_qwen_image.yaml`
- `text-to-image/sglang-agg_flux_image.yaml`
- `text-to-video/vllm-agg_wan_t2v.yaml`
- `image-to-video/vllm-agg_wan_i2v.yaml`
- `llm-diffusion/sglang-agg_llada2_mini.yaml`
- `trtllm-video-diffusion/trtllm-agg_wan_t2v.yaml`

## Notes

- Official Dynamo docs for these GenAI workflows are currently split between `latest` and `dev` paths.
- The commands in each `cmd.txt` are either adapted to the repo-local manifests above or kept as explicit notes where the public docs remain incomplete.
- `text-to-audio/` is included because you asked for it, but I still did not find a stable, first-class Kubernetes-focused Dynamo `v1.1.0` text-to-audio guide in the official docs used here even though the `v1.1.0` release notes expand diffusion coverage into audio and TTS.
- First startup for these examples can be slow because models are downloaded from Hugging Face. Transient `429 Too Many Requests` errors are possible and usually require waiting and retrying.
- For image/video generation examples, the response often contains a `file://` URL to media stored inside the worker pod. The corresponding `cmd.txt` files show how to copy those files out with `kubectl cp`.
- The `trtllm-video-diffusion/` example is the least portable path here. In practice it may fail on clusters where the GPU node driver stack is older than what the `tensorrtllm-runtime:1.1.0` container expects, even if the cluster can run the vLLM and SGLang examples.

## Sources

- vLLM-Omni: https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- TRT-LLM Video Diffusion: https://docs.nvidia.com/dynamo/user-guides/diffusion/trt-llm-diffusion
- SGLang Diffusion: https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
- TensorRT-LLM backend docs: https://docs.nvidia.com/dynamo/dev/backends/tensor-rt-llm
