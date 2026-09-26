# 🐙 GitOps Continuous Delivery: Deploying ArgoCD on Kubernetes

[![ArgoCD](https://img.shields.io/badge/Argo%20CD-v2.13+-EF6035?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![CNCF](https://img.shields.io/badge/CNCF-Graduated%20Project-4B7C98?logo=cncf&logoColor=white)](https://www.cncf.io/)
[![GitOps](https://img.shields.io/badge/GitOps-Declarative%20Delivery-purple)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-v3+-0F1689?logo=helm&logoColor=white)](https://helm.sh/)

> **A complete step-by-step tutorial to deploy ArgoCD (the CNCF-graduated GitOps continuous delivery engine) on your local Kubernetes cluster, access the Web UI, manage apps via the CLI, and deploy your first self-healing GitOps application.**

---

## 📑 Table of Contents

1. [What is ArgoCD & The GitOps Philosophy?](#1-what-is-argocd--the-gitops-philosophy)
2. [ArgoCD Architecture & Component Breakdown](#2-argocd-architecture--component-breakdown)
3. [Prerequisites](#3-prerequisites)
4. [Method 1: Deploy with Helm (Recommended)](#method-1-deploy-with-helm-recommended)
   - [Step 1: Add Argo Helm Repository](#step-1-add-argo-helm-repository)
   - [Step 2: Custom Values & Install](#step-2-custom-values--install)
5. [Method 2: Deploy with Plain Kubernetes Manifests](#method-2-deploy-with-plain-kubernetes-manifests)
6. [Step 3: Verify ArgoCD Installation](#step-3-verify-argocd-installation)
7. [Step 4: Access the ArgoCD Web Dashboard](#step-4-access-the-argocd-web-dashboard)
   - [Retrieve the Initial Admin Password](#retrieve-the-initial-admin-password)
   - [Start Port-Forwarding](#start-port-forwarding)
   - [Login to the Web UI](#login-to-the-web-ui)
8. [Step 5: Install & Use the ArgoCD CLI](#step-5-install--use-the-argocd-cli)
9. [Step 6: Deploy Your First GitOps Application](#step-6-deploy-your-first-gitops-application)
   - [Create the Application Manifest (CRD)](#create-the-application-manifest-crd)
   - [Test GitOps Drift Detection & Self-Healing](#test-gitops-drift-detection--self-healing)
10. [Useful ArgoCD Administration Commands](#useful-argocd-administration-commands)
11. [Troubleshooting & Pro Tips](#troubleshooting--pro-tips)
12. [Cleanup](#cleanup)

---

## 1. What is ArgoCD & The GitOps Philosophy?

**GitOps** is an operational framework where **Git is the single source of truth** for your infrastructure and application definitions.

```
┌─────────────────────────────────┐       ┌─────────────────────────────────────────┐
│     Traditional CI/CD (Push)    │  VS   │             GitOps (Pull)               │
├─────────────────────────────────┤       ├─────────────────────────────────────────┤
│ • CI pipeline (GitHub Actions)  │       │ • CI only tests and commits to Git     │
│   holds cluster admin credentials│       │ • Cluster credentials NEVER leave K8s   │
│ • "Pushes" kubectl apply to K8s │       │ • ArgoCD runs inside K8s & "Pulls" code │
│ • Fails if cluster is offline   │       │ • Continuously corrects manual drift    │
│ • No auto-reconciliation        │       │ • Automated self-healing 24/7           │
└─────────────────────────────────┘       └─────────────────────────────────────────┘
```

### The 4 Principles of GitOps:
1. **Declarative:** Entire desired cluster state is described in Git (YAML, Helm, Kustomize).
2. **Version Controlled:** All changes are audited via Git commits, PRs, and commit history.
3. **Automated Pull:** Software agents automatically pull the desired state from Git.
4. **Continuous State Reconciliation:** If someone manually modifies a pod or service with `kubectl`, ArgoCD notices the drift and **automatically overwrites it back to the Git state!**

---

## 2. ArgoCD Architecture & Component Breakdown

```mermaid
flowchart TB
    subgraph GitWorld["🐙 Git Ecosystem"]
        GIT_REPO[("📦 Git Repository<br/>(GitHub / GitLab)<br/><i>Target State</i>")]
    end

    subgraph Developers["👥 Users & Tooling"]
        DEV["👨‍💻 Developer / DevOps"]
        CLI["💻 argocd CLI"]
        WEB["🌐 ArgoCD Web UI"]
    end

    subgraph K8s["☸️ Kubernetes Cluster (Namespace: argocd)"]
        subgraph ArgoCore["ArgoCD Control Plane"]
            SERVER["🚪 argocd-server<br/><i>(REST / gRPC API & Web Dashboard)</i>"]
            REPO_SRV["📚 argocd-repo-server<br/><i>(Clones repos, builds Helm/Kustomize)</i>"]
            CONTROLLER["🧠 argocd-application-controller<br/><i>(Compares Target State vs Live State)</i>"]
            REDIS[("⚡ argocd-redis<br/><i>(Cache for state & manifests)</i>")]
            DEX["🔐 argocd-dex-server<br/><i>(SSO / OpenID Connect Identity)</i>"]
        end

        subgraph Workloads["🎯 Target Application Namespaces"]
            APP_PROD["🚀 Production Workloads<br/>(Pods, Deployments, Services)"]
        end

        SERVER <--> REDIS
        CONTROLLER <--> REDIS
        CONTROLLER <--> REPO_SRV
        SERVER <--> DEX
    end

    DEV --> WEB
    DEV --> CLI
    WEB --> SERVER
    CLI --> SERVER

    REPO_SRV <==|1. Outbound Git Pull| GIT_REPO
    CONTROLLER <==|2. Monitors Cluster State| APP_PROD
    CONTROLLER ==|3. Auto-Heals / Synchronizes Drift| APP_PROD
```

### Core Components:
- **`argocd-server`**: Exposes the Web UI and gRPC/REST API consumed by CLI and developers.
- **`argocd-repo-server`**: Maintains a local cache of Git repositories and renders Kubernetes manifests using Helm, Kustomize, or Jsonnet.
- **`argocd-application-controller`**: The brain that compares live cluster state against Git. Triggers automatic synchronization when drift occurs.
- **`argocd-redis`**: Caching layer for high performance and reduced Git API rate limits.
- **`argocd-dex-server`**: Authentication proxy for integrating enterprise SSO (GitHub, Google, Okta, SAML).

---

## 3. Prerequisites

Before starting, ensure you have:
- [x] A running **KinD cluster** (see [Local Kubernetes Setup Guide](../README.md)).
- [x] `kubectl` configured (`kubectl get nodes`).
- [x] `helm` installed (v3+).
- [x] At least **2 GB free RAM** allocated to Docker.

---

## Method 1: Deploy with Helm (Recommended)

### Step 1: Add Argo Helm Repository
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Step 2: Custom Values & Install
For local development, we want an easy-to-use configuration without complex TLS checks:

```bash
# Create namespace
kubectl create namespace argocd

# Deploy ArgoCD via Helm
helm install argocd argo/argo-cd   --namespace argocd   --set server.service.type=ClusterIP   --set configs.params."server\.insecure"=true
```

> [!TIP]
> Setting `server.insecure=true` disables internal TLS redirection inside the pod so you can easily access the Web UI over standard HTTP through `port-forward` without SSL browser warnings!

---

## Method 2: Deploy with Plain Kubernetes Manifests

If you prefer applying the official pre-packaged Kubernetes manifests directly:

```bash
# 1. Create dedicated namespace
kubectl create namespace argocd

# 2. Apply official manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## Step 3: Verify ArgoCD Installation

Wait until all pods in the `argocd` namespace are `Running` and `Ready`:

```bash
kubectl get pods -n argocd -w
```

**Expected output:**
```text
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          1m
argocd-dex-server-79469588c8-j8wzp                  1/1     Running   0          1m
argocd-redis-59755f949c-f9xpw                       1/1     Running   0          1m
argocd-repo-server-677ff8bd87-5z2d8                 1/1     Running   0          1m
argocd-server-54bb688849-h8q7v                      1/1     Running   0          1m
```

---

## Step 4: Access the ArgoCD Web Dashboard

### Retrieve the Initial Admin Password
ArgoCD generates a random initial admin password during the first boot and saves it in a Kubernetes secret:

```bash
# Linux / macOS
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Windows PowerShell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

**Sample Output:**
```text
k8jF02_X9abCD1234
```
*(Copy this password for the next step!)*

### Start Port-Forwarding
Expose the ArgoCD server to your local machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
# If you didn't use server.insecure:
# kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Login to the Web UI
1. Open your browser and navigate to: **`http://localhost:8080`** (or `https://localhost:8080` if using SSL).
2. Enter credentials:
   - **Username:** `admin`
   - **Password:** *(The decoded password from above)*
3. Click **Sign In**! 🎉 You are now in the ArgoCD GitOps Dashboard!

---

## Step 5: Install & Use the ArgoCD CLI

The `argocd` CLI allows you to create applications, trigger manual syncs, and roll back deployments from your terminal.

### 🐧 Linux:
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 🍎 macOS:
```bash
brew install argocd
```

### 🪟 Windows:
```powershell
winget install Argo.ArgoCD
# or
choco install argocd-cli
```

### Log in via CLI:
```bash
# Login (use password retrieved from secret)
argocd login localhost:8080 --username admin --insecure

# Update the admin password to something memorable
argocd account update-password
```

---

## Step 6: Deploy Your First GitOps Application

In GitOps, we define applications declaratively using the **`Application`** Custom Resource Definition (CRD).

### Create the Application Manifest (CRD)

Save the following file as `guestbook-app.yaml`:

```yaml
# guestbook-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-demo
  namespace: argocd
spec:
  project: default
  # Source: Where the desired manifests live
  source:
    repoURL: "https://github.com/argoproj/argocd-example-apps.git"
    targetRevision: HEAD
    path: "guestbook"
  # Destination: Where to deploy the application
  destination:
    server: "https://kubernetes.default.svc"
    namespace: default
  # Sync Policy: Automated GitOps reconciliation
  syncPolicy:
    automated:
      prune: true     # Deletes K8s resources if they are deleted from Git
      selfHeal: true  # Reverts manual kubectl changes automatically
    syncOptions:
      - CreateNamespace=true
```

Apply the application to ArgoCD:
```bash
kubectl apply -f guestbook-app.yaml
```

Now look at your ArgoCD Web Dashboard (`http://localhost:8080`):
- You will see a new green application card: **`guestbook-demo`**.
- Status: **`Synced`** and **`Healthy`**!
- Click on the card to see the beautiful live visual tree of the Deployment, ReplicaSet, Pods, and Services!

---

### Test GitOps Drift Detection & Self-Healing 🧪

Let's test the superpower of GitOps! What happens if someone tries to manually tamper with the live cluster using `kubectl`?

#### 1. Manually scale the deployment down to 0:
```bash
kubectl scale deployment guestbook-ui --replicas=0 -n default
```

#### 2. Check pod status:
```bash
kubectl get pods -n default
```
Notice what happens: ArgoCD's application controller detects that the live state (`replicas: 0`) does NOT match the Git target state (`replicas: 1`).  
Because `selfHeal: true` is enabled, **ArgoCD immediately resurrects the pod back to 1 replica!** 🛡️

---

## Useful ArgoCD Administration Commands

| Task | Command |
| :--- | :--- |
| **List all deployed applications** | `argocd app list` |
| **Get detailed status of an app** | `argocd app get guestbook-demo` |
| **Manually trigger sync** | `argocd app sync guestbook-demo` |
| **View application logs** | `argocd app logs guestbook-demo` |
| **Delete an application and its pods** | `argocd app delete guestbook-demo` |
| **Delete application without deleting pods**| `argocd app delete guestbook-demo --cascade=false` |

---

## Troubleshooting & Pro Tips

> [!WARNING]
> **Secret "argocd-initial-admin-secret" not found:**  
> Once you change the default password or if you deployed via certain Helm charts, this secret is automatically deleted for security. If you lost the password, reset it using:
> ```bash
> kubectl -n argocd patch secret argocd-secret >   -p '{"stringData": {"admin.password": "$2a$10$rRyBsGSHK6.IG8fhdKyMu.OWJhp39KnQDxgDm3jUkqVDzpG.V6S.W"}}'
> # Password is now reset to: password
> ```

> [!TIP]
> **Git Webhooks for Instant Sync:**  
> By default, ArgoCD polls Git repositories every **3 minutes**. In production, configure a GitHub Webhook pointing to `https://argocd.mycompany.com/api/webhook`. Every `git push` will trigger an **instant zero-second deployment**!

---

## Cleanup

To completely remove ArgoCD from your cluster:

### If using Helm:
```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

### If using Manifests:
```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```
