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
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌───────────┐  │
│   │  Cluster A  │   │  Cluster B  │   │  Cluster C  │   │  Cluster  │  │
│   │  (Prod-US)  │   │  (Prod-EU)  │   │  (Staging)  │   │     N     │  │
│   │             │   │             │   │             │   │    ...    │  │
│   │  3 replicas │   │  5 replicas │   │  1 replica  │   │           │  │
│   │  8GB RAM    │   │  16GB RAM   │   │  2GB RAM    │   │           │  │
│   │  IP: 10.1.x │   │  IP: 10.2.x │   │  IP: 10.3.x │   │           │  │
│   └─────────────┘   └─────────────┘   └─────────────┘   └───────────┘  │
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
Commit to Git
Fleet syncs all clusters
Git history = audit log
Guaranteed consistency
git revert = instant
Git blame = accountability
Time to update 1000 clusters: Minutes (automatic sync)
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
# Final values for prod-us cluster running v1.4.0:
replicaCount: 5              # from clusters/prod-us.yaml
loadBalancerIP: 10.1.1.1     # from clusters/prod-us.yaml
storageClass: fast-ssd       # from clusters/prod-us.yaml
image:
  repository: nginx          # from values.yaml (base)
  tag: "1.27.0"              # from versions/v1.4.0.yaml
```

---

## Why Different Clusters Need Different Configurations

Every cluster is unique. Here's why one-size-fits-all doesn't work:

```
┌─────────────────────────────────────────────────────────────────────────┐
│              WHY CLUSTERS NEED DIFFERENT CONFIGURATIONS                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 🖥️  HARDWARE CAPACITY                                           │   │
│  │                                                                  │   │
│  │  Production Cluster        │    Edge/Small Cluster              │   │
│  │  • 5 replicas              │    • 1 replica                     │   │
│  │  • 4GB memory per pod      │    • 512MB memory per pod          │   │
│  │  • CPU: 2 cores            │    • CPU: 0.5 cores                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 🌐 NETWORK CONFIGURATION                                        │   │
│  │                                                                  │   │
│  │  US Region                 │    EU Region                       │   │
│  │  • LB IP: 10.100.1.x       │    • LB IP: 10.200.1.x             │   │
│  │  • NodePort: 30080         │    • NodePort: 30081               │   │
│  │  • Ingress: us.example.com │    • Ingress: eu.example.com       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 🏢 ENVIRONMENT DIFFERENCES                                      │   │
│  │                                                                  │   │
│  │  Production                │    Staging                         │   │
│  │  • LOG_LEVEL: warn         │    • LOG_LEVEL: debug              │   │
│  │  • Monitoring: enabled     │    • Monitoring: minimal           │   │
│  │  • Backups: hourly         │    • Backups: disabled             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 👥 CUSTOMER/TENANT REQUIREMENTS                                 │   │
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
gitops-fleet/
│
├── apps/
│   └── store-stack/                    # Application bundle
│       │
│       ├── Chart.yaml                  # Umbrella chart (pulls published Helm charts)
│       ├── values.yaml                 # Base defaults for all clusters
│       ├── fleet.yaml                  # Fleet targeting & customization rules
│       │
│       ├── versions/                   # Release-specific configurations
│       │   ├── v1.4.0.yaml             # Stable release image tags
│       │   ├── v1.5.0.yaml             # New release image tags
│       │   └── v1.6.0.yaml             # Future release
│       │
│       └── clusters/                   # Per-cluster configurations
│           ├── prod-us-cluster.yaml    # Production US settings
│           ├── prod-eu-cluster.yaml    # Production EU settings
│           ├── staging-cluster.yaml    # Staging environment
│           └── pilot-cluster.yaml      # Pilot/canary cluster
│
└── README.md
```

**How It Helps:**

| Directory | Purpose | Benefit |
|-----------|---------|---------|
| `versions/` | Image tags per release | Change versions without touching cluster configs |
| `clusters/` | Per-cluster settings | Each cluster's unique needs in one place |
| `fleet.yaml` | Targeting rules | Control which cluster gets which configuration |

---

## Release Train: Upgrade Strategy

Upgrading applications across multiple clusters requires a controlled approach. Here's our release train model:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RELEASE TRAIN WORKFLOW                             │
│                                                                         │
│   STEP 1: CI Creates Version File                                       │
│   ════════════════════════════════                                      │
│                                                                         │
│   Git Tag: v1.6.0                                                       │
│        │                                                                │
│        ▼                                                                │
│   ┌─────────────┐     Creates      ┌─────────────────────────────┐     │
│   │ CI Pipeline │ ───────────────► │ versions/v1.6.0.yaml        │     │
│   └─────────────┘                  │                             │     │
│                                    │ nginx-service:              │     │
│                                    │   image:                    │     │
│                                    │     tag: "1.29.0"           │     │
│                                    │ redis-service:              │     │
│                                    │   image:                    │     │
│                                    │     tag: "7.4.0"            │     │
│                                    └─────────────────────────────┘     │
│                                                                         │
│   STEP 2: Deploy to Pilot Clusters (Manual/DevOps)                     │
│   ════════════════════════════════════════════════                     │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ fleet.yaml                                                      │  │
│   │                                                                 │  │
│   │ targetCustomizations:                                           │  │
│   │   - name: pilot-cluster           # ◄── Updated to v1.6.0      │  │
│   │     clusterSelector: ...                                        │  │
│   │     helm:                                                       │  │
│   │       valuesFiles:                                              │  │
│   │         - versions/v1.6.0.yaml    # ◄── NEW VERSION            │  │
│   │         - clusters/pilot.yaml                                   │  │
│   │                                                                 │  │
│   │   - name: prod-clusters           # Still on v1.5.0            │  │
│   │     helm:                                                       │  │
│   │       valuesFiles:                                              │  │
│   │         - versions/v1.5.0.yaml    # ◄── UNCHANGED              │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   STEP 3: Validate & Test                                              │
│   ═══════════════════════                                              │
│                                                                         │
│   ┌────────────────┐                                                   │
│   │ Pilot Cluster  │ ◄── Running v1.6.0                               │
│   │                │                                                   │
│   │ ✓ Health checks│                                                   │
│   │ ✓ Smoke tests  │                                                   │
│   │ ✓ Customer UAT │                                                   │
│   └────────────────┘                                                   │
│                                                                         │
│   STEP 4: Gradual Rollout (CI or DevOps)                               │
│   ══════════════════════════════════════                               │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                                                                 │  │
│   │  Week 1:  pilot-cluster ──────► v1.6.0                          │  │
│   │                                                                 │  │
│   │  Week 2:  staging-cluster ────► v1.6.0                          │  │
│   │                                                                 │  │
│   │  Week 3:  prod-eu-cluster ────► v1.6.0                          │  │
│   │                                                                 │  │
│   │  Week 4:  prod-us-cluster ────► v1.6.0                          │  │
│   │                                                                 │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   ROLLBACK (If Issues Found)                                           │
│   ══════════════════════════                                           │
│                                                                         │
│   Simply revert the valuesFiles reference:                             │
│                                                                         │
│   versions/v1.6.0.yaml  ──►  versions/v1.5.0.yaml                      │
│                                                                         │
│   Commit & push. Fleet automatically rolls back.                       │
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

#### 1. Create GitRepo in Fleet (Required)

Before Fleet can deploy anything, you must create a `GitRepo` resource that tells Fleet where to find your configuration. This is a **one-time setup** that points Fleet to your gitops-fleet repository:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: store-stack
  namespace: fleet-default
spec:
  repo: https://github.com/your-org/gitops-fleet
  branch: main
  paths:
    - apps/store-stack
  targets:
    - name: all-clusters
      clusterSelector:
        matchLabels:
          environment: production
```

#### 2. Configure Cluster Targeting in fleet.yaml

```yaml
# apps/store-stack/fleet.yaml

defaultNamespace: default

helm:
  releaseName: store-stack

targetCustomizations:
  - name: prod-us-cluster
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: prod-us-cluster
    helm:
      valuesFiles:
        - values.yaml
        - versions/v1.4.0.yaml
        - clusters/prod-us-cluster.yaml

  - name: pilot-cluster
    clusterSelector:
      matchLabels:
        management.cattle.io/cluster-display-name: pilot-cluster
    helm:
      valuesFiles:
        - values.yaml
        - versions/v1.5.0.yaml
        - clusters/pilot-cluster.yaml
```

#### 3. Define Your Value Layers

**Base defaults (`values.yaml`):**
```yaml
nginx-service:
  replicaCount: 1
  image:
    repository: nginx
    tag: "latest"
  service:
    type: ClusterIP
```

**Version-specific (`versions/v1.4.0.yaml`):**
```yaml
nginx-service:
  image:
    tag: "1.27.0"

redis-service:
  image:
    tag: "7.2.0"
```

**Cluster-specific (`clusters/prod-us-cluster.yaml`):**
```yaml
nginx-service:
  replicaCount: 5
  service:
    type: LoadBalancer
    loadBalancerIP: "10.100.10.1"
  resources:
    limits:
      cpu: "500m"
      memory: "512Mi"
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
