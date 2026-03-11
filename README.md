# VKS GitOps — App of Apps on vSphere Kubernetes Service

GitOps-driven lifecycle management for VKS clusters, add-ons, and workloads using Argo CD on the vSphere Supervisor.

This repository implements the **Argo CD App of Apps pattern** to declaratively manage the full stack — from vSphere Namespace registration, through VKS cluster provisioning with standard package add-ons, to application deployment — all from a single Git repository.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Git Repository                             │
│                                                                     │
│  root-app.yaml ──► bootstrap/                                       │
│                      ├── vSphereNS/          (sync-wave: 0)         │
│                      │   └── workload-ns     Register namespace     │
│                      ├── argoApps/           (sync-wave: 10, 12)    │
│                      │   ├── cl01-app        Provision cl01         │
│                      │   └── cl02-app        Provision cl02         │
│                      └── workloads/          (sync-wave: 20)        │
│                          └── app-1           Deploy workloads       │
│                                                                     │
│  clusters/                                                          │
│    ├── cl01/                                                        │
│    │   ├── cl01.yaml              Cluster API v1beta2 manifest      │
│    │   ├── argo-register.yaml     Auto-register with Argo CD        │
│    │   ├── cert-manager-addon.yaml  VKS AddonInstall                │
│    │   └── contour-addon.yaml       VKS AddonInstall                │
│    └── cl02/                                                        │
│        ├── cl02.yaml              Cluster with autoscaler enabled    │
│        └── argo-register.yaml     Auto-register with Argo CD        │
│                                                                     │
│  workloads/cl01/nginx/                                              │
│    ├── namespace.yaml      Application namespace                    │
│    ├── deployment.yaml     nginx-unprivileged deployment            │
│    └── service.yaml        LoadBalancer service                     │
└─────────────────────────────────────────────────────────────────────┘
```

## How It Works

A single `root-app.yaml` is applied to the Argo CD instance running on the Supervisor. Argo CD scans the `bootstrap/` directory and discovers child Applications, each with a sync wave that controls ordering:

| Wave | Resource | What Happens |
|------|----------|-------------|
| **0** | `ArgoNamespace` CR | Registers `workload-ns` with the Argo CD instance via the auto-attach service |
| **10** | Application `cl01-app` | Syncs `clusters/cl01/` — provisions VKS cluster, registers with Argo CD, installs cert-manager and contour add-ons |
| **12** | Application `cl02-app` | Syncs `clusters/cl02/` — provisions VKS cluster with cluster autoscaler enabled, registers with Argo CD |
| **20** | Application `app-1` | Deploys nginx workload into the `cl01` VKS cluster |

**The result:** Push YAML to Git → namespace registered → clusters provisioned with add-ons → clusters auto-registered → workloads deployed. No manual CLI commands after the initial root app.

## Clusters

### cl01 — Standard Cluster with Add-ons

| Parameter | Value |
|-----------|-------|
| ClusterClass | `builtin-generic-v3.6.0` |
| Kubernetes Version | `v1.35.0---vmware.2-vkr.4` |
| Control Plane | 1 replica |
| Worker Pool | 1 node (`node-pool-1`) |
| VM Class | `best-effort-medium` |
| Storage Class | `vsan-esa-default-policy-raid5` |
| CNI | Antrea |
| Add-ons | cert-manager, contour |
| Autoscaler | Disabled |

### cl02 — Autoscaler-Enabled Cluster

| Parameter | Value |
|-----------|-------|
| ClusterClass | `builtin-generic-v3.6.0` |
| Kubernetes Version | `v1.35.0---vmware.2-vkr.4` |
| Control Plane | 1 replica |
| Worker Pool | Autoscaled (min: 1, max: 3) |
| VM Class | `best-effort-medium` |
| Storage Class | `vsan-esa-default-policy-raid5` |
| CNI | Antrea |
| Add-ons | None (base cluster) |
| Autoscaler | **Enabled** via MachineDeployment annotations |

Cluster autoscaler is enabled by setting annotations on the MachineDeployment:

```yaml
metadata:
  annotations:
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "3"
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "1"
```

## VKS Add-on Management

Standard packages are installed declaratively via `AddonInstall` resources applied to the **Supervisor** (not inside the workload cluster). VKS automatically selects the latest compatible version for the cluster's VKr and handles lifecycle management during cluster upgrades.

```yaml
# Example: clusters/cl01/cert-manager-addon.yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cl01-cert-manager
  namespace: workload-ns
spec:
  addonRef:
    name: cert-manager
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: cl01
```

### Installed Add-ons

| Cluster | Add-on | AddonInstall | Installs Into Namespace |
|---------|--------|-------------|------------------------|
| cl01 | cert-manager | `cl01-cert-manager` | `cert-manager` (auto-created) |
| cl01 | contour | `cl01-contour` | `projectcontour` (auto-created) |

> **Note:** cert-manager is a prerequisite for contour and most other standard packages. VKS handles dependency ordering automatically.

### Available Add-ons

To add more standard packages, create additional `AddonInstall` files in the cluster directory:

| Add-on | `addonRef.name` | Purpose |
|--------|-----------------|---------|
| cert-manager | `cert-manager` | TLS certificate automation |
| contour | `contour` | Ingress controller (Envoy-based) |
| prometheus | `prometheus` | Metrics and alerting |
| cluster-autoscaler | `cluster-autoscaler` | Automatic node scaling |
| external-dns | `external-dns` | DNS record automation |
| ako | `ako` | NSX ALB integration |
| telegraf | `telegraf` | Node metrics collection |

## Prerequisites

- **VCF 9.0+** with Supervisor enabled
- **VKS 3.5.0+** (for add-on management; ClusterClass `builtin-generic-v3.5.0` or later)
- **Argo CD Supervisor Service** — [VMware Argo CD Operator](https://vsphere-tmm.github.io/Supervisor-Services/#argocd-operator)
- **Argo CD Auto-Attach Service** — [warroyo/argocd-attach-service](https://github.com/warroyo/argocd-attach-service)
- **Broadcom customized Argo CD CLI** (version must contain `vcf` suffix)
- A vSphere Namespace for the Argo CD instance (e.g., `argocd-instance-1`)
- Git repository accessible from the Supervisor network

## Quick Start

### 1. Verify Argo CD is Running

```bash
kubectl get pods -n argocd-instance-1
```

All pods should be `Running`: `argocd-server`, `argocd-application-controller`, `argocd-repo-server`, `argocd-redis`.

### 2. Log in to Argo CD

```bash
# Get the LoadBalancer IP
kubectl get svc argocd-server -n argocd-instance-1

# Get the admin password
kubectl get secret -n argocd-instance-1 argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

# Log in (use the customized Broadcom CLI)
argocd login <ARGOCD-LB-IP>
```

### 3. Register the Argo CD Instance's Own Namespace

```bash
argocd cluster add <SUPERVISOR-IP-OR-HOSTNAME> \
  --namespace argocd-instance-1 \
  --kubeconfig sc.kubeconfig
```

### 4. Apply the Root Application

This is the **only manual step**. Everything else is GitOps from here.

```bash
kubectl apply -f root-app.yaml
```

Or via CLI:

```bash
argocd app create vks-gitops-root \
  --repo https://github.com/skandpurohit/vks-gitops.git \
  --path bootstrap \
  --dest-name sp-lab-supervisor \
  --dest-namespace argocd-instance-1 \
  --directory-recurse \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 5. Watch the Cascade

```bash
# Watch all apps
argocd app list

# Watch cluster provisioning
kubectl get clusters -n workload-ns -w

# Verify add-ons are installed
kubectl get clusteraddon -n workload-ns

# Watch workload deployment (after cl01 is healthy)
argocd app get app-1
```


## Repository Structure

```
.
├── root-app.yaml                              # Entry point — apply once
├── bootstrap/                                 # Child Application definitions
│   ├── vSphereNS/
│   │   └── workload-ns.yaml                   # ArgoNamespace CR (wave 0)
│   ├── argoApps/
│   │   ├── cl01-app.yaml                      # App → clusters/cl01 (wave 10)
│   │   └── cl02-app.yaml                      # App → clusters/cl02 (wave 12)
│   └── workloads/
│       └── app-1.yaml                         # App → workloads/cl01 (wave 20)
├── clusters/                                  # VKS cluster definitions
│   ├── cl01/
│   │   ├── cl01.yaml                          # Cluster API v1beta2 manifest
│   │   ├── argo-register.yaml                 # ArgoCluster CR (auto-attach)
│   │   ├── cert-manager-addon.yaml            # AddonInstall: cert-manager
│   │   └── contour-addon.yaml                 # AddonInstall: contour
│   └── cl02/
│       ├── cl02.yaml                          # Cluster with autoscaler annotations
│       └── argo-register.yaml                 # ArgoCluster CR (auto-attach)
└── workloads/                                 # Application workloads per cluster
    └── cl01/
        └── nginx/
            ├── namespace.yaml                 # Creates demo-app namespace
            ├── deployment.yaml                # nginx-unprivileged deployment
            └── service.yaml                   # LoadBalancer service
```
