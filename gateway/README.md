# Gateway API

official docs: **[RKE2 Networking Services](https://docs.rke2.io/networking/networking_services)**

Traefik is used as the **Gateway Controller** for the cluster through Kubernetes Gateway API.

Instead of using the traditional Kubernetes `Ingress` resource, we use:

```text
Gateway
   +
HTTPRoute
```

to expose applications and internal services.

## Traefik configuration

RKE2 is configured to use Traefik:

```yaml id="fg4n2k"
ingress-controller: traefik
```

The Traefik Helm values configure it to use Gateway API and disable the traditional Kubernetes Ingress provider:

```yaml id="17o5op"
service:
  enabled: true
  type: LoadBalancer
  ExternalTrafficPolicy: Cluster

providers:
  kubernetesIngress:
    enabled: false
  kubernetesGateway:
    enabled: true

ports:
  traefik:
    port: 8000
  web:
    port: 80
  websecure:
    port: 443
    http:
      tls:
        enabled: true

gateway:
  enabled: true
```

The important points are:

* `kubernetesIngress.enabled: false`: disables Traefik's traditional Kubernetes Ingress provider.
* `kubernetesGateway.enabled: true`: enables Traefik's Gateway API provider.
* `gateway.enabled: true`: enables Traefik Gateway support.
* `service.type: LoadBalancer`: exposes Traefik through a Kubernetes LoadBalancer service.
* `web` / `websecure`: provide HTTP and HTTPS entrypoints on ports `80` and `443`.

## Gateway API CRDs

The Gateway API CRDs are installed separately because they are not automatically installed with our RKE2 setup.

```bash id="p0ozpz"
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
```

This installs the Gateway API resource definitions required for resources such as:

```text
Gateway
HTTPRoute
GatewayClass
```

## Application routing

The Mates application uses a Gateway and an HTTPRoute in the `mates-app` namespace.

```text id="4rj14z"
                    Traefik
                       │
                 Gateway API
                       │
              mates.kube.local
                       │
              ┌────────┴────────┐
              │                 │
           /api, /ws             /
              │                 │
              ▼                 ▼
           backend           front-service
```

The application Gateway:

```yaml id="qks7dr"
kind: Gateway
metadata:
  name: mates-gateway
  namespace: mates-app
spec:
  gatewayClassName: traefik
```

The HTTPS listener terminates TLS using the `gw-cert` Secret.

The HTTPRoute forwards:

```text
/api  → backend:8000
/ws   → backend:8000
/     → front-service:80
```

The route is available through:

```text
mates.kube.local
```

## Hubble UI

Hubble UI is exposed using another Gateway and HTTPRoute:

```text id="fdw8tp"
hubble.kube.local
        │
        ▼
     kube-gw
        │
        ▼
   hubble-ui:80
```

The Gateway and HTTPRoute are deployed in the `kube-system` namespace.

## Longhorn

Longhorn's frontend is exposed in the same way:

```text id="w2lpy0"
longhorn.kube.local
        │
        ▼
   longhorn-gw
        │
        ▼
longhorn-frontend:80
```

The Gateway and HTTPRoute are deployed in the `longhorn-system` namespace.

## Overall flow

```text id="k6h1ts"
Client
  │
  ▼
LoadBalancer IP
  │
  ▼
Traefik
  │
  ▼
Gateway
  │
  ▼
HTTPRoute
  │
  ├── Mates Frontend
  ├── Mates Backend
  ├── Hubble UI
  └── Longhorn
```

Traefik acts as the common **Gateway Controller**, while each application or service defines its own `Gateway` and `HTTPRoute` resources.
