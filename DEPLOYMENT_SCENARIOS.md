# Deployment Scenarios

Updated against NVIDIA Dynamo documentation on March 31, 2026. This reflects the current `latest` docs for local install, Kubernetes deployment, operator behavior, and disaggregated serving, plus the current GAIE guide published under the Dynamo `dev` docs path.

## Scenario Map

### 1. Local single-machine: file discovery

**Use case:** Fastest path for local development on one machine or VM.

| Aspect | Description |
|--------|-------------|
| **Discovery** | Use `--discovery-backend file` for all components. No etcd setup required. |
| **Infra dependencies** | None required. For vLLM, disable KV event publishing explicitly with `--kv-events-config '{"enable_kv_cache_events": false}'` to avoid needing NATS. |
| **Runtime shape** | Frontend and one or more workers on the same machine. Can be run in separate terminals or one container with background processes. |
| **Best for** | Bring-up, debugging, CLI-based testing, and small single-node experiments. |

### 2. Local single-machine: Docker Compose infra

**Use case:** Higher-fidelity local topology with infrastructure services present.

| Aspect | Description |
|--------|-------------|
| **Discovery** | etcd is the default local discovery backend when you do not override it. |
| **Infra dependencies** | Start etcd and optionally NATS with Docker Compose. NATS is only needed when using KV routing with events. |
| **Runtime shape** | Frontend and workers still run on a single machine, but coordination is closer to a distributed deployment. |
| **Best for** | Testing KV routing, event-driven flows, local observability stacks, and multi-process behavior before moving to Kubernetes. |

### 3. Kubernetes platform: Dynamo operator-managed

**Use case:** Recommended Kubernetes deployment model for most cluster environments.

| Aspect | Description |
|--------|-------------|
| **Deployments** | Declarative custom resources: `DynamoGraphDeployment` (DGD), `DynamoGraphDeploymentRequest` (DGDR), `DynamoModel`, and lower-level `DynamoComponentDeployment` when needed. |
| **Discovery** | Kubernetes-native discovery is the default and recommended mode. The operator injects `DYN_DISCOVERY_BACKEND=kubernetes` and uses `DynamoWorkerMetadata` plus `EndpointSlices`; external etcd is not required for standard operator-managed discovery. |
| **Operator modes** | Supports cluster-wide mode, namespace-scoped mode, and hybrid mode. Namespace-scoped mode is the current shared-cluster path; hybrid mode allows a cluster-wide operator plus isolated namespace-scoped operators. |
| **Observability** | Operator metrics are exposed via Prometheus and ServiceMonitor integration; Grafana dashboards are available. |
| **Platform install** | Install CRDs and the `dynamo-platform` chart. The platform deploys the operator and core services such as etcd and NATS; multinode orchestrators are optional. |

**This repository** is built for this scenario: the `text`, `image`, `audio`, `video`, `profiler`, and `GlobalPlanner` directories all assume operator-managed DGD or DGDR workflows on Kubernetes. The `GlobalPlanner` folder includes a shared-GPU-budget control-plane example.

### 4. Kubernetes disaggregated serving: RDMA-first specialized path

**Use case:** Prefill/decode separation for workloads where disaggregation materially improves throughput or scaling behavior.

| Aspect | Description |
|--------|-------------|
| **Architecture** | Separate prefill and decode workers, with KV cache transfer between them. |
| **Decision flow** | Current docs recommend using AIConfigurator to compare aggregated vs disaggregated layouts and deploy the better one for the target SLA. |
| **Hard requirements** | RDMA-capable network, RDMA device plugin, and deployed etcd + NATS for coordination. |
| **Performance note** | The current docs are explicit: without RDMA, KV transfer becomes a severe bottleneck and can degrade performance by roughly 40x. |
| **Best for** | Long-input workloads, independent prefill/decode scaling, and SLA-driven tuning on the right hardware. |

This is not the default recommendation for balanced workloads. The current disaggregated guide notes that aggregated serving is often better when the workload is relatively balanced; disaggregated serving shines when long inputs or scaling asymmetry justify the extra complexity.

### 5. Kubernetes + Inference Gateway (GAIE on kGateway)

**Use case:** Gateway-layer routing and traffic steering in front of Dynamo-managed backends.

| Aspect | Description |
|--------|-------------|
| **Gateway** | Integrates Dynamo with the Gateway API Inference Extension through kGateway. |
| **Support scope** | The current GAIE guide says this setup supports both aggregated and disaggregated serving, and currently only supports the kGateway-based inference gateway. |
| **Routing model** | Routing happens in the EPP, not in the Dynamo frontend. The frontend must run with `--router-mode direct` so it respects the worker selections passed in headers. |
| **Backend relationship** | This complements operator-managed DGDs; the gateway sits in front of the Dynamo deployment rather than replacing it. |
| **Important limitation** | The current GAIE guide explicitly says to deploy Dynamo without the Inference Gateway if you need LoRA. |

## Scenario Comparison

| Scenario | Primary purpose | Discovery and coordination | Network requirements | Strengths | Tradeoffs |
|----------|-----------------|----------------------------|----------------------|-----------|-----------|
| **Local single-machine (file discovery)** | Fast local dev and debugging | File discovery; no external infra. For vLLM, disable KV events to avoid NATS. | None beyond localhost | Lowest setup friction; easiest way to get an OpenAI-compatible endpoint running | Single-machine only; not representative of cluster coordination behavior |
| **Local single-machine (Docker Compose infra)** | Higher-fidelity local topology | etcd by default; NATS optional depending on KV routing/events | None beyond localhost | Better rehearsal for distributed coordination and observability | More moving parts than file discovery, without production-grade HA |
| **Kubernetes operator-managed** | Standard production Kubernetes deployment | Kubernetes-native discovery by default via `DynamoWorkerMetadata` and `EndpointSlices`; platform chart also deploys core services | Standard Kubernetes networking | Declarative workflows, operator reconciliation, DGDR-based profiling, autoscaling, native K8s lifecycle management | Requires CRDs, Helm-managed platform install, and cluster-specific RBAC choices |
| **Kubernetes disaggregated (RDMA)** | Maximize performance for the right SLA and workload shape | Requires etcd + NATS for coordination in addition to the deployment itself | RDMA-capable network, device plugin, UCX/RDMA settings | Independent prefill/decode scaling and best upside for long-context workloads | Hardware-sensitive, more tuning, and severe performance penalty if RDMA is missing |
| **Kubernetes + GAIE/kGateway** | Gateway-layer KV-aware routing and ingress | Gateway EPP performs routing; operator-managed backends still serve the model graph | Standard gateway networking; backend still inherits any disagg RDMA needs | Adds Gateway API integration and centralized traffic steering | Extra gateway layer, kGateway-only support today, and no LoRA support through GAIE |

## Latest Doc Changes That Affect This Repo

- Kubernetes-native service discovery is now the recommended default on Kubernetes; etcd discovery is the legacy or opt-in path there.
- Local single-machine development is now clearly documented around file discovery first, with Compose-based infra as an optional higher-fidelity setup.
- The operator now documents three concrete deployment modes: cluster-wide, namespace-scoped, and hybrid.
- Multinode orchestration is optional and not installed by default with the platform chart; Grove + KAI Scheduler or LWS + Volcano are separate decisions.
- The current disaggregated guide is much stricter about RDMA being required for good performance and recommends AIConfigurator for deciding between aggregated and disaggregated topologies.
- The current GAIE guide is more specific that support is kGateway-only, uses EPP-side routing with `--router-mode direct`, and should be avoided if you need LoRA.

## Summary

| Scenario | Focus | Fit for this repo |
|----------|-------|-------------------|
| **Local single-machine (file or Compose)** | CLI workflows, small-scale validation, local debugging | Useful for understanding Dynamo, but not what this repo automates |
| **Kubernetes operator-managed** | Declarative DGD/DGDR workflows on a cluster | Yes, this is the main target, including the shared-budget `GlobalPlanner` example |
| **Kubernetes disaggregated (RDMA)** | Specialized high-performance prefill/decode separation | Partially, mainly through the profiler and disaggregated examples |
| **Kubernetes + GAIE/kGateway** | Gateway-layer routing in front of Dynamo | Adjacent pattern; not what the manifests in this repo currently deploy |
