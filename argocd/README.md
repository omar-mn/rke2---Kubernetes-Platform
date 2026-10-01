# Argo CD

official docs: **[Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)**

## what is Argo CD?

Argo CD is a **GitOps continuous delivery tool for Kubernetes**.

Instead of manually applying Kubernetes manifests to the cluster, Argo CD continuously compares the desired state stored in Git with the actual state in Kubernetes and synchronizes them.

```text
Git Repository
      │
      ▼
   Argo CD
      │
      ▼
Kubernetes Cluster
```

## Installation

Argo CD was installed using the official **Getting Started** guide.

The Argo CD CLI was also installed following the official documentation.

---

# Argo CD Application

We created an Argo CD `Application` to manage the Mates deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: mates
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/omar-mn/mates-config.git
    targetRevision: HEAD
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: mates-app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Main configuration

* `source`: defines where Argo CD gets the Kubernetes manifests from.

* `repoURL`: the Git repository containing the application manifests.

* `targetRevision: HEAD`: tracks the latest commit on the repository's default branch.

* `path: .`: uses the repository root as the manifest path.

* `destination`: defines where Argo CD deploys the application.

* `namespace: mates-app`: deploys the application resources into the `mates-app` namespace.

* `automated`: enables automatic synchronization with Git.

* `prune: true`: removes Kubernetes resources that were deleted from Git.

* `selfHeal: true`: automatically restores resources if their state is changed manually in the cluster.

---

# Private GitHub Repository

The `mates-config` repository is private, so Argo CD needs credentials to access it.

A GitHub token was created and registered in Argo CD using:

```bash
argocd repo add <repo-url> \
  --username <username> \
  --password <password>
```

This allows Argo CD to authenticate to the private repository and continuously pull the manifests.

---

# Private Docker Registry

The application images are stored in a private Docker Hub repository.

A Kubernetes `docker-registry` Secret was created to allow Kubernetes to authenticate to Docker Hub when pulling the images:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=3omar0 \
  --docker-password=<token> \
  --docker-email=<email> \
  --namespace=mates-app
```

The workloads reference this secret through `imagePullSecrets`:

```yaml
imagePullSecrets:
  - name: dockerhub-secret
```

This allows the cluster to pull private application images during deployment.

---

# GitOps Flow

```text
Developer
   │
   ▼
GitHub
mates-config
   │
   ▼
Argo CD
   │
   ├── pulls manifests
   ├── detects changes
   └── synchronizes cluster state
          │
          ▼
    Kubernetes Cluster
          │
          ▼
   Private Docker Hub
          │
          ▼
    Application Images
```

The Git repository is the **desired state**, while Argo CD continuously keeps the Kubernetes cluster synchronized with it.
