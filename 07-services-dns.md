# 07. Services, DNS

[Index](README.md) | Prev: [06](06-statefulset-daemonset-job-cronjob.md) | Next: [08. Ingress](08-ingress-gateway-api.md)

## Problem

Pods mortal. Restart aithe kotha IP. 3 replicas unte 3 IPs. Client e IP ki call cheyyali?
**Service** oka stable virtual IP (ClusterIP) + DNS name isthundi, labels tho pods select chesi traffic balance chesthundi.

```
client pod --> web (DNS) --> 10.100.12.34 (ClusterIP) --> kube-proxy rules --> pod A / pod B / pod C
```

## Endpoints / EndpointSlices

Service create chesthe, EndpointSlice controller selector match ayye **Ready** pods IPs ni EndpointSlice objects lo pedathundi.
kube-proxy vaatini chusi node meeda iptables/IPVS rules set chesthundi.

```bash
kubectl get svc web
kubectl get endpointslices -l kubernetes.io/service-name=web -o wide
```

Readiness probe fail ayina pod endpoints nundi remove avthundi. Traffic radhu, kani pod restart avvadu.

## Service types

### ClusterIP (default)

Cluster lopala matrame reachable. Internal microservices ki.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80          # Service port
    targetPort: 8080  # container port (number leda container port name)
```

### NodePort

Prathi node meeda oka port (30000-32767) open chesi Service ki forward chesthundi.

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080     # ivvakapothe random
```

`<any-node-ip>:30080`. Direct ga production lo vadam. LoadBalancer / Ingress lu internally deenini vadatayi.

### LoadBalancer

Cloud load balancer create chesthundi. EKS Auto Mode lo **NLB** automatic ga create avthundi.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-public
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing   # public kavalante explicit ga
spec:
  type: LoadBalancer
  loadBalancerClass: eks.amazonaws.com/nlb   # Auto Mode lo default, omit cheyyochu
  loadBalancerSourceRanges:
  - 203.0.113.10/32                           # nee IP matrame allow. 0.0.0.0/0 vaddu
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

Prathi LoadBalancer Service ki separate NLB = separate cost. 10 HTTP services unte 10 NLBs kaakunda okka ALB + Ingress vadu ([08](08-ingress-gateway-api.md)).

### ExternalName

DNS CNAME alias. Proxy ledu, selector ledu.

```yaml
spec:
  type: ExternalName
  externalName: mydb.abc123.us-east-1.rds.amazonaws.com
```

App `mydb` ki connect avthe RDS endpoint ki resolve avthundi. Environment ki environment external host marchadaniki use.

### Headless Service

`clusterIP: None`. Virtual IP ledu, load balancing ledu. DNS query ki **pod IPs direct ga** return chesthundi.
StatefulSets ki (prathi pod ki DNS), client-side load balancing (gRPC), service discovery ki.

```yaml
spec:
  clusterIP: None
  selector:
    app: db
```

## port vs targetPort vs nodePort vs containerPort

```
client --> Service:port (80) --> Pod:targetPort (8080) == container listens on containerPort (8080)
external --> Node:nodePort (30080) --> Service:port --> Pod:targetPort
```

- `containerPort`: informational. App nijanga e port meeda listen chesthundo adi `targetPort` tho match avvali.
- `targetPort` lo port **name** vadochu (`targetPort: http`). Different pods different port numbers vadina pani chesthundi.

## DNS (CoreDNS)

Prathi Service ki DNS record:

```
<service>.<namespace>.svc.cluster.local
```

- Same namespace lo: `http://web`
- Vere namespace nundi: `http://web.shop` leda full `web.shop.svc.cluster.local`
- StatefulSet pod: `db-0.db.shop.svc.cluster.local`

Pod `/etc/resolv.conf` lo search domains untayi, anduke short names pani chesthayi:

```bash
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup web
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- cat /etc/resolv.conf
```

`ndots:5` valla external names (`api.github.com`) ki modata search domains try chesthundi, extra DNS queries. High-traffic apps lo FQDN chivara `.` pettadam (`api.github.com.`) leda `dnsConfig` tho `ndots` thagginchadam.

## Session affinity, traffic policies

```yaml
spec:
  sessionAffinity: ClientIP            # same client -> same pod
  internalTrafficPolicy: Local         # same node pods ki matrame (latency)
  externalTrafficPolicy: Local         # NodePort/LB: client source IP preserve, kani node lo pod lekapothe drop
  trafficDistribution: PreferClose     # same zone pods ki prefer (cross-AZ cost thaggisthundi), 1.31+
```

## Selector leni Service

Selector ivvakapothe EndpointSlice automatic ga create avvadu. Nuvve manual ga external IPs tho EndpointSlice create cheyyochu. Cluster bayata unna database ni Service name tho access cheyyadaniki.

## Debug checklist (Service pani cheyyatledu)

```bash
kubectl get svc web -o wide                          # selector, ports
kubectl get pods -l app=web -o wide                  # selector tho pods vasthunnaya, Ready aa?
kubectl get endpointslices -l kubernetes.io/service-name=web   # IPs unnaya?
kubectl run curl --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- curl -sv http://web
kubectl exec -it <pod> -- sh -c 'wget -qO- localhost:8080'     # app pod lopala listen chesthonda?
```

1. Endpoints empty: selector/labels mismatch leda pods not Ready.
2. Endpoints unnayi kani connection refused: targetPort tappu leda app `127.0.0.1` meeda listen chesthondi (`0.0.0.0` kavali container lopala).
3. DNS resolve avvatledu: namespace tappu, CoreDNS pods check.
4. Timeout: NetworkPolicy block chesthondi ([15](15-networking-deep-dive.md)).

## Interview one-liners

- Service enduku: pods ki stable IP/DNS + load balancing.
- ClusterIP ela pani chesthundi: virtual IP, kube-proxy iptables/IPVS rules tho DNAT chesi pod IP ki pampisthundi.
- Headless service: `clusterIP: None`, DNS direct pod IPs return chesthundi.
- LoadBalancer vs Ingress: LB = L4 per service; Ingress = L7, okka LB tho chala services, host/path routing.

Lab: [labs/README.md#lab-05](labs/README.md#lab-05-services-and-dns)
