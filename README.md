# VKS GitOps — App of Apps on vSphere Kubernetes Service

GitOps-driven lifecycle management for VKS clusters and workloads using Argo CD on the vSphere Supervisor.

This repository implements the **Argo CD App of Apps pattern** to declaratively manage the full lifecycle — from vSphere Namespace registration, through VKS cluster provisioning, to application deployment — all from a single Git repository.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Git Repository                            │
│                                                                  │
│  root-app.yaml ──► bootstrap/                                    │
│                      ├── vSphereNS/        (sync-wave: 0)        │
│                      │   └── workload-ns   Register namespace    │
│                      ├── argoApps/         (sync-wave: 10)       │
│                      │   └── cl01-app      Provision VKS cluster │
│                      └── applications/     (sync-wave: 20)       │
│                          └── app-1         Deploy workloads      │
│                                                                  │
│  clusters/cl01/                                                  │
│    ├── cl01.yaml           Cluster API v1beta2 manifest          │
│    └── argo-register.yaml  Auto-register cluster with Argo CD    │
│                                                                  │
│  workloads/cl01/nginx/                                           │
│    ├── namespace.yaml      Application namespace                 │
│    ├── deployment.yaml     nginx-unprivileged deployment         │
│    └── service.yaml        LoadBalancer service                  │
└──────────────────────────────────────────────────────────────────┘
```

### How It Works

A single `root-app.yaml` is applied to the Argo CD instance running on the Supervisor. Argo CD scans the `bootstrap/` directory and discovers three child Applications, each with a sync wave that controls ordering:

| Wave | What Happens | Directory |
|------|-------------|-----------|
| **0** | `ArgoNamespace` CR registers `workload-ns` with the Argo CD instance via the auto-attach service | `bootstrap/vSphereNS/` |
| **10** | Argo CD Application `cl01-app` syncs the cluster manifest and `ArgoCluster` CR to the Supervisor, provisioning a VKS cluster and registering it back to Argo CD | `bootstrap/argoApps/` → `clusters/cl01/` |
| **20** | Argo CD Application `app-1` deploys workloads into the now-running VKS cluster | `bootstrap/applications/` → `workloads/cl01/` |

The result is a **fully closed GitOps loop**: push YAML to Git → namespace registered → cluster provisioned → cluster auto-registered → workloads deployed. No manual `argocd` CLI commands after the initial root app.

## Prerequisites

- **VCF 9.0+** with Supervisor enabled
- **VKS 3.3.0+** (ClusterClass `builtin-generic-v3.3.0` or later)
- **Argo CD Supervisor Service** installed ([VMware Argo CD Operator](https://vsphere-tmm.github.io/Supervisor-Services/#argocd-operator))
- **Argo CD Auto-Attach Service** installed ([warroyo/argocd-attach-service](https://github.com/warroyo/argocd-attach-service))
- **Broadcom customized Argo CD CLI** (version must contain `vcf` suffix)
- A vSphere Namespace for the Argo CD instance (e.g., `argocd-instance-1`)
- Git repository accessible from the Supervisor network

## Quick Start

### 1. Verify Argo CD is running

```bash
kubectl get pods -n argocd-instance-1
```

All pods (`argocd-server`, `argocd-application-controller`, `argocd-repo-server`, `argocd-redis`) should be `Running`.

### 2. Log in to Argo CD

```bash
# Get the LoadBalancer IP
kubectl get svc argocd-server -n argocd-instance-1

# Get the admin password
kubectl get secret -n argocd-instance-1 argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

# Log in
argocd login <ARGOCD-LB-IP>
```

### 3. Register the Argo CD instance's own namespace

```bash
argocd cluster add <SUPERVISOR-IP-OR-HOSTNAME> \
  --namespace argocd-instance-1 \
  --kubeconfig sc.kubeconfig
```

### 4. Apply the root application

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

### 5. Watch the cascade

```bash
# Watch all apps
argocd app list

# Watch cluster provisioning
kubectl get clusters -n workload-ns -w

# Watch workload deployment (after cluster is healthy)
argocd app get app-1
```

## Repository Structure

```
.
├── root-app.yaml                          # Entry point — apply this once
├── bootstrap/                             # Child Application definitions
│   ├── vSphereNS/
│   │   └── workload-ns.yaml               # ArgoNamespace CR (wave 0)
│   ├── argoApps/
│   │   └── cl01-app.yaml                  # App targeting clusters/cl01 (wave 10)
│   └── applications/
│       └── app-1.yaml                     # App targeting workloads/cl01 (wave 20)
├── clusters/                              # VKS cluster definitions
│   └── cl01/
│       ├── cl01.yaml                      # Cluster API v1beta2 manifest
│       └── argo-register.yaml             # ArgoCluster CR (auto-attach)
└── workloads/                             # Application workloads per cluster
    └── cl01/
        └── nginx/
            ├── namespace.yaml
            ├── deployment.yaml
            └── service.yaml
```
