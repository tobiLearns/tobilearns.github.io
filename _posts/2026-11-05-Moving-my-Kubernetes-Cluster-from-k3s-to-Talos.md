---
title: Moving my Kubernetes Cluster from k3s to Talos
date: 2026-11-05
categories: [Homelab, Kubernetes]
tags: [talos, kubernetes]
---
When I moved my homelab Kubernetes cluster from k3s to Talos Linux, I had to set up some things manually on Talos, that k3s already includes:

### Storage
- Applications that keep data (settings, uploads, databases) need storage that survives Pod restarts and rescheduling. Kubernetes provides this through *PersistentVolumes* (PV), which live outside the container’s file system, for example on the node’s disk or on an external disk. An application requests a PV with a *PersistentVolumeClaim* (PVC).
- k3s ships Rancher’s local-path-provisioner and defines a default StorageClass named `local-path`. For each PVC, the provisioner creates a directory on the node’s disk and a matching PV.
- Talos doesn't come with a built-in storage provider, so I had to deploy one myself. To let my workloads still use the *local-path* storage, I installed Rancher's local-path-provisioner.

### Ingress
- By default, a Service is reachable only from inside the cluster. An Ingress is a set of HTTP/HTTPS rules that map hostnames and paths to Services (for example `app-a.example.com` -> application A), so many web applications can share one entry point.
  The Ingress resource only contains the rules. An ingress controller, running as a Pod in the cluster, applies them. The controller itself must be made reachable from outside, for example through a LoadBalancer Service (see below).
- k3s automatically deploys *Traefik* as its default ingress controller and registers it as an IngressClass named `traefik`. Ingress resources that reference this class are picked up and routed by Traefik.
- Talos comes without a default ingress controller, so you have to install one on your own.
  To avoid further changes, I installed Traefik here as well. Other popular Ingress controllers are, for example, Cilium or Istio. And since the Ingress API is frozen, Kubernetes recommends to use *Gateway API*.

### LoadBalancer
- A Service of type `LoadBalancer` gives the Service an IP address that is reachable from outside the cluster. It works at the TCP/UDP level, so it is independent of the protocol, but it provides no hostname- or path-based routing. Cloud providers supply this automatically; on bare metal something else has to.
- k3s ships ServiceLB (klipper-lb). For each Service of type `LoadBalancer`, it starts a small Pod on the node that forwards traffic to the Service using the node’s own IP address.
- Talos ships no equivalent, so a LoadBalancer Service would stay in `<pending>`. I installed MetalLB in Layer 2 mode, which assigns addresses from a range I reserved in the home network.
  In my setup, the LoadBalancer exposes Traefik, and Traefik routes by hostname. A request travels: client -> LoadBalancer IP -> Traefik -> Service -> Pod.

### Pod security
- _Pod Security Standards_ (PSS) define what a Pod may request that lets it reach beyond its container sandbox and touch the underlying node. This is enforced by the _Pod Security Admission_ (PSA) controller. There are three levels: `privileged`, `baseline` and `restricted`.
- On k3s no PSS are enforced by default, so pods can request privileges such as host networking or host paths.
- By default, Talos enforces the PSS profile `baseline` cluster-wide ([docs.siderolabs.com/kubernetes-guides/security/pod-security](https://docs.siderolabs.com/kubernetes-guides/security/pod-security)). But some Pods legitimately need to access the underlying node, e.g., for the local-path-storage or the node-exporter in the monitoring stack. So I had to define some exemptions (which can be made only by namespace).
- This is actually not a very good solution and has to be improved in a next step. The tool *Kyverno* provides the possibility to define more fine grained security policies.

---

## The technical changes
### Overview

| Component    | k3s                            | Talos setup                  | Where in the repo                                                                  |
| ------------ | ------------------------------ |------------------------------| ---------------------------------------------------------------------------------- |
| Storage      | local-path-provisioner bundled | own deployment, changed path | `infrastructure/controllers/base/local-path-provisioner/`                          |
| Ingress      | Traefik bundled                | Traefik HelmRelease          | `infrastructure/controllers/base/traefik/`                                         |
| LoadBalancer | ServiceLB (klipper)            | MetalLB HelmRelease, L2 mode | `infrastructure/controllers/base/metallb/`, `infrastructure/configs/base/metallb/` |
| Pod security | permissive                     | PSA labels per namespace     | namespace manifests                                                                |

### 1. Storage: local-path-provisioner
I deployed Rancher's local-path-provisioner (controller, RBAC, ConfigMap) and defined the `local-path` StorageClass as the cluster default (the same as used in the k3s cluster before):

```yaml
# infrastructure/controllers/base/local-path-provisioner/storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

I had to adapt the storage path. By default, Rancher's local-path-provisioner is configured to create its Persistent Volumes under `/opt`. 
The Talos documentation recommends using `/var/mnt` for the PersistentVolumes. 
This directory is documented as the persistent EPHEMERAL partition on Talos, 
so volumes survive reboots and upgrades of the node (it is wiped only on a reset):

```yaml
# infrastructure/controllers/base/local-path-provisioner/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap":[
      {
        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
        "paths":["/var/mnt/local-path-provisioner"]
      }
      ]
    }
```

To make this directory visible to the kubelet, a kubelet extraMounts entry in the Talos machine config is required. 
I applied this with a patch:
````yaml
machine:
    kubelet:
      # Required for local-path-provisioner's hostPath volumes to write under
      # /var/mnt — kubelet's own container doesn't see that path by default.
      extraMounts:
        - destination: /var/mnt
          type: bind
          source: /var/mnt
          options:
            - bind
            - rshared
            - rw
````
The Talos documentation for local-path-provisioner uses instead a UserVolumeConfig (see [docs.siderolabs.com/kubernetes-guides/csi/local-storage](https://docs.siderolabs.com/kubernetes-guides/csi/local-storage)).

(see pull request [Adaptations for moving from k3s to talos](https://github.com/tobiLearns/kubernetes-homelab/pull/35/changes))

### 2. Ingress: Traefik
In my old k3s cluster I had several Ingress resources defined, which all used k3s default IngressClass ``traefik``. 
To keep these existing resources working on the new Talos node without any further changes, 
I installed Traefik from its official Helm chart and registered it as the default `IngressClass`.  
I did this in two steps. At first there was no LoadBalancer, so Traefik was exposed through `hostPort` (binding ports 80/443 directly on the node). That needed a privileged PSA label (see section 4).

```yaml
# infrastructure/controllers/base/traefik/release.yaml (excerpt)
values:
    ports:
      web:
        hostPort: 80
      websecure:
        hostPort: 443
    service:
      spec:
        type: ClusterIP
    ingressClass:
      enabled: true
      isDefaultClass: true
      name: traefik
```

Once MetalLB was running, I removed the `hostPort` bindings and switched the Service to `type: LoadBalancer` with a fixed IP:
```yaml
# infrastructure/controllers/base/traefik/release.yaml (excerpt)
values:
  service:
    annotations:
      metallb.io/loadBalancerIPs: <my-LoadBalancer-IP>
    spec:
      type: LoadBalancer
  ingressClass:
    enabled: true
    isDefaultClass: true
    name: traefik
```
Also the ``privileged`` label on the ``traefik`` Namespace was no longer needed, instead one component of MetalLB now needed privileged rights (see section 4).


(see pull requests [Adaptations for moving from k3s to Talos](https://github.com/tobiLearns/kubernetes-homelab/pull/35/changes) 
and [Switch Traefik Service to type LoadBalancer](https://github.com/tobiLearns/kubernetes-homelab/pull/43/changes))

### 3. LoadBalancer: MetalLB
MetalLB assigns IP addresses from a pool to Services of type `LoadBalancer` and announces them to the network, so other devices know where to send the traffic. It can do this in two modes.

#### Layer 2 mode
In Layer 2 mode, MetalLB uses the *Address Resolution Protocol* (ARP). ARP defines how devices on a local network map IP addresses to hardware (MAC) addresses. A device that wants to reach IP x.x.x.x broadcasts a “who has this address?” query, and the device that owns it replies. MetalLB lets one node give this reply for each assigned IP, so all other devices send that IP’s traffic to this node. If the node fails, another node takes over the IP.
- It needs no special router configuration. The assigned IPs must come from the same subnet as the nodes, and clients in other VLANs reach them through the router like any other device in that subnet.
- It does not distribute load across nodes: all traffic for one IP enters the cluster through a single node, which therefore limits the bandwidth. Inside the cluster, Kubernetes still spreads the traffic across the Pods (replicas) behind a Service, even if they run on other nodes.

#### BGP mode
In BGP mode, each node opens a session with a router and announces the assigned IPs using the *Border Gateway Protocol* (BGP), the routing protocol that routers use to tell each other which IP ranges they can reach. The router can then spread the traffic for one IP across several nodes.
- It needs a router that supports BGP (some home routers and firewalls do, many don’t) and additional configuration for the peers and announcements.
- FRR-K8s is the default backend MetalLB uses for BGP.

#### Decision to use Layer 2 mode
My cluster has one node, so there is only one possible entry point and load distribution across nodes is irrelevant. The configuration for L2 mode is minimal and doesn't require any additional config on the router.

For a multi-node cluster, Layer 2 mode keeps working and gives failover. BGP becomes interesting when traffic for a single IP should be spread across nodes, or when one node’s bandwidth is no longer enough.
A switch to BGP mode is a configuration change: the IP pool and the Services stay as they are, and the `L2Advertisement` is replaced by BGP resources (`BGPPeer`, `BGPAdvertisement`).

#### Setup
MetalLB is installed through a `HelmRelease` with two non-default settings:

```yaml
# infrastructure/controllers/base/metallb/release.yaml (excerpt)
values:
  frrk8s:
    enabled: false      # BGP backend, not needed in L2 mode
  speaker:
    ignoreExcludeLB: true
```

- **`frrk8s.enabled: false`**: disables the BGP backend FRR-K8s, which is not needed in Layer 2 mode.
- **`speaker.ignoreExcludeLB: true`**: Kubernetes labels control-plane nodes with `node.kubernetes.io/exclude-from-external-load-balancers`. In a multi-node cluster this keeps external traffic away from the control plane. My cluster has one node that is both control plane and worker, so the label has to be ignored. Otherwise no node would be eligible to announce IPs.

The address pool and the L2 advertisement are separate resources:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
    - <IP-addr-range-begin>-<IP-addr-range-end>
  autoAssign: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - homelab-pool
```

With `autoAssign: false`, no Service gets an IP implicitly. A Service has to request one with the `metallb.io/loadBalancerIPs` annotation, as Traefik does with `metallb.io/loadBalancerIPs: <my-LoadBalancer-IP>`.

**Ordering with Flux.** These two resources are custom resources, and their definitions (CRDs) only exist after the MetalLB chart is installed. Applying them in the same step would fail. So the infrastructure is split into two Flux Kustomizations where the ``infrastructure-configs`` (which hold the two custom resources) wait for the ``infrastructure-controllers`` to be ready:

```yaml
# clusters/staging/infrastructure.yaml (excerpt)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-controllers
  namespace: flux-system
spec:
  [...]
  # Wait until all resources, defined under '/infrastructure/controllers/', 
  # are really running, before marking this Kustomization as 'Ready':
  wait: true
  [...]

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-configs
  namespace: flux-system
spec:
  [...]
  # Check, if the referenced Kustomization is in state 'Ready':
  dependsOn:
    - name: infrastructure-controllers
  [...]
```

(see pull request [Add LoadBalancer Metallb](https://github.com/tobiLearns/kubernetes-homelab/pull/42/changes))

### 4. Pod security: PSA labels
On Talos PSA enforces the security level ``baseline`` by default. 
PSA is configured per namespace via labels and has no per-workload exceptions. 
To allow one Pod more privileges, the security level for its whole namespace has to be relaxed.  
Three resources in my cluster needed this privileged access to the node that the ``baseline`` profile forbids:
- **local-path-provisioner:** The helper pods that create and delete the directories for PersistenVolumes on the node's filesystem use the `hostPath`.
- **node-exporter** (part of the monitoring stack) needs `hostNetwork`, `hostPID`, `hostPath` and `hostPort` to read host-level metrics.
- **MetalLB speaker:** In Layer 2 mode, it needs `hostNetwork` and the `NET_RAW` capability to send ARP replies on the node’s network interface.

The namespaces `local-path-storage`, `monitoring` and `metallb-system` therefore carry this label:
```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

The exceptions differ in how much they open up. `local-path-storage` and `metallb-system` are dedicated namespaces containing only the component that needs the privilege, so the exception is as narrow as PSA allows.
The `monitoring` namespace is the problem case: The privileged label also relaxes the security level for Grafana, Prometheus and Alertmanager, which don’t need it. Moving node-exporter into a separate namespace would fix that.

Traefik needed the privileged label only temporarily. Before MetalLB was deployed, I exposed it with `hostPort`, which required the `traefik` namespace to be privileged. Once MetalLB ran, I switched Traefik’s Service to `type: LoadBalancer` and removed both the `hostPort` bindings and the label. The privilege effectively moved from Traefik to the MetalLB speaker, in a namespace of its own.
The `local-path-storage` label can go if the provisioner is ever replaced by a real CSI storage driver.

With a policy engine like *Kyverno* the security standards could be "re-applied" to the privileged namespaces, except some fine-grained exceptions for exactly these workloads, which need them.

