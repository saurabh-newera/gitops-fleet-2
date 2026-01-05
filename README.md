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
├── clusters/                           # Per-cluster configurations (shared)
│   ├── edge-store-001.yaml             # Store 001 specific settings
│   ├── edge-store-002.yaml             # Store 002 specific settings
│   ├── edge-store-010.yaml             # Pilot store settings
│   └── ... (100s-1000s of files)
│
└── apps/
    ├── store-stack-v1.4/               # Version 1.4 (stable)
    │   ├── Chart.yaml                  # Helm chart deps (nginx 0.2.0, redis 0.2.0)
    │   ├── fleet.yaml                  # Lists clusters on v1.4
    │   └── versions/
    │       └── v1.4.0.yaml             # Image tags for v1.4
    │
    ├── store-stack-v1.5/               # Version 1.5
    │   ├── Chart.yaml                  # Helm chart deps (nginx 0.3.0, redis 0.3.0)
    │   ├── fleet.yaml                  # Lists clusters on v1.5
    │   └── versions/
    │       └── v1.5.0.yaml             # Image tags for v1.5
    │
    └── store-stack-v1.6/               # Version 1.6 (latest/pilot)
        ├── Chart.yaml                  # Helm chart deps (nginx 0.4.0, redis 0.4.0)
        ├── fleet.yaml                  # Lists clusters on v1.6
        └── versions/
            └── v1.6.0.yaml             # Image tags for v1.6
```

### Key Concepts:

| Directory/File | Purpose | Shared Across Versions? |
|----------------|---------|-------------------------|
| clusters/*.yaml | Per-cluster config (resources, IPs, store IDs) | Yes - reused by all versions |
| apps/store-stack-vX.X/ | Version-specific application bundle | No - isolated per version |
| versions/*.yaml | Image tags for a release | No - specific to version |
| Chart.yaml | Helm chart dependencies | No - different charts per version |
| fleet.yaml | Which clusters run this version | No - different per version |

---

## Release Train: Upgrade Strategy

Upgrading applications across multiple edge clusters requires isolation and controlled rollouts. Our version-based approach provides complete separation between releases.

### Version Isolation Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VERSION ISOLATION ARCHITECTURE                       │
│                                                                         │
│   Each version gets its own folder with:                                │
│   • Separate Chart.yaml (different Helm chart versions)                 │
│   • Separate fleet.yaml (different cluster assignments)                 │
│   • Separate versions/*.yaml (different image tags)                     │
│                                                                         │
│   Shared across all versions:                                           │
│   • clusters/*.yaml (cluster-specific configs reused)                   │
│                                                                         │
│   ┌───────────────────┐   ┌───────────────────┐   ┌─────────────────┐ │
│   │ store-stack-v1.4  │   │ store-stack-v1.5  │   │ store-stack-v1.6│ │
│   │                   │   │                   │   │                 │ │
│   │ Chart: nginx 0.2.0│   │ Chart: nginx 0.3.0│   │ Chart: nginx 0.4│ │
│   │ Images: 1.27.0    │   │ Images: 1.28.0    │   │ Images: 1.29.0  │ │
│   │                   │   │                   │   │                 │ │
│   │ Clusters:         │   │ Clusters:         │   │ Clusters:       │ │
│   │ • edge-store-001  │   │ • edge-store-005  │   │ • edge-store-010│ │
│   │ • edge-store-002  │   │ • edge-store-006  │   │   (pilot)       │ │
│   │ • edge-store-003  │   │                   │   │                 │ │
│   │ • ... (900+)      │   │                   │   │                 │ │
│   └───────────────────┘   └───────────────────┘   └─────────────────┘ │
│            │                       │                       │            │
│            └───────────────────────┴───────────────────────┘            │
│                                    │                                    │
│                          All use same cluster configs:                  │
│                          clusters/edge-store-001.yaml                   │
│                          clusters/edge-store-002.yaml                   │
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
│   • Publishes nginx-service 0.4.0, redis-service 0.4.0 to GitHub Pages │
│   • New Docker images: nginx:1.29.0, redis:7.5.0                       │
│                                                                         │
│   STEP 2: Platform Team Creates v1.6 Folder                            │
│   ════════════════════════════════════════════                         │
│                                                                         │
│   mkdir apps/store-stack-v1.6                                           │
│   • Create Chart.yaml with chart version 0.4.0 dependencies             │
│   • Create versions/v1.6.0.yaml with image tags                         │
│   • Create fleet.yaml listing pilot cluster only                        │
│   • Create GitRepo CRD pointing to store-stack-v1.6                     │
│                                                                         │
│   STEP 3: Deploy to Pilot (Automatic)                                  │
│   ════════════════════════════════════                                 │
│                                                                         │
│   # apps/store-stack-v1.6/fleet.yaml                                    │
│   targetCustomizations:                                                 │
│     - name: edge-store-010                                              │
│       helm:                                                             │
│         valuesFiles:                                                    │
│           - versions/v1.6.0.yaml         # New image tags              │
│           - ../../clusters/edge-store-010.yaml  # Same cluster config  │
│                                                                         │
│   Fleet deploys v1.6 to edge-store-010 within 30 seconds                │
│                                                                         │
│                                                                         │
│   STEP 4: Validate on Pilot                                            │
│   ═══════════════════════                                              │
│                                                                         │
│   • Health checks, smoke tests, customer UAT                            │
│   • Monitor for 2-3 days                                                │
│                                                                         │
│   STEP 5: Gradual Migration (Manual)                                   │
│   ════════════════════════════════                                     │
│                                                                         │
│   Week 2: Migrate 5-10 stores                                           │
│   • Remove from apps/store-stack-v1.4/fleet.yaml                        │
│   • Add to apps/store-stack-v1.6/fleet.yaml                             │
│                                                                         │
│   # apps/store-stack-v1.4/fleet.yaml (REMOVE)                           │
│   targetCustomizations:                                                 │
│     - name: edge-store-001                                              │
│     - name: edge-store-002                                              │
│     # edge-store-003 REMOVED ◄──                                        │
│                                                                         │
│   # apps/store-stack-v1.6/fleet.yaml (ADD)                              │
│   targetCustomizations:                                                 │
│     - name: edge-store-010  # pilot                                     │
│     - name: edge-store-003  # ◄── ADDED                                 │
│       helm:                                                             │
│         valuesFiles:                                                    │
│           - versions/v1.6.0.yaml                                        │
│           - ../../clusters/edge-store-003.yaml  # Same config!          │
│                                                                         │
│   Week 3-12: Continue batch migrations                                  │
│   • 10-50 stores per week                                               │
│   • Monitor after each batch                                            │
│                                                                         │
│   ROLLBACK (If Issues Found)                                           │
│   ══════════════════════════                                           │
│                                                                         │
│   Move cluster back from v1.6 to v1.4:                                 │
│   • Remove from apps/store-stack-v1.6/fleet.yaml                        │
│   • Add back to apps/store-stack-v1.4/fleet.yaml                        │
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
| Version file creation | CI Pipeline | Automatically creates `versions/v1.x.0.yaml` on git tag |
| Pilot deployment | DevOps/Platform Team | Updates `fleet.yaml` to point pilot cluster to new version |
| Validation | QA/Customer Success | Tests the pilot cluster |
| Production rollout | CI Pipeline or DevOps | Updates remaining clusters in `fleet.yaml` |

---

## Getting Started

### Prerequisites

1. **Rancher with Fleet** installed on management cluster
2. **Helm charts published** to a Helm repository (e.g., GitHub Pages, ChartMuseum, OCI registry)
3. **Downstream clusters** registered with Rancher
4. **GitRepo resource created** in Fleet pointing to your gitops-fleet repository (see below)

### Setup Steps

#### 1. Create GitRepo Resources in Fleet (Required)

Before Fleet can deploy anything, you must create `GitRepo` resources for each version. Each version gets its own GitRepo pointing to its folder:

```yaml
# gitrepos/store-stack-v1.4.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: store-stack-v1.4
  namespace: fleet-default
spec:
  repo: https://github.com/your-org/gitops-fleet-2
  branch: main
  paths:
    - apps/store-stack-v1.4
  pollingInterval: 30s
```

```yaml
# gitrepos/store-stack-v1.6.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: store-stack-v1.6
  namespace: fleet-default
spec:
  repo: https://github.com/your-org/gitops-fleet-2
  branch: main
  paths:
    - apps/store-stack-v1.6
  pollingInterval: 15s      # Faster sync for pilot testing
```

#### 2. Configure Cluster Targeting in fleet.yaml

```yaml
# apps/store-stack-v1.4/fleet.yaml

defaultNamespace: default

helm:
  releaseName: store-stack

targetCustomizations:
  - name: edge-store-001
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: edge-store-001
    helm:
      valuesFiles:
        - versions/v1.4.0.yaml
        - ../../clusters/edge-store-001.yaml

  - name: edge-store-002
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: edge-store-002
    helm:
      valuesFiles:
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

**Version-specific (`apps/store-stack-v1.4/versions/v1.4.0.yaml`):**
```yaml
nginx-service:
  image:
    repository: nginx
    tag: "1.27.0"

httpbin-service:
  image:
    tag: "latest"

redis-service:
  image:
    tag: "7.2.0"

postgres:
  image:
    tag: "15.4"
```

**Cluster-specific (`clusters/edge-store-001.yaml`):**
```yaml
nginx-service:
  replicaCount: 3
  service:
    type: LoadBalancer
    loadBalancerIP: "10.100.1.10"
  resources:
    limits:
      cpu: "1000m"
      memory: "2Gi"
  env:
    - name: STORE_ID
      value: "001"
    - name: REGION
      value: "us-east"

postgres:
  persistence:
    size: "100Gi"
    storageClass: "fast-ssd"
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
      cpu: "500m"
      memory: "512Mi"           # Less RAM
  env:
    - name: STORE_ID
      value: "002"

postgres:
  persistence:
    size: "50Gi"                # Less storage
```

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