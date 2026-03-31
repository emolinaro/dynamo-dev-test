# GenAI

This directory collects generation-focused examples discussed in the current Dynamo documentation, organized as a staging area for future testing.

Unlike the `Text/`, `Image/`, `Audio/`, and `Video/` folders in this repo, most of the official GenAI examples are currently documented as local launch scripts in the upstream Dynamo repository rather than as Kubernetes manifests. The files here capture those examples in a repo-local structure.

## Folders

| Folder | Scope |
|--------|-------|
| `text-to-image/` | vLLM-Omni and SGLang image generation |
| `text-to-video/` | vLLM-Omni and SGLang text-to-video |
| `text-to-audio/` | Placeholder for text-to-audio generation; no first-class Dynamo guide was found in the docs used for this scaffold |
| `text-to-text/` | vLLM-Omni text generation and SGLang LLM diffusion |
| `image-to-video/` | vLLM-Omni image-to-video and notes on SGLang image-prompt video generation |
| `llm-diffusion/` | SGLang diffusion language models |
| `fastvideo/` | FastVideo custom-worker deployment notes |
| `trtllm-video-diffusion/` | TensorRT-LLM video diffusion |

## Notes

- Official Dynamo docs for these GenAI workflows are currently split between `latest` and `dev` paths.
- The commands in each `cmd.txt` are copied or adapted from the official docs and upstream launch scripts. They are not yet adapted into Kubernetes manifests in this repo unless explicitly stated.
- `text-to-audio/` is included because you asked for it, but I did not find a first-class, user-facing Dynamo `v1.0.1` text-to-audio generation guide in the official docs used here.

## Sources

- vLLM-Omni: https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- FastVideo: https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/fastvideo
- TRT-LLM Video Diffusion: https://docs.nvidia.com/dynamo/user-guides/diffusion/trt-llm-diffusion
- SGLang Diffusion: https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
