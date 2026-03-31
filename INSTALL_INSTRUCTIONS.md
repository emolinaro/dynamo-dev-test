# Dynamo Install / Upgrade Instructions

This document installs Dynamo `v1.0.1` on a fresh cluster or upgrades an
existing `dynamo-platform` release, including the `0.8.1` layout that this repo
originally documented.

The commands below preserve the current repo topology:

- namespace-restricted operator in `dynamo-system`
- bundled `etcd`
- Grove disabled
- no `kai-scheduler`
- `csi-rbd-sc` storage class for etcd and NATS JetStream
- Prometheus endpoint at `http://prometheus-server.monitoring.svc.cluster.local`

Do not rely on `--reuse-values` for a `0.8.1 -> 1.0.1` upgrade. Several Helm
keys moved in `1.0.x`, so the explicit flags below preserve the existing
behavior.

This flow intentionally disables Grove. If you are upgrading an older release
that included bundled Grove, Helm will remove the Grove operator and its webhook
resources during the upgrade. That also avoids the `grove-webhook-server-cert`
server-side-apply conflict seen when the old Grove operator is still rotating
that secret.

Because this repo uses `dynamo-operator.namespaceRestriction.enabled=true`, do
not let the operator pod perform CRD creation during upgrade. The namespace-
restricted service account cannot create new cluster-scoped CRDs. Apply the CRDs
first as your own cluster identity, then run Helm with
`dynamo-operator.upgradeCRD=false`.

## Prerequisites: Hugging Face token and GPU nodes

Set the target release details first:

```bash
# Optional if you keep multiple kubeconfigs.
# export KUBECONFIG=/path/to/your/kubeconfig

export NAMESPACE=dynamo-system
export RELEASE_NAME=dynamo-platform
export RELEASE_VERSION=1.0.1
export CHART_URL="https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-${RELEASE_VERSION}.tgz"
```

Create or update the Hugging Face token secret:

```bash
kubectl create ns "$NAMESPACE" 2>/dev/null || true

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<insert_huggingface_token> \
  --dry-run=client -o yaml | kubectl apply -f -
```

Check nodes with GPUs:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUS:.status.allocatable.nvidia\\.com/gpu
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}' | awk '$2!="" && $2!="0"{print}'
```

---

## Snapshot the current release before upgrading

Recommended when upgrading an existing cluster:

```bash
helm list -n "$NAMESPACE"
helm history "$RELEASE_NAME" -n "$NAMESPACE"
helm get values "$RELEASE_NAME" -n "$NAMESPACE" -o yaml > /tmp/${RELEASE_NAME}.values.before-${RELEASE_VERSION}.yaml || true

kubectl -n "$NAMESPACE" get pods,svc
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration | egrep 'dynamo|podclique|grove' || true
kubectl get crd | grep -i dynamo
```

---

## Fetch the v1.0.1 chart

```bash
helm pull "$CHART_URL" -d /tmp
helm show chart /tmp/dynamo-platform-${RELEASE_VERSION}.tgz
mkdir -p /tmp/dynamo-platform-${RELEASE_VERSION}
tar -xf /tmp/dynamo-platform-${RELEASE_VERSION}.tgz -C /tmp/dynamo-platform-${RELEASE_VERSION}
```

---

## Apply CRDs manually before Helm

This step is required for the namespace-restricted operator flow used here.
Use server-side apply so `kubectl` does not try to write a huge
`last-applied-configuration` annotation onto large CRDs, and use
`--force-conflicts` because older CRDs may still be owned by the previous
`dynamo-crds` field manager.

```bash
kubectl apply --server-side --force-conflicts \
  -f /tmp/dynamo-platform-${RELEASE_VERSION}/dynamo-platform/charts/dynamo-operator/crds/

kubectl get crd | grep -E 'dynamo(checkpoints|componentdeployments|graphdeploymentrequests|graphdeployments|graphdeploymentscalingadapters|models|workermetadatas)\.nvidia\.com'
```

---

## Dry-run the install / upgrade

```bash
helm upgrade --install "$RELEASE_NAME" /tmp/dynamo-platform-${RELEASE_VERSION}.tgz \
  -n "$NAMESPACE" --create-namespace \
  --dry-run --debug \
  --set dynamo-operator.upgradeCRD=false \
  --set dynamo-operator.namespaceRestriction.enabled=true \
  --set global.etcd.install=true \
  --set etcd.persistence.storageClass="csi-rbd-sc" \
  --set global.grove.install=false \
  --set global.grove.enabled=false \
  --set 'global.kai-scheduler.install=false' \
  --set 'global.kai-scheduler.enabled=false' \
  --set nats.config.jetstream.fileStore.pvc.storageClassName="csi-rbd-sc" \
  --set-string dynamo-operator.dynamo.metrics.prometheusEndpoint="http://prometheus-server.monitoring.svc.cluster.local"
```

`1.0.x` key changes covered by the command above:

- `grove.enabled` -> `global.grove.install` and `global.grove.enabled`
- `kai-scheduler.enabled` -> `global.kai-scheduler.install` and `global.kai-scheduler.enabled`
- `prometheusEndpoint` -> `dynamo-operator.dynamo.metrics.prometheusEndpoint`
- `etcd` is no longer installed by default, so `global.etcd.install=true` preserves the old layout
- `dynamo-operator.upgradeCRD=false` is intentional here because CRDs are applied manually before Helm

---

## Install or upgrade Dynamo to v1.0.1

```bash
helm upgrade --install "$RELEASE_NAME" /tmp/dynamo-platform-${RELEASE_VERSION}.tgz \
  -n "$NAMESPACE" --create-namespace \
  --wait --timeout 30m \
  --set dynamo-operator.upgradeCRD=false \
  --set dynamo-operator.namespaceRestriction.enabled=true \
  --set global.etcd.install=true \
  --set etcd.persistence.storageClass="csi-rbd-sc" \
  --set global.grove.install=false \
  --set global.grove.enabled=false \
  --set 'global.kai-scheduler.install=false' \
  --set 'global.kai-scheduler.enabled=false' \
  --set nats.config.jetstream.fileStore.pvc.storageClassName="csi-rbd-sc" \
  --set-string dynamo-operator.dynamo.metrics.prometheusEndpoint="http://prometheus-server.monitoring.svc.cluster.local"
```

---

## Immediate post-install / post-upgrade checks

```bash
helm list -n "$NAMESPACE"
helm history "$RELEASE_NAME" -n "$NAMESPACE"

kubectl get crd | grep -i dynamo

kubectl -n "$NAMESPACE" get pods -o wide
kubectl -n "$NAMESPACE" get svc | egrep -i 'webhook|dynamo-operator|etcd|nats' || true
kubectl -n "$NAMESPACE" get statefulset

kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration | egrep 'dynamo|podclique|grove' || true

kubectl -n "$NAMESPACE" rollout status deploy/dynamo-platform-dynamo-operator-controller-manager --timeout=10m
```

If Grove was present before the upgrade, you should no longer see the
`grove-operator` deployment after this flow:

```bash
kubectl -n "$NAMESPACE" get deploy | egrep 'dynamo-platform-dynamo-operator|grove' || true
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration | egrep 'podclique|grove' || true
```

If you already have `DynamoGraphDeployment` or `DynamoGraphDeploymentRequest`
objects in the cluster, re-check them after the platform is healthy:

```bash
kubectl -n "$NAMESPACE" get dynamographdeployments,dynamographdeploymentrequests
```

If you are recovering from a failed rollout where the operator is stuck in
`Init:CrashLoopBackOff`, the usual reason is the `crd-apply` init container
trying to create a new CRD without cluster-scope permissions. In that case:

```bash
kubectl apply --server-side --force-conflicts \
  -f /tmp/dynamo-platform-${RELEASE_VERSION}/dynamo-platform/charts/dynamo-operator/crds/

helm upgrade --install "$RELEASE_NAME" /tmp/dynamo-platform-${RELEASE_VERSION}.tgz \
  -n "$NAMESPACE" --create-namespace \
  --wait --timeout 30m \
  --set dynamo-operator.upgradeCRD=false \
  --set dynamo-operator.namespaceRestriction.enabled=true \
  --set global.etcd.install=true \
  --set etcd.persistence.storageClass="csi-rbd-sc" \
  --set global.grove.install=false \
  --set global.grove.enabled=false \
  --set 'global.kai-scheduler.install=false' \
  --set 'global.kai-scheduler.enabled=false' \
  --set nats.config.jetstream.fileStore.pvc.storageClassName="csi-rbd-sc" \
  --set-string dynamo-operator.dynamo.metrics.prometheusEndpoint="http://prometheus-server.monitoring.svc.cluster.local"
```

---

## Roll back the upgrade if needed

```bash
helm history "$RELEASE_NAME" -n "$NAMESPACE"

# Replace <previous_revision> with the revision you want to return to.
helm rollback "$RELEASE_NAME" <previous_revision> -n "$NAMESPACE" --wait --timeout 30m

kubectl -n "$NAMESPACE" get pods -o wide
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration | egrep 'dynamo|podclique|grove' || true
```

If this was a fresh install and you want to remove the platform entirely:

```bash
helm uninstall "$RELEASE_NAME" -n "$NAMESPACE"
```

---

## Important follow-up for this repository

Upgrading the platform does not automatically update the example graph manifests
in this repo. Many of them still pin old runtime tags such as `0.8.1` and one
LLaVA example pins `0.9.0`.

If you later need Grove for multinode orchestration, install or enable it
separately after the platform upgrade instead of bundling it into this default
platform flow.

Before reapplying the example manifests from this repo, inspect the pinned image
tags and bump them to the runtime version you actually want to run:

```bash
rg -n 'nvcr.io/nvidia/ai-dynamo/.+:(0\\.8\\.1|0\\.9\\.0)' text image audio video profiler
```
