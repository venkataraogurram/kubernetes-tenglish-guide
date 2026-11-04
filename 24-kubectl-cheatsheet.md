# 24. kubectl cheatsheet

[Index](README.md) | Prev: [23](23-troubleshooting-playbook.md)

## Setup

- `alias k=kubectl`

  Typing thaggisthundi. `~/.zshrc` lo pettu. Completion: `source <(kubectl completion zsh)` + `compdef k=kubectl`.

- `kubectl config get-contexts`

  kubeconfig lo unna clusters/contexts. `*` current.

- `kubectl config use-context <ctx>`

  Vere cluster ki switch.

- `kubectl config set-context --current --namespace=<ns>`

  Default namespace marchu, prathi command ki `-n` avasaram ledu.

## Get, describe

- `kubectl get pods -o wide`

  Pod IP, node kuda chupisthundi.

- `kubectl get all -n <ns>`

  Pods, services, deployments, replicasets okesari (ConfigMaps, Secrets, Ingress, PVC raavu).

- `kubectl get pods -A --field-selector status.phase!=Running`

  Anni namespaces lo Running kaani pods.

- `kubectl get pods -l app=web --show-labels`

  Label filter + labels column.

- `kubectl get pods -w`

  Watch: changes live ga.

- `kubectl get deploy web -o yaml`

  Full object (spec + status).

- `kubectl get pods -o jsonpath='{.items[*].metadata.name}'`

  Specific fields matrame. Scripts ki.

- `kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,IMAGE:.spec.containers[0].image'`

  Own columns table.

- `kubectl get pods --sort-by=.status.startTime`

  Sort.

- `kubectl describe pod <pod>`

  Human-readable details + Events. Debug first step.

- `kubectl api-resources`

  Anni resource types, short names, apiVersion, namespaced aa.

- `kubectl explain pod.spec.containers --recursive`

  Field documentation.

## Create, apply, edit

- `kubectl apply -f file.yaml`

  Create leda update (declarative). Ide main.

- `kubectl apply -f dir/ -R`

  Folder recursive.

- `kubectl apply -k overlays/dev`

  Kustomize.

- `kubectl apply -f file.yaml --dry-run=server`

  Server validation, create cheyyadu.

- `kubectl diff -f file.yaml`

  Cluster vs file difference.

- `kubectl create deployment web --image=nginx:1.27 --dry-run=client -o yaml > d.yaml`

  YAML skeleton generate ([02](02-manifest-rayadam-ela.md)).

- `kubectl edit deploy web`

  Live edit (editor open). Git tho drift avthundi, learning/emergency ki matrame.

- `kubectl patch deploy web -p '{"spec":{"replicas":3}}'`

  Small change command line nundi.

- `kubectl delete -f file.yaml`

  File lo unnavi delete.

- `kubectl delete pod <pod> --grace-period=0 --force`

  Force delete. Last resort.

## Deployments

- `kubectl scale deploy web --replicas=5`

  Scale.

- `kubectl set image deploy/web web=nginx:1.28`

  Image update (container=image).

- `kubectl rollout status deploy/web`

  Rollout complete ayye varaku wait.

- `kubectl rollout history deploy/web`

  Revisions.

- `kubectl rollout undo deploy/web`

  Previous revision ki rollback.

- `kubectl rollout restart deploy/web`

  Anni pods rolling restart (ConfigMap change tarvata).

## Debug

- `kubectl logs <pod> -c <container> --previous --tail=100`

  Crash ayina previous container last 100 lines.

- `kubectl logs -l app=web -f --prefix`

  Label unna anni pods logs, pod name prefix tho.

- `kubectl exec -it <pod> -- sh`

  Container lo shell.

- `kubectl debug -it <pod> --image=busybox:1.36 --target=<container>`

  Ephemeral debug container (shell leni distroless images ki).

- `kubectl debug node/<node> -it --image=busybox:1.36`

  Node meeda debug pod, host FS `/host` lo.

- `kubectl run t --rm -it --image=busybox:1.36 --restart=Never -- sh`

  Throwaway pod: DNS/network test, exit chesthe delete.

- `kubectl port-forward svc/web 8080:80`

  Local `localhost:8080` -> Service. LB lekunda test. Default ga `127.0.0.1` matrame bind.

- `kubectl cp <pod>:/path/file ./file`

  Pod nundi file copy (container lo `tar` kavali).

- `kubectl get events -n <ns> --sort-by=.lastTimestamp`

  Recent events.

- `kubectl top pods --containers -n <ns>`

  CPU/memory usage.

- `kubectl auth can-i create pods -n <ns> --as=system:serviceaccount:<ns>:<sa>`

  Permissions check.

## Nodes

- `kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type`

  Labels as columns.

- `kubectl cordon <node>` / `kubectl uncordon <node>`

  Schedule stop / resume.

- `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`

  Pods evict (PDB respect).

- `kubectl taint nodes <node> key=value:NoSchedule`

  Taint add. Chivara `-` pedithe remove.

## Labels, annotations

- `kubectl label pod <pod> env=dev --overwrite`

  Label add/update. `env-` remove.

- `kubectl annotate deploy web kubernetes.io/change-cause="v2" --overwrite`

  Annotation.

## Output tricks

- `kubectl get pods -o name`

  `pod/web-xxx` format, xargs ki.

- `kubectl get pods -o name -l app=web | xargs -I{} kubectl delete {}`

  Bulk action (jagratha).

- `kubectl get deploy web -o yaml | grep -v -E 'creationTimestamp|resourceVersion|uid|managedFields'`

  Rough clean YAML. Better: `kubectl neat` plugin (krew).

## Plugins (krew)

- `kubectl krew install neat ctx ns stern`

  `neat` clean YAML, `ctx`/`ns` fast switching, `stern` multi-pod logs (`stern web -n shop`).

## Common short names

```
po=pods  deploy=deployments  rs=replicasets  sts=statefulsets  ds=daemonsets
svc=services  ing=ingress  cm=configmaps  ns=namespaces  no=nodes
pv=persistentvolumes  pvc=persistentvolumeclaims  sa=serviceaccounts
hpa=horizontalpodautoscalers  pdb=poddisruptionbudgets  netpol=networkpolicies
crd=customresourcedefinitions  sc=storageclasses  ep=endpoints
```
