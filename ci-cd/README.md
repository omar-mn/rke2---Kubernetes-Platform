# CI/CD Flow

This repository uses GitHub Actions for CI and Docker image publishing, with Argo CD handling deployments through a GitOps repository.

## Development Flow

Changes pushed to the `dev` branch trigger the development pipelines.

```text
Developer
   │
   ▼
Push to dev
   │
   ├───────────────┐
   ▼               ▼
Backend CI      Frontend CI
   │               │
   ▼               ▼
Tests          Build Docker Image
   │               │
   ▼               ▼
Build Image    Push to Docker Hub
   │
   ▼
Push to Docker Hub
```

Development images use the commit SHA with a `dev` prefix.

```text
Backend:  3omar0/mates:dev-<short-sha>
Frontend: 3omar0/mates-front:dev-<short-sha>
```

A separate development GitOps environment can consume these images and Argo CD will keep the cluster synchronized with the GitOps repository.

## Production Flow

Production releases are triggered by pushing a Git tag starting with `v`.

```text
Developer
   │
   ▼
Merge changes into main
   │
   ▼
Create Git Tag
   │
   ▼
v1.4.0
   │
   ├───────────────┐
   ▼               ▼
Backend CI      Frontend CI
   │               │
   ▼               ▼
Tests          Build Docker Image
   │               │
   ▼               ▼
Build Image    Push to Docker Hub
   │
   ▼
Push to Docker Hub
   │
   ▼
Update GitOps Repository
   │
   ▼
Argo CD detects the change
   │
   ▼
Kubernetes
```

Production images use the Git tag directly.

```text
Backend:  3omar0/mates:v1.4.0
Frontend: 3omar0/mates-front:v1.4.0
```

The production pipeline automatically updates the image tag in the GitOps repository, commits the change, and pushes it.

Argo CD detects the Git change and synchronizes the new version to the Kubernetes cluster.

## Overall Architecture

```text
                   GitHub Application Repo
                            │
             ┌──────────────┴──────────────┐
             │                             │
          dev branch                    v1.x.x tag
             │                             │
             ▼                             ▼
         CI: Test/Build                CI: Test/Build
             │                             │
             ▼                             ▼
       Docker Hub                    Docker Hub
             │                             │
             └──────────────┬──────────────┘
                            │
                            ▼
                    GitOps Repository
                            │
                            ▼
                         Argo CD
                            │
                            ▼
                       Kubernetes
```

### Key Principles

* `dev` images are identified by commit SHA.
* Production images are identified by Git release tags.
* Docker images are immutable versioned artifacts.
* The GitOps repository is the source of truth for deployments.
* CI builds and publishes images.
* Argo CD is responsible for applying the desired state to Kubernetes.
* Production image updates are automated through Git commits to the GitOps repository.
