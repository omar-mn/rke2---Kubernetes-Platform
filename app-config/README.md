# Mates App

Kubernetes manifests for deploying the monolithic **Mates** application.

## Architecture

```text
                         Traefik
                            │
                     Gateway API
                            │
                    mates.kube.local
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
        Frontend Service             Backend Service
              │                           │
              ▼                           ▼
        React Frontend              Django Backend
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                              ▼                       ▼
                         PostgreSQL                 Redis
```

## Components

* **Frontend**: serves the React application through Nginx.
* **Backend**: Django application serving the API.
* **PostgreSQL**: persistent database for the application.
* **Redis**: in-memory data store used by the application.
* **Gateway**: Traefik Gateway API configuration for exposing the application over HTTP/HTTPS.
* **Secrets**: TLS certificate and application/database credentials used by the workloads.

## Kubernetes Resources

| File            | Purpose                           |
| --------------- | --------------------------------- |
| `frontend.yaml` | Frontend Deployment and Service   |
| `backend.yaml`  | Backend Deployment and Service    |
| `postgres.yaml` | PostgreSQL Deployment and Service |
| `redis.yaml`    | Redis Deployment and Service      |
| `gateWay/`      | Gateway API configuration         |
| `secrets/`      | TLS and application secrets       |

## Deployment

The manifests are stored in Git and synchronized to the Kubernetes cluster through **Argo CD**.

The application runs inside the:

```text
mates-app namespace.
```

