# 03. Pods

[Index](README.md) | Prev: [02](02-manifest-rayadam-ela.md) | Next: [04. Labels](04-labels-selectors-annotations.md)

## Pod ante enti

- Kubernetes lo **smallest deployable unit**. Container kaadu, Pod.
- Oka Pod lo okati leda ekkuva containers. Avi:
  - Same **network namespace** share chesthayi: same IP, `localhost` tho okadanni okati reach avthayi, same ports space.
  - **Volumes** share cheyyochu.
  - Same node meeda kalisi schedule avthayi, kalisi start/stop avthayi.
- Pod ki oka IP vasthundi (EKS lo VPC subnet nundi real IP).

Rule of thumb: **okka Pod = okka app instance**. Two different apps (web + database) ni okka pod lo pettaku. Scale cheyyalante separate ga scale avvali kada.

## Pod ni direct ga create cheyyam enduku

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
spec:
  containers:
  - name: hello
    image: nginx:1.27
```

Idi pani chesthundi, kani Pod **mortal**. Node chanipothe, delete chesthe, malli radhu. Self-healing, scaling, rolling update emi ledu.
Anduke real apps ki **Deployment / StatefulSet / DaemonSet / Job** vadatham. Avi Pods ni create chesi manage chesthayi.
Standalone Pod debugging ki, quick test ki matrame.

## Pod lifecycle (phases)

| Phase | Ardham |
|---|---|
| `Pending` | Pod accept ayyindi, kani inka run avvaledu. Node select avvaledu leda image pull avthondi |
| `Running` | Node ki bind ayyindi, kanisam okka container run avthondi |
| `Succeeded` | Anni containers exit 0 tho complete (Jobs lo) |
| `Failed` | Kanisam okka container non-zero exit tho fail, restart cheyyadu |
| `Unknown` | Node tho contact poyindi |

`kubectl get pods` lo `STATUS` column lo vachhe `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating`, `OOMKilled` ivi phases kaavu, **container state reasons**. Debug ki [23](23-troubleshooting-playbook.md).

### Container states

- `Waiting`: start avvaledu (image pull, `CrashLoopBackOff` wait).
- `Running`: run avthondi.
- `Terminated`: exit ayyindi (`exitCode`, `reason` chudu).

```bash
kubectl get pod hello -o jsonpath='{.status.containerStatuses[0].state}'
kubectl get pod hello -o jsonpath='{.status.containerStatuses[0].lastState}'   # previous crash reason
```

## restartPolicy

Pod spec level lo:
- `Always` (default): container exit aithe eppudu restart. Deployments ki idi matrame allowed.
- `OnFailure`: non-zero exit aithe matrame restart. Jobs ki.
- `Never`: restart cheyyadu.

Restart lu exponential backoff tho jarugutayi (10s, 20s, 40s ... max 5 min). Ade `CrashLoopBackOff`.

## Multi-container patterns

### Init containers

Main containers start ayye **mundu**, order lo okkokati run ayyi complete avvali.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ["sh", "-c", "until nc -z db 5432; do echo waiting; sleep 2; done"]
  containers:
  - name: app
    image: myapp:1.0
```

Use cases: DB ready ayye varaku wait, config files download, DB migration, permissions fix.
Init container fail aithe Pod `Init:CrashLoopBackOff` lo untundi, main container start avvadu.

### Sidecar containers

Main app tho paatu side lo run ayye helper: log shipper, proxy (Istio Envoy), config reloader.

Kotha (native) way, Kubernetes 1.29+ lo default ga undi: `initContainers` lo `restartPolicy: Always` pedithe adi sidecar avthundi.

```yaml
spec:
  initContainers:
  - name: log-shipper
    image: fluent/fluent-bit:3.1
    restartPolicy: Always          # idi sidecar ani cheptundi
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

Native sidecar enduku better: main container kante mundu start avthundi, main tarvata stop avthundi, Jobs complete avvadaniki block cheyyadu.
Old way (`containers` lo rendu containers pettadam) kuda pani chesthundi, kani start/stop order guarantee ledu.

### Shared volume example

Paina `emptyDir` volume: pod tho paatu create avthundi, pod delete aithe poyindi. Containers madhya files share ki.

## Container spec lo important fields

```yaml
containers:
- name: app
  image: nginx:1.27
  imagePullPolicy: IfNotPresent     # Always / IfNotPresent / Never. ":latest" tag ki default Always
  command: ["nginx"]                # image ENTRYPOINT ni override
  args: ["-g", "daemon off;"]       # image CMD ni override
  workingDir: /app
  env:
  - name: APP_ENV
    value: "dev"
  - name: POD_NAME                  # Downward API: pod info ni env lo
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  ports:
  - containerPort: 80
    name: http                      # port ki name, Service lo targetPort: http ani vadachu
  resources: {...}                  # 11 lo
  readinessProbe: {...}             # 11 lo
  securityContext: {...}            # 16 lo
  volumeMounts: [...]               # 10 lo
```

`command` vs `args` confusion:

| Docker | Kubernetes |
|---|---|
| `ENTRYPOINT` | `command` |
| `CMD` | `args` |

## Pod lo login, debug

```bash
kubectl exec -it hello -- sh                  # container lo shell
kubectl exec -it mypod -c sidecar -- sh       # multi-container lo specific container
kubectl logs hello                            # logs
kubectl logs hello --previous                 # crash ayina previous container logs
kubectl logs mypod -c app -f                  # follow
kubectl describe pod hello                    # events: scheduling, pull, probe failures
kubectl debug -it hello --image=busybox:1.36 --target=hello   # ephemeral debug container (distroless images ki)
kubectl port-forward pod/hello 8080:80        # local ga access
```

## Graceful shutdown

Pod delete chesinappudu:
1. Pod `Terminating` avthundi, Service endpoints nundi remove avthundi.
2. `preStop` hook (unte) run avthundi.
3. Container ki `SIGTERM` vasthundi.
4. `terminationGracePeriodSeconds` (default 30s) wait.
5. Inka run avthunte `SIGKILL`.

App `SIGTERM` handle cheyyakapothe in-flight requests drop avthayi. Load balancer deregistration ki time ivvadaniki common trick:

```yaml
lifecycle:
  preStop:
    sleep:
      seconds: 10        # Kubernetes 1.30+ native sleep action. Paatha versions lo exec: ["sleep","10"]
```

## Interview one-liners

- Pod vs container: Pod ante shared network/storage unna containers group. Kubernetes containers ni direct ga manage cheyyadu, Pods ni manage chesthundi.
- Pod lo containers ela matladukuntayi: `localhost` tho.
- Init container vs sidecar: init run ayyi complete avthundi mundu; sidecar app tho paatu run avthune untundi.
- CrashLoopBackOff ante: container malli malli crash avthondi, kubelet backoff tho restart chesthondi.

Lab: [labs/README.md#lab-02](labs/README.md#lab-02-pod-debugging-and-init-container)
