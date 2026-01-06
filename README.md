# Multi-Cluster GitOps with Fleet: A Production-Ready Approach

Managing applications across multiple Kubernetes clusters is challenging. Different clusters have different requirements—varying hardware capacities, network configurations, and customer-specific needs. This guide presents an approach to multi-cluster GitOps using **Fleet by Rancher**.

> **Perfect for Edge Computing**: This approach is particularly well-suited for managing edge clusters at scale—from retail stores, manufacturing plants, warehouses, to IoT deployments. When you're managing 100s to 1000s of clusters, manual management becomes impossible. GitOps provides the automation, auditability, and consistency you need at scale.

---

## Table of Contents

1. [What Are We Trying to Achieve?](#what-are-we-trying-to-achieve)
2. [GitOps Principles](#gitops-principles)
3. [Architecture Overview](#architecture-overview)
4. [Why Configuration Needs to Come From Different Places](#why-configuration-needs-to-come-from-different-places)
5. [Why Different Clusters Need Different Configurations](#why-different-clusters-need-different-configurations)
6. [Repository Structure](#repository-structure)
7. [Release Train: Upgrade Strategy](#release-train-upgrade-strategy)
8. [Getting Started](#getting-started)

---

## What Are We Trying to Achieve?

When managing a fleet of Kubernetes clusters, we need to solve several challenges:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         THE MULTI-CLUSTER CHALLENGE                     │
│                                                                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────┐  │
│   │ edge-store-  │   │ edge-store-  │   │ edge-store-  │   │  ...   │  │
│   │     001      │   │     002      │   │     010      │   │ 1000s  │  │
│   │              │   │              │   │   (pilot)    │   │        │  │
│   │  3 replicas  │   │  1 replica   │   │  2 replicas  │   │        │  │
│   │  8GB RAM     │   │  512MB RAM   │   │  2GB RAM     │   │        │  │
│   │  IP: 10.1.10 │   │  IP: 10.1.20 │   │  IP: 10.1.30 │   │        │  │
│   │  v1.4.0      │   │  v1.4.0      │   │  v1.6.0      │   │        │  │
│   └──────────────┘   └──────────────┘   └──────────────┘   └────────┘  │
│                                                                         │
│   SAME APPLICATION ──► DIFFERENT CONFIGURATIONS ──► CONTROLLED ROLLOUT │
└─────────────────────────────────────────────────────────────────────────┘
```

**Our Goals:**

| Goal | Description |
|------|-------------|
| **Single Source of Truth** | All configuration lives in Git—auditable, versioned, reviewable |
| **Per-Cluster Customization** | Each cluster can have unique settings while sharing common config |
| **Controlled Rollouts** | Upgrade pilot clusters first, then gradually roll out to production |
| **Separation of Concerns** | Helm charts (templates) are separate from deployment configuration |

### Why GitOps for Edge at Scale?

```

┌─────────────────────────────────────────────────────────────────────────┐
│                    EDGE CLUSTER MANAGEMENT AT SCALE                     │
│                         GitOps Approach Benefits                        │
│                                                                         │
│   + Commit to Git - All changes version controlled                     │
│   + Fleet syncs all clusters automatically                             │
│   + Git history provides complete audit log                            │
│   + Guaranteed consistency across all clusters                         │
│   + Instant rollback with git revert                                   │
│   + Full accountability via Git blame                                  │
│                                                                         │
│   Time to update 1000 clusters: Minutes (automatic sync)               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real-World Edge Scenarios:**
- **Retail**: 500+ store locations, each with its own K3s cluster
- **Manufacturing**: 200+ factory floor edge nodes running ML models
- **Logistics**: 1000+ warehouse clusters with inventory systems
- **Healthcare**: Regional clinics with patient data systems

---

## GitOps Principles

GitOps is an operational framework that takes DevOps best practices for application development and applies them to infrastructure automation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GitOps WORKFLOW                                │
│                                                                         │
│    Developer/Ops                                                        │
│         │                                                               │
│         │  1. Commit changes                                            │
│         ▼                                                               │
│    ┌─────────┐      2. Detect changes      ┌──────────────────┐        │
│    │   Git   │ ─────────────────────────► │  Fleet Controller │        │
│    │  Repo   │                             │  (on Management   │        │
│    └─────────┘                             │     Cluster)      │        │
│         │                                  └────────┬─────────┘        │
│         │                                           │                   │
│         │  Source of Truth                          │ 3. Reconcile     │
│         │                                           │                   │
│         ▼                                           ▼                   │
│    ┌─────────────────────────────────────────────────────────────┐     │
│    │                    Downstream Clusters                       │     │
│    │  ┌───────────┐    ┌───────────┐    ┌───────────┐           │     │
│    │  │ Cluster A │    │ Cluster B │    │ Cluster C │           │     │
│    │  │  Desired  │    │  Desired  │    │  Desired  │           │     │
│    │  │  State    │    │  State    │    │  State    │           │     │
│    │  └───────────┘    └───────────┘    └───────────┘           │     │
│    └─────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Core Principles:**

1. **Declarative Configuration**: Define *what* you want, not *how* to get there
2. **Version Controlled**: Every change is tracked in Git history
3. **Automated Reconciliation**: System continuously ensures actual state matches desired state
4. **Pull-Based Deployment**: Clusters pull their configuration (more secure than push)

---

## Architecture Overview

Our architecture separates **Helm charts** (application templates) from **GitOps configuration** (deployment specifics):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURE OVERVIEW                            │
│                                                                         │
│  ┌─────────────────────┐              ┌─────────────────────────────┐  │
│  │   helm-charts repo  │              │     gitops-fleet repo       │  │
│  │                     │              │                             │  │
│  │  • Chart templates  │   Published  │  • fleet.yaml (targeting)   │  │
│  │  • Default values   │ ───────────► │  • versions/*.yaml          │  │
│  │  • Reusable across  │   to Helm    │  • clusters/*.yaml          │  │
│  │    projects         │   Registry   │  • Environment-specific     │  │
│  │                     │              │                             │  │
│  └─────────────────────┘              └──────────────┬──────────────┘  │
│                                                      │                  │
│                                                      │ GitRepo         │
│                                                      │ points here     │
│                                                      ▼                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Fleet Controller                               │  │
│  │                  (Management Cluster)                             │  │
│  │                                                                   │  │
│  │   Reads fleet.yaml → Builds Bundle → Deploys to target clusters  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                       │                                 │
│              ┌────────────────────────┼────────────────────────┐       │
│              ▼                        ▼                        ▼       │
│       ┌────────────┐          ┌────────────┐          ┌────────────┐  │
│       │ Cluster A  │          │ Cluster B  │          │ Cluster C  │  │
│       │ v1.4.0     │          │ v1.4.0     │          │ v1.5.0     │  │
│       │ Production │          │ Production │          │   Pilot    │  │
│       └────────────┘          └────────────┘          └────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why Separate Repositories?**

| Repository | Purpose | Changes When |
|------------|---------|--------------|
| **helm-charts** | Application templates, Kubernetes manifests | Application code/structure changes |
| **gitops-fleet** | Deployment configuration, cluster-specific values | Environment/cluster config changes |

This separation allows:
- Helm charts to be **reusable** across multiple projects
- Deployment config to change **without** modifying application code
- Different teams to own different repositories

---

## Why Configuration Needs to Come From Different Places

Real-world deployments require **layered configuration**. Not everything should be in one place:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION LAYERING                               │
│                                                                         │
│   Layer 3: Cluster-Specific          clusters/prod-us.yaml             │
│   (Highest Priority)                 ┌─────────────────────────┐       │
│                                      │ replicaCount: 5         │       │
│   • Network settings (IPs, ports)    │ loadBalancerIP: 10.1.1.1│       │
│   • Resource allocations             │ storageClass: fast-ssd  │       │
│   • Customer-specific config         └───────────┬─────────────┘       │
│                                                  │ overrides           │
│                                                  ▼                      │
│   Layer 2: Version-Specific          versions/v1.4.0.yaml              │
│   (Medium Priority)                  ┌─────────────────────────┐       │
│                                      │ image:                  │       │
│   • Image tags                       │   tag: "1.27.0"         │       │
│   • Version-locked dependencies      │ redis:                  │       │
│   • Release-specific features        │   tag: "7.2.0"          │       │
│                                      └───────────┬─────────────┘       │
│                                                  │ overrides           │
│                                                  ▼                      │
│   Layer 1: Base Defaults             values.yaml                       │
│   (Lowest Priority)                  ┌─────────────────────────┐       │
│                                      │ replicaCount: 1         │       │
│   • Sane defaults                    │ image:                  │       │
│   • Common settings                  │   repository: nginx     │       │
│   • Works for local dev              │   tag: "latest"         │       │
│                                      └─────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

**The Merge Result:**

```yaml
# Final values for edge-store-001 running v1.4.0:
replicaCount: 3              # from clusters/edge-store-001.yaml
loadBalancerIP: 10.100.1.10  # from clusters/edge-store-001.yaml
resources:
  limits:
    memory: 2Gi              # from clusters/edge-store-001.yaml
image:
  repository: nginx          # from Helm chart defaults
  tag: "1.27.0"              # from versions/v1.4.0.yaml
env:
  STORE_ID: "001"            # from clusters/edge-store-001.yaml
```

---

## Why Different Clusters Need Different Configurations

Every cluster is unique. Here's why one-size-fits-all doesn't work:

```
┌─────────────────────────────────────────────────────────────────────────┐
│              WHY CLUSTERS NEED DIFFERENT CONFIGURATIONS                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ HARDWARE CAPACITY                                                │   │
│  │                                                                  │   │
│  │  Production Cluster        │    Edge/Small Cluster              │   │
│  │  • 5 replicas              │    • 1 replica                     │   │
│  │  • 4GB memory per pod      │    • 512MB memory per pod          │   │
│  │  • CPU: 2 cores            │    • CPU: 0.5 cores                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ NETWORK CONFIGURATION                                            │   │
│  │                                                                  │   │
│  │  US Region                 │    EU Region                       │   │
│  │  • LB IP: 10.100.1.x       │    • LB IP: 10.200.1.x             │   │
│  │  • NodePort: 30080         │    • NodePort: 30081               │   │
│  │  • Ingress: us.example.com │    • Ingress: eu.example.com       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ENVIRONMENT DIFFERENCES                                          │   │
│  │                                                                  │   │
│  │  Production                │    Staging                         │   │
│  │  • LOG_LEVEL: warn         │    • LOG_LEVEL: debug              │   │
│  │  • Monitoring: enabled     │    • Monitoring: minimal           │   │
│  │  • Backups: hourly         │    • Backups: disabled             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ CUSTOMER/TENANT REQUIREMENTS                                     │   │
│  │                                                                  │   │
│  │  Customer A                │    Customer B                      │   │
│  │  • Storage: 100GB          │    • Storage: 500GB                │   │
│  │  • Features: standard      │    • Features: premium             │   │
│  │  • SLA: 99.9%              │    • SLA: 99.99%                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
gitops-fleet-2/
│
├── clusters/                           # Per-cluster configurations
│   ├── saurabh-kubernetes-cluster.yaml # Complete cluster-specific settings
│   ├── edge-store-001.yaml             # Store 001 specific settings
│   ├── edge-store-002.yaml             # Store 002 specific settings
│   └── ... (100s-1000s of cluster files)
│
└── apps/
    ├── store-stack/                    # Application stack
    │   ├── fleet.yaml                  # Per-cluster targeting logic
    │   └── versions/                   # Version-specific values
    │       ├── v1.4.0.yaml            # Image tags for v1.4
    │       ├── v1.5.0.yaml            # Image tags for v1.5
    │       └── v1.6.0.yaml            # Image tags for v1.6
    │
    └── storage/                        # Storage infrastructure
        ├── fleet.yaml                  # Per-cluster targeting logic
        └── versions/
            ├── v1.0.0.yaml            # Storage version 1.0
            └── v1.1.0.yaml            # Storage version 1.1
```

### Key Concepts:

| Directory/File | Purpose | Contents |
|----------------|---------|----------|
| `clusters/*.yaml` | **Per-cluster configuration** | Replicas, IPs, resources, env vars - everything cluster-specific |
| `apps/store-stack/fleet.yaml` | **Deployment targeting** | Per-cluster entries specifying which version + which config |
| `versions/*.yaml` | **Version-specific values** | Image tags and version-specific defaults ONLY |

### How It Works:

**Values Merge Order** (later overrides earlier):
1. **Helm chart defaults** (`store-stack/values.yaml` in helm-charts repo)
2. **Cluster-specific config** (`clusters/saurabh-kubernetes-cluster.yaml`) 
3. **Version-specific values** (`versions/v1.4.0.yaml`)

This layering provides:
- **Cluster configs are reusable** across versions
- **Image tags are isolated** per version
- **fleet.yaml scales** - one entry per cluster (not per version per cluster)
- **Full GitOps auditability** - all config in Git

---

## Release Train: Upgrade Strategy

Upgrading applications across multiple edge clusters requires controlled rollouts and clear version management. Our per-cluster targeting approach provides flexibility while maintaining a single source of truth.

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT ARCHITECTURE                              │
│                                                                         │
│   Single store-stack folder with:                                      │
│   • fleet.yaml (per-cluster entries specify version)                   │
│   • versions/*.yaml (image tags per version)                           │
│                                                                         │
│   Shared across all versions:                                          │
│   • clusters/*.yaml (cluster-specific configs)                         │
│                                                                         │
│   apps/store-stack/fleet.yaml                                          │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ targetCustomizations:                                            │  │
│   │                                                                  │  │
│   │   - name: saurabh-kubernetes-cluster                            │  │
│   │     helm:                                                        │  │
│   │       version: 1.4.0                                            │  │
│   │       valuesFiles:                                              │  │
│   │         - ../../clusters/saurabh-kubernetes-cluster.yaml        │  │
│   │         - versions/v1.4.0.yaml                                  │  │
│   │                                                                  │  │
│   │   - name: edge-store-010  # Pilot cluster                       │  │
│   │     helm:                                                        │  │
│   │       version: 1.6.0                                            │  │
│   │       valuesFiles:                                              │  │
│   │         - ../../clusters/edge-store-010.yaml                    │  │
│   │         - versions/v1.6.0.yaml                                  │  │
│   │                                                                  │  │
│   │   - name: edge-store-001                                        │  │
│   │     helm:                                                        │  │
│   │       version: 1.4.0                                            │  │
│   │       valuesFiles:                                              │  │
│   │         - ../../clusters/edge-store-001.yaml                    │  │
│   │         - versions/v1.4.0.yaml                                  │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│            ┌───────────────────────┼──────────────────────┐            │
│            ▼                       ▼                      ▼            │
│     ┌─────────────┐       ┌─────────────┐       ┌─────────────┐      │
│     │ saurabh-k8s │       │ edge-010    │       │ edge-001    │      │
│     │   v1.4.0    │       │   v1.6.0    │       │   v1.4.0    │      │
│     │ Production  │       │   Pilot     │       │ Production  │      │
│     └─────────────┘       └─────────────┘       └─────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Upgrade Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RELEASE TRAIN WORKFLOW                             │
│                                                                         │
│   STEP 1: App Team Releases New Helm Charts                            │
│   ════════════════════════════════════════════                         │
│                                                                         │
│   • Publishes store-stack 1.6.0 to GitHub Pages                        │
│   • New Docker images: nginx:1.29.0, redis:7.5.0                       │
│                                                                         │
│   STEP 2: Platform Team Creates Version File                           │
│   ════════════════════════════════════════════                         │
│                                                                         │
│   # Create apps/store-stack/versions/v1.6.0.yaml                       │
│   nginx-service:                                                        │
│     image:                                                              │
│       tag: "1.29.0"                                                     │
│   redis-service:                                                        │
│     image:                                                              │
│       tag: "7.5.0"                                                      │
│                                                                         │
│   STEP 3: Deploy to Pilot (Manual Edit)                                │
│   ════════════════════════════════════════                             │
│                                                                         │
│   # Edit apps/store-stack/fleet.yaml - Add pilot cluster               │
│   targetCustomizations:                                                 │
│     - name: edge-store-010  # NEW PILOT ENTRY                          │
│       clusterSelector:                                                  │
│         matchLabels:                                                    │
│           management.cattle.io/cluster-display-name: edge-store-010    │
│       helm:                                                             │
│         version: 1.6.0                    # New version!                │
│         valuesFiles:                                                    │
│           - ../../clusters/edge-store-010.yaml  # Same cluster config  │
│           - versions/v1.6.0.yaml          # New image tags!            │
│                                                                         │
│   Commit & push → Fleet deploys v1.6 to pilot within 30 seconds        │
│                                                                         │
│   STEP 4: Validate on Pilot                                            │
│   ═══════════════════════                                              │
│                                                                         │
│   • Health checks, smoke tests, customer UAT                            │
│   • Monitor for 2-3 days                                                │
│                                                                         │
│   STEP 5: Gradual Migration                                            │
│   ════════════════════════                                             │
│                                                                         │
│   Week 2: Migrate first batch                                           │
│   • Edit targetCustomizations for edge-store-003                        │
│   • Change version: 1.4.0 → 1.6.0                                      │
│   • Change valuesFiles: v1.4.0.yaml → v1.6.0.yaml                      │
│                                                                         │
│   # BEFORE                          # AFTER                             │
│   - name: edge-store-003            - name: edge-store-003              │
│     helm:                             helm:                             │
│       version: 1.4.0                    version: 1.6.0                  │
│       valuesFiles:                      valuesFiles:                    │
│         - ../../clusters/...              - ../../clusters/...          │
│         - versions/v1.4.0.yaml            - versions/v1.6.0.yaml        │
│                                                                         │
│   Week 3-12: Continue batch migrations                                  │
│   • 10-50 stores per week                                               │
│   • Monitor after each batch                                            │
│                                                                         │
│   ROLLBACK (If Issues Found)                                           │
│   ══════════════════════════                                           │
│                                                                         │
│   Simply change version back in fleet.yaml:                             │
│   • helm.version: 1.6.0 → 1.4.0                                        │
│   • valuesFiles: v1.6.0.yaml → v1.4.0.yaml                             │
│   • Commit & push → Fleet redeploys v1.4 automatically                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Upgrade Workflow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                          UPGRADE FLOW                                   │
│                                                                         │
│   ┌──────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────┐  │
│   │   New    │    │   CI/CD      │    │   DevOps   │    │   CI/CD  │  │
│   │ Release  │───►│   Creates    │───►│  Updates   │───►│ Promotes │  │
│   │  Tagged  │    │ Version File │    │   Pilot    │    │   All    │  │
│   └──────────┘    └──────────────┘    └────────────┘    └──────────┘  │
│                                              │                          │
│                                              ▼                          │
│                                       ┌────────────┐                   │
│                                       │  Validate  │                   │
│                                       │  & Test    │                   │
│                                       └────────────┘                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Who Does What?**

| Stage | Actor | Action |
|-------|-------|--------|
| Version file creation | DevOps/Platform Team | Creates `versions/v1.x.0.yaml` with new image tags |
| Pilot deployment | DevOps/Platform Team | Adds pilot cluster entry in `fleet.yaml` with new version |
| Validation | QA/Customer Success | Tests the pilot cluster |
| Production rollout | DevOps/Platform Team | Updates cluster entries in `fleet.yaml` to new version |

---

## Getting Started

### Prerequisites

1. **Rancher with Fleet** installed on management cluster
2. **Helm charts published** to a Helm repository (e.g., GitHub Pages, ChartMuseum, OCI registry)
3. **Downstream clusters** registered with Rancher
4. **GitRepo resource created** in Fleet pointing to your gitops-fleet repository (see below)

### Setup Steps

#### 1. Create GitRepo Resource in Fleet (Required)

Before Fleet can deploy anything, you must create a `GitRepo` resource pointing to the store-stack folder:

```yaml
# Apply this to your Fleet management cluster
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: store-stack
  namespace: fleet-default
spec:
  repo: https://github.com/your-org/gitops-fleet-2
  branch: main
  paths:
    - apps/store-stack
  pollingInterval: 30s
```

#### 2. Configure Per-Cluster Targeting in fleet.yaml

```yaml
# apps/store-stack/fleet.yaml

defaultNamespace: default

helm:
  repo: https://saurabh-newera.github.io/helm-charts
  chart: store-stack

targetCustomizations:
  - name: saurabh-kubernetes-cluster
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: saurabh-kubernetes-cluster
    helm:
      version: 1.4.0
      valuesFiles:
        - ../../clusters/saurabh-kubernetes-cluster.yaml
        - versions/v1.4.0.yaml

  - name: edge-store-001
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: edge-store-001
    helm:
      version: 1.4.0
      valuesFiles:
        - ../../clusters/edge-store-001.yaml
        - versions/v1.4.0.yaml
        - versions/v1.4.0.yaml
        - ../../clusters/edge-store-002.yaml
  
  # ... more clusters on v1.4
```

```yaml
# apps/store-stack-v1.6/fleet.yaml

defaultNamespace: default

helm:
  releaseName: store-stack

targetCustomizations:
  - name: edge-store-010
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: edge-store-010
    helm:
      valuesFiles:
        - versions/v1.6.0.yaml
        - ../../clusters/edge-store-010.yaml
```

#### 3. Define Your Value Layers

**Version-specific (`apps/store-stack/versions/v1.4.0.yaml`):**
```yaml
# ONLY image tags and version-specific defaults
nginx-service:
  image:
    repository: nginx
    tag: "1.26.0"

httpbin-service:
  image:
    repository: kennethreitz/httpbin
    tag: "latest"

redis-service:
  image:
    repository: redis
    tag: "7.2.0"

postgres:
  image:
    repository: postgres
    tag: "15.4"
```

**Cluster-specific (`clusters/saurabh-kubernetes-cluster.yaml`):**
```yaml
# ALL cluster-specific configuration
nginx-service:
  replicaCount: 2
  service:
    type: LoadBalancer
    loadBalancerIP: "10.100.10.1"
  resources:
    limits:
      cpu: "320m"
      memory: "356Mi"
    requests:
      cpu: "95m"
      memory: "94Mi"
  env:
    APP_ENV: "production"

httpbin-service:
  replicaCount: 2
  service:
    type: NodePort
    nodePort: 30080
  env:
    APP_ENV: "production"
    LOG_LEVEL: "info"

redis-service:
  replicaCount: 1
  env:
    REDIS_MAXMEMORY: "256mb"

postgres:
  replicaCount: 1
  service:
    type: LoadBalancer
    loadBalancerIP: "10.100.10.2"
  storage:
    size: "20Gi"
    storageClass: "longhorn"
```

**Small edge cluster (`clusters/edge-store-002.yaml`):**
```yaml
nginx-service:
  replicaCount: 1              # Smaller store
  service:
    type: LoadBalancer
    loadBalancerIP: "10.100.1.20"
  resources:
    limits:
      cpu: "200m"
      memory: "256Mi"           # Less RAM
  env:
    APP_ENV: "production"
    STORE_ID: "002"

postgres:
  persistence:
    size: "10Gi"                # Less storage
    storageClass: "local-path"
```

---

## Key Advantages of This Approach

| Advantage | Benefit |
|-----------|---------|
| **Single app folder** | No duplication - one `store-stack` folder for all versions |
| **Cluster configs are reusable** | Same cluster file used across versions - no duplication |
| **Clean separation** | Image tags in `versions/*.yaml`, cluster config in `clusters/*.yaml` |
| **Scalable fleet.yaml** | One entry per cluster (not per version per cluster) |
| **Simple upgrades** | Just change `version` and `valuesFiles` reference in fleet.yaml |
| **Full GitOps** | All configuration tracked in Git with complete audit history |

---

## Alternative Approaches

This guide presents one approach to multi-cluster GitOps with Fleet. Depending on your enterprise structure, business unit organization, or OEM requirements, there may be other valid patterns that better fit your needs. The key principles—GitOps, layered configuration, and controlled rollouts—can be implemented in various ways.

---

## Conclusion

Managing multiple Kubernetes clusters doesn't have to be chaotic. With a well-structured GitOps approach:

- **All configuration is version-controlled** and auditable
- **Clusters can have unique configurations** while sharing common defaults
- **Upgrades are controlled** through a release train model
- **Rollbacks are simple**—just revert a file reference

The separation of concerns between Helm charts (what to deploy) and GitOps configuration (how to deploy it) provides flexibility and maintainability at scale.

---