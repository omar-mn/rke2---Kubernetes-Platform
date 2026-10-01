# Cilium installation

official docs: **[RKE2 network options](https://docs.rke2.io/networking/basic_network_options?CNIplugin=Cilium+CNI+Plugin)**

## What is a CNI?

CNI (Container Network Interface) is responsible for providing networking for Kubernetes Pods. It handles things such as assigning IP addresses to Pods and connecting Pod traffic to the rest of the cluster.

RKE2 supports several CNI plugins, and in our cluster we use **Cilium**. Cilium uses **eBPF** for its networking datapath, allowing it to handle Pod networking, Kubernetes Service load-balancing, network policies, and other networking features directly in the Linux kernel.

In our setup, Cilium also replaces `kube-proxy` and provides additional features such as native routing, LoadBalancer IP management, L2 announcements, and Hubble for network observability.

## rke2-config

The RKE2 configuration enables Cilium as the primary CNI and disables `kube-proxy`.

```yaml
cni: cilium

disable-kube-proxy: true
```

* `cni: cilium`: tells RKE2 to use Cilium as the cluster CNI.
* `disable-kube-proxy: true`: disables `kube-proxy` because Cilium will handle Kubernetes Service networking.

For kube-proxy replacement, Cilium is also configured with `kubeProxyReplacement: true`.

---

# Helm Chart Values

```yaml
# /var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChartConfig

metadata:
  name: rke2-cilium
  namespace: kube-system

spec:
  valuesContent: |-
    kubeProxyReplacement: true

    gatewayAPI:
      enabled: false

    k8sServiceHost: 172.16.4.100
    k8sServicePort: 6443

    routingMode: native
    autoDirectNodeRoutes: true
    ipv4NativeRoutingCIDR: 10.42.0.0/16

    ipam:
      mode: cluster-pool
      operator:
        clusterPoolIPv4PodCIDRList:
          - 10.42.0.0/16
        clusterPoolIPv4MaskSize: 22

    MTU: 1400

    ipv4:
      enabled: true

    ipv6:
      enabled: false

    enableIPv4Masquerade: true

    devices:
      - eth0
      - eth1
      - eth2

    nodePort:
      directRoutingDevice: eth2

    loadBalancer:
      mode: snat

    l2announcements:
      enabled: true
      leaseDuration: 15s
      leaseRenewDeadline: 5s
      leaseRetryPeriod: 2s

    hubble:
      enabled: true

      relay:
        enabled: true

      ui:
        enabled: true
        baseUrl: "/kube/hubble/"
```

---

## kube-proxy replacement

```yaml
kubeProxyReplacement: true
```

This enables Cilium to replace `kube-proxy` and handle Kubernetes Service traffic using eBPF.

This works together with:

```yaml
disable-kube-proxy: true
```

in the RKE2 configuration.

---

## Gateway API

```yaml
gatewayAPI:
  enabled: false
```

This disables **Cilium's own Gateway API implementation**.

Gateway API is handled separately by **Traefik** in this cluster.

This setting does not disable Traefik's Gateway API support.

---

## Kubernetes API server connection

```yaml
k8sServiceHost: 172.16.4.100
k8sServicePort: 6443
```

These values tell Cilium how to connect to the Kubernetes API server.

In the final cluster configuration:

```text
172.16.4.100:6443
        |
        v
kube-vip
        |
        v
Kubernetes API Server
```

The API VIP is used after the initial bootstrap stage.

---

## Routing mode

```yaml
routingMode: native
```

This enables **native routing** instead of tunneling.

Cilium does not encapsulate Pod-to-Pod traffic in VXLAN or another overlay. Instead, traffic is routed through the underlying node network.

```text
Pod
10.42.x.x
   |
   v
Cilium
   |
   v
Linux routing
   |
   v
eth2 / node fabric
```

---

## Automatic node routes

```yaml
autoDirectNodeRoutes: true
```

This allows Cilium to automatically create the direct routes required to reach the Pod CIDRs of other nodes.

---

## Native routing CIDR

```yaml
ipv4NativeRoutingCIDR: 10.42.0.0/16
```

This defines the IPv4 CIDR used for native Pod routing.

In our cluster:

```text
10.42.0.0/16
        |
        v
Pod network
```

---

## IP Address Management (IPAM)

```yaml
ipam:
  mode: cluster-pool
```

IPAM is responsible for allocating IP addresses for Cilium-managed Pods and endpoints.

The `cluster-pool` mode uses a cluster-wide Pod CIDR pool and assigns a separate CIDR to each node.

### Cluster Pod CIDR pool

```yaml
clusterPoolIPv4PodCIDRList:
  - 10.42.0.0/16
```

This defines the overall IPv4 pool.

### Per-node Pod CIDR

```yaml
clusterPoolIPv4MaskSize: 22
```

This divides the `/16` pool into `/22` blocks for individual nodes.

```text
10.42.0.0/16
      |
      +-- /22 -> Node 1
      +-- /22 -> Node 2
      +-- /22 -> Node 3
```

---

## MTU

```yaml
MTU: 1400
```

This sets the MTU used by Cilium's internal datapath interfaces.

It does not change the MTU of the underlying host interfaces themselves.

---

## IP families

```yaml
ipv4:
  enabled: true

ipv6:
  enabled: false
```

The cluster uses IPv4 networking only.

---

## IPv4 masquerading

```yaml
enableIPv4Masquerade: true
```

This enables IPv4 SNAT/masquerading for traffic leaving the native routing CIDR.

A simplified flow is:

```text
Pod
10.42.x.x
   |
   v
Cilium SNAT
   |
   v
Node IP
172.17.x.x
   |
   v
External network
```

---

## Network devices

```yaml
devices:
  - eth0
  - eth1
  - eth2
```

Cilium is allowed to use the three host interfaces:

```text
eth0 -> NAT network
eth1 -> Control Plane network
eth2 -> Pod/Fabric network
```

---

## NodePort direct routing device

```yaml
nodePort:
  directRoutingDevice: eth2
```

`eth2` is selected as the direct-routing interface because it is the dedicated Pod/Fabric network.

```text
rke2-1 eth2 -> 172.17.0.3
rke2-2 eth2 -> 172.17.0.34
rke2-3 eth2 -> 172.17.0.35
```

---

## LoadBalancer mode

```yaml
loadBalancer:
  mode: snat
```

This configures Cilium's LoadBalancer forwarding to use SNAT mode.

For traffic forwarded to a backend on another node, Cilium performs source NAT so that the return traffic can be handled by the original node.

---

# LoadBalancer IP Management

In this private/bare-metal environment there is no cloud LoadBalancer to allocate external IPs for Kubernetes `LoadBalancer` Services.

Cilium **LB IPAM** is therefore used to allocate LoadBalancer IPs from a predefined pool.

## CiliumLoadBalancerIPPool

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool

metadata:
  name: main-pool

spec:
  blocks:
    - start: 172.16.4.101
      stop: 172.16.4.150
```

This defines the IP range that Cilium can use for `LoadBalancer` Services.

```text
172.16.4.101 - 172.16.4.150
              |
              v
       LoadBalancer IP pool
```

For example, the Traefik Service currently receives:

```text
172.16.4.101
```

It is important to separate two responsibilities:

```text
LB IPAM
   |
   +-- allocates the IP

L2 Announcement
   |
   +-- advertises the IP
```

LB IPAM allocates addresses but does not make those addresses reachable on the physical network by itself.

---

# L2 LoadBalancer Announcements

After a LoadBalancer IP is allocated, Cilium needs to advertise that IP on the private Layer 2 network so external clients can reach it.

This is provided by **L2 Announcements**. Cilium responds to ARP requests for the configured LoadBalancer IPs.

## CiliumL2AnnouncementPolicy

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy

metadata:
  name: traefik-l2

spec:
  interfaces:
    - eth1

  externalIPs: true
  loadBalancerIPs: true
```

### `interfaces`

```yaml
interfaces:
  - eth1
```

Specifies that the LoadBalancer IPs should be advertised through `eth1`, which is our Control Plane network.

### `loadBalancerIPs`

```yaml
loadBalancerIPs: true
```

Tells Cilium to announce IPs assigned to `LoadBalancer` Services.

### `externalIPs`

```yaml
externalIPs: true
```

Also enables announcement of Service `ExternalIPs`.

In our setup, the important one is `loadBalancerIPs`.

---

## Why both are required

The complete LoadBalancer path is:

```text
        LoadBalancer Service
                 |
                 v
           Cilium LB IPAM
                 |
                 | allocates
                 v
           172.16.4.101
                 |
                 v
       Cilium L2 Announcement
                 |
                 | ARP on eth1
                 v
          Private Network
                 |
                 v
              Traefik
```

So:

```text
LB IPAM
    = "Which IP does the Service get?"

L2 Announcement
    = "How do devices on the network reach that IP?"
```

Together, they provide LoadBalancer IP allocation and local Layer 2 reachability in the private lab environment.

### L2 lease configuration

```yaml
leaseDuration: 15s
leaseRenewDeadline: 5s
leaseRetryPeriod: 2s
```

These settings control the lease used to determine which Cilium instance is responsible for advertising a given LoadBalancer IP.

---

# Hubble

Hubble is Cilium's network observability layer.

It provides visibility into network flows and service communication handled by Cilium.

## Enable Hubble

```yaml
hubble:
  enabled: true
```

Enables Hubble.

## Hubble Relay

```yaml
relay:
  enabled: true
```

Enables Hubble Relay for cluster-wide Hubble access.

## Hubble UI

```yaml
ui:
  enabled: true
```

Enables the Hubble web interface.

## Hubble UI base path

```yaml
baseUrl: "/kube/hubble/"
```

The UI is exposed under:

```text
/kube/hubble/
```

This allows it to be published through the existing Gateway API setup.
