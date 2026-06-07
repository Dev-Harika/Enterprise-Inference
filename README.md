# OPEA Enterprise Inference — Engineering Contributions

This fork tracks my upstream infrastructure and platform engineering contributions to **[Intel® AI for Enterprise Inference](https://github.com/opea-project/Enterprise-Inference)**, an open-source solution built on the **OPEA (Open Platform for Enterprise AI)** framework.

The project automates the deployment and lifecycle management of LLM inference services on Intel Xeon and Intel Gaudi hardware using Kubernetes, Ansible, and Helm.

---

## Industry Recognition

My deployment automation code has been adopted and cited by **Dell Technologies** in an official enterprise reference architecture:

> [Dell InfoHub: Bare-Metal Ubuntu Automation for Enterprise Inference (CPU/Gaudi3)](https://infohub.delltechnologies.com/en-us/p/bare-metal-ubuntu-automation-for-enterprise-inference-cpu-gaudi3/)

---

## System Architecture

```
                        ┌─────────────────────────────────────┐
                        │         Enterprise Clients           │
                        └─────────────┬───────────────────────┘
                                      │ HTTPS / OpenAI-compatible API
                        ┌─────────────▼───────────────────────┐
                        │   Ingress NGINX  +  APISIX Gateway   │
                        │   (auth, rate-limit, routing)         │
                        └────┬──────────────┬──────────────────┘
                             │              │
               ┌─────────────▼──┐   ┌──────▼──────────────────┐
               │   Keycloak      │   │   GenAI Gateway           │
               │   (OIDC / SSO)  │   │   (LiteLLM + Langfuse)   │
               └─────────────────┘   └──────────────────────────┘
                                              │
                     ┌────────────────────────┼────────────────────────┐
                     │                        │                        │
        ┌────────────▼─────────┐ ┌────────────▼─────────┐ ┌───────────▼──────────┐
        │  vLLM / TGI / SGLang │ │  TEI Embedding Svc    │ │  TEI Reranking Svc   │
        │  (LLM inference)      │ │  (vector embeddings)  │ │  (RAG reranking)     │
        └────────────┬─────────┘ └──────────────────────┘ └──────────────────────┘
                     │
       ┌─────────────┴──────────────────┐
       │                                │
┌──────▼──────────┐           ┌─────────▼────────┐
│  Intel Gaudi3   │           │   Intel Xeon      │
│  (Habana Ops)   │           │   (CPU inference) │
└─────────────────┘           └──────────────────┘
```

**Observability**: Prometheus + Grafana stack deployed via Helm across the cluster.

---

## Deployment Stack

| Layer | Technology |
|---|---|
| Orchestration | Kubernetes (bare-metal, RKE2) / OpenShift |
| Automation | Ansible playbooks + Helm charts |
| Inference engines | vLLM, TGI (text-generation-inference), SGLang |
| Embedding / Reranking | TEI (text-embeddings-inference) |
| API gateway | APISIX |
| Identity | Keycloak (OIDC) |
| Observability | Prometheus, Grafana, Fluentbit |
| Storage | Ceph (distributed), local PVCs |
| Hardware | Intel Gaudi 3, Intel Xeon Scalable |

---

## My Merged Pull Requests

### [PR #49 — Script Modification for Clean Environment Target Bootstrapping](https://github.com/opea-project/Enterprise-Inference/pull/49)

**Problem:** Bootstrap scripts suffered from path conflicts and environmental variable bleeding across target nodes, causing unstable initialization when setting up the inference substrate.

**Solution:** Restructured script execution flags and setup parameters to isolate system-level dependencies. Added strict host architectural readiness checks before downstream steps run.

**Impact:** Eliminated initialization failures across deployment nodes, producing a drift-free substrate for the AI runtime.

---

### [PR #67 — Ubuntu OS Installation/Deployment Script](https://github.com/opea-project/Enterprise-Inference/pull/67)

**Problem:** Manual host preparation introduced inconsistent directory permissions, path errors, and brittle orchestration dependencies across Ubuntu nodes.

**Solution:** Built an idempotent end-to-end automation script covering directory provisioning, system configuration, and environment path definitions — ensuring a repeatable, immutable installation sequence.

**Impact:** Eliminated manual runbook execution and significantly reduced node onboarding time. This script is the foundation cited in the Dell Technologies whitepaper.

---

### [PR #72 — RHEL Deployment Adaptations and Hardening](https://github.com/opea-project/Enterprise-Inference/pull/72)

**Problem:** Deploying the inference engine into RHEL environments failed due to non-standard installation paths, SELinux constraints, and enterprise filesystem policies.

**Solution:** Overhauled the deployment script to adapt to RHEL directory layouts and security baselines. Added explicit path validation, error-handling routines, and file permission schemes compliant with enterprise security policy.

**Impact:** Unlocked RHEL and OpenShift environments — organizations running zero-trust, SELinux-enforced infrastructure can now deploy OPEA runtimes without modifying their security posture.

---

### [PR #73 — NVIDIA Configuration Support and Device Runtime Orchestration](https://github.com/opea-project/Enterprise-Inference/pull/73)

**Problem:** The deployment framework had no clean way to switch between Intel Gaudi and NVIDIA GPU hardware, leading to container runtime failures and device plugin conflicts.

**Solution:** Added conditional configuration logic that detects the host hardware platform at deploy time, injects the correct NVIDIA container toolkit hooks, maps device plugins, and mounts the required execution directories — all without manual intervention.

**Impact:** True multi-hardware support from a single deployment pipeline. Operations teams can target Gaudi3 or NVIDIA clusters using the same automation layer.

---

### [PR #100 — Microservice Dependency Synchronization and Race Condition Resolution](https://github.com/opea-project/Enterprise-Inference/pull/100)

**Problem:** Cold-start pod restart loops occurred because downstream services attempted connections before core platform services were healthy — a classic distributed system initialization race.

**Solution:** Introduced deterministic startup synchronization: health-check wait loops with progressive dependency resolution, ensuring core services reach `Ready` state before subordinate microservices start.

**Impact:** Eliminated initialization race conditions during cluster scale-up events, directly improving uptime SLAs for production inference clusters.

---

## Deployment Overview

```bash
# 1. Define cluster topology
vim inventory/hosts.yaml

# 2. Configure components (inference engine, hardware, models)
vim inference-config.cfg

# 3. Deploy the full stack
bash inference-stack-deploy.sh
```

The Ansible playbook sequence:

```
inference-precheck.yml          ← validate hosts and prerequisites
deploy-cluster-config.yml       ← Kubernetes cluster baseline
deploy-habana-ai-operator.yml   ← Intel Gaudi device plugin (Gaudi targets only)
deploy-ingress-controller.yml   ← NGINX ingress
deploy-keycloak-*.yml           ← Keycloak identity provider
deploy-observability.yml        ← Prometheus + Grafana
deploy-inference-models.yml     ← vLLM / TGI / SGLang model pods
deploy-genai-gateway.yml        ← LiteLLM + Langfuse gateway
```

Supported deployment topologies:

| Topology | Use case |
|---|---|
| Single node (vLLM Docker) | Quick local testing |
| Single node (full stack) | Dev / lightweight workloads |
| Single master + N workers | Higher throughput |
| Multi-master + N workers | HA production clusters |

---

## Sample Solutions

The repo ships with pre-built OPEA application blueprints:

| Blueprint | Description |
|---|---|
| `RAGChatbot` | Retrieval-augmented generation with TEI embeddings + reranking |
| `DocSummarization` | Document summarization pipeline |
| `MultiAgentQnA` | Multi-agent question answering |
| `HybridSearch` | Dense + sparse search hybrid |
| `PDFToPodcast` | PDF ingestion to audio |
| `agenticai` | Agentic AI workflow samples |

---

## Attribution Note

My upstream contributions were developed and merged via a corporate GitHub account during my role as a Platform/MLOps Engineer. This fork exists to connect those public PR records to my personal portfolio for architectural review purposes.

All linked PRs are public at [github.com/opea-project/Enterprise-Inference](https://github.com/opea-project/Enterprise-Inference).