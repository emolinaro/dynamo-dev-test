# GlobalPlanner

This folder adapts the upstream Dynamo global planner examples into the same
repo-local Kubernetes style used elsewhere in this repository.

It currently includes:

- **Shared GPU budget:** `global-planner-shared-gpu-budget.yaml` — one shared
  `GlobalPlanner` plus two independent model DGDs, each with its own frontend.

## Files

| File | Purpose |
|------|---------|
| `global-planner-shared-gpu-budget.yaml` | `envsubst`-driven mixed-topology manifest containing the planner `ClusterRoleBinding`, shared RWX model-cache PVC, and the three DGDs (`gp-ctrl`, `model-a`, `model-b`) |
| `cmd-gp-shared-gpu-budget.txt` | Render, validate, deploy, test, inspect, and delete flow for `global-planner-shared-gpu-budget.yaml` |

## Prerequisites

- Dynamo `1.0.1` installed on Kubernetes with the operator and CRDs available
- target namespace available, usually `dynamo-system`
- `hf-token-secret` present in that namespace
- the operator-installed ClusterRole `dynamo-platform-dynamo-operator-planner`
- Prometheus scraping available for planner metrics
- an RWX-capable `StorageClass` for the shared `hf-model-cache` PVC
  - `csi-rbd-sc` is not suitable for these examples on clusters where it does
    not provide `ReadWriteMany`
- GPU nodes matching this repo's current examples:
  - `runtimeClassName: nvidia`
  - `ucloud.dk/gpu: "true"`

## Environment

`cmd-gp-shared-gpu-budget.txt` uses the following variables:

- required:
  - `STORAGE_CLASS_NAME`
  - `DYNAMO_IMAGE`
  - `DYNAMO_VLLM_IMAGE`
- optional with defaults:
  - `K8S_NAMESPACE` defaults to `dynamo-system`
  - `MODEL_A` defaults to `meta-llama/Llama-3.2-3B-Instruct`
  - `MODEL_B` defaults to `Qwen/Qwen2.5-7B-Instruct`
  - `MODEL_A_MAX_GPU_BUDGET` defaults to `2`
  - `MODEL_B_MAX_GPU_BUDGET` defaults to `1`
  - `PROFILE_RESULTS_STAGING_ROOT` defaults to `/profile-results`
  - `MODEL_A_PROFILE_RESULTS_SOURCE_DIR` defaults to `/workspace/tests/planner/profiling_results/H200_TP1P_TP1D`
  - `MODEL_A_PROFILE_RESULTS_ALIAS_NAME` defaults to `H100_TP1P_TP1D`
  - `MODEL_A_PROFILE_RESULTS_DIR`

If you do not have a separate base image for the frontend and planner, it is
fine to point both `DYNAMO_IMAGE` and `DYNAMO_VLLM_IMAGE` at
`nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.1`, which matches the current repo
examples.

By default, the disaggregated `model-a` planner stages the bundled Dynamo
`v1.0.1` profiling fixture into a dedicated alias directory inside the
container:

- source fixture directory:
  `/workspace/tests/planner/profiling_results/H200_TP1P_TP1D`
- staged alias directory used by the planner config:
  `/profile-results/H100_TP1P_TP1D`

This keeps the visible path less confusing on H100 clusters, but it is still
copying the bundled fixture data unless you override the source directory. If
you need hardware-specific results, override
`MODEL_A_PROFILE_RESULTS_SOURCE_DIR` with your real profiler output, or adjust
the alias name and target directory before rendering the manifest.

If you previously exported the old path
`/workspace/components/src/dynamo/planner/tests/data/profiling_results/H100_TP1P_TP1D`
in your shell, unset it before rendering again:

`unset MODEL_A_PROFILE_RESULTS_DIR`

The planner expects `profile_results_dir` to contain:

- `selected_prefill_interpolation/raw_data.npz` or `prefill_raw_data.json`
- `selected_decode_interpolation/raw_data.npz` or `decode_raw_data.json`

If those files are missing, the planner will fail during startup with the
`FileNotFoundError` you saw from `perf_interpolation.py`.

## Shared GPU Budget

This example keeps the control plane minimal while fitting a 3-GPU cluster:

- one `gp-ctrl` DGD running only `GlobalPlanner`
- one `model-a` DGD with its own frontend, disaggregated workers, and a global-planner-aware planner
- one `model-b` DGD with its own frontend, one aggregated vLLM worker, and a local aggregated planner

Important: this is a shared-budget multi-model example with two independent
frontend services, not a single routed endpoint.

### Workflow

1. Export your environment variables and render the manifest with `envsubst`.
2. Confirm the rendered file contains exactly 1 `ClusterRoleBinding`, 1 PVC, and
   3 `DynamoGraphDeployment` objects with no unresolved `${...}` variables.
3. Run `kubectl apply --dry-run=client` against the rendered manifest, then
   `kubectl apply`.
4. Wait for `gp-ctrl`, `model-a`, and `model-b` to reconcile.
5. Port-forward `svc/model-a-frontend` and `svc/model-b-frontend`, then send one
   OpenAI-compatible chat completion request to each frontend.

### Budget Behavior

This runtime does not accept the upstream `--max-total-gpus` CLI flag on
`dynamo.global_planner`, which is why `gp-ctrl` crashed with:

```text
__main__.py: error: unrecognized arguments: --max-total-gpus 4
```

To keep the example runnable on Dynamo `1.0.1`, the shared-budget manifest now
uses planner-side `max_gpu_budget` values instead. With the documented defaults
`MODEL_A_MAX_GPU_BUDGET=2` and `MODEL_B_MAX_GPU_BUDGET=1`, the steady-state
baseline is:

- `model-a`: 1 prefill GPU + 1 decode GPU
- `model-b`: 1 aggregated vLLM GPU

That keeps the combined baseline at 3 GPUs, which fits clusters that only expose
three allocatable full `nvidia.com/gpu` devices.

To inspect this behavior under load:

- watch the DGDs and pods in the namespace while requests are in flight
- inspect the `model-a` and `model-b` planner pod logs for attempted scale
  requests that cannot be admitted because they would exceed
  `max_gpu_budget`
- inspect the `gp-ctrl` pod logs to confirm the shared planner itself is up and
  seeing the namespace participants

Use `kubectl get pods -n $K8S_NAMESPACE | rg 'gp-ctrl|model-a|model-b'` to
find the pod names to inspect.

## Notes

- This example keeps the global planner scope implicit. Because it does not pass
  `--managed-namespaces`, all DGDs in the same Kubernetes namespace are visible
  to the shared `GlobalPlanner`.
- `global-planner-shared-gpu-budget.yaml` uses both current vLLM command
  shapes:
  - `model-a` uses the Dynamo `1.0.1` disaggregation syntax
    `--disaggregation-mode decode|prefill` plus
    `--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both"}'`
  - `model-b` uses the aggregated syntax `python3 -m dynamo.vllm --model ...`
- The manifest mounts a shared RWX Hugging Face cache PVC into the worker pods at
  `/home/dynamo/.cache/huggingface/hub`.
- `global-planner-shared-gpu-budget.yaml` is intentionally mixed-topology:
  `model-a` is disaggregated, while `model-b` uses a single aggregated vLLM
  worker so the example can fit on a 3-GPU cluster.
- `global-planner-shared-gpu-budget.yaml` uses planner-side
  `max_gpu_budget` values because the current `dynamo.global_planner` CLI does
  not accept the upstream `--max-total-gpus` flag.
