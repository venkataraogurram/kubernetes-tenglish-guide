# 22. EKS specifics, Auto Mode

[Index](README.md) | Prev: [21](21-service-mesh-istio.md) | Next: [23. Troubleshooting](23-troubleshooting-playbook.md)

Ee guide lo labs anni EKS Auto Mode cluster meeda test chesinavi. Ikkada EKS lo "Kubernetes + AWS" kalise points.

## EKS modes

| | Standard EKS | EKS Auto Mode | Fargate |
|---|---|---|---|
| Nodes | Managed node groups / self-managed (nuvvu) | AWS (Karpenter-based) | Node ledu, pod = micro VM |
| Node OS patching | Nuvvu (AMI updates) | AWS (Bottlerocket, auto replace) | AWS |
| VPC CNI, CoreDNS, kube-proxy | Add-ons nuvvu manage | Built-in | Built-in |
| Load balancer controller | Install (Helm) | Built-in (`eks.amazonaws.com/alb`, `nlb`) | Install |
| EBS CSI | Add-on install | Built-in (`ebs.csi.eks.amazonaws.com`) | EBS ledu |
| DaemonSets | Yes | Yes | No |

## Auto Mode lo em built-in, em nuvvu cheyyali

Mana cluster lo check chesinavi (`kubectl get nodepools`, `nodeclasses`):

```
nodepool.karpenter.sh/general-purpose   (on-demand, c/m/r gen>4, amd64, no taint)
nodepool.karpenter.sh/system            (CriticalAddonsOnly taint, amd64+arm64)
nodeclass.eks.amazonaws.com/default
```

- Nee apps `general-purpose` NodePool meeda run avthayi. Pods Pending aithe Auto Mode kotha node launch chesthundi (1 min daggara). `system` nodes meeda taint valla mee pods vellavu.
- `general-purpose` amd64 matrame. arm64 kavali ante own NodePool create cheyyi ([14](14-autoscaling-hpa-vpa-karpenter.md)). Images multi-arch aithe problem ledu.
- Nodes Bottlerocket. SSH ledu. Node debug ki `kubectl debug node/<name>` leda `NodeDiagnostic`.
- Nodes `expireAfter: 336h` (14 days) tarvata replace. PDBs + 2+ replicas pettu.

### Nuvve create cheyyalsinavi

| Need | Create |
|---|---|
| EBS PVCs | StorageClass `provisioner: ebs.csi.eks.amazonaws.com` (Auto Mode create cheyyadu; old `gp2` in-tree class Auto Mode storage vadadu) |
| ALB Ingress | `IngressClassParams` + `IngressClass` (`controller: eks.amazonaws.com/alb`) |
| NetworkPolicy | `kube-system/amazon-vpc-cni` ConfigMap `enable-network-policy-controller: "true"` |
| Custom nodes (spot, arm64, GPU) | Own NodePool (+ NodeClass if subnets/SG veru) |

NLB (Service type LoadBalancer) ki emi avasaram ledu: `loadBalancerClass: eks.amazonaws.com/nlb` cluster default.

## IAM <-> Kubernetes

Rendu directions ([12](12-namespaces-rbac-serviceaccounts.md)):

```
Human/CI (IAM role) --Access Entry--> Kubernetes groups/policies --> RBAC
Pod (ServiceAccount) --Pod Identity / IRSA--> IAM role --> AWS APIs (S3, DynamoDB)
```

```bash
aws eks list-access-entries --cluster-name <cluster>
aws eks list-pod-identity-associations --cluster-name <cluster>
kubectl auth whoami
```

## kubeconfig

```bash
aws eks update-kubeconfig --name <cluster> --region us-east-1
kubectl config get-contexts
kubectl config use-context <context>
```

kubeconfig lo token ledu: `aws eks get-token` prathi sari run avthundi (exec plugin). AWS credentials expire aithe `kubectl` kuda fail (`the server has asked for the client to provide credentials`).

## Networking (EKS)

- Pods ki VPC IPs (VPC CNI). Subnet size plan cheyyi ([15](15-networking-deep-dive.md)).
- Subnet tags: public `kubernetes.io/role/elb=1`, private `kubernetes.io/role/internal-elb=1`. LB ekkada create avvalo ivi decide chesthayi.
- Cluster endpoint: public / private / both. Prod lo private leda CIDR restricted.
- Cross-AZ traffic charge: `trafficDistribution: PreferClose` / topology-aware routing.

## Cost tips

- Requests right-size cheyyi (Karpenter requests batti node size chusthundi). Over-requested pods = waste.
- Spot NodePool (stateless workloads).
- Prathi LoadBalancer Service = oka NLB. HTTP services ki shared ALB Ingress (IngressGroup).
- Lab ayyaka LBs, PVCs (EBS) delete cheyyi. Namespace delete chesthe PVC/Service delete avthayi, AWS resources controller clean chesthundi. Konni minutes tarvata console lo confirm cheyyi.

## Upgrades

- Control plane: `aws eks update-cluster-version` (okka minor version at a time).
- Auto Mode lo nodes automatic ga kotha version ki replace.
- Mundu deprecated APIs check: `kubectl` warnings, EKS upgrade insights (`aws eks list-insights`).
- Standard support ~14 months, extended support extra cost.

## Useful commands

```bash
aws eks describe-cluster --name <cluster> --query 'cluster.{v:version,status:status,auto:computeConfig.enabled}'
kubectl get nodes -L eks.amazonaws.com/compute-type,node.kubernetes.io/instance-type,topology.kubernetes.io/zone
kubectl get nodeclaims
kubectl get nodepools
kubectl describe nodeclaim <name>          # e instance enduku launch ayyindi
kubectl get events -A --field-selector reason=FailedScheduling
```

## Interview one-liners

- EKS Auto Mode em manage chesthundi: nodes (Karpenter), OS, core add-ons, LB, EBS CSI, Pod Identity agent.
- IRSA vs Pod Identity: rendu pod IAM; Pod Identity simpler (no OIDC trust per cluster, no SA annotation).
- EKS lo pod IPs: VPC IPs, subnet exhaustion risk.
- Access entries vs aws-auth: access entries API-based, aws-auth ConfigMap legacy.

Lab: [labs/README.md](labs/README.md) (anni labs Auto Mode meeda)
