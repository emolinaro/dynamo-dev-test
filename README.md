# dynamo-dev-test

Kubernetes manifests and quick commands for deploying **NVIDIA AI Dynamo** graph deployments using the **vLLM runtime**, covering:

- Text (Qwen3)
- Image/Vision (LLaVA 1.5, Qwen2.5-VL)
- Audio (Qwen2-Audio)
- Video (LLaVA-NeXT-Video)
- Profiler: SLA profiling examples for rapid and thorough DGDR (DynamoGraphDeploymentRequest) runs against Qwen3-0.6B — **runs only with disaggregated** (decode + prefill) deployments
- GenAI: doc-based scaffolding for generation workflows from the current Dynamo docs, including text-to-image, text-to-video, image-to-video, and diffusion examples

Each modality lives in its own folder and includes:

- A `cmd.txt` with a minimal deploy + port-forward + test `curl`
- One or more `DynamoGraphDeployment` YAML manifests

**This repository targets the first scenario below** (Kubernetes + Dynamo operator + DGD).

## Deployment scenarios

Dynamo can be deployed in two main ways:

| Scenario | Use case | Characteristics |
|----------|----------|-----------------|
| **Kubernetes (Dynamo operator + DGD)** | Production-style cluster deployments | **Declarative deployments** via `DynamoGraphDeployment` (DGD) and `DynamoGraphDeploymentRequest` (DGDR). Autoscaling adapters, metrics integration (e.g. Prometheus), operator-managed lifecycle. This repo’s manifests and profiler follow this model. |
| **Kubernetes + GAIE/kGateway (Inference Gateway)** | K8s-native ingress + token-aware KV routing at the gateway | **Token-aware KV routing** in the gateway EPP; **header-based deterministic routing** to backends; aligns with the **Gateway API** model. Suited when routing and request affinity are handled at the ingress/gateway layer rather than purely by the operator. |

- **Scenario 1** is what the examples in this repo assume: apply DGD/DGDR manifests, use the Dynamo operator, and optionally run the profiler for disaggregated SLA tuning.
- **Scenario 2** adds an inference gateway (GAIE/kGateway) in front of the cluster for gateway-level routing and token-aware behavior; it complements rather than replaces DGD-based deployments.

For more detail and a **comparison table** (local file-KV, Compose, K8s operator-managed, disaggregated RDMA, GAIE/kGateway), see **[DEPLOYMENT_SCENARIOS.md](DEPLOYMENT_SCENARIOS.md)**.

## Repository layout

- `Text/`
  - `vllm-agg.yaml`: single-worker text deployment
  - `cmd.txt`: deploy + sample OpenAI-compatible request
- `Image/`
  - `vllm-agg_llava.yaml`: LLaVA image chat
  - `vllm-agg_qwen_vision_e_pd.yaml`: Qwen2.5-VL with processor + encode + PD workers
  - `cmd.txt`: deploy + sample image chat requests
- `Audio/`
  - `vllm-agg_qwen_audio_e_pd.yaml`: Qwen2-Audio with processor + audio encode + PD workers
  - `cmd.txt`: deploy + sample audio chat request
- `Video/`
  - `vllm-agg_llava_video_e_pd.yaml`: LLaVA-NeXT-Video with processor + encode + PD workers
  - `cmd.txt`: deploy + sample video chat request
- `Profiler/`
  - Profiler runs **only with disaggregated** deployments (decode + prefill); aggregated graphs are not profiled.
  - `disagg.yaml`: disaggregated vLLM graph (Frontend + decode + prefill workers) updated to `vllm-runtime:1.0.1`
  - `profile_sla_aic_dgdr.yaml`: `nvidia.com/v1beta1` DGDR using `searchStrategy: rapid`
  - `profile_sla_online_dgdr.yaml`: `nvidia.com/v1beta1` DGDR using `searchStrategy: thorough`
  - `cmd.txt`: apply DGDR, watch request state, tail profiler job logs, and inspect the generated DGD
- `GenAI/`
  - doc-oriented scaffold for generation workflows discussed in the current Dynamo docs
  - `text-to-image/`, `text-to-video/`, `text-to-audio/`, `text-to-text/`, `image-to-video/`, `llm-diffusion/`, `fastvideo/`, `trtllm-video-diffusion/`

## Prerequisites

- A Kubernetes cluster with NVIDIA GPU nodes (these manifests assume a GPU `runtimeClassName: nvidia`)
- NVIDIA AI Dynamo installed (CRD `kind: DynamoGraphDeployment` available)
- Namespace `dynamo-system` exists
- A Hugging Face token Kubernetes secret referenced by the manifests:
  - `hf-token-secret`

> **Install instructions:** **INSTALL_INSTRUCTIONS.md** now covers both fresh install and platform upgrade to Dynamo `v1.0.1` for the **Kubernetes + Dynamo operator + DGD** scenario, including the Helm value changes needed to upgrade an older `0.8.1` deployment safely, post-upgrade checks, and rollback steps. Run that flow before applying the manifests in this repo.

> Note: Some multimodal examples prefer TCP request plane to avoid payload limits (see the image LLaVA manifest comments and `DYN_REQUEST_PLANE` usage).
>
> If UCX / RDMA is not available in your environment (for example, `nixl` cannot find RDMA devices), you can safely fall back to **TCP-only UCX** by setting the UCX-related env vars in the manifests (as shown in the Qwen vision/audio/video examples) so that UCX uses `tcp` instead of attempting RDMA transports.

## Quick start

Pick a modality folder and run the commands in its `cmd.txt`.

Example (`Text`):

```bash
cd Text
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
