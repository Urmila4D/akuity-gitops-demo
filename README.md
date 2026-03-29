# akuity-gitops-demo

A GitOps-based progressive delivery system built on Argo CD, Kargo, and the Akuity Platform. Demonstrates multi-environment application promotion across dev, staging, and prod using namespace isolation on a single Kubernetes cluster.

---

## 1. Overview

This repository implements a GitOps deployment pipeline for a sample web application (`helm-guestbook`) across three isolated environments. The system uses:

- **Argo CD** (via Akuity Platform) to continuously sync application state from Git to Kubernetes
- **Kargo** to orchestrate progressive promotion across environments
- **Namespace isolation** to simulate multi-cluster behavior on a single local cluster
- **Git as the single source of truth** for all application and infrastructure configuration

The goal is to demonstrate a scalable, auditable delivery workflow where no change reaches production without passing through dev and staging first.

---

## 2. Architecture

### Component Interaction

```
GitHub Repository (source of truth)
        │
        │  Argo CD watches repo continuously
        ▼
Akuity Platform (cloud control plane)
  ├── Argo CD Instance (demo-argocd)
  │     └── Manages 3 Applications → syncs to cluster namespaces
  └── Kargo Instance (demo-kargo)
        └── Manages promotion pipeline → dev → staging → prod
        │
        ▼
Akuity Agent (installed in local cluster)
        │
        ▼
OrbStack Kubernetes Cluster (local-cluster)
  ├── namespace: dev      ← guestbook-dev
  ├── namespace: staging  ← guestbook-staging
  └── namespace: prod     ← guestbook-prod
```

### Multi-Environment Setup

Rather than provisioning three separate Kubernetes clusters, this implementation uses **namespace isolation** on a single OrbStack cluster. Each namespace represents a distinct environment with its own Argo CD Application, sync policy, and Kargo stage. This is equivalent in architecture to a real multi-cluster setup, the only difference is the destination server URL.

In a production deployment, each namespace would be replaced by a dedicated cluster with its own Akuity agent, providing independent RBAC, blast radius isolation, and the ability to deploy across cloud regions.

### Application Definition and Sync

Each environment is defined as a separate Argo CD `Application` manifest stored in the `apps/` directory. Argo CD continuously reconciles the desired state in Git against the live state in the cluster. Any drift is automatically corrected via `selfHeal`.

<img width="468" height="169" alt="Argo CD UI" src="https://github.com/user-attachments/assets/e80b4334-e456-4193-a3c9-559d97b1bec3" />

---

## 3. Setup

### Prerequisites

- [OrbStack](https://orbstack.dev) with Kubernetes enabled (or any local/cloud Kubernetes cluster)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) configured to target your cluster
- [Kargo CLI](https://docs.kargo.io/user-guide/installing-the-cli/) installed
- An [Akuity Platform](https://akuity.cloud) trial account
- A public GitHub repository

### Reproducing The Environment

#### Step 1 — Create Namespaces
```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

#### Step 2 — Set Up Akuity Platform
1. Log in to [akuity.cloud](https://akuity.cloud)
2. Create an Argo CD instance — name it `demo-argocd`
3. Connect your cluster:
   - Navigate to Argo CD → your instance → Clusters → Connect Cluster
   - Run the generated `kubectl apply` command in your terminal
4. Create a Kargo instance — name it `demo-kargo`
5. Register a Kargo agent:
   - Select **Akuity Platform-managed** (no local install required)
   - Link it to `demo-argocd` in the Remote Argo CD field

#### Step 3 — Deploy Argo CD Applications
Create the three applications via the Argo CD UI or by applying the manifests:

```bash
kubectl apply -f apps/dev/application.yaml
kubectl apply -f apps/staging/application.yaml
kubectl apply -f apps/prod/application.yaml
```

> **Note:** If applying via `kubectl`, ensure Argo CD CRDs are installed in your cluster first.
> Alternatively, create each application directly in the Argo CD UI using the values defined in each manifest.

#### Step 4 — Configure Kargo Pipeline
```bash
kargo login https://<your-kargo-url>.kargo.akuity.cloud --admin

kargo apply -f kargo/pipeline.yaml
```

---

## 4. Application Deployment

### Sample Application

The deployed application is [`helm-guestbook`](https://github.com/argoproj/argocd-example-apps/tree/master/helm-guestbook) a simple web frontend packaged as a Helm chart. It was chosen because it is lightweight, well-maintained, and demonstrates Helm-based deployment patterns.

### Application Configuration

Each Argo CD Application points to the same source repository and Helm chart path, but targets a different namespace:

```yaml
source:
  repoURL: https://github.com/argoproj/argocd-example-apps
  targetRevision: HEAD
  path: helm-guestbook
destination:
  server: http://cluster-local-cluster:8001
  namespace: dev  # staging / prod for respective apps
```

### Sync Behavior

| Environment | Sync Policy | Self-Heal | Rationale |
|---|---|---|---|
| dev | Automatic | Enabled | Fast feedback loop for developers |
| staging | Automatic | Enabled | Mirrors dev for QA validation |
| prod | Manual | Disabled | Requires explicit human approval |

### Healthy State Definition

An application is considered healthy when:
- All pods are in `Running` state with `1/1` containers ready
- Argo CD reports `Synced` live state matches Git state
- No resources are in `Degraded` or `Missing` health status

---

## 5. Promotion Workflow

### How Kargo Works

Kargo sits above Argo CD and controls **what version** reaches **which environment** and **when**. It does not replace Argo CD, it orchestrates it.

### Stages

<img width="468" height="187" alt="Stages" src="https://github.com/user-attachments/assets/60832678-e26a-435a-ae57-4868f9295aa0" />

Each stage maps to a distinct Argo CD application. A stage cannot receive freight that has not been verified in all upstream stages.

### Promotion Trigger

Promotions are **manually triggered** via the Kargo UI. When a new commit is detected in the source repository, the Warehouse creates a `Freight` resource. An operator then promotes that Freight through each stage sequentially.

This is intentional automated promotion to production without human approval is considered a risk in enterprise environments.

### Warehouse Configuration

The Warehouse subscribes to the `master` branch of the `argocd-example-apps` repository. Any new commit detected on that branch produces a new Freight resource available for promotion.

---

## 6. Design Decisions

### Namespace Isolation Over Separate Clusters
Chosen for simplicity and cost in a local demo context. The assignment explicitly suggests this pattern. The Argo CD Application manifests and Kargo pipeline are identical to what a real multi-cluster setup would use, only the destination server URL changes.

### Akuity Platform-Managed Kargo Agent
Selected over the self-hosted agent because all dependencies (GitHub, Argo CD) are publicly accessible. This eliminates operational overhead and is the recommended approach for public Git providers and Akuity-managed Argo CD.

### Git-Push Promotion Step In Kargo
The Kargo pipeline uses `git-push` to write promotion state to environment-specific branches (`env/dev`, `env/staging`, `env/prod`). This avoids direct Kargo-to-Argo CD API authorization requirements and keeps all state changes auditable in Git.

### Manual Production Sync
Production is set to manual sync in Argo CD. This ensures that even if Kargo promotes freight to prod, the actual cluster update requires a deliberate human action. This is a common enterprise control for production environments.

### Separate Application Manifests Per Environment
Rather than using a single ApplicationSet (which would be the next step for scale), three separate Application manifests were defined. This makes each environment's configuration explicit and independently reviewable.

---

## 7. Tradeoffs

### Single cluster with namespace isolation
I used one local cluster with separate namespaces for dev, staging, and prod instead of creating three separate clusters. This kept the setup lightweight and easier to manage for the assignment. The tradeoff is that it does not provide the same level of isolation as a true multi cluster setup. A cluster level issue would still affect all three environments.

### Separate Argo CD Applications per environment
I created one Argo CD Application per environment because it makes the targets easy to understand and review in the UI. The downside is that this approach creates some repetition. If the number of environments or applications grows, ApplicationSet would be a better fit to reduce manual work.

### Sample application instead of a custom application
I used the guestbook sample application so I could focus on the GitOps and promotion workflow rather than application development. This made the assignment easier to reproduce, but it also means the demo does not show environment specific application behavior or a more realistic service deployment.

### Manual promotion flow
The promotion path is intentionally simple and easy to follow. This works well for a demo because it makes each stage visible and easy to explain. The tradeoff is that a production setup would usually include stronger validation between stages, such as automated tests, policy checks, or approval gates.

### Platform managed setup
Using Akuity managed Argo CD and Kargo reduced the amount of infrastructure I had to run myself, which helped keep the focus on delivery and promotion patterns. The tradeoff is that some lower level platform behavior is abstracted away compared with a fully self managed installation.

---

## 8. Assumptions

- Local Kubernetes cluster is available and running via OrbStack
- All three environments (dev, staging, prod) are logically isolated via namespaces, physical cluster isolation is out of scope for this demo
- The Akuity Platform trial account has sufficient permissions to create Argo CD and Kargo instances
- GitHub repository is public. No authentication required for Argo CD to read source manifests
- Promotions are operator-initiated, no CI/CD trigger integration is assumed

---

## 9. Repository Structure

```
akuity-gitops-demo/
│
├── apps/
│   ├── dev/
│   │   └── application.yaml    # Argo CD Application for dev namespace
│   ├── staging/
│   │   └── application.yaml    # Argo CD Application for staging namespace
│   └── prod/
│       └── application.yaml    # Argo CD Application for prod namespace
│
├── kargo/
│   └── pipeline.yaml           # Kargo Project, Warehouse, and Stage definitions
│
├── .gitignore                  # Excludes .DS_Store and sensitive files
└── README.md                   # This file
```

### Key Files

**`apps/*/application.yaml`**
Defines an Argo CD Application resource for each environment. Specifies the source repository, Helm chart path, target cluster, and sync policy. The namespace field is the key differentiator between environments.

**`kargo/pipeline.yaml`**
Defines the complete Kargo promotion pipeline including the Project, Warehouse (source subscription), and three sequential Stages. Each stage uses `git-push` steps to write promotion state to environment-specific branches, which Argo CD then syncs from.

---
