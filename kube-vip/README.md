# kube-vip installation

official docs: **[kube-vip - DaemonSet](https://kube-vip.io/docs/installation/daemonset/)**

## why kube-vip?

We use kube-vip to provide a **Virtual IP (VIP)** for the Kubernetes control plane.

Our three control-plane nodes use:

```text
rke2-1 → 172.16.0.4
rke2-2 → 172.16.0.35
rke2-3 → 172.16.0.36
```

and kube-vip provides a shared control-plane VIP:

```text
172.16.4.100
```

The VIP can move between the control-plane nodes through leader election, so clients can always use the same address to access the Kubernetes API.

---

## RBAC

Since kube-vip is running as a DaemonSet and needs access to Kubernetes resources, we first apply the required RBAC resources:

```bash
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

---

## Set configuration details

Before generating the DaemonSet manifest, we assign the required configuration values as environment variables.

```bash
export VIP=172.16.4.100
export INTERFACE=eth1
export KVVERSION=v1.2.3
```

* `VIP`: the Virtual IP that will be used by the Kubernetes control plane.

* `INTERFACE`: the network interface used by kube-vip to advertise the VIP. In our setup, `eth1` is the Control Plane network.

* `KVVERSION`: the kube-vip image version used to generate the manifest.

Since we are using RKE2 with containerd, we also define the RKE2 `ctr` binary and the containerd socket:

```bash
export CTR="/var/lib/rancher/rke2/bin/ctr"
export CONTAINERD_SOCK="/run/k3s/containerd/containerd.sock"
```

---

## Generate the DaemonSet manifest

Pull the kube-vip image into the `k8s.io` containerd namespace:

```bash
$CTR --address $CONTAINERD_SOCK \
  --namespace k8s.io \
  image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
```

Then create the `kube-vip` command using the pulled image:

```bash
alias kube-vip="$CTR --address $CONTAINERD_SOCK \
  --namespace k8s.io \
  run --rm --net-host \
  ghcr.io/kube-vip/kube-vip:$KVVERSION \
  vip /kube-vip"
```

Generate the DaemonSet manifest:

```bash
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection
```

The generated manifest is then placed in:

```text
/var/lib/rancher/rke2/server/manifests/kube-vip.yaml
```

so RKE2 can automatically apply it to the cluster.
