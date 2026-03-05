# Deployment scenarios

Two main deployment models for **NVIDIA AI Dynamo** on Kubernetes are detailed below. For a comparison across **all** scenarios (including local and disaggregated), see the [Scenario comparison](#scenario-comparison) table.

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

## Scenario comparison

| Scenario | Primary purpose | Scale suitability | HA posture | Multi-model support | Key coordination / discovery | Network requirements | Typical cost drivers |
|----------|-----------------|-------------------|------------|----------------------|------------------------------|----------------------|----------------------|
| **Local single-node (file-KV)** | Fast dev/test without infra | Single machine | None (single process/host) | Possible, but dev-focused | File KV store; requires shared disk between frontend/workers | None beyond localhost | 1 GPU + local storage; model downloads each run if not cached |
| **Local single-node (etcd + NATS via Docker Compose)** | Higher-fidelity local topology; basic multi-process | Single machine (can simulate multiple workers) | Limited (Compose is usually not HA) | Yes (run multiple workers/models) | etcd discovery default; NATS+JetStream infrastructure | Localhost; optional host networking constraints | 1 machine + infra containers + storage for caches |
| **Kubernetes operator-managed (Dynamo operator + DGD/DynamoModel)** | Production-style deployments with CRDs | Single-node to multi-node | Kubernetes replicas + scheduler; operator modes | Strong (multiple DGDs/namespaces; DynamoModel for LoRA) | Kubernetes-native discovery with operator (no external etcd required for discovery) | Standard K8s networking; high-speed recommended at scale | GPU fleet + storage/PVC + observability stack |
| **Kubernetes multi-node disaggregated (RDMA)** | Max throughput under SLA; prefill/decode split | Multi-node GPU clusters | Depends on orchestration and redundancy | Per-model topology; can run multiple DGDs | Disaggregated guide explicitly requires etcd+NATS for coordination; RDMA for KV transfer | RDMA required; InfiniBand/RoCE recommended | RDMA fabric (NICs/switch), GPU count for separate prefill/decode, tuning time |
| **Kubernetes + GAIE/kGateway** | Gateway-layer token-aware routing / traffic management | Cluster-scale ingress for many endpoints/models | Gateway HA + backend HA | Yes; gateway routes to multiple model backends | Dynamo router embedded in EPP (token-aware); routing via headers | As per backend; RDMA still needed for disagg KV transfer | Gateway infra + EPP compute + operational complexity |

---

## Summary

| Scenario | Focus | This repo |
|----------|--------|-----------|
| **K8s + Dynamo operator + DGD** | Declarative DGD/DGDR, autoscaling, metrics, production clusters | ✅ Manifests and profiler target this |
| **K8s + GAIE/kGateway** | Gateway-level ingress, token-aware KV routing, Gateway API | Refer to gateway/inference-gateway docs |
