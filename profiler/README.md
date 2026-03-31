# Profiler

Manifests and commands for **SLA profiling** and **DynamoGraphDeploymentRequest (DGDR)** workflows on Dynamo `1.0.1`. The profiler uses a disaggregated base graph plus SLA and hardware constraints to generate an optimized `DynamoGraphDeployment`.

**The profiler can be executed only in the disaggregated deployment scenario.** Aggregated single-worker graphs are not profiled. In `1.0.1`, the profiler request API is `nvidia.com/v1beta1`, and the base disaggregated graph is passed inline through `spec.overrides.dgd`.

## Manifest files

| File | Purpose |
|------|--------|
| **disagg.yaml** | Standalone disaggregated vLLM graph: Frontend + `VllmDecodeWorker` + `VllmPrefillWorker` (Qwen3-0.6B), updated to `vllm-runtime:1.0.1`. The same layout is embedded into the DGDR examples through `spec.overrides.dgd`. |
| **profile_sla_aic_dgdr.yaml** | `nvidia.com/v1beta1` DGDR using `searchStrategy: rapid` for the fast profiling path. Keeps the cluster-specific runtime class, GPU node selector, and `hf-token-secret` wiring through the embedded base DGD. |
| **profile_sla_online_dgdr.yaml** | `nvidia.com/v1beta1` DGDR using `searchStrategy: thorough` for online profiling against real GPU-backed deployments. Uses the same embedded disaggregated base DGD. |

## Workflow

1. Apply one DGDR: `profile_sla_aic_dgdr.yaml` for the rapid path or `profile_sla_online_dgdr.yaml` for the thorough online path.
2. Watch request state with `kubectl get dgdr` or `kubectl describe dgdr <name>`.
3. Tail the profiler job: `kubectl logs -f job/profile-<dgdr-name> -n $NAMESPACE`.
4. Inspect the generated output ConfigMap: `dgdr-output-<dgdr-name>`.
5. Extract the generated `DynamoGraphDeployment` from `.status.profilingResults.selectedConfig`, or inspect the auto-applied DGD directly if `autoApply: true`.

All commands assume `NAMESPACE=dynamo-system`; set it at the top of `cmd.txt` or in your shell.
