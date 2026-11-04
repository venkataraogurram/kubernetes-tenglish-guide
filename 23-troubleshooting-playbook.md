# 23. Troubleshooting playbook

[Index](README.md) | Prev: [22](22-eks-auto-mode.md) | Next: [24. kubectl cheatsheet](24-kubectl-cheatsheet.md)

## Golden rule: 3 commands first

```bash
kubectl get pods -o wide                # STATUS, RESTARTS, NODE
kubectl describe pod <pod>              # Events section (bottom) chaduvu
kubectl logs <pod> --previous           # crash ayina container logs
```

90% problems ee moodu tho telusthayi. Guess cheyyaku, events chaduvu.

## Flow chart

```
Pod status?
├── Pending ---------------> scheduling problem (A)
├── ContainerCreating -----> volume / network / image pull slow (B)
├── ImagePullBackOff ------> image problem (C)
├── CrashLoopBackOff ------> app crash (D)
├── Running kani 0/1 Ready -> readiness probe (E)
├── OOMKilled -------------> memory limit (F)
├── Terminating stuck -----> finalizers / node (G)
└── Running, Ready, kani traffic radhu -> Service/Ingress/NetworkPolicy (H)
```

## A. Pending

```bash
kubectl describe pod <pod> | sed -n '/Events/,$p'
```

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests ki fit ayye node ledu | Requests thagginchu, Auto Mode/Karpenter NodePool limits check |
| `node(s) had untolerated taint` | Taint unna nodes matrame | Toleration add leda vere NodePool |
| `didn't match Pod's node affinity/selector` | nodeSelector/affinity labels tappu | `kubectl get nodes --show-labels` tho compare |
| `pod has unbound immediate PersistentVolumeClaims` | PVC bind avvaledu | `kubectl describe pvc`, StorageClass check |
| `volume node affinity conflict` | EBS vere AZ lo | Pod same AZ lo run avvali, topology check |
| `didn't satisfy existing pods anti-affinity` | Required anti-affinity | preferred ki marchu leda nodes penchu |
| Event emi ledu, Pending | Scheduler inka chudaledu leda ResourceQuota | `kubectl get events`, `kubectl describe quota` |

Auto Mode lo node launch avvatledu ante: `kubectl get nodeclaims`, `kubectl get events -A | grep -i nodeclaim`.

## B. ContainerCreating stuck

```bash
kubectl describe pod <pod>
```

- `FailedMount` / `FailedAttachVolume`: PVC/EBS attach problem, Multi-Attach (RWO vere node lo attached), Secret/ConfigMap ledu.
- `MountVolume.SetUp failed for volume "x": configmap "y" not found`: ConfigMap create cheyyi.
- `failed to assign an IP address`: subnet IPs ayipoyayi (VPC CNI).
- Pedda image pull avthondi: wait, `kubectl get events`.

## C. ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod> | grep -A3 -i "failed"
```

| Message | Cause |
|---|---|
| `code = NotFound` / `failed to resolve image` / `manifest unknown` | Image name/tag tappu |
| `unauthorized` / `no basic auth credentials` | Private registry, imagePullSecrets ledu / ECR permissions |
| `no matching manifest for linux/arm64` | Image arch support ledu (arm64 node meeda amd64-only image) |
| `i/o timeout` | Node internet/NAT/VPC endpoint ledu (private subnets lo ECR endpoints kavali) |
| `toomanyrequests` | Docker Hub rate limit. ECR pull-through cache / public.ecr.aws vadu |

## D. CrashLoopBackOff

Container start avthondi, exit avthondi, malli malli.

```bash
kubectl logs <pod> --previous
kubectl describe pod <pod> | grep -A10 "Last State"
```

Exit codes:

| Code | Ardham |
|---|---|
| `0` | Normal exit. Deployment lo process exit avvakudadu (daemon ga run avvali). Command tappu? |
| `1` | App error. Logs chudu (config missing, DB connect fail, exception) |
| `126` / `127` | Command not executable / not found (`command:` tappu, binary ledu) |
| `137` | SIGKILL: OOMKilled leda liveness fail tarvata kill |
| `139` | Segfault |
| `143` | SIGTERM (graceful stop) |

Logs emi levu, container ventane exit: debug ki command override:

```bash
kubectl debug <pod> -it --copy-to=debug-pod --container=<container> -- sh
# leda deployment lo temporarily: command: ["sleep","3600"], tarvata exec chesi manual ga start
```

Liveness probe valla restart avthunda? `describe` lo `Liveness probe failed` events chudu.

## E. Running, kani READY 0/1

```bash
kubectl describe pod <pod> | grep -i readiness
```

- Probe path/port tappu (app 8080 meeda, probe 80 ki).
- App slow start: `startupProbe` add, `initialDelaySeconds`.
- Readiness lo dependency (DB) down.
- Multi-container lo e container not ready: `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].ready}'`.

## F. OOMKilled

```bash
kubectl describe pod <pod> | grep -B2 -A5 OOMKilled
kubectl top pod <pod> --containers
```

Memory limit penchu, leak check, JVM/Node heap settings container limit ki align (`-XX:MaxRAMPercentage`, `--max-old-space-size`).
Node level eviction (`Evicted`, `The node was low on resource: memory`) vere: node pressure, BestEffort/Burstable pods first.

## G. Terminating stuck

```bash
kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'
kubectl get node <node>                   # NotReady aa?
```

- Node NotReady: kubelet report cheyyatledu, pod Terminating lo untundi. Node recover/replace.
- Finalizer: aa controller cleanup cheyyatledu. Controller logs chudu.
- Last resort (data/AWS resources orphan risk ardham chesukuni): `kubectl delete pod <pod> --grace-period=0 --force`.

Namespace Terminating stuck: `kubectl get ns <ns> -o yaml` lo `status.conditions` e resources/finalizers block chesthunnayo cheptundi. Usually unavailable APIService leda CRD controller ledu.

## H. Pod fine, traffic radhu

Layer by layer, inside to outside:

```bash
# 1. App pod lopala listen chesthonda
kubectl exec -it <pod> -- sh -c 'netstat -tlnp 2>/dev/null || ss -tlnp'
kubectl port-forward pod/<pod> 8080:<port>       # direct ga pani chesthe app ok

# 2. Service endpoints
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl port-forward svc/<svc> 8080:80

# 3. DNS + Service from inside cluster
kubectl run t --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- curl -sv http://<svc>.<ns>

# 4. NetworkPolicy
kubectl get networkpolicy -n <ns>

# 5. Ingress / LB
kubectl describe ingress <ing>                   # events, backend errors
kubectl get svc <svc>                            # EXTERNAL-IP pending?
```

| Symptom | Likely cause |
|---|---|
| Endpoints empty | Selector/labels mismatch, pods not Ready |
| Connection refused | targetPort tappu, app 127.0.0.1 meeda listen |
| Timeout | NetworkPolicy, security group, NACL |
| Ingress ADDRESS empty | IngressClass ledu/tappu, controller events, subnet tags |
| ALB 502/503 | Target health fail: health check path, pod readiness, SG |
| LoadBalancer `<pending>` | LB controller events, subnet tags, quota |

## Node problems

```bash
kubectl get nodes
kubectl describe node <node>               # Conditions: MemoryPressure, DiskPressure, PIDPressure, Ready
kubectl top node
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
kubectl debug node/<node> -it --image=busybox:1.36    # node filesystem /host lo
```

## Deployment / rollout problems

```bash
kubectl rollout status deploy/<d>
kubectl describe deploy <d>                # Conditions
kubectl get rs -l app=<label>              # kotha RS pods count
kubectl rollout undo deploy/<d>
```

## RBAC / access problems

```bash
kubectl auth whoami
kubectl auth can-i <verb> <resource> -n <ns>
kubectl auth can-i --list -n <ns> --as=system:serviceaccount:<ns>:<sa>
```

`Forbidden: User "x" cannot list resource "pods"`: Role/RoleBinding missing. EKS lo access entry check.
`You must be logged in to the server (Unauthorized)`: IAM credentials expire / access entry ledu / wrong context.

## Interview: "pod start avvatledu, ela debug chestharu?"

Answer structure:
1. `kubectl get pods` status chusi category identify.
2. `kubectl describe pod` events.
3. `kubectl logs --previous`.
4. Category batti (Pending -> scheduling, ImagePull -> registry, CrashLoop -> app logs/exit code, NotReady -> probes).
5. Fix, verify, root cause prevent (alerts, limits, probes).

Lab: [labs/README.md#lab-20](labs/README.md#lab-20-break-and-fix-challenges)
