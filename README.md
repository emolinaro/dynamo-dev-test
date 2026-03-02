# dynamo-dev-test

Kubernetes manifests and quick commands for deploying **NVIDIA AI Dynamo** graph deployments using the **vLLM runtime**, covering:

- Text (Qwen3)
- Image/Vision (LLaVA 1.5, Qwen2.5-VL)
- Audio (Qwen2-Audio)
- Video (LLaVA-NeXT-Video)

Each modality lives in its own folder and includes:

- A `cmd.txt` with a minimal deploy + port-forward + test `curl`
- One or more `DynamoGraphDeployment` YAML manifests

## Repository layout

- `text/`
  - `vllm-agg.yaml`: single-worker text deployment
  - `cmd.txt`: deploy + sample OpenAI-compatible request
- `image/`
  - `vllm-agg_llava.yaml`: LLaVA image chat
  - `vllm-agg_qwen_vision_e_pd.yaml`: Qwen2.5-VL with processor + encode + PD workers
  - `cmd.txt`: deploy + sample image chat requests
- `audio/`
  - `vllm-agg_qwen_audio_e_pd.yaml`: Qwen2-Audio with processor + audio encode + PD workers
  - `cmd.txt`: deploy + sample audio chat request
- `video/`
  - `vllm-agg_llava_video_e_pd.yaml`: LLaVA-NeXT-Video with processor + encode + PD workers
  - `cmd.txt`: deploy + sample video chat request

## Prerequisites

- A Kubernetes cluster with NVIDIA GPU nodes (these manifests assume a GPU `runtimeClassName: nvidia`)
- NVIDIA AI Dynamo installed (CRD `kind: DynamoGraphDeployment` available)
- Namespace `dynamo-system` exists
- A Hugging Face token Kubernetes secret referenced by the manifests:
  - `hf-token-secret`

> **Install instructions:** For full steps to install Dynamo (CRDs, platform Helm chart), create the Hugging Face token secret, check GPU nodes, run post-install checks, and rollback if needed, see **[INSTALL_INSTRUCTIONS.md](INSTALL_INSTRUCTIONS.md)**.

> Note: Some multimodal examples prefer TCP request plane to avoid payload limits (see the image LLaVA manifest comments and `DYN_REQUEST_PLANE` usage).
>
> If UCX / RDMA is not available in your environment (for example, `nixl` cannot find RDMA devices), you can safely fall back to **TCP-only UCX** by setting the UCX-related env vars in the manifests (as shown in the Qwen vision/audio/video examples) so that UCX uses `tcp` instead of attempting RDMA transports.

## Quick start

Pick a modality folder and run the commands in its `cmd.txt`.

Example (text):

```bash
cd text
kubectl apply -f vllm-agg.yaml
kubectl -n dynamo-system port-forward svc/<deployment-name>-frontend 8000:8000 &
curl -s http://127.0.0.1:8000/v1/models | head
```

Then send a request:

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role":"user","content":"Say hello in one short sentence."}],
    "temperature": 0.2,
    "max_tokens": 64
  }' | head -c 2000
echo
```
