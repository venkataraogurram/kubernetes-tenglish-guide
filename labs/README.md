# Labs (EKS Auto Mode meeda test chesinavi)

[Index](../README.md)

## Before you start

```bash
kubectl config current-context        # correct cluster aa?
kubectl get nodes
cd ~/path/to/kubernetes-tenglish-guide
kubectl apply -f labs/00-namespace.yaml
kubectl config set-context --current --namespace=k8s-lab   # prathi command ki -n avasaram ledu
```

Notes:
- Auto Mode `general-purpose` NodePool amd64 nodes launch chesthundi. Pods Pending lo 1 min undochu, Auto Mode node launch chesthondi. Normal.
- Cost unna labs (ALB, EBS) lo **COST** ani mark chesa. Ayyaka ventane delete.
- Services ni local ga test cheyyadaniki `kubectl port-forward` vadatham. Adi `127.0.0.1` meeda matrame bind avthundi, internet ki expose avvadu.

## Full cleanup (eppudaina)

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace k8s-lab k8s-lab-secure --ignore-not-found
kubectl delete -f labs/06-ingress-alb.yaml --ignore-not-found      # IngressClass, params (cluster-level)
kubectl delete storageclass lab-auto-ebs --ignore-not-found
kubectl delete -f labs/15-crd.yaml --ignore-not-found
```

Namespace delete chesthe Ingress (ALB), PVC (EBS) delete avthayi, Auto Mode AWS resources clean chesthundi. 5 min tarvata EC2 console lo Load Balancers, Volumes check cheyyi.

---

## Lab 01: First Deployment and Service

Topics: [02](../02-manifest-rayadam-ela.md), [04](../04-labels-selectors-annotations.md), [05](../05-deployment-replicaset.md)

```bash
kubectl apply -f labs/01-web.yaml --dry-run=server     # validate first
kubectl apply -f labs/01-web.yaml
kubectl get deploy,rs,pods -l app=web                  # Deployment -> ReplicaSet -> Pods chain
kubectl get pods -l app=web --show-labels              # pod-template-hash label chudu
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl port-forward svc/web 8080:80                   # browser: http://localhost:8080, Ctrl+C tho stop
```

Self-healing chudu:

```bash
kubectl delete pod -l app=web --wait=false
kubectl get pods -l app=web -w                         # kotha random names tho malli vasthayi
```

**Break it** (labels lesson): `01-web.yaml` lo Service `selector` ni `app: wrong` ga marchi apply cheyyi.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web   # endpoints empty
kubectl port-forward svc/web 8080:80                           # error: no running pods / connection fail
```

Error message radhu, traffic matrame aagipothundi. Selector malli `app: web` ki marchi apply.

## Lab 02: Pod debugging and init container

Topics: [03](../03-pods.md)

```bash
kubectl apply -f labs/02-pod-init.yaml
kubectl get pod init-demo -w            # Init:0/2 -> PodInitializing -> Running (2/2: app + sidecar)
kubectl logs init-demo -c wait-for-web
kubectl logs init-demo -c log-tailer -f # sidecar app log file tail chesthondi
kubectl exec -it init-demo -c app -- sh -c 'ls -l /shared; tail -3 /shared/app.log'
kubectl describe pod init-demo | sed -n '/Init Containers/,/Containers:/p'
```

Try: Lab 01 Service delete chesi (`kubectl delete svc web`) init-demo malli create cheyyi. `Init:0/2` lo wait chesthundi. Service malli apply chesthe start avthundi.

## Lab 03: Rolling update and rollback

Topics: [05](../05-deployment-replicaset.md)

```bash
kubectl annotate deploy web kubernetes.io/change-cause="nginx 1.27 initial" --overwrite
kubectl set image deploy/web web=nginx:1.28
kubectl annotate deploy web kubernetes.io/change-cause="nginx 1.28" --overwrite
kubectl rollout status deploy/web
kubectl get rs -l app=web                 # old RS 0 replicas, kotha RS 2
kubectl rollout history deploy/web

# Bad rollout
kubectl set image deploy/web web=nginx:9.99-nope
kubectl rollout status deploy/web --timeout=60s     # stuck
kubectl get pods -l app=web                          # ImagePullBackOff, kani old pods inka serve chesthunnayi
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
```

Gamaninchu: bad image tho kuda old pods delete avvaledu (maxUnavailable 25% of 2 = 0 round down, maxSurge 1). Ade rolling update safety.
Lab ayyaka `labs/01-web.yaml` malli apply chesi image `1.27` ki teesukuraa (Git = truth).

## Lab 04: Job, CronJob, DaemonSet

Topics: [06](../06-statefulset-daemonset-job-cronjob.md)

```bash
kubectl apply -f labs/04-job-cronjob-daemonset.yaml
kubectl get pods -l job-name=pi-count -w      # 2 parallel, total 3 completions
kubectl get job pi-count
kubectl get ds node-hello -o wide             # DESIRED = schedulable nodes count
kubectl get pods -l app=node-hello -o wide    # prathi node meeda okati
kubectl get cronjob every-2-min
kubectl create job manual-tick --from=cronjob/every-2-min
kubectl logs job/manual-tick
```

DaemonSet `system` nodes meeda raadu (`CriticalAddonsOnly` taint), endukante toleration ledu. `kubectl describe node <system-node> | grep Taints` tho chudu.
Cleanup: `kubectl delete -f labs/04-job-cronjob-daemonset.yaml`

## Lab 05: Services and DNS

Topics: [07](../07-services-dns.md)

```bash
kubectl run dns --rm -it --image=busybox:1.36 --restart=Never -- sh
# lopala:
nslookup web
nslookup web.k8s-lab.svc.cluster.local
cat /etc/resolv.conf
wget -qO- http://web | head -5
exit
```

Headless chudu:

```bash
kubectl create service clusterip web-headless --clusterip=None --tcp=80:80 --dry-run=client -o yaml \
  | sed 's/app: web-headless/app: web/' | kubectl apply -f -
kubectl run dns2 --rm -it --image=busybox:1.36 --restart=Never -- nslookup web-headless   # pod IPs direct
kubectl delete svc web-headless
```

## Lab 06: Ingress with EKS Auto Mode ALB

Topics: [08](../08-ingress-gateway-api.md). **COST + public exposure**: internet-facing ALB. Lab 01 apply chesi undali.

```bash
kubectl apply -f labs/06-ingress-alb.yaml
kubectl get ingress web -w                # ADDRESS 2-4 min lo vasthundi
kubectl describe ingress web              # events
curl -s http://<ADDRESS>/ | head -5
```

ADDRESS raakapothe: `kubectl describe ingress web` events, subnet tags (`kubernetes.io/role/elb=1`) check.
**Ayyaka ventane**: `kubectl delete -f labs/06-ingress-alb.yaml`

## Lab 07: ConfigMap and Secret

Topics: [09](../09-configmap-secret.md)

```bash
kubectl apply -f labs/07-config-secret.yaml
kubectl logs config-demo -f
kubectl exec config-demo -- ls -l /etc/app /etc/secret
kubectl exec config-demo -- cat /etc/secret/DB_PASSWORD; echo
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo   # base64 matrame, encryption kaadu
```

Update behavior test:

```bash
kubectl patch configmap app-config --type merge -p '{"data":{"APP_ENV":"qa","app.properties":"greeting=Hello\n"}}'
# ~1 min wait
kubectl logs config-demo --tail=2
```

File (`greeting=Hello`) update ayyindi, env `APP_ENV` inka `dev`. Env ki pod restart kavali.
Cleanup: `kubectl delete -f labs/07-config-secret.yaml`

## Lab 08: Persistent storage with EBS

Topics: [10](../10-storage-pv-pvc-storageclass.md). **COST**: 1Gi gp3 volume.

```bash
kubectl get storageclass                   # gp2 (in-tree) undi kani Auto Mode di kaadu
kubectl apply -f labs/08-storage.yaml
kubectl get pvc data -w                    # Pending (WaitForFirstConsumer) -> Bound pod schedule ayyaka
kubectl get pv
kubectl logs writer

kubectl delete pod writer
kubectl apply -f labs/08-storage.yaml      # pod malli create
kubectl logs writer                        # rendu lines: data survive ayyindi
```

Cleanup: `kubectl delete -f labs/08-storage.yaml` (PVC delete -> EBS volume delete).

## Lab 09: Probes, resources and OOMKilled

Topics: [11](../11-resources-qos-probes.md). Lab 01 apply chesi undali.

```bash
kubectl apply -f labs/09-probes-oom.yaml
kubectl get pods not-ready oom-demo -w
kubectl describe pod not-ready | grep -i readiness      # Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl get endpointslices -l kubernetes.io/service-name=web -o yaml | grep -B2 -A2 "ready: false"
kubectl describe pod oom-demo | grep -A6 "Last State"   # Reason: OOMKilled, Exit Code: 137
kubectl get pod oom-demo -o jsonpath='{.status.qosClass}'; echo
kubectl top pods
```

Fix try: `oom-demo` limit `300Mi` ki penchi (pod delete chesi malli apply). Running avthundi.
Cleanup: `kubectl delete -f labs/09-probes-oom.yaml`

## Lab 10: RBAC and ServiceAccount

Topics: [12](../12-namespaces-rbac-serviceaccounts.md)

```bash
kubectl apply -f labs/10-rbac.yaml
kubectl auth can-i list pods -n k8s-lab --as=system:serviceaccount:k8s-lab:pod-viewer      # yes
kubectl auth can-i delete pods -n k8s-lab --as=system:serviceaccount:k8s-lab:pod-viewer    # no
kubectl auth can-i list secrets -n k8s-lab --as=system:serviceaccount:k8s-lab:pod-viewer   # no
kubectl auth can-i list pods -n default --as=system:serviceaccount:k8s-lab:pod-viewer      # no (Role namespace-scoped)
```

Pod lopala nundi API call:

```bash
kubectl exec -it api-client -- sh
# lopala:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CA=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl -s --cacert $CA -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/k8s-lab/pods | head -20
curl -s --cacert $CA -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/k8s-lab/secrets   # 403 Forbidden
exit
```

Cleanup: `kubectl delete -f labs/10-rbac.yaml`

## Lab 11: Scheduling: spread and PDB

Topics: [13](../13-scheduling-affinity-taints.md)

```bash
kubectl apply -f labs/11-spread-pdb.yaml
kubectl get pods -l app=spread -o wide -w           # Auto Mode zones lo nodes launch chesthondi
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -l app=spread -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName'
kubectl get pdb spread-pdb                           # ALLOWED DISRUPTIONS 1
```

Taints chudu:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints[*].key,POOL:.metadata.labels.karpenter\.sh/nodepool'
```

Experiment: `whenUnsatisfiable: DoNotSchedule` tho `replicas: 7` chesi chudu, zones madhya 3-2-2 / skew 1 maintain.
Cleanup: `kubectl delete -f labs/11-spread-pdb.yaml` (Auto Mode khaali nodes ni konchem sepu tarvata remove chesthundi)

## Lab 12: HPA autoscaling

Topics: [14](../14-autoscaling-hpa-vpa-karpenter.md)

```bash
kubectl apply -f labs/12-hpa.yaml
kubectl get hpa php-apache -w            # TARGETS cpu: 0%/50% (konchem sepu <unknown> normal)
```

Vere terminal lo load:

```bash
kubectl run load --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c "while true; do wget -q -O- http://php-apache > /dev/null; done"
```

HPA replicas 1 -> 4/5/6 perugutayi, kotha pods Pending aithe Auto Mode node add chesthundi (`kubectl get nodeclaims -w`).
Load aapu (Ctrl+C), 1-2 min lo scale down.
Cleanup: `kubectl delete -f labs/12-hpa.yaml`

## Lab 13: NetworkPolicy

Topics: [15](../15-networking-deep-dive.md). Lab 01 apply chesi undali.

**Cluster-level prerequisite (okkasari)**: Auto Mode network policy controller enable. Cluster lo already `amazon-vpc-cni` ConfigMap unte daanni merge cheyyi, overwrite cheyyaku:

```bash
kubectl get configmap amazon-vpc-cni -n kube-system -o yaml 2>/dev/null || echo "not present"
kubectl patch configmap amazon-vpc-cni -n kube-system --type merge -p '{"data":{"enable-network-policy-controller":"true"}}' \
  || kubectl create configmap amazon-vpc-cni -n kube-system --from-literal=enable-network-policy-controller=true
```

Before policy (anni allow):

```bash
kubectl run t1 --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://web | head -3
```

Apply:

```bash
kubectl apply -f labs/13-netpol.yaml
kubectl run t2 --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://web            # timeout (blocked)
kubectl run t3 --rm -it --image=busybox:1.36 --restart=Never -l role=frontend -- wget -qO- -T 3 http://web | head -3   # pass
```

Cleanup: `kubectl delete -f labs/13-netpol.yaml` (ConfigMap unchochu, policies lekapothe effect ledu)

## Lab 14: Pod Security restricted

Topics: [16](../16-security.md)

```bash
kubectl apply -f labs/14-pod-security.yaml
kubectl get pods -n k8s-lab-secure                 # secure-web Running
kubectl run bad -n k8s-lab-secure --image=nginx:1.27   # REJECT: violates PodSecurity "restricted"
kubectl -n k8s-lab-secure port-forward deploy/secure-web 8081:8080   # http://localhost:8081
```

Error message lo e fields missing (`allowPrivilegeEscalation`, `runAsNonRoot`, `capabilities`, `seccompProfile`) ani cheptundi. Aa list ne checklist.
Cleanup: `kubectl delete namespace k8s-lab-secure`

## Lab 15: CRD without and with a controller

Topics: [17](../17-controllers-crds-operators.md)

```bash
kubectl apply -f labs/15-crd.yaml
kubectl api-resources | grep -i website
kubectl explain website.spec
kubectl apply -f labs/15-website.yaml
kubectl get websites                      # MESSAGE, REPLICAS columns
kubectl get ws hello -o yaml
kubectl get pods                          # kotha pods emi levu
```

Lesson: CRD = API + storage matrame. Pods/Deployments create cheyyadaniki controller kavali. Validation chudu:

```bash
kubectl patch ws hello --type merge -p '{"spec":{"replicas":10}}'   # rejected: maximum 5
```

Next step (optional, advanced): Kubebuilder tutorial tho `Website` controller rasi, Deployment + ConfigMap create cheyyinchu.
Cleanup: `kubectl delete -f labs/15-website.yaml -f labs/15-crd.yaml`

## Lab 16: Kustomize overlays

Topics: [18](../18-helm-kustomize.md)

```bash
kubectl kustomize labs/16-kustomize/overlays/dev     # rendered YAML chudu: names -dev, env label, configmap hash suffix
kubectl kustomize labs/16-kustomize/overlays/prod    # replicas 3, image 1.28
diff <(kubectl kustomize labs/16-kustomize/overlays/dev) <(kubectl kustomize labs/16-kustomize/overlays/prod)
kubectl apply -k labs/16-kustomize/overlays/dev
kubectl apply -k labs/16-kustomize/overlays/prod
kubectl get deploy,svc,cm -l app=kweb
kubectl get endpointslices -l kubernetes.io/service-name=kweb-dev    # dev pods matrame (includeSelectors)
```

Overlay lo `LOG_LEVEL` marchi apply: ConfigMap kotha hash name tho create, Deployment reference maari automatic rollout.
Cleanup: `kubectl delete -k labs/16-kustomize/overlays/dev && kubectl delete -k labs/16-kustomize/overlays/prod`

## Lab 17: Argo CD (optional)

Topics: [19](../19-gitops-argocd.md)

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8443:443      # https://localhost:8443  user: admin
```

Ee repo ne source ga Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kweb-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/venkataraogurram/kubernetes-tenglish-guide.git
    targetRevision: main
    path: labs/16-kustomize/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: k8s-lab
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

selfHeal test: `kubectl scale deploy kweb-dev -n k8s-lab --replicas=5`, Argo CD malli 1 ki teesukuvasthundi.
Cleanup: Application delete, `kubectl delete namespace argocd`.

## Lab 18: Events, logs and top

Topics: [20](../20-observability.md)

```bash
kubectl get events --sort-by=.lastTimestamp | tail -20
kubectl get events -A --field-selector type=Warning
kubectl logs -l app=web --prefix --tail=5
kubectl top nodes
kubectl top pods -A --sort-by=memory | head
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/k8s-lab/pods | head -c 600; echo
```

## Lab 19: Istio canary (optional)

Topics: [21](../21-service-mesh-istio.md). Istio control plane + sidecars resources vadatayi, lab ayyaka uninstall.

```bash
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PWD/$(ls -d istio-*/ | head -1)bin:$PATH"
istioctl install --set profile=demo -y
kubectl label namespace k8s-lab istio-injection=enabled
kubectl rollout restart deploy/web
kubectl get pods -l app=web               # 2/2
```

Rendu versions:

```bash
kubectl create deployment web-v2 --image=nginx:1.28 --dry-run=client -o yaml \
  | sed 's/app: web-v2/app: web/' > /tmp/web-v2.yaml
# /tmp/web-v2.yaml lo template labels ki "version: v2" add, labs/01-web.yaml template ki "version: v1" add, apply
```

Tarvata [21](../21-service-mesh-istio.md) lo DestinationRule (subsets v1/v2) + VirtualService (90/10) apply chesi:

```bash
# istio-proxy container lo curl undi; Server header lo nginx/1.27 (v1) vs nginx/1.28 (v2) ~90/10 ratio
for i in $(seq 1 20); do kubectl exec deploy/web -c istio-proxy -- curl -sI http://web | grep -i '^server'; done
```

Cleanup: `istioctl uninstall --purge -y && kubectl delete namespace istio-system && kubectl label namespace k8s-lab istio-injection-`

## Lab 20: Break-and-fix challenges

Topics: [23](../23-troubleshooting-playbook.md)

```bash
kubectl apply -f labs/20-broken.yaml
kubectl get pods
```

4 bugs unnayi. Prathi daaniki symptom kanukko, root cause cheppu, fix cheyyi. Hints:
1. `broken-api` pods start avvatledu. STATUS chudu.
2. Image fix chesaka kuda pods `0/1 Ready`. describe lo probe event chudu.
3. Pods Ready ayyaka kuda Service ki endpoints levu.
4. `broken-worker` `ContainerCreating` lo stuck.

<details>
<summary>Answers (modata nuvve try cheyyi)</summary>

1. `image: nginx:1.27-doesnotexist` -> `ErrImagePull` / `ImagePullBackOff` (`code = NotFound ... failed to resolve image`). Fix: `nginx:1.27`.
2. `readinessProbe.port: 8080`, nginx 80 meeda listen. Fix: `port: 80`.
3. Service `selector: app: brokenapi`, pods label `broken-api`. Fix: `app: broken-api`. Check: `kubectl get endpointslices -l kubernetes.io/service-name=broken-api`.
4. ConfigMap `worker-settings` ledu -> `FailedMount ... configmap "worker-settings" not found`. Fix: `kubectl create configmap worker-settings --from-literal=settings.env=MODE=lab` (pod automatic ga start avthundi).

</details>

Cleanup: `kubectl delete -f labs/20-broken.yaml; kubectl delete configmap worker-settings --ignore-not-found`
