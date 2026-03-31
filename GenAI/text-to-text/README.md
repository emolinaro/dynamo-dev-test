# Text-to-Text

Current Dynamo docs discuss two generation-oriented text paths:

- vLLM-Omni text-to-text
- SGLang LLM diffusion

## Documented examples

| Backend | Example models | Endpoint |
|---------|----------------|----------|
| vLLM-Omni | `Qwen/Qwen2.5-Omni-7B` | `/v1/chat/completions` |
| SGLang LLM diffusion | `inclusionAI/LLaDA2.0-mini-preview` | `/v1/chat/completions`, `/v1/completions` |

## Sources

- https://docs.nvidia.com/dynamo/latest/user-guides/diffusion/v-llm-omni
- https://docs.nvidia.com/dynamo/dev/backends/sg-lang/diffusion
