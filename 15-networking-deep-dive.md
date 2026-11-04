# 15. Networking deep dive: pod networking, kube-proxy, VPC CNI, NetworkPolicy

[Index](README.md) | Prev: [14](14-autoscaling-hpa-vpa-karpenter.md) | Next: [16. Security](16-security.md)

Prerequisites: [03 pods](03-pods.md), [07 services](07-services-dns.md)

Advanced lo chala mandi ikkade stuck avtharu. Layer by layer chuddam.

## Kubernetes networking model (3 rules)

1. Prathi pod ki **own unique IP**.
2. Anni pods anni pods tho **NAT lekunda** direct ga matladagalavu (any node).
3. Node meeda agents (kubelet) aa node pods tho matladagalavu.

Ee rules ni implement chesedi **CNI plugin** (Container Network Interface). Kubernetes tanu network create cheyyadu.

## 4 types of communication

```
1. Container <-> container (same pod)   : localhost (same network namespace)
2. Pod <-> pod                          : pod IP direct (CNI)
3. Pod <-> Service                      : ClusterIP -> kube-proxy rules -> pod IP
4. External <-> Service                 : LoadBalancer/Ingress -> node/pod
```

## 1. Same pod lo

Pod create ayyinappudu oka "pause" container network namespace hold chesthundi. Pod lo anni containers aa namespace join avthayi. Same IP, `localhost`, ports conflict avvakudadu.

## 2. Pod to pod: CNI

### Overlay CNIs (Flannel, Calico VXLAN)

Pods ki separate CIDR (e.g. 10.244.0.0/16). Node to node packets encapsulate (VXLAN) chesi pamputhayi. VPC ki pod IPs teliyavu.

### Amazon VPC CNI (EKS default, Auto Mode lo built-in)

- Pods ki **VPC subnet nundi real IPs**. Overlay ledu. Pod IP VPC lo first-class: security groups, VPC flow logs, RDS anni pod IP chusthayi.
- Node ENIs ki secondary IPs / prefixes attach chesi pods ki isthundi.

```
Node (EC2)
 ├── ENI-0 (primary)   IP 172.31.2.34
 │     └── secondary IPs / /28 prefixes -> pods
 └── ENI-1             -> more pods
```

**Max pods per node** limit ENIs x IPs per ENI batti untundi. Prefix delegation (/28 blocks) tho chala ekkuva pods. Auto Mode idi manage chesthundi.

**Subnet IP exhaustion**: pods VPC IPs thintayi. Chinna subnets (/24) lo pods `Pending` / `failed to assign an IP address`. Pedda subnets leda secondary CIDR (100.64.0.0/10) custom networking plan cheyyi.

```bash
kubectl get pods -o wide            # pod IPs VPC CIDR lo ne untayi chudu
```

## 3. Pod to Service: kube-proxy

Service ClusterIP **virtual**. E interface ki assign avvadu. Ping cheyyalemu (ICMP pani cheyyadu, normal).

kube-proxy prathi node meeda Services + EndpointSlices watch chesi rules rasthundi:

- **iptables mode** (common): `KUBE-SERVICES` chain lo ClusterIP:port match -> random ga oka pod IP ki DNAT.
- **IPVS mode**: kernel load balancer, pedda clusters lo fast.
- **nftables mode**: kotha Kubernetes versions lo.
- **eBPF (Cilium)**: kube-proxy replace.

```
pod A sends to 10.100.12.34:80
  -> node iptables: DNAT to 172.31.5.20:8080 (pod B)
  -> VPC CNI routing -> pod B
```

Connection-level balancing (prathi request kaadu). HTTP/2, gRPC long-lived connections okka pod ki stick avvochu. Daaniki headless Service + client-side LB leda service mesh.

## 4. External traffic (EKS)

### NLB (Service type LoadBalancer)

- **instance target**: NLB -> node NodePort -> kube-proxy -> pod (extra hop, vere node ki vellochu).
- **ip target**: NLB -> pod IP direct (VPC CNI valla possible). Better latency, source IP preserve easy.

### ALB (Ingress)

- ALB -> pod IPs (`target-type: ip`) leda NodePort. Auto Mode ALB default ga ip targets.

```
Internet -> ALB (public subnets) -> pod IPs (private subnets)
```

## DNS flow

```
pod -> /etc/resolv.conf nameserver (kube-dns Service IP) -> CoreDNS pods
   -> cluster name (*.svc.cluster.local) aithe answer
   -> external name aithe upstream (VPC Route 53 Resolver)
```

## NetworkPolicy (pod firewall)

**Default: anni pods anni pods tho matladagalavu.** Security ki NetworkPolicy tho restrict cheyyali.

Key rules:
- Policy oka pod ni select chesina ventane, aa direction (ingress/egress) lo **explicitly allow chesinavi tappa anni deny**.
- Policies **additive** (allow union). Deny rule ledu.
- CNI enforce cheyyali. Enforce cheyyani CNI lo policy create chesina emi jaragadu, error kuda radhu.

EKS Auto Mode lo enable cheyyadaniki `kube-system` lo ConfigMap kavali ([AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/auto-net-pol.html)):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: amazon-vpc-cni
  namespace: kube-system
data:
  enable-network-policy-controller: "true"
```

### Pattern 1: default deny ingress (namespace lo)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: shop
spec:
  podSelector: {}          # namespace lo anni pods
  policyTypes: [Ingress]   # ingress rules emi levu -> anni ingress deny
```

### Pattern 2: specific allow

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-web
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: api                     # ee policy api pods ki
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web                 # same namespace web pods nundi
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring   # monitoring namespace anni pods nundi
    ports:
    - protocol: TCP
      port: 8080
```

### `from` lo AND vs OR (most confusing part)

```yaml
# OR: web pods (same ns) LEDA monitoring namespace lo anni pods
- from:
  - podSelector: { matchLabels: { app: web } }
  - namespaceSelector: { matchLabels: { team: monitoring } }

# AND: monitoring namespace lo unna prometheus pods matrame
- from:
  - namespaceSelector: { matchLabels: { team: monitoring } }
    podSelector: { matchLabels: { app: prometheus } }
```

Okka `-` item lo rendu selectors = AND. Separate `-` items = OR. Okka dash difference tho security maruthundi.

### Pattern 3: egress restrict (DNS marchipovaddu)

```yaml
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Egress]
  egress:
  - to:                              # DNS allow, lekapothe name resolve avvadu
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - { protocol: UDP, port: 53 }
    - { protocol: TCP, port: 53 }
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - { protocol: TCP, port: 5432 }
```

Egress deny chesi DNS allow cheyyakapothe "app connect avvatledu" ani gantalu debug chestharu. Classic mistake.

### Testing

```bash
kubectl run t --rm -it --image=busybox:1.36 --restart=Never -l app=web -n shop -- wget -qO- -T 3 http://api:8080
kubectl run t --rm -it --image=busybox:1.36 --restart=Never -l app=other -n shop -- wget -qO- -T 3 http://api:8080   # timeout avvali
kubectl get networkpolicy -n shop
kubectl describe networkpolicy api-allow-from-web -n shop
```

## Security groups for pods (EKS)

VPC CNI tho pods ki AWS security groups attach cheyyochu (`SecurityGroupPolicy` CRD). RDS SG lo "ee pods SG nundi matrame 5432" lanti AWS-level rules.

## Debug toolkit

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot --restart=Never -- bash
#   lopala: dig web.shop.svc.cluster.local, curl -v, nc -zv host port, traceroute, tcpdump
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>   # pod network namespace lo
```

Checklist: DNS resolve avthunda -> Service endpoints unnaya -> pod direct IP ki reach avthunda -> NetworkPolicy -> SG/NACL -> app listen address.

## Interview one-liners

- Kubernetes networking model: unique pod IP, NAT-less pod-to-pod.
- VPC CNI special enti: pods ki real VPC IPs, overlay ledu.
- ClusterIP ping enduku avvadu: virtual IP, iptables DNAT TCP/UDP ports ki matrame.
- NetworkPolicy default: policy select chesthe aa direction lo default deny.
- NetworkPolicy pani cheyyatledu: CNI enforce chesthunda? (Auto Mode lo controller enable chesava?)

Lab: [labs/README.md#lab-13](labs/README.md#lab-13-networkpolicy)
