# GAIE

This folder contains a **tenant-scoped** GAIE example for the same two models
used in the GlobalPlanner examples:

- `MODEL_A`: aggregated vLLM behind EPP
- `MODEL_B`: aggregated vLLM behind EPP

The important boundary is:

- **cluster-global prerequisites** are documented here but are **not** managed by
  the command script
- the **runnable example** lives entirely in one namespace that you can remove
  with `kubectl delete namespace <name>`

## Safety Model

The disposable tenant flow in this folder only creates namespace-scoped
resources:

- `Namespace`
- `PersistentVolumeClaim`
- `Gateway`
- optional `hf-token-secret`
- `DynamoGraphDeployment`
- `HTTPRoute`

Deleting that namespace removes the whole example.

It does **not** install or modify cluster-global components such as:

- Gateway API CRDs
- Gateway API Inference Extension CRDs
- `kgateway` controller
- `GatewayClass`

That separation is what makes this setup non-disruptive for an already-running
cluster.

## Cluster Prerequisites

These are **one-time cluster admin steps**, not part of the disposable tenant
install:

1. Install Gateway API CRDs
2. Install Gateway API Inference Extension CRDs
3. Install `kgateway` and its CRDs in `kgateway-system`
4. Ensure a `GatewayClass` named `kgateway` exists

This follows the upstream GAIE cluster-prerequisite flow. The links below track
the release-aligned `v1.1.0` install guidance, while the tenant manifests in
this folder follow the same aggregated example shape and `1.1.0` image set
called out later in this README:

- [install_gaie_crd_kgateway.sh](https://github.com/ai-dynamo/dynamo/blob/v1.1.0/deploy/inference-gateway/scripts/install_gaie_crd_kgateway.sh)

Upstream references:

- [Dynamo recipes — GAIE integration](https://github.com/ai-dynamo/dynamo/blob/v1.1.0/recipes/README.md#inference-gateway-gaie-integration-optional)
- [Inference Gateway guide](https://github.com/ai-dynamo/dynamo/blob/v1.1.0/docs/kubernetes/inference-gateway.md)

Once those prerequisites are present, everything else in this folder can live in
its own namespace.

## Files

| File | Purpose |
|------|---------|
| `cmd-gaie-cluster-prereqs.txt` | One-time cluster-global install for Gateway API, GAIE CRDs, and `kgateway` |
| `cmd-gaie-cluster-remove.txt` | Cluster-global rollback helper for `kgateway` and, optionally, GAIE/Gateway API CRDs |
| `gaie-tenant-base.yaml` | Namespace + RWX cache PVC + namespaced `Gateway` |
| `gaie-model-a-agg.yaml` | `MODEL_A` DGD with `Epp` + aggregated worker |
| `gaie-model-b-agg.yaml` | `MODEL_B` DGD with `Epp` + aggregated worker |
| `http-route-model-a.yaml` | `HTTPRoute` for `${GAIE_DGD_MODEL_A}-pool` |
| `http-route-model-b.yaml` | `HTTPRoute` for `${GAIE_DGD_MODEL_B}-pool` |
| `cmd-gaie-two-models.txt` | Render, validate, create the tenant namespace stack, and inspect it |

## Defaults

The command flow now defaults to a disposable namespace instead of
`dynamo-system`:

- `K8S_NAMESPACE=gaie-demo`
- `GAIE_GATEWAY_NAMESPACE=$K8S_NAMESPACE`
- `GAIE_GATEWAY_NAME=inference-gateway`
- `GAIE_GATEWAY_CLASS_NAME=kgateway`
- `STORAGE_CLASS_NAME=csi-cephfs-sc`
- `DYNAMO_VLLM_IMAGE=nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.0`
- `DYNAMO_EPP_IMAGE=nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.0`
- `GAIE_CACHE_PVC_NAME=hf-model-cache`
- `GAIE_HF_SECRET_NAME=hf-token-secret`

So the default shape is:

- one tenant namespace: `gaie-demo`
- one tenant `Gateway` in `gaie-demo`
- both `HTTPRoute`s in `gaie-demo`
- both DGDs in `gaie-demo`
- one tenant-local cache PVC in `gaie-demo`
- both model DGDs follow the current aggregated GAIE pattern using
  `extraPodSpec.containers` for the direct worker frontend

## Controlled Namespace Workflow

1. Confirm the cluster-global GAIE prerequisites already exist.
   If they do not, run:

   ```bash
   export CONFIRM_CLUSTER_PREREQS=yes
   bash cmd-gaie-cluster-prereqs.txt
   ```

   That script prints a warning and requires explicit confirmation because it
   installs cluster-global CRDs and the shared `kgateway` controller.

2. Choose a tenant namespace, for example:

   ```bash
   export K8S_NAMESPACE=gaie-demo
   ```

3. If you want the script to create the tenant-local Hugging Face secret too:

   ```bash
   export HF_TOKEN=hf_...
   ```

4. Run:

   ```bash
   bash cmd-gaie-two-models.txt
   ```

The script will:

- render `gaie-tenant-base.yaml`
- create the namespace, PVC, and namespaced `Gateway`
- create or update `hf-token-secret` in that namespace if `HF_TOKEN` is set
- apply the two DGDs
- apply the two `HTTPRoute`s
- leave you with a gateway URL that can be smoke-tested with the commands in
  `../API_TESTING.md` by adding the appropriate `Host:` header

## Teardown

The safe teardown path is the whole point of this layout:

```bash
kubectl delete namespace "$K8S_NAMESPACE"
```

That removes:

- the tenant `Gateway`
- the tenant PVC
- the tenant secret
- the two DGDs
- the two `HTTPRoute`s

It does **not** touch:

- `kgateway-system`
- cluster CRDs
- `GatewayClass`
- any existing Dynamo workloads in other namespaces

## Cluster Rollback

If you want to undo the shared GAIE installation layer too, use:

```bash
export CONFIRM_CLUSTER_REMOVE=yes
bash cmd-gaie-cluster-remove.txt
```

That script is intentionally conservative:

- it removes the `kgateway` Helm releases
- it can optionally delete tenant namespaces first if you set
  `GAIE_TENANT_NAMESPACES="ns1 ns2"`
- it removes the `GatewayClass` by default
- it does **not** remove Gateway API or GAIE CRDs unless you opt in

For full rollback of the cluster-global APIs as well:

```bash
export CONFIRM_CLUSTER_REMOVE=yes
export REMOVE_GAIE_CRDS=yes
export REMOVE_GATEWAY_API_CRDS=yes
bash cmd-gaie-cluster-remove.txt
```

Use that full CRD removal only if nothing else in the cluster depends on those
APIs.

## Notes

- The model manifests now use `${GAIE_HF_SECRET_NAME}` and
  `${GAIE_CACHE_PVC_NAME}` so they can stay fully tenant-local.
- The GAIE command now defines both `DYNAMO_VLLM_IMAGE` and `DYNAMO_EPP_IMAGE`
  explicitly, so worker images and the EPP image can be overridden
  independently.
- This tenant example now follows the release-aligned
  [aggregated GAIE example](https://github.com/ai-dynamo/dynamo/blob/v1.1.0/examples/backends/vllm/deploy/gaie/agg.yaml),
  which models the worker-local direct frontend via `extraPodSpec.containers`
  rather than `frontendSidecar`.
- On the current cluster, the worker `main` containers are also pinned to the
  base graph `DYN_NAMESPACE` with an empty `DYN_NAMESPACE_WORKER_SUFFIX` so
  backend endpoint registration stays aligned with the namespace EPP uses for
  router initialization.
- The `HTTPRoute`s now default to the tenant `Gateway`, not to a shared gateway
  namespace.
- This folder is intentionally separate from `GlobalPlanner/` now. GAIE is an
  adjacent deployment pattern, not part of the shared-budget planner flow.
- If you need LoRA, upstream guidance is still to avoid the inference gateway
  path.
