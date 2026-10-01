# configuration files

official docs: **[server conf](https://docs.rke2.io/reference/server_config)**

eng. Ahmed Maher report: **[medium](https://medium.com/@ahmedmaherbf/building-a-distributed-system-for-private-cloud-part-1-installing-rke2-cilium-ebpf-native-routing-a71a82a71f99/share/omar.ali.hassan02?source=social.linkedin&_nonce=MBwIR5Bm)**

## initial node configuration

we have three RKE2 nodes, and each of them will be a server node (control plane) to provide high availability.

we first have an initial node, then we join the other two nodes to the cluster as server nodes as well.

```yaml
# /etc/rancher/rke2/config.yaml

token: <Token>

node-name: rke2-1

node-ip: 172.17.0.3

advertise-address: 172.16.0.4

tls-san:
  - 172.16.4.100
  - 172.16.0.4
  - 172.16.0.35
  - 172.16.0.36
  - 172.17.0.3
  - 172.17.0.34
  - 172.17.0.35

cluster-cidr: 10.42.0.0/16

service-cidr: 10.43.0.0/16

cluster-dns: 10.43.0.10

cluster-domain: cluster.local

cni: cilium

disable-kube-proxy: true

write-kubeconfig-mode: 0644

ingress-controller: traefik

disable:
  - rke2-ingress-nginx
```

#### Cluster bootstrap & access

* `token`: a shared cluster secret used to join the other server nodes to the cluster after bootstrapping the initial node.

* `write-kubeconfig-mode: 0644` (`-rw-r--r--`): sets the permissions of the generated kubeconfig file.

#### Node identity + networking

* `node-name: <node-name>`: the Kubernetes node name.

* `node-ip: <internal-ip>` (`172.17.0.3`): the internal IP address used by the node, which is on the Pod/Fabric network (`eth2`).

#### Kubernetes API / TLS

* `advertise-address: <API-server address>` (`172.16.0.4`): the address advertised by the Kubernetes API server, which is on the Control Plane network (`eth1`).

* `tls-san: <API server certificate>`: IPs/hostnames that the Kubernetes API Server's TLS certificate is valid for.

#### Pod & Service networking

* `cluster-cidr: <network for pods>`: defines the IP address range used by Kubernetes Pods (`10.42.0.0/16`).

* `service-cidr: <network for ClusterIP services>`: defines the IP address range used by Kubernetes Services (`10.43.0.0/16`).

* `cluster-dns: <CoreDNS IP>`: the ClusterIP used by the CoreDNS service (`10.43.0.10`).

* `cluster-domain: <DNS suffix>`: the DNS suffix used inside the cluster (`cluster.local`), for example:
  `<svc-name>.<namespace>.svc.cluster.local`

#### CNI / kube-proxy

* `cni: <CNI-name>`: specifies the CNI used by the cluster, in our case Cilium.

* `disable-kube-proxy: true/false`: disables kube-proxy. In our case, it is disabled because Cilium will handle Kubernetes service networking.

#### Ingress

* `ingress-controller: traefik`: configures RKE2 to use Traefik as the ingress controller. Kubernetes Ingress support will be disabled later in the Traefik Helm values because Traefik is used through Gateway API instead.

* `disable: rke2-ingress-nginx`: disables the built-in RKE2 Ingress NGINX controller since Traefik is used instead.

## Server node bootstrap

The other nodes that will join the cluster as control-plane nodes are also called server nodes.

The main difference is the `server` parameter, which specifies the existing RKE2 server that the new node should join.

```yaml
token: "<Token>"

server: https://172.16.4.100:9345

node-name: rke2-2

node-ip: 172.17.0.34

kubelet-arg:
  - "node-ip=172.17.0.34"

advertise-address: 172.16.0.35

tls-san:
  - 172.16.4.100
  - 172.16.0.4
  - 172.16.0.35
  - 172.16.0.36
  - 172.17.0.3
  - 172.17.0.34
  - 172.17.0.35

cluster-cidr: 10.42.0.0/16

service-cidr: 10.43.0.0/16

cluster-dns: 10.43.0.10

cluster-domain: cluster.local

cni: cilium

disable-kube-proxy: true

write-kubeconfig-mode: "0644"

disable:
  - rke2-ingress-nginx
```

---

# RKE2 - installation

from the docs: **[RKE2 docs](https://docs.rke2.io/install/quickstart)**

used the following commands:

### installer

```bash
curl -sfL https://get.rke2.io | sh -
```

### systemctl

```bash
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

## setup (kubectl & k8s config file)

binary files exist in `/var/lib/rancher/rke2/bin/`

```text
containerd  containerd-shim-runc-v2  crictl  ctr  kubectl  kubelet  runc
```

so the `kubectl` command isn't accessible (`kubectl: command not found`), so we have to add the binaries directory to the `$PATH` in the `.bashrc` file:

```bash
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> $HOME/.bashrc
```

also, to have access to the Kubernetes cluster, we have to configure `kubectl` to use the correct kubeconfig path, which exists in `/etc/rancher/rke2/rke2.yaml`.

This can be done by exporting the `KUBECONFIG` environment variable or using the `--kubeconfig` command-line flag.

```bash
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >> $HOME/.bashrc
```

without this step, you will get the following issue:

```text
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

now run:

```bash
kubectl get nodes -o wide
```

to see the cluster nodes and check the `INTERNAL-IP` for each node to make sure it is using the Pod/Fabric network.

Note: all the nodes will initially be `NotReady` because the CNI has not been installed/configured yet.

```text
root@kirito:~# kubectl get no -o wide

NAME     STATUS    ROLES                AGE   VERSION          INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
rke2-1   NotReady  control-plane,etcd   22d   v1.35.5+rke2r2   172.17.0.3      <none>        Ubuntu 24.04.4 LTS   6.8.0-142-generic   containerd://2.2.3-k3s1
rke2-2   NotReady  control-plane,etcd   22d   v1.35.5+rke2r2   172.17.0.34     <none>        Ubuntu 24.04.4 LTS   6.8.0-142-generic   containerd://2.2.3-k3s1
rke2-3   NotReady  control-plane,etcd   22d   v1.35.5+rke2r2   172.17.0.35     <none>        Ubuntu 24.04.4 LTS   6.8.0-142-generic   containerd://2.2.3-k3s1
```
---

