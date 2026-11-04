# 01. Kubernetes enduku, architecture

[Index](README.md) | Next: [02. Manifest rayadam](02-manifest-rayadam-ela.md)

## Problem enti

Docker tho oka container run cheyyadam easy. Kani production lo:

- 50 containers, 10 servers meeda run avvali. Edi ekkada run avvalo evaru decide chestharu?
- Oka server crash aithe aa containers ni vere server meeda evaru start chestharu?
- Traffic perigithe copies penchadam, thaggithe thaggincham evaru chestharu?
- Kotha version deploy chesetappudu downtime lekunda ela?
- Containers IPs marutu untayi, okadanni okati ela kanukkuntayi?

Ivanni manually cheyyadam impossible. **Kubernetes ee pani antha automate chese container orchestrator.**

## Core idea: desired state

Kubernetes ki "ila cheyyi" ani steps cheppavu. "Naaku ila undali" ani **final state** cheptavu.

```
Nuvvu:       "web app ki 3 copies eppudu run avvali"      (desired state)
Kubernetes:  ippudu 2 matrame unnayi -> inkokati create    (actual state ni fix)
```

Idi **declarative** model. Imperative (step-by-step commands) kaadu.
Ee concept okkati ardham aithe Kubernetes lo 50% ardham ayinatte.

## Reconcile loop (most important concept)

Kubernetes lo prathi controller ee loop lo ne run avthundi:

```
while true:
    desired = etcd lo unna spec chadavu
    actual  = cluster lo nijanga em undo chudu
    if desired != actual:
        difference fix cheyyi
```

- Pod delete chesava? ReplicaSet controller chusi kotha pod create chesthundi.
- Node chanipoyinda? Node controller daanni NotReady ani mark chesthundi. Default ga ~5 min (toleration 300s) tarvata aa pods evict avthayi, Deployment/ReplicaSet lanti controller unte vere nodes meeda kotha pods create avthayi. Standalone pod (controller lenidi) malli radhu.
- Replicas 3 nundi 5 ki marchava? Controller 2 extra pods create chesthundi.

Advanced topics (operators, GitOps, Karpenter) anni ide loop meeda build ayyayi. Gurthu pettuko.

## Architecture

```
                    +-------------------- Control Plane --------------------+
 kubectl  ------->  |  kube-apiserver  <---->  etcd                          |
                    |       ^                                                |
                    |       |---- kube-scheduler                             |
                    |       |---- kube-controller-manager                    |
                    |       |---- cloud-controller-manager                   |
                    +-------|------------------------------------------------+
                            |
          +-----------------+------------------+
          |                                    |
   +------v------ Worker Node ----+    +------v------ Worker Node ----+
   | kubelet                      |    | kubelet                      |
   | container runtime(containerd)|    | container runtime            |
   | kube-proxy                   |    | kube-proxy                   |
   | [pod] [pod] [pod]            |    | [pod] [pod]                  |
   +------------------------------+    +------------------------------+
```

### Control plane components

**kube-apiserver**
- Cluster ki front door. `kubectl`, controllers, kubelet anni ide API tho matladatayi.
- Authentication, authorization (RBAC), admission checks ikkade jarugutayi.
- Direct ga etcd ni touch chesedi apiserver okkate.

**etcd**
- Distributed key-value database. Cluster motham state (anni objects) ikkade store avthundi.
- etcd poyithe cluster memory poyinattu. Anduke backups important. EKS lo AWS manage chesthundi.

**kube-scheduler**
- Kotha pod create ayyaka, "ee pod e node meeda run avvali" ani decide chesthundi.
- Node lo enough CPU/memory unda, taints, affinity rules, anni check chesi best node select chesthundi.
- Scheduler pod ni run cheyyadu. Pod object lo `nodeName` set chesthundi anthe.

**kube-controller-manager**
- Chala controllers okka process lo: Deployment controller, ReplicaSet controller, Node controller, Job controller, etc.
- Prathi okati reconcile loop run chesthundi.

**cloud-controller-manager**
- Cloud provider tho integration. `Service type: LoadBalancer` create chesthe AWS lo load balancer create cheyyadam lanti pani.

### Worker node components

**kubelet**
- Prathi node meeda agent. "Naa node ki assign ayina pods ni run cheyyi" ani chusukuntundi.
- Container runtime ki "ee container start cheyyi" ani cheptundi, probes run chesthundi, status ni apiserver ki report chesthundi.

**Container runtime**
- Actual ga containers run chesedi. Ippudu mostly `containerd`. (Docker runtime support Kubernetes 1.24 lo remove ayyindi, kani Docker build chesina images ekkadaina run avthayi.)

**kube-proxy**
- Service IPs ki vachhe traffic ni correct pod ki pampadaniki iptables/IPVS rules set chesthundi.
- Konni CNIs (Cilium lanti) kube-proxy lekunda kuda ee pani chesthayi.

## `kubectl apply` chesinappudu em jarugutundi (end to end)

Deployment apply chesav anuko:

1. `kubectl` YAML ni apiserver ki pampisthundi.
2. apiserver authenticate, authorize (RBAC), admission checks chesi object ni etcd lo save chesthundi.
3. Deployment controller kotha Deployment chusi oka ReplicaSet create chesthundi.
4. ReplicaSet controller "3 pods kavali, 0 unnayi" ani chusi 3 Pod objects create chesthundi (inka e node ledu, `Pending`).
5. Scheduler aa pods ki nodes select chesi `nodeName` set chesthundi.
6. Aa node meeda kubelet "naaku kotha pod vachindi" ani chusi containerd tho image pull chesi container start chesthundi.
7. kubelet status (`Running`, IP) ni apiserver ki report chesthundi.
8. Service unte, endpoints controller kotha pod IP ni Service endpoints lo add chesthundi.

Gamaninchu: **evaru evariki direct ga order ivvaru**. Andaru apiserver ni watch chesi tama pani chesukuntaru. Idi Kubernetes design lo key point.

## EKS lo evaru em manage chestharu

| Component | Self-managed cluster | EKS (standard) | EKS Auto Mode |
|---|---|---|---|
| Control plane (apiserver, etcd, scheduler) | Nuvvu | AWS | AWS |
| Worker nodes (EC2) | Nuvvu | Nuvvu (managed node groups help chesthayi) | AWS (Karpenter-based, automatic) |
| VPC CNI, kube-proxy, CoreDNS add-ons | Nuvvu | Nuvvu (add-ons) | AWS |
| Load balancer, EBS CSI controllers | Nuvvu | Nuvvu install cheyyali | AWS built-in |
| Nee apps, manifests | Nuvvu | Nuvvu | Nuvvu |

Auto Mode lo nuvvu app manifests meeda focus chesthe chalu. Details [22-eks-auto-mode.md](22-eks-auto-mode.md) lo.

## Practice

```bash
kubectl cluster-info                 # apiserver endpoint
kubectl get nodes -o wide            # worker nodes
kubectl get pods -n kube-system      # system components (EKS Auto Mode lo chala hidden ga untayi)
kubectl api-resources | head -30     # cluster lo unna resource types
kubectl get events -A --sort-by=.lastTimestamp | tail -20   # cluster lo recent ga em jarigindi
```

## Interview lo adigite (one-liners)

- Kubernetes ante: declarative container orchestrator, desired state ni reconcile loop tho maintain chesthundi.
- etcd enduku: cluster state store, single source of truth.
- Scheduler pod run chesthunda: ledu, node select chesthundi anthe. kubelet run chesthundi.
- Controller ante: oka resource type ni watch chesi actual state ni desired state ki theesukelle loop.
