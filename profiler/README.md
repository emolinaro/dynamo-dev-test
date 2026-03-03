# Profiler

Manifests and commands for **SLA profiling** and **DynamoGraphDeploymentRequest (DGDR)** workflows: the controller uses a base disaggregated graph and SLA/hardware config to produce an optimized `DynamoGraphDeployment`.

**The profiler can be executed only in the disaggregated deployment scenario.** You must provide a disaggregated graph (e.g. Frontend + decode + prefill workers) as input; aggregated (single-worker) deployments are not used for profiling.

## Manifest files

| File | Purpose |
|------|--------|
| **disagg.yaml** | Base disaggregated vLLM graph: Frontend + `VllmDecodeWorker` + `VllmPrefillWorker` (Qwen3-0.6B). Used as the graph template for profiling. |
| **disagg-configmap.yaml** | ConfigMap that embeds `disagg.yaml` under the key `disagg.yaml`. Name: `qwen-config`, namespace: `dynamo-system`. Use this if you prefer `kubectl apply -f` instead of `kubectl create configmap --from-file`. |
| **profile_sla_aic_dgdr.yaml** | **DynamoGraphDeploymentRequest** for SLA profiling **with AiConfigurator**: `useAiConfigurator: true`, `aicSystem: h100_sxm`, `aicHfId`, `aicBackendVersion`. References `qwen-config` / `disagg.yaml` via `profilingConfig.configMapRef`. |
| **profile_sla_online_dgdr.yaml** | **DynamoGraphDeploymentRequest** for SLA profiling **without** AiConfigurator (`useAiConfigurator: false`). Same SLA/hardware and configMapRef; useful when AiConfigurator is not available. |

## Workflow

1. Create the ConfigMap from `disagg.yaml` (see `cmd.txt`: `kubectl create configmap` or `kubectl apply -f disagg-configmap.yaml`).
2. Apply one DGDR: `profile_sla_aic_dgdr.yaml` (with AIC) or `profile_sla_online_dgdr.yaml` (online only).
3. Tail the profiler job: `kubectl logs -f job/profile-qwen-sla-aic -n $NAMESPACE`.
4. Inspect outputs: `dgdr-output-qwen-sla-aic` and `planner-profile-data` ConfigMaps.
5. Extract the recommended `DynamoGraphDeployment` from the planner output (see last command in `cmd.txt`).

All commands assume `NAMESPACE=dynamo-system`; set it at the top of `cmd.txt` or in your shell.
