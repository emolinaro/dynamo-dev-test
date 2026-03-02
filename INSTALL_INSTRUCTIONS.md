# Dynamo Install Instructions

## Prerequisites: Hugging Face token and GPU nodes

Create the Hugging Face token secret:

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<insert_huggingface_token>
```

Check nodes with GPUs:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUS:.status.allocatable.nvidia\\.com/gpu
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}' | awk '$2!="" && $2!="0"{print}'
```

---

## Install

```bash
export NAMESPACE=dynamo-system
export RELEASE_VERSION=0.8.1

kubectl create ns $NAMESPACE 2>/dev/null || true

# CRDs (rendered apply preferred)
helm template dynamo-crds "dynamo-crds-${RELEASE_VERSION}.tgz" > /tmp/dynamo-crds.rendered.yaml
kubectl apply -f /tmp/dynamo-crds.rendered.yaml

# Platform
helm install dynamo-platform "dynamo-platform-${RELEASE_VERSION}.tgz" \
  -n "$NAMESPACE" --create-namespace \
  --set dynamo-operator.namespaceRestriction.enabled=true \
  --set grove.enabled=true \
  --set kai-scheduler.enabled=false \
  --set prometheusEndpoint="http://prometheus-server.monitoring.svc.cluster.local" \
  --set etcd.persistence.storageClass="csi-rbd-sc" \
  --set nats.config.jetstream.fileStore.pvc.storageClassName="csi-rbd-sc" \
  --wait --timeout 30m
```

---

## Immediate post-install checks (read-only)

```bash
kubectl get crd | grep grove

kubectl -n $NAMESPACE get pods -o wide
kubectl -n $NAMESPACE get svc | egrep -i 'webhook|grove-operator|dynamo-operator' || true

kubectl get mutatingwebhookconfiguration podcliqueset-defaulting-webhook -o yaml | egrep -n 'failurePolicy|namespaceSelector|clientConfig|resources|apiGroups'
kubectl get validatingwebhookconfiguration podcliqueset-validating-webhook -o yaml | egrep -n 'failurePolicy|namespaceSelector|clientConfig|resources|apiGroups'
kubectl get validatingwebhookconfiguration dynamo-platform-dynamo-operator-validating-dynamo-system -o yaml | egrep -n 'failurePolicy|namespaceSelector|clientConfig|resources|apiGroups'
```

---

## Stop and rollback (if something feels wrong)

```bash
kubectl delete mutatingwebhookconfiguration podcliqueset-defaulting-webhook
kubectl delete validatingwebhookconfiguration podcliqueset-validating-webhook
kubectl delete validatingwebhookconfiguration dynamo-platform-dynamo-operator-validating-dynamo-system

helm uninstall dynamo-platform -n $NAMESPACE
```
