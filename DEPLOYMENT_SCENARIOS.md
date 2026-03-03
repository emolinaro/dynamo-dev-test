# Deployment scenarios

Two main deployment models for **NVIDIA AI Dynamo** on Kubernetes:

---

## 1. Kubernetes (Dynamo operator + DGD)

**Use case:** Production-style cluster deployments with operator-managed graphs.

| Aspect | Description |
|--------|-------------|
| **Deployments** | Declarative: `DynamoGraphDeployment` (DGD) and `DynamoGraphDeploymentRequest` (DGDR). Apply YAML; operator reconciles desired state. |
| **Autoscaling** | Autoscaling adapters integrate with the operator and workload metrics. |
| **Observability** | Metrics integration (e.g. Prometheus); operator and Grove expose cluster/graph metrics. |
| **Lifecycle** | CRDs, Helm (platform), and manifests drive install and updates. |

**This repository** (text, image, audio, video, profiler) is built for this scenario: apply DGD/DGDR manifests, use the Dynamo operator, and optionally run the profiler for disaggregated SLA tuning.

---

## 2. Kubernetes + GAIE/kGateway (Inference Gateway)

**Use case:** K8s-native ingress with **token-aware KV routing** at the gateway.

| Aspect | Description |
|--------|-------------|
| **Gateway** | Inference gateway (GAIE/kGateway) in front of the cluster. Handles ingress and request routing. |
| **Routing** | **Token-aware KV routing** in the gateway EPP (entry point / edge). Deterministic routing based on request/session identity for cache affinity. |
| **API model** | **Header-based deterministic routing** so the gateway can steer requests to the same backend when beneficial. Fits the **Gateway API** model and K8s-native ingress patterns. |

This scenario complements DGD-based deployments: the gateway layer adds routing and token-aware behavior; backend graphs can still be managed by the Dynamo operator (DGD/DGDR).

---

## Summary

| Scenario | Focus | This repo |
|----------|--------|-----------|
| **K8s + Dynamo operator + DGD** | Declarative DGD/DGDR, autoscaling, metrics, production clusters | ✅ Manifests and profiler target this |
| **K8s + GAIE/kGateway** | Gateway-level ingress, token-aware KV routing, Gateway API | Refer to gateway/inference-gateway docs |
