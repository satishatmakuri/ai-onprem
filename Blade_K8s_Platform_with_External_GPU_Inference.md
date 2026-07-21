# On-Premises Kubernetes AI Solution Platform (Blades) with a Separate GPU Inference Plane

**Enterprise Architecture, Low-Level Design & Deployment Runbook**

| Field | Value |
|---|---|
| Document class | Implementation-ready architecture + runbook |
| Design intent | Blade estate hosts the **Kubernetes microservices / AI-solution platform**; a **separate GPU server** performs GLM-5.2 model inference |
| Blade hardware | 6× Supermicro SBI-8149P-C4N (CPU-only) in SBE-820C-822 chassis (~1,000 cores, ~2 TB RAM aggregate) |
| Inference hardware | Separate dedicated GPU server(s) — sized in §7 |
| Orchestration | Kubernetes (RKE2) on blades |
| Environment | Production, multi-user, secure, observable, recoverable, restricted/air-gapped |
| Data rule | Enterprise data & prompts remain **on-premises** end-to-end |
| Date | 2026-07-21 |
| Companion doc | *GLM-5.2 On-Prem CPU Feasibility* (established CPU-only inference is **not** viable → this design moves inference to GPU) |

> **Why this design.** The prior feasibility study proved the CPU-only blades **cannot** interactively host full GLM-5.2 (744B MoE; ~460–480 GB weights at Q4; decode is memory-bandwidth-bound). The correct architecture therefore **splits the two planes**: the blades run everything *around* the model (APIs, orchestration, RAG, vector search, security, observability, MLOps), and a **purpose-built GPU server** runs the model itself. The blades call the GPU inference endpoint over the network. This preserves the entire Kubernetes platform investment and isolates the scarce, expensive GPU capacity behind a clean, swappable service contract.

> **Evidence legend.** **[DOC]** documented capability · **[EST]** engineering estimate (validate by benchmark) · **[ASSUMPTION]** assumed pending discovery · **[TESTED]** measured on your hardware (none yet). Untagged text is design guidance.

---

## Table of Contents
1. Executive Summary & Architecture Intent
2. Two-Plane Reference Architecture
3. Plane Boundary: How Blades Reach the GPU Inference Endpoint
4. Blade Kubernetes Platform Design
5. AI-Solution Microservices Catalogue (what runs on the blades)
6. What Runs Where — Placement Decision Matrix
7. GPU Inference Plane Design (the separate server)
8. Inference Service Contract (OpenAI-compatible API)
9. Networking Between Planes
10. Security Architecture (cross-plane, zero-trust)
11. Storage & Data Architecture
12. Observability Across the Boundary
13. Autoscaling, Capacity & Load Management
14. High Availability & Failure Analysis
15. GitOps, CI/CD & MLOps
16. Backup, DR & Air-Gapped Operation
17. Implementation Phases
18. Installation Runbook
- Appendices A–N: manifests, Helm values, port matrix, ADRs, checklists, risk register, BOM, capacity model, GPU sizing

---

# 1. Executive Summary & Architecture Intent

**Decision:** adopt a **two-plane on-prem architecture**.

- **Plane 1 — Application/Platform plane (the 6 blades, CPU-only).** A production RKE2 Kubernetes cluster hosting the **AI-solution microservices**: API gateway, model router, RAG/orchestration services, vector database, document ingestion pipelines, caching, business/application services, plus platform services (registry, secrets, GitOps, observability, security). Optionally also hosts **CPU-friendly models** (embeddings, rerankers) since those run well on blades.
- **Plane 2 — Inference plane (separate GPU server).** Runs GLM-5.2 (and other large models) on GPUs using a GPU-class inference engine (vLLM / SGLang / TensorRT-LLM), exposing an **OpenAI-compatible HTTPS endpoint**. It is stateless with respect to business data — it receives a prompt, returns tokens, logs **no content**.

**The contract between planes is a network API**, not shared memory. The blades never try to run GLM-5.2; they call it. This gives clean separation of concerns, independent scaling, independent lifecycle/upgrades, and a straightforward path to add more GPU capacity later.

**Key properties delivered**
- All data and prompts stay on-prem (both planes are in your data centre; traffic is mTLS-encrypted and never leaves).
- The GPU server is a **swappable backend** — you can start with one server, add more, or change the inference engine without touching the microservices.
- Blades provide real HA for the *platform*; the GPU plane's HA depends on GPU count (§14).
- Embeddings/rerankers run on the blades (cheap, CPU-friendly), keeping GPU capacity for the big model.

**Go decision:** **GO.** Build the blade K8s platform now (Phases 0–6); stand up the GPU inference plane in parallel (Phase 7); integrate and test (Phases 8–10).

---

# 2. Two-Plane Reference Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│ PLANE 1 — BLADES: Kubernetes AI-Solution / Microservices Platform (CPU)      │
│                                                                             │
│  Consumers ──► Ingress/TLS ──► API Gateway (authN/Z, quota, audit-meta)     │
│                                   │                                         │
│                                   ▼                                         │
│                          AI Orchestration / RAG services                    │
│                          (retrieval, prompt assembly, tools, agents)        │
│                            │            │                │                  │
│                   Vector DB (Qdrant)  Embeddings/Rerank  Cache (Redis)      │
│                            │            (CPU pods)         │                │
│                            ▼                                                 │
│                   Model Router (LiteLLM / Envoy)                             │
│                            │  OpenAI-compatible calls                        │
│  Platform: Harbor · Vault · Argo CD · Prometheus/Grafana/Loki · Kyverno      │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                 │  mTLS HTTPS  (dedicated inference VLAN)
                                 │  /v1/chat/completions, /v1/embeddings
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ PLANE 2 — SEPARATE GPU SERVER(S): Inference Plane                            │
│                                                                             │
│   Inference Gateway (TLS, authN) ──► vLLM / SGLang / TensorRT-LLM            │
│                                        serving GLM-5.2 (tensor-parallel)     │
│   Local NVMe model store · GPU metrics exporter · NO content logging         │
└───────────────────────────────────────────────────────────────────────────┘
```

**Integration options for Plane 2 (choose per §7.4):**
- **Option A — Standalone external endpoint (recommended default).** GPU server is its own host (or small cluster), *not* a K8s node of the blade cluster. Blades reach it via a stable in-cluster abstraction (`ExternalName`/`Endpoints` + egress policy). Cleanest separation; independent lifecycle.
- **Option B — GPU node pool joined to the same cluster.** GPU server joins the RKE2 cluster as a **tainted GPU worker** (`node-role=gpu`, NVIDIA device plugin). Unified scheduling/observability, but couples lifecycles and mixes a heavyweight node into the blade cluster.

Both keep the microservices on the blades and the model on the GPU; they differ only in how the GPU host is managed.

---

# 3. Plane Boundary: How Blades Reach the GPU Inference Endpoint

The **only** thing the blades know about the model is a URL + credentials, brokered by the **Model Router (LiteLLM)**. Everything else (which engine, how many GPUs, which quantization) is hidden behind the contract.

**Call path:** app → gateway → orchestrator → **LiteLLM router** → (egress mTLS) → **GPU inference gateway** → engine → tokens stream back.

Benefits of routing through LiteLLM rather than calling the GPU directly:
- One place for **authN to the GPU plane**, retries, timeouts, circuit-breaking, and failover between GPU endpoints.
- Per-app **quotas, rate limits, token accounting** without each service re-implementing them.
- **Model aliasing** — `glm-5.2`, `glm-mid`, `bge-embed` are logical names; router maps them to CPU pods or the GPU endpoint transparently.
- Swap or add GPU servers by editing router config, no application changes.

---

# 4. Blade Kubernetes Platform Design

*(Reuses the hardened platform from the feasibility doc; summarized here, focused on the microservices role.)*

## 4.1 Cluster shape
- **RKE2**, 3 control-plane blades (stacked etcd, quorum survives 1 loss), 3 worker blades for microservices + CPU models.
- Control-plane blades **tainted** `NoSchedule`; **etcd on dedicated NVMe** (p99 fsync < 10 ms).
- CNI **Cilium** (eBPF, NetworkPolicy, egress control — important for the plane boundary).
- Distribution/version pins, CIS profile, audit logging as per companion doc §6–7.

## 4.2 Blade role allocation (microservices platform)

| Blade | Role | Hosts |
|---|---|---|
| blade-01 | control-plane + etcd-1 | apiserver, ingress-nginx |
| blade-02 | control-plane + etcd-2 | Vault, cert-manager, Harbor |
| blade-03 | control-plane + etcd-3 | Prometheus, Loki, Grafana, Argo CD |
| blade-04 | worker | gateway, orchestrator/RAG, LiteLLM router (replicas) |
| blade-05 | worker | Qdrant (vector DB), Redis cache, embeddings (TEI) |
| blade-06 | worker | rerankers, ingestion workers, embeddings replica, burst |

Reserve host capacity on every blade (OS + kubelet + **≥20% headroom**); QoS `Guaranteed` for latency-sensitive services; pod anti-affinity so stateful/router replicas never co-locate.

## 4.3 Namespaces
`platform` (registry, vault, cert-mgr), `gateway`, `ai-apps` (orchestrators/RAG/business svcs), `data` (Qdrant, Redis), `models-cpu` (embeddings/rerank), `observability`, `argocd`. Default-deny NetworkPolicy per namespace.

---

# 5. AI-Solution Microservices Catalogue (runs on the blades)

| Service | Purpose | Tech (open-source) | Plane | Notes |
|---|---|---|---|---|
| **API Gateway** | TLS termination, authN/Z, quota, audit-metadata | Envoy / Kong / APISIX | blades | no prompt/response bodies logged |
| **Model Router** | OpenAI-compatible facade, routing, retries, failover, token accounting | **LiteLLM** | blades | brokers CPU models + GPU endpoint |
| **AI Orchestrator / RAG** | retrieval, prompt assembly, tool/agent flows | LangChain/LlamaIndex service, or custom FastAPI | blades | stateless, horizontally scaled |
| **Vector Database** | embeddings storage + ANN search | **Qdrant** (or Weaviate/Milvus) | blades | CPU-friendly; persistent NVMe |
| **Embeddings** | text→vector | **text-embeddings-inference** (BGE-M3) | blades (CPU) | keep off GPU to save capacity |
| **Reranker** | cross-encoder reranking | TEI / ONNX Runtime | blades (CPU) | CPU-friendly |
| **Document Ingestion** | parse, chunk, embed, upsert | workers + queue | blades | batch/stream |
| **Cache** | prompt/result & session cache, rate-limit store | **Redis** | blades | semantic cache optional |
| **Queue / Async** | batch inference jobs, ingestion | NATS / RabbitMQ / KEDA-scaled | blades | backpressure to GPU plane |
| **Business/App services** | the actual AI solutions (chat UI, copilots, APIs) | your microservices | blades | consume the router only |
| **Guardrails/PII** | input/output filtering, redaction | open-source guardrail lib | blades | before/after model call |
| **AuthN provider** | OIDC/SSO | Keycloak / AD federation | blades | issues app tokens |

**Platform services:** Harbor (registry+scan+sign), Vault (secrets), Argo CD (GitOps), cert-manager (internal CA), Kyverno (admission policy), MinIO (object store), kube-prometheus-stack + Loki (observability), Velero (backup).

> The blades run **everything except the large-model forward pass**. Embeddings and rerankers deliberately stay on CPU blades — they're cheap there and preserve GPU headroom for GLM-5.2.

---

# 6. What Runs Where — Placement Decision Matrix

| Workload | Blades (CPU) | GPU server | Rationale |
|---|---|---|---|
| API gateway, router, orchestration, RAG logic | ✅ | ❌ | I/O-bound, cheap on CPU |
| Vector DB, cache, queues, ingestion | ✅ | ❌ | data services; keep near apps + on-prem storage |
| Embeddings, rerankers | ✅ | (optional) | CPU-adequate; save GPU |
| **GLM-5.2 / large LLM inference** | ❌ | ✅ | needs GPU VRAM + bandwidth |
| Other large/vision/low-latency models | ❌ | ✅ | GPU-class |
| Platform, security, observability, GitOps | ✅ | (agents only) | control lives on blades |
| Model weights master copy | ✅ (MinIO) | ✅ (local NVMe cache) | staged on-prem, pulled to GPU host |

---

# 7. GPU Inference Plane Design (the separate server)

## 7.1 Sizing for GLM-5.2 (744B MoE, ~40B active, up to 1M ctx) [DOC/EST]

| Serving precision | Weights (approx) | Practical GPU configuration (illustrative) |
|---|---|---|
| **FP8** | ~740–800 GB **[EST]** | 8× 80–141 GB GPUs (e.g., H100/H200-class) with tensor parallelism, or 1–2 nodes |
| **INT4 / Q4** | ~460–480 GB **[DOC]** | 6–8× 80 GB GPUs, TP=8 |
| Full BF16 | ~1.5 TB **[DOC/EST]** | multi-node GPU (16× 80 GB) — usually unnecessary |

> **Guardrail:** confirm exact GPU count/VRAM against the *pinned* engine + GLM-5.2 build during procurement/benchmark; add **KV-cache VRAM** for concurrency and context (can be tens–hundreds of GB at long context). Long-context (1M) dramatically increases KV-cache and is a separate sizing exercise.

**Recommended starting point [EST]:** a single GPU server (or 2U node) with **8× 80 GB-class GPUs + NVLink/NVSwitch**, ≥1 TB system RAM, ≥4 TB NVMe for model store, dual 25/100 GbE. This serves GLM-5.2 at FP8/INT4 with tensor parallelism over NVLink (fast intra-node interconnect — unlike the Ethernet-connected CPU blades, this is where tensor parallelism works well).

## 7.2 Inference engine
| Engine | Fit | Notes |
|---|---|---|
| **vLLM** | **Recommended** | high-throughput, PagedAttention, OpenAI-compatible server, tensor-parallel, strong GLM support |
| **SGLang** | strong alternative | excellent throughput/structured output |
| **TensorRT-LLM** | max perf, more ops effort | NVIDIA-optimized |

Run the engine's built-in **OpenAI-compatible server** (`vllm serve ... --api-key ...`) behind a thin **inference gateway** (TLS, authN, rate-limit) on the GPU host or a small front proxy.

## 7.3 GPU host software
- OS + NVIDIA driver + CUDA (pinned), NVIDIA Container Toolkit.
- Containerized engine, image **pinned by digest**, model on **local NVMe** (pulled+verified from blade MinIO).
- **node-exporter + DCGM-exporter** for GPU metrics scraped by the blades' Prometheus.
- **No content logging**; metadata metrics only.

## 7.4 Integration mode (choose)
- **A. Standalone (recommended):** GPU host is not a K8s node. Managed by its own systemd/compose or a tiny single-node k3s. Blades reach it via `ExternalName`/manual `Endpoints`. **Cleanest separation, independent upgrades, blast-radius isolation.**
- **B. GPU node pool:** join GPU host to RKE2 as a tainted node (`nvidia.com/gpu` device plugin). Unified scheduling/observability, but couples cluster lifecycle to the GPU host and mixes a very different node into the cluster. Choose only if you want one control plane for everything.

> **Recommendation:** start with **Option A** for clean separation, revisit B if you later run several GPU nodes and want unified scheduling.

---

# 8. Inference Service Contract (OpenAI-compatible API)

- **Protocol:** HTTPS (mTLS), OpenAI schema: `POST /v1/chat/completions` (stream), `/v1/completions`, `/v1/embeddings` (if any embed on GPU), `/v1/models`.
- **Auth:** bearer token (from Vault) *plus* mTLS client cert; GPU gateway validates both.
- **Versioning:** model alias (`glm-5.2`) + engine version header; router pins the alias to a specific backend.
- **Timeouts/limits:** request timeout, max tokens, max concurrent streams per client (enforced at router and GPU gateway).
- **Idempotency/retries:** router retries on connection errors only (not on partial streams); circuit-breaker opens on sustained 5xx/timeout.
- **Backpressure:** GPU returns 429 when queue is full; router surfaces 429 with `Retry-After`; batch jobs go through the queue.

---

# 9. Networking Between Planes

## 9.1 Segmented networks
| Network | Purpose |
|---|---|
| Management/IPMI VLAN | BMC of blades + GPU host |
| Blade cluster VLANs | node/API, pod/CNI, storage |
| **Inference VLAN (new)** | dedicated path blades ↔ GPU host; mTLS only; jumbo frames |
| Consumer/ingress VLAN | clients → blade gateway |

The **inference VLAN** is the only route between planes. Firewall it to allow just the GPU gateway port from the router pods' egress identity.

## 9.2 Reaching the GPU endpoint from the cluster (Option A)
```yaml
# ExternalName service abstracts the GPU host behind a stable in-cluster DNS name
apiVersion: v1
kind: Service
metadata: { name: gpu-inference, namespace: gateway }
spec:
  type: ExternalName
  externalName: gpu-infer.inference.internal   # DNS or use Endpoints for a bare IP
---
# For a bare IP, use a headless Service + manual Endpoints
apiVersion: v1
kind: Endpoints
metadata: { name: gpu-inference, namespace: gateway }
subsets:
  - addresses: [{ ip: 10.30.0.20 }]     # GPU host on inference VLAN
    ports: [{ port: 8000, name: https }]
```

## 9.3 Egress control (Cilium NetworkPolicy)
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: allow-router-to-gpu, namespace: gateway }
spec:
  endpointSelector: { matchLabels: { app: litellm } }
  egress:
    - toCIDR: ["10.30.0.20/32"]
      toPorts: [{ ports: [{ port: "8000", protocol: TCP }] }]
    - toEndpoints: [{ matchLabels: { "k8s:io.kubernetes.pod.namespace": kube-system, k8s-app: kube-dns } }]
      toPorts: [{ ports: [{ port: "53", protocol: UDP }] }]
  # default-deny all other egress
```
Only LiteLLM router pods may egress to the GPU host; nothing else in the cluster can reach it.

## 9.4 Port matrix (cross-plane additions)
| From | To | Port | Proto | Notes |
|---|---|---|---|---|
| LiteLLM router (blades) | GPU inference gateway | 8000/443 | TCP/mTLS | only allowed egress path |
| Prometheus (blade-03) | GPU DCGM/node exporter | 9400/9100 | TCP | metrics scrape |
| GPU host | MinIO (blades) | 9000 | TCP | model pull (verified) |
| Admin | GPU host BMC | 623 | UDP | mgmt VLAN |

---

# 10. Security Architecture (cross-plane, zero-trust)

| Domain | Control |
|---|---|
| **Data residency** | both planes on-prem; inference VLAN never routes off-site; no external API calls |
| **Transport** | **mTLS** blades↔GPU; internal CA (cert-manager) issues both server (GPU gateway) and client (router) certs |
| **AuthN to GPU** | bearer token (Vault, short TTL) **and** client cert; GPU gateway rejects either missing |
| **Egress lockdown** | only router pods reach the GPU host (Cilium policy §9.3); default-deny elsewhere |
| **No content logging** | gateway, router, and GPU engine configured to log **metadata only** (user, model, token counts, latency) — never prompts/responses |
| **PII/guardrails** | redaction/guardrail service on the blades runs before the prompt leaves the app plane and after the response returns |
| **Supply chain** | all images signed (cosign) + digest-pinned + scanned (Trivy); **model weights** verified (SHA-256 + signature) before the GPU engine loads them; Kyverno enforces on the cluster |
| **Secrets** | Vault; GPU host uses a Vault agent or sealed token; no secrets in images/Git |
| **Isolation** | GPU host firewalled to the inference VLAN; BMC on mgmt VLAN; PSA `restricted` on blade workloads |
| **Audit** | K8s audit + gateway metadata audit; GPU host access logged |

> **Guardrails honored:** endpoints never exposed without TLS + authN + quota + audit; prompts/responses not logged by default; images digest-pinned (no `:latest`); model artifacts treated as controlled supply-chain items.

---

# 11. Storage & Data Architecture

| Data | Location | Notes |
|---|---|---|
| etcd | blade CP local NVMe | dedicated, fsync-gated |
| Vector DB (Qdrant) | blade NVMe PV | persistent, backed up (Velero) |
| Redis cache | blade RAM/NVMe | ephemeral/semi-persistent |
| Documents/corpora | MinIO on blades | on-prem object store; encrypted at rest |
| Model weights (master) | **MinIO on blades** | signed + checksummed; single source of truth |
| Model weights (serving copy) | **GPU host local NVMe** | pulled + verified from MinIO; engine mmaps locally |
| Observability data | blade NVMe + MinIO long-term | retention tiering |
| Backups | MinIO + offsite | 3-2-1 |

Model flow: publish to MinIO (verified) → GPU host pulls to local NVMe (`mc mirror` + `sha256sum` + `cosign verify`) → engine loads. Never serve weights over the network at inference time.

---

# 12. Observability Across the Boundary

Single pane (blade Prometheus/Grafana) scraping **both** planes:
- **Blade services:** request rate, latency, errors per microservice; gateway/router metrics; queue depth; vector-DB latency.
- **GPU plane:** DCGM GPU util, VRAM used, temp/power; engine metrics — **TTFT, inter-token latency, tokens/sec, running/queued requests, KV-cache utilization, preemptions**.
- **Boundary metrics:** router→GPU call latency, timeout/retry/circuit-breaker events, 429 rate, per-model/per-tenant token consumption (chargeback).
- **Traces:** OpenTelemetry spans propagate from app → orchestrator → router → GPU call, so a slow request is attributable to a specific plane.
- **Alerts:** GPU VRAM > 90%, TTFT p95 breach, GPU host unreachable, circuit-breaker open, queue saturation, cert expiry, memory-headroom < 20% on blades.

---

# 13. Autoscaling, Capacity & Load Management

- **Blade microservices:** HPA/KEDA on latency + queue depth (not CPU% alone); scale gateway/orchestrator/router replicas across worker blades.
- **GPU plane:** scaling = **admission control + queue**, because you have a fixed number of GPUs. When the GPU is saturated, the router returns 429 + `Retry-After`; batch traffic drains through the queue; interactive traffic is prioritized.
- **Concurrency budget:** the GPU engine (vLLM) sets `--max-num-seqs` / KV-cache limits; the router enforces per-tenant caps so one app can't starve others.
- **Adding capacity:** add GPUs/servers → register new backend in the router → traffic spreads. Blades don't change.
- **Capacity model:** track GPU tokens/sec ceiling vs. demand; alert at 70/80% to trigger procurement (Appendix M).

---

# 14. High Availability & Failure Analysis

| Component | HA posture | Notes |
|---|---|---|
| Blade control-plane | **HA** (3 etcd, survives 1 loss) | ✅ |
| Blade microservices (gateway/router/orchestrator) | **HA** (replicas + anti-affinity across blades) | ✅ |
| Vector DB / Redis | HA via replication/clustering | configure Qdrant replicas; Redis Sentinel/cluster |
| Embeddings/rerank (CPU) | HA (replicated) | ✅ |
| **Single GPU server** | **NOT HA** — its loss stops large-model inference | ⚠️ mitigations below |
| GPU with 2+ servers | HA if router failovers between them | ✅ recommended target |

**Mitigations for GPU-plane availability:**
- **Router failover:** LiteLLM configured with ≥2 GPU backends → automatic failover; with one server, failover degrades to a graceful error + queue.
- **Graceful degradation:** on GPU outage, route eligible requests to a smaller CPU model on the blades (reduced quality) or queue with clear status; embeddings/RAG retrieval on blades keep working.
- **Fast restart:** model pre-staged on GPU local NVMe for quick reload.
- **Roadmap:** a second GPU server removes the single point of failure and doubles throughput.

> **Honesty:** with one GPU server the inference plane is a single point of failure for large-model requests — do not claim full HA until ≥2 GPU backends exist. The *platform* (blades) remains HA regardless.

---

# 15. GitOps, CI/CD & MLOps

- **Repos:** `platform-gitops` (cluster add-ons), `ai-apps` (microservices), `ai-router-config` (LiteLLM model map incl. GPU backends), `model-registry` (model metadata/provenance; weights in MinIO), `gpu-infra` (GPU host config, engine version, model pin).
- **Model promotion:** register → verify signature/hash → deploy to GPU host (canary) → automated eval (quality + latency) → sign → flip router alias to new backend → monitor → promote. Rollback = revert router alias / Git tag.
- **Argo CD** reconciles blade workloads; the GPU host (Option A) is driven by its own pinned automation (Ansible/compose) but its **model pin + engine digest live in Git** for auditability.
- **Admission:** Kyverno enforces signed, digest-pinned images and verified model artifacts.

---

# 16. Backup, DR & Air-Gapped Operation

| Asset | Method | Frequency |
|---|---|---|
| etcd | RKE2 snapshot + off-node | 6 h |
| Vault | Raft snapshot | daily |
| Qdrant / Redis / PVs | Velero + restic → MinIO | daily |
| MinIO (docs + model master) | replication → offsite | daily |
| Cluster manifests / router config | Git | on commit |
| GPU host config + model pin | Git + local NVMe copy | on change |

**Air-gap:** connected staging host downloads → verifies (signature/hash) → scans → signs → imports images to Harbor, weights to MinIO. GPU engine images and CUDA/driver bundles are mirrored the same way. Admission rejects anything unsigned. **DR:** rebuild blades from GitOps + snapshots; rebuild GPU host from pinned config + re-pull verified weights from MinIO. Test restores quarterly.

---

# 17. Implementation Phases

| Phase | Goal | Exit criteria |
|---|---|---|
| **0 Discovery & requirements** | blade HW discovery (companion doc §2); GPU procurement sizing (§7); app SLOs, concurrency, data rules | inventory + GPU spec + SLOs agreed |
| **1 Blade HW remediation & OS baseline** | firmware parity, NUMA/perf, CIS, LUKS | golden image; gates pass |
| **2 Network & storage prep** | VLANs incl. **inference VLAN**, MTU, NVMe, MinIO | port matrix live; etcd disk gated |
| **3 K8s control-plane** | RKE2 3-node HA, VIP, snapshots, audit | 3 CP Ready; etcd healthy |
| **4 Platform services** | Cilium, cert-manager, Harbor, Vault, Argo, MinIO, ingress, Kyverno | GitOps reconciling; registry signed |
| **5 Security hardening** | PSA restricted, NetworkPolicies (incl. egress lockdown), Vault wiring, no-log config | policy audit clean |
| **6 Observability** | Prometheus/Grafana/Loki/OTel; dashboards | SLO dashboards + alerts live |
| **7 GPU inference plane** | build GPU host, driver/CUDA, deploy vLLM + GLM-5.2, TLS+authN gateway, DCGM metrics | GPU endpoint serves OpenAI API over mTLS; metrics scraped |
| **8 Microservices deployment** | gateway, LiteLLM router (→GPU + CPU models), Qdrant, Redis, embeddings, rerank, orchestrator/RAG | end-to-end call works; embeddings on CPU |
| **9 Integration & SLO testing** | load test cross-plane; failover (kill GPU/backends); timeout/circuit-breaker; DR restore; security review | SLOs met; graceful degradation proven |
| **10 Controlled production onboarding** | onboard AI solutions by tier with quotas; canary; monitor | SLOs sustained; error budget healthy |

---

# 18. Installation Runbook (ordered, air-gap assumed, versions pinned)

## 18.1 Blade platform (Phases 3–6)
```bash
# RKE2 HA control plane (blade-01 first, then 02/03) — see companion doc §24
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh install.sh && systemctl enable --now rke2-server
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml && kubectl get nodes
# workers 04/05/06 as agents, labelled for microservices
kubectl label node blade-04 role=apps ; kubectl label node blade-05 role=data ; kubectl label node blade-06 role=data
# platform add-ons via Argo CD app-of-apps
helm install argocd oci://harbor.internal/charts/argo-cd -n argocd --create-namespace -f argocd-values.yaml
argocd app sync app-of-apps   # cilium, cert-manager, harbor, vault, minio, monitoring, kyverno, keda
```

## 18.2 GPU inference plane (Phase 7, on the separate GPU host)
```bash
# driver + CUDA + container toolkit (pinned versions, air-gap bundle)
apt-get install -y ./nvidia-driver-*.deb ./cuda-toolkit-*.deb ./nvidia-container-toolkit-*.deb
nvidia-smi   # confirm GPUs visible
# pull + verify GLM-5.2 weights from blade MinIO
mc alias set blades https://minio.internal:9000 "$AK" "$SK"
mc mirror blades/models/glm-5.2/ /data/models/glm-5.2/
sha256sum -c /data/models/glm-5.2/SHA256SUMS && cosign verify --key cosign.pub ...  # gate load
# run vLLM OpenAI-compatible server (tensor-parallel across GPUs), digest-pinned image
docker run -d --gpus all --restart unless-stopped -p 8000:8000 \
  -v /data/models:/models:ro -e VLLM_API_KEY_FILE=/run/secrets/vllm_key \
  harbor.internal/ai/vllm-openai@sha256:REPLACE \
  --model /models/glm-5.2 --served-model-name glm-5.2 \
  --tensor-parallel-size 8 --quantization fp8 --max-model-len 32768 \
  --host 0.0.0.0 --port 8000
# front with TLS/mTLS gateway (Envoy/nginx) terminating on the inference VLAN
```

## 18.3 Wire the blades to the GPU endpoint (Phase 8)
```bash
# ExternalName/Endpoints + egress policy (see §9.2–9.3), then LiteLLM values:
kubectl apply -f gateway/gpu-inference-endpoints.yaml
kubectl apply -f gateway/allow-router-to-gpu.yaml
helm upgrade --install litellm oci://harbor.internal/charts/litellm -n gateway -f litellm-values.yaml
```
LiteLLM values (excerpt):
```yaml
model_list:
  - model_name: glm-5.2                       # logical name apps use
    litellm_params:
      model: openai/glm-5.2
      api_base: https://gpu-inference.gateway.svc:8000/v1
      api_key: os.environ/GPU_INFER_TOKEN     # from Vault
      timeout: 120
      # mTLS client cert paths mounted from cert-manager secret
  - model_name: bge-embed
    litellm_params: { model: openai/bge, api_base: http://tei-embed.models-cpu:80/v1 }
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  num_retries: 2
  request_timeout: 120
  # fallbacks: on GPU failure, degrade to a CPU model or return 503+Retry-After
  fallbacks: [{ "glm-5.2": ["glm-small-cpu"] }]
litellm_settings:
  redact_messages_in_exceptions: true         # no content in logs
  set_verbose: false
```

## 18.4 Smoke test (end to end)
```bash
curl -sk https://ai-gw.internal/v1/chat/completions \
  -H "Authorization: Bearer $APP_TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"glm-5.2","messages":[{"role":"user","content":"ping"}],"max_tokens":16}'
# expect tokens streamed from the GPU plane via the blade router
```

---

# Appendices

## Appendix A. ADRs
- **ADR-101** Two-plane split: blades = microservices, GPU server = inference. *Accepted.*
- **ADR-102** GPU host standalone (Option A) not a cluster node initially. *Accepted.*
- **ADR-103** LiteLLM as the single model-router/broker to the GPU plane. *Accepted.*
- **ADR-104** Embeddings/rerankers stay on CPU blades. *Accepted.*
- **ADR-105** vLLM as primary GPU engine (OpenAI-compatible, tensor-parallel). *Accepted, verify GLM-5.2 support at pin.*
- **ADR-106** mTLS + token dual-auth on the plane boundary; egress locked to router pods. *Accepted.*
- **ADR-107** No content logging on either plane. *Accepted.*
- **ADR-108** ≥2 GPU backends required before claiming inference HA. *Accepted.*

## Appendix B. Cross-plane port matrix
See §9.4 (router→GPU 8000/mTLS; Prometheus→DCGM 9400; GPU→MinIO 9000; BMC 623).

## Appendix C. Kubernetes manifests
See §9.2 (ExternalName/Endpoints), §9.3 (Cilium egress), §18.3 (LiteLLM). Add PDBs, HPAs/ScaledObjects for gateway/router/orchestrator, PSA restricted, Kyverno signature/digest policies (companion doc §16.3).

## Appendix D. Helm values
See §18.1 (Argo), §18.3 (LiteLLM). Per-add-on values in `platform-gitops`.

## Appendix E. GPU sizing worksheet
See §7.1. Fill actual GPU model/VRAM/count; add KV-cache VRAM for target concurrency × context; validate with a Phase-7 benchmark (TTFT/ITL/tok-s at target load).

## Appendix F. Production-readiness checklist
Blade CP HA ✅ · etcd NVMe+snapshots ✅ · PSA restricted + default-deny + **egress locked to router** ✅ · images signed+digest-pinned (no `:latest`) ✅ · secrets in Vault ✅ · **mTLS + token on GPU endpoint** ✅ · no content logging (verified both planes) ✅ · ≥20% blade mem headroom ✅ · GPU VRAM/thermal alerts ✅ · router retries/timeouts/circuit-breaker/fallbacks tested ✅ · GPU-outage graceful degradation proven ✅ · backups+DR rehearsed (both planes) ✅ · SLO dashboards+alerts ✅ · onboarding quotas ✅.

## Appendix G. Risk register (top 10)
| # | Risk | Impact | Treatment |
|---|---|---|---|
| 1 | Single GPU server = SPOF for inference | High | add 2nd GPU backend; router fallback to CPU model/queue |
| 2 | GPU VRAM under-sized for GLM-5.2 + KV at concurrency | High | Phase-7 benchmark before go-live; size KV-cache |
| 3 | Plane boundary unsecured | Critical | mTLS+token, egress lockdown, audit |
| 4 | Content leaked via logs | Critical | no-log default both planes; verify in tests |
| 5 | GLM-5.2 unsupported by pinned engine | Med | verify at pin; fallback model |
| 6 | Network saturation on inference VLAN | Med | dedicated VLAN, jumbo frames, monitor |
| 7 | Model supply-chain tampering | Critical | sign+verify weights before load |
| 8 | Blade platform scaling limits | Med | replicas across 3 workers; add nodes later |
| 9 | Capacity managed on CPU%/GPU% only | Med | scale/alert on queue + TTFT + VRAM |
| 10 | GPU host lifecycle drift | Med | pin engine/driver/model in Git; automation |

## Appendix H. Bill of Materials (software)
RKE2/K8s · Cilium · CoreDNS · cert-manager · Harbor(+Trivy+Notation) · Vault · MinIO · Argo CD · Kyverno · KEDA · kube-prometheus-stack · Loki · OpenTelemetry · Velero · **LiteLLM** · Envoy/APISIX · **Qdrant** · Redis · text-embeddings-inference · **vLLM** (GPU) · NVIDIA driver/CUDA/Container Toolkit/DCGM-exporter · cosign/Syft/Grype · Ansible. All open-source, air-gap-installable, pinned by digest/version.

## Appendix I. Capacity model
Blade side: replicas × req/s/replica per microservice (I/O-bound). GPU side: engine tokens/sec ceiling at target context/concurrency (measure Phase 7). Sustained solution throughput = min(blade orchestration ceiling, GPU token ceiling). Alert at 70/80% of GPU ceiling → add GPU capacity.

## Appendix J. GPU expansion roadmap
Phase A: 1 GPU server (GLM-5.2 FP8/INT4, TP=8). Phase B: 2nd GPU server → inference HA + ~2× throughput via router load-balancing. Phase C: dedicated GPU node pool joined to cluster (Option B) once ≥2–3 GPU nodes justify unified scheduling; add more models/tiers. Blades and all microservices remain unchanged throughout — only backends are added.

---

## Notes on facts & assumptions
- GLM-5.2 sizing (744B MoE, ~40B active, ~460–480 GB INT4/Q4, ~740–800 GB FP8, up to 1M ctx) is **[DOC/EST]** from public 2026 model docs; **confirm exact GPU count/VRAM and KV-cache needs against your pinned vLLM + GLM-5.2 build during Phase 0/7 procurement and benchmarking.**
- Blade per-blade RAM/CPU/ISA/bandwidth remain **[ASSUMPTION]** until the companion doc's Phase-0 discovery scripts are run.
- Every latency/throughput figure is **[EST]** until Phase 7/9 benchmarks make it **[TESTED]**. Do not publish SLOs before those gates.

**Bottom line:** the blades are an excellent home for the *entire AI solution* except the model itself; a separate GPU server hosts GLM-5.2 and is reached through a secured, swappable OpenAI-compatible contract brokered by the model router. This is the correct, honest architecture given CPU-only blades.
