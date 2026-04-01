# Profiler

Manifests and commands for **SLA profiling** and **DynamoGraphDeploymentRequest (DGDR)** workflows on Dynamo `1.0.1`. The profiler uses a disaggregated base graph plus SLA and hardware constraints to generate an optimized `DynamoGraphDeployment`.

**The profiler can be executed only in the disaggregated deployment scenario.** Aggregated single-worker graphs are not profiled. In `1.0.1`, the profiler request API is `nvidia.com/v1beta1`, and the base disaggregated graph is passed inline through `spec.overrides.dgd`.

## Manifest files

| File | Purpose |
|------|--------|
| **disagg.yaml** | Standalone disaggregated vLLM graph: Frontend + `VllmDecodeWorker` + `VllmPrefillWorker` (Qwen3-0.6B), updated to `vllm-runtime:1.0.1`. For current vLLM disaggregation, the workers use `--disaggregation-mode {decode|prefill}` plus an explicit `--kv-transfer-config` for `NixlConnector`. The same layout is embedded into the DGDR examples through `spec.overrides.dgd`. |
| **profile_sla_aic_dgdr.yaml** | `nvidia.com/v1beta1` DGDR using `searchStrategy: rapid`. Keep this as a reference manifest only for now: current Dynamo `1.0.1` docs/runtime are inconsistent about vLLM rapid-path support, and the stable profiler feature matrix still marks offline AIC support as TensorRT-LLM-only. The worker overrides intentionally do not pin GPU limits so the profiler can choose GPUs per engine, while `hardware.totalGpus: 4` keeps the search bounded to a single 4-GPU node budget. |
| **profile_sla_online_dgdr.yaml** | `nvidia.com/v1beta1` DGDR using `searchStrategy: thorough` for online profiling against real GPU-backed deployments. This is the recommended and reliable path for vLLM on Dynamo `1.0.1`. Uses the same embedded disaggregated base DGD, but leaves worker GPU limits unset so the profiler can inject TP/engine sizing, while `hardware.totalGpus: 4` constrains the final selected deployment to your 1-node/4-GPU budget. During profiling, the controller still benchmarks standalone prefill and decode candidates up to that size. |

## Workflow

1. For vLLM on Dynamo `1.0.1`, apply `profile_sla_online_dgdr.yaml` and use the thorough online path. Treat `profile_sla_aic_dgdr.yaml` as reference-only until the rapid vLLM path is fixed upstream.
2. Watch request state with `kubectl get dgdr` or `kubectl describe dgdr <name>`.
3. Tail the profiler job: `kubectl logs -f job/profile-<dgdr-name> -n $NAMESPACE`.
4. Inspect the generated output ConfigMap: `dgdr-output-<dgdr-name>`.
5. Extract the generated `DynamoGraphDeployment` from `.status.profilingResults.selectedConfig`, or inspect the auto-applied DGD directly if `autoApply: true`.

All commands assume `NAMESPACE=dynamo-system`; set it at the top of `cmd.txt` or in your shell.

For Dynamo `1.0.1`, vLLM disaggregation no longer accepts the older `--is-decode-worker` / `--is-prefill-worker` flags. Use `--disaggregation-mode decode|prefill` and set `--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both"}'` explicitly on both workers.

If the rapid profiler job fails with a pandas `KeyError` while selecting candidate configs, that is consistent with the current support gap for vLLM offline profiling. In practice, switch to `searchStrategy: thorough` instead of trying to tune the rapid request.

For a single-node 4-GPU environment, `searchStrategy: thorough` still creates temporary TP1/TP2/TP4 prefill and decode candidates as separate benchmark deployments. That means the node must have 4 GPUs free for the TP4 candidate runs even though the final chosen prefill+decode deployment is constrained by `hardware.totalGpus: 4`.
