# Kubernetes Cheatsheet

> **K8s** = container orchestration. It **schedules** containers onto machines, **scales** them on demand, **self-heals** crashed ones, and **load-balances** traffic — so apps run reliably at scale without micromanaging containers. This sheet targets local dev with **Minikube**.

---

## 1. Core Objects & Cluster Architecture

A cluster splits into a **Control Plane** (decides) and **Worker Nodes** (run the containers). Everything — `kubectl`, k9s, the components themselves — talks through the API Server.

| Component | What it does |
|---|---|
| **Control Plane** | The brain that manages the cluster. |
| ↳ **API Server** | Front door — every request and every component goes through it. |
| ↳ **Scheduler** | Picks which Node a new Pod runs on. |
| ↳ **Controller Manager** | Reconciles reality against desired state (this is what recreates a deleted Pod). |
| ↳ **etcd** | Key-value store holding the cluster's true state. |
| **Node** | A worker machine that runs Pods. |
| ↳ **kubelet** | Agent that runs and watches its Node's Pods. |
| ↳ **kube-proxy** | Routes network traffic to the right Pods. |
| ↳ **container runtime** | Actually runs the containers (containerd, CRI-O). |

> **Desired state** is the whole model: you *declare* the target ("10 replicas"), and the Controller Manager keeps fixing reality until it matches. That is where self-healing comes from — nothing restarts anything by hand.

| Object | What it is |
|---|---|
| **Pod** | Smallest deployable unit — wraps 1+ containers that run together. Own IP. **Ephemeral** (won't self-recover alone). |
| **Deployment** | Controller that manages Pods — keeps desired replica count, does rolling updates & rollbacks. Use this, not raw Pods, in prod. |
| **Service** | Stable network endpoint (fixed DNS + virtual IP) for a set of Pods, since Pod IPs change. |
| **ConfigMap** | Non-secret config data injected into Pods (env vars, files). |
| **Secret** | Sensitive data (passwords, tokens). Values are **base64-encoded, not encrypted** — anyone with `get secret` RBAC can decode them. Enable [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) or an external store (Vault, cloud CSI driver) for real secrets. |
| **Job** | Runs a Pod to **completion** (batch task) — one-shot ETL, backfill, migration. |
| **CronJob** | Creates Jobs on a **schedule** — the K8s-native way to run a recurring pipeline. |
| **PVC** | PersistentVolumeClaim — durable storage that survives Pod restarts. |
| **Namespace** | Logical isolation boundary for grouping resources. |

**Service types:** `ClusterIP` (default, internal only) · `NodePort` (exposes on each node's IP:port) · `LoadBalancer` (cloud-managed public LB).

---

## 2. Local Setup (Minikube)

```bash
# macOS
brew install kubectl minikube

# Windows (choco)
choco install kubernetes-cli minikube -y

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl
```

```bash
minikube version
kubectl version --client
minikube start --driver=docker      # or --driver=hyperv / virtualbox / hyperkit
kubectl get nodes                   # expect: minikube  Ready  control-plane
```

### Lifecycle

```bash
minikube stop      # pause cluster (keeps state)
minikube start     # resume
minikube delete    # wipe cluster entirely (fresh start next time)
minikube ip        # cluster IP (for NodePort access)
minikube service <svc-name>   # open a NodePort service in browser
```

### Using a locally-built image

The cluster can't see images in your laptop's Docker — hand it over, no registry needed:

```bash
docker build -t my-app:v1 .
minikube image load my-app:v1     # copy the image INTO the cluster
```

> **Don't tag it `:latest`.** With `:latest` the default pull policy makes K8s fetch from a registry and fail (`ErrImagePull`) instead of using your local image. Use a real tag, or set `imagePullPolicy: IfNotPresent`.

### k9s — terminal dashboard

Live, auto-refreshing TUI over the cluster — beats re-running `kubectl get pods` in a loop.

```bash
brew install k9s     # Windows: winget install k9s  |  choco install k9s
k9s                  # opens on the current context
```

| Key | Action |
|---|---|
| `:pods` `:svc` `:deploy` `:nodes` `:ns` | switch resource view (**colon = view**) |
| `0` | show all namespaces |
| `l` / `d` | logs / describe selected resource |
| `s` | scale (or shell, on a pod) |
| `Ctrl-D` | delete selected resource |
| `?` / `Esc` | help / go back |

---

## 3. Essential kubectl Commands

```bash
# Imperative shortcuts (fast for demos/debugging; YAML is what you commit)
kubectl create deployment redis --image=redis:7   # Deployment without writing YAML
kubectl expose deployment redis --port=6379       # Service in front of it (ClusterIP)
kubectl create deployment <n> --image=<img> --dry-run=client -o yaml > d.yaml  # scaffold YAML

# Apply / delete manifests
kubectl apply -f pod.yaml               # create/update from file
kubectl apply -f k8s/                    # apply a whole directory
kubectl delete -f deployment.yaml        # delete resources in file
kubectl delete namespace mini-project    # delete a namespace + everything in it

# Inspect
kubectl get pods                         # list pods
kubectl get pods -l app=hello --show-labels   # filter by label
kubectl get deployments
kubectl get svc                          # services (+ NodePort mappings)
kubectl get endpoints hello-service      # pod IPs behind a service (empty = selector mismatch)
kubectl get all -n mini-project          # everything in a namespace

# Debug
kubectl describe pod hello-pod           # events + full spec
kubectl logs <pod>                       # container logs
kubectl logs -f <pod>                    # follow logs
kubectl logs -f deployment/<name>        # follow via the Deployment (no pod name needed)
kubectl logs -f -l app=hello --prefix --max-log-requests=10   # ALL matching pods at once
kubectl exec -it <pod> -- bash           # shell into a pod
kubectl delete pod <pod>                 # managed pods are recreated → self-healing check

# Access
kubectl port-forward pod/hello-pod 8080:80          # localhost:8080 → pod:80
kubectl port-forward -n mini-project svc/mini-api 5000:5000
```

Add `-n <namespace>` to scope any command; `-A` / `--all-namespaces` to span all.

---

## 4. Pod

Smallest unit — usually managed by a Deployment, not created directly.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
spec:
  containers:
  - name: hello
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl port-forward pod/hello-pod 8080:80   # → http://localhost:8080
```

---

## 5. Deployment

Keeps N replicas running, handles rolling updates. The `selector.matchLabels` must match the Pod `template.metadata.labels`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello                 # must match template labels below
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl get deployments          # hello-deployment  2/2  READY
kubectl scale deployment hello-deployment --replicas=4
kubectl rollout status deployment hello-deployment
kubectl rollout undo deployment hello-deployment   # rollback
```

---

## 6. Service

Gives Pods a stable endpoint. The `selector` must match Pod labels.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello                   # routes to Pods with label app=hello
  ports:
    - protocol: TCP
      port: 80                   # service port
      targetPort: 80             # container port
  type: NodePort                 # ClusterIP | NodePort | LoadBalancer
```

```bash
kubectl get svc hello-service           # e.g. 80:32055/TCP
kubectl get endpoints hello-service     # must list pod IPs (empty → selector mismatch)
minikube service hello-service          # open NodePort in browser
curl http://$(minikube ip):<NodePort>
```

---

## 7. Deployment + Service Together

Common pattern — one manifest, `---` separates objects:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

```bash
kubectl apply -f deployment-and-service.yaml
kubectl delete -f deployment-and-service.yaml   # cleanup
```

---

## 8. Namespace, Secret, ConfigMap, PVC

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mini-project
```

### Secret (values are base64-encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-secret
  namespace: mini-project
type: Opaque
data:
  password: ZXhhbXBsZXBhc3M=    # echo -n "examplepass" | base64
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-init-sql
  namespace: mini-project
data:
  01-init.sql: |
    CREATE TABLE items (...);
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: mini-project
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
```

---

## 9. Consuming Secrets & Config in a Pod

Inject a Secret value as an env var via `secretKeyRef`:

```yaml
spec:
  containers:
  - name: mini-api
    image: mini-project-api:latest
    imagePullPolicy: IfNotPresent     # use locally-loaded image, don't pull
    env:
      - name: DB_HOST
        value: "postgres"             # a Service name → DNS resolves to Pods
      - name: DB_PASS
        valueFrom:
          secretKeyRef:
            name: pg-secret           # references the Secret above
            key: password
    ports:
      - containerPort: 5000
```

> Pods reach other Pods by **Service name** as hostname (e.g. `postgres:5432`), not Pod IP.

---

## 10. Cleanup

```bash
kubectl delete -f deployment-and-service.yaml    # by file
kubectl delete namespace <name>                  # a namespace + all its resources
minikube stop                                    # pause cluster
minikube delete                                  # wipe everything
```

---

## 11. Quick Reference

| Task | Command |
|---|---|
| Start cluster | `minikube start --driver=docker` |
| Cluster IP | `minikube ip` |
| Load local image | `minikube image load <image>` |
| Dashboard (TUI) | `k9s` |
| Quick Deployment | `kubectl create deployment <n> --image=<img>` |
| Quick Service | `kubectl expose deployment <n> --port=<port>` |
| Apply manifest | `kubectl apply -f file.yaml` |
| Apply a folder | `kubectl apply -f k8s/` |
| List pods | `kubectl get pods` |
| Pod logs (follow) | `kubectl logs -f <pod>` |
| Logs, all matching pods | `kubectl logs -f -l app=<v> --prefix` |
| Shell into pod | `kubectl exec -it <pod> -- bash` |
| Describe (events) | `kubectl describe pod <pod>` |
| Port-forward | `kubectl port-forward svc/<name> 8080:80` |
| Scale replicas | `kubectl scale deployment <name> --replicas=4` |
| Rollback | `kubectl rollout undo deployment <name>` |
| Check endpoints | `kubectl get endpoints <svc>` |
| Open NodePort | `minikube service <svc>` |
| Delete namespace | `kubectl delete namespace <name>` |
| Encode a secret | `echo -n "value" \| base64` |
