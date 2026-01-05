# GitOps Fleet Configuration

This repository contains Fleet (by Rancher) configurations for deploying applications to multiple Kubernetes clusters.

## Repository Structure

```
gitops-fleet-2/
│
├── apps/                           # Application deployments
│   └── store-stack/                # Store Stack application
│       ├── Chart.yaml              # Umbrella Helm chart
│       ├── values.yaml             # Default values
│       ├── fleet.yaml              # Fleet targeting configuration
│       ├── versions/               # Version-specific image tags
│       │   ├── v1.4.0.yaml
│       │   └── v1.5.0.yaml
│       └── clusters/               # Per-cluster configurations
│           ├── anakin-cluster.yaml
│           ├── anakin-cluster-2.yaml
│           └── saurabh-kubernetes-cluster.yaml
│
├── gitrepos/                       # GitRepo CRD definitions
│   ├── fleet.yaml                  # Target local cluster
│   └── store-stack.yaml            # GitRepo for store-stack
│
└── README.md
```

## How It Works

### Value Layering

Values are applied in this order (later overrides earlier):

1. `values.yaml` - Base defaults
2. `versions/<release-version>.yaml` - Image tags for the release
3. `clusters/<cluster-name>.yaml` - Cluster-specific configuration

### Cluster Labels

Each cluster needs these labels set (via Rancher UI or kubectl):

| Label | Purpose | Example Values |
|-------|---------|----------------|
| `release-version` | Which version to deploy | `v1.4.0`, `v1.5.0` |

**To set labels via kubectl:**
```bash
kubectl label cluster.fleet.cattle.io <cluster-name> -n fleet-default \
  release-version=v1.4.0
```

### Current Cluster Assignments

| Cluster | Release Version | Environment |
|---------|-----------------|-------------|
| `anakin-cluster` | v1.4.0 | Production |
| `saurabh-kubernetes-cluster` | v1.4.0 | Production |
| `anakin-cluster-2` | v1.5.0 | Pilot/Testing |

## Promotion Workflow

### Promoting a Cluster to a New Version

1. **Test on pilot cluster first** (anakin-cluster-2 runs v1.5.0)

2. **When ready to promote**, change the cluster's label:
   ```bash
   kubectl label cluster.fleet.cattle.io anakin-cluster -n fleet-default \
     release-version=v1.5.0 --overwrite
   ```

3. Fleet automatically deploys the new version

### Creating a New Release

1. Create `apps/store-stack/versions/v1.6.0.yaml` with new image tags

2. Update a pilot cluster to test:
   ```bash
   kubectl label cluster.fleet.cattle.io anakin-cluster-2 -n fleet-default \
     release-version=v1.6.0 --overwrite
   ```

3. Update GitRepo targeting (if needed) in `gitrepos/store-stack.yaml`

## Setup Instructions

### 1. Add GitRepo definitions to Fleet

In Rancher UI:
1. Go to **Continuous Delivery** → **Git Repos**
2. Switch namespace to **fleet-local**
3. Click **Add Repository**
4. Configure:
   - **Name**: `gitops-fleet-gitrepos`
   - **Repository URL**: `https://github.com/saurabh-newera/gitops-fleet-2`
   - **Branch**: `main`
   - **Paths**: `gitrepos`

### 2. Set Cluster Labels

For each downstream cluster, set the required labels:

```bash
# For anakin-cluster (v1.4.0)
kubectl label cluster.fleet.cattle.io anakin-cluster -n fleet-default \
  release-version=v1.4.0

# For saurabh-kubernetes-cluster (v1.4.0)
kubectl label cluster.fleet.cattle.io saurabh-kubernetes-cluster -n fleet-default \
  release-version=v1.4.0

# For anakin-cluster-2 (v1.5.0 - pilot)
kubectl label cluster.fleet.cattle.io anakin-cluster-2 -n fleet-default \
  release-version=v1.5.0
```

### 3. Verify Deployment

```bash
# Check GitRepo status
kubectl get gitrepo -n fleet-default

# Check Bundle status
kubectl get bundle -n fleet-default

# Check BundleDeployment status
kubectl get bundledeployment -n fleet-default
```

## Updating Cluster Configuration

### Change Cluster-Specific Settings

Edit the appropriate file in `apps/store-stack/clusters/`:

```yaml
# clusters/anakin-cluster.yaml
nginx-service:
  replicaCount: 5  # Changed from 3
```

Commit and push. Fleet will apply the changes.

### Change Image Versions (New Release)

Edit or create a version file in `apps/store-stack/versions/`:

```yaml
# versions/v1.6.0.yaml
nginx-service:
  image:
    tag: "1.29.0"
```

Then update cluster labels to use the new version.

## Troubleshooting

### Check Fleet Logs

```bash
kubectl logs -n cattle-fleet-system -l app=fleet-controller
```

### Check Bundle Status

```bash
kubectl describe bundle store-stack -n fleet-default
```

### Force Reconciliation

```bash
kubectl annotate gitrepo store-stack -n fleet-default \
  reconcile.fleet.cattle.io/force=true --overwrite
```
