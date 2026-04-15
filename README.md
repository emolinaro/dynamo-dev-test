# dynamo-dev-test

Kubernetes manifests and quick commands for deploying **NVIDIA AI Dynamo** graph deployments on Kubernetes, covering:

- Text (Qwen3)
- Image/Vision (LLaVA 1.5, Qwen2.5-VL)
- Audio (Qwen2-Audio)
- Video (LLaVA-NeXT-Video)
- Profiler: SLA profiling examples for rapid and thorough DGDR (DynamoGraphDeploymentRequest) runs against Qwen3-0.6B — **runs only with disaggregated** (decode + prefill) deployments
- GlobalPlanner: shared-GPU-budget global-planner example
- GAIE: tenant-scoped Gateway API Inference Extension example for the same two models, designed to live in a removable namespace
- GenAI: repo-local generation examples adapted from the current Dynamo docs across vLLM-Omni, SGLang, and experimental TensorRT-LLM paths

Each example family lives in its own folder and includes:

- A command helper file (`cmd.txt` or `cmd-*.txt`) with a minimal deploy + test flow
- One or more `DynamoGraphDeployment` YAML manifests where the public docs provide a stable path

**This repository mainly targets the first scenario below** (Kubernetes + Dynamo
operator + DGD), and it also includes a `GAIE/` example for the second
scenario.

## Deployment scenarios

This repository focuses on two Kubernetes-oriented deployment patterns:

| Scenario | Use case | Characteristics |
|----------|----------|-----------------|
| **Kubernetes (Dynamo operator + DGD)** | Production-style cluster deployments | **Declarative deployments** via `DynamoGraphDeployment` (DGD) and `DynamoGraphDeploymentRequest` (DGDR). Autoscaling adapters, metrics integration (e.g. Prometheus), operator-managed lifecycle. Most examples in this repo follow this model. |
| **Kubernetes + GAIE/kGateway (Inference Gateway)** | K8s-native ingress + token-aware KV routing at the gateway | **Token-aware KV routing** in the gateway EPP; **header-based deterministic routing** to backends; aligns with the **Gateway API** model. Suited when routing and request affinity are handled at the ingress/gateway layer rather than purely by the operator. |

- **Scenario 1** is the main repo path: `Text/`, `Image/`, `Audio/`, `Video/`,
  `Profiler/`, `GlobalPlanner/`, and most `GenAI/` examples assume DGD/DGDR
  manifests managed by the Dynamo operator.
- **Scenario 2** is represented by `GAIE/`, which adds an inference gateway
  (GAIE/kGateway) in front of Dynamo-managed backends rather than replacing the
  DGD-based deployment model.

For more detail and a broader **comparison table** covering local file-KV,
Compose, K8s operator-managed, disaggregated RDMA, and GAIE/kGateway, see
**[DEPLOYMENT_SCENARIOS.md](DEPLOYMENT_SCENARIOS.md)**.

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
- `GlobalPlanner/`
  - `global-planner-shared-gpu-budget.yaml`: shared GPU-budget manifest with one `gp-ctrl` DGD plus independent `model-a` and `model-b` DGDs
  - `cmd-gp-shared-gpu-budget.txt`: render, validate, deploy, inspect, test both frontends, and delete for the GPU-budget example
  - `README.md`: prerequisites, environment variables, and behavior notes
- `GAIE/`
  - tenant-scoped Gateway API Inference Extension setup for the same two models (`MODEL_A` agg, `MODEL_B` agg)
  - `gaie-tenant-base.yaml`: namespace + tenant gateway + RWX cache PVC
  - `cmd-gaie-two-models.txt`: render, validate, deploy, inspect, and tear down the whole tenant namespace
  - `README.md`: separates cluster-global prerequisites from the disposable namespace workflow
- `GenAI/`
  - generation-focused examples adapted from the current Dynamo docs
  - `text-to-image/`
    - `vllm-agg_qwen_image.yaml`: vLLM-Omni image generation with `Qwen/Qwen-Image`
    - `sglang-agg_flux_image.yaml`: SGLang image generation with `black-forest-labs/FLUX.1-dev`
  - `text-to-video/`
    - `vllm-agg_wan_t2v.yaml`: vLLM-Omni text-to-video with `Wan-AI/Wan2.1-T2V-1.3B-Diffusers`
  - `image-to-video/`
    - `vllm-agg_wan_i2v.yaml`: vLLM-Omni image-to-video with `Wan-AI/Wan2.2-TI2V-5B-Diffusers`
  - `llm-diffusion/`
    - `sglang-agg_llada2_mini.yaml`: SGLang diffusion LLM example with `inclusionAI/LLaDA2.0-mini-preview`
  - `trtllm-video-diffusion/`
    - `trtllm-agg_wan_t2v.yaml`: experimental TensorRT-LLM video diffusion example
  - `text-to-audio/`
    - placeholder only; no first-class Dynamo text-to-audio guide was found in the current public docs

## Prerequisites

- A Kubernetes cluster with NVIDIA GPU nodes (these manifests assume a GPU `runtimeClassName: nvidia`)
- NVIDIA AI Dynamo installed (CRD `kind: DynamoGraphDeployment` available)
- Namespace `dynamo-system` exists
- A Hugging Face token Kubernetes secret referenced by the manifests:
  - `hf-token-secret`

> **Install instructions:** **INSTALL_INSTRUCTIONS.md** covers both fresh install and platform upgrade to Dynamo `v1.0.1` for the **Kubernetes + Dynamo operator + DGD** scenario, including the Helm value changes needed to upgrade an older `0.8.1` deployment safely, post-upgrade checks, and rollback steps. Run that flow before applying the manifests in this repo.

> Note: Some multimodal examples prefer TCP request plane to avoid payload limits (see the image LLaVA manifest comments and `DYN_REQUEST_PLANE` usage).
>
> If UCX / RDMA is not available in tested environment (for example, `nixl` cannot find RDMA devices), you can safely fall back to **TCP-only UCX** by setting the UCX-related env vars in the manifests (as shown in the Qwen vision/audio/video examples) so that UCX uses `tcp` instead of attempting RDMA transports.
>
> GenAI note: first startup can be slow because models are downloaded from Hugging Face, and transient `429 Too Many Requests` errors are possible. The `trtllm-video-diffusion/` example is also more sensitive to host GPU driver compatibility than the vLLM and SGLang examples.

## Quick start

Pick an example folder and run the commands in its command helper file (`cmd.txt`
or `cmd-*.txt`).

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
