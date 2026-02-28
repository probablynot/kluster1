# Kind-lab
A place to learn about kube
# Getting started with KinD on RHEL 10 (and deploying NGINX from a ConfigMap)

You’ve got years of Solaris muscle-memory — nice. Think of this as building a **little “zones-like” Kubernetes playground** right on your laptop: KinD runs each Kubernetes “node” as a container.

This guide sets up:
- **KinD (Kubernetes in Docker/Podman)**
- an **NGINX Deployment** serving a web page stored in a **ConfigMap**
- a **NodePort** Service mapped so you can hit it from **localhost**

---

## 0) Prereqs (what you need installed)

### Packages
On RHEL 10, Podman is the standard container engine. KinD can use **rootless Podman** (via the Podman socket / Docker-compatible API). KinD supports rootless Podman and requires cgroup v2 on the host.

Install baseline tools:

```bash
sudo dnf install -y \
  podman podman-docker \
  curl wget git \
  tar gzip
```

> `podman-docker` provides Docker CLI compatibility (so tools that expect `docker` can still work).

---

## 1) Enable the Podman API socket (Docker-compatible endpoint)

### Rootless (recommended for a laptop)
Enable the user socket:

```bash
systemctl --user enable --now podman.socket
```

Point Docker-compatible tools at the Podman socket:

```bash
export DOCKER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"
```

Add that export to your shell profile so it persists (pick one):

```bash
echo 'export DOCKER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"' >> ~/.bashrc
# or
echo 'export DOCKER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"' >> ~/.zshrc
```

These steps follow Red Hat’s RHEL 10 container-tools guidance for enabling `podman.socket` and using `DOCKER_HOST` for rootless Podman compatibility.

Quick sanity check:

```bash
podman info
podman ps
```

---

## 2) Install kubectl

Kubernetes upstream provides a simple “download the stable binary” method.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

> If your laptop is ARM64, swap `amd64` for `arm64`.

---

## 3) Install kind

KinD’s quick start recommends installing from the release binaries.

Example for x86_64:

```bash
# Pick a version from the GitHub releases page here: https://github.com/kubernetes-sigs/kind/releases
# Replace <VERSION> with something like v0.31.0 (or whatever is current).
curl -Lo kind "https://kind.sigs.k8s.io/dl/<VERSION>/kind-linux-amd64"
chmod +x kind
sudo mv kind /usr/local/bin/
kind version
```

> If you prefer, you can also install via package managers (as available), but the release binary is the most universal path.

---

## 4) Create a KinD cluster with NodePort mapped to localhost

### Why the extra port mapping?
In KinD, “nodes” are containers. A **NodePort** opens on the *node*, but that’s still inside the container network.
So to reach it via `localhost`, you map that NodePort from the node-container to your host using `extraPortMappings`.

Create `kind-cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        listenAddress: "127.0.0.1"
        protocol: TCP
```

This is the same mechanism documented in KinD’s configuration docs (`extraPortMappings`).

Create the cluster:

```bash
kind create cluster --name lab --config kind-cluster.yaml
kubectl cluster-info --context kind-lab
kubectl get nodes -o wide
```

---

## 5) Deploy NGINX serving a page from a ConfigMap

Create `nginx-cm-nodeport.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
data:
  index.html: |
    <!doctype html>
    <html>
      <head>
        <meta charset="utf-8" />
        <title>Hello from KinD</title>
      </head>
      <body>
        <h1>Hakuna Matata — your NGINX page is live 🎉</h1>
        <p>Served from a Kubernetes ConfigMap, running on KinD.</p>
      </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          volumeMounts:
            # Replace the default index.html with our ConfigMap's index.html
            - name: html
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: html
          configMap:
            name: nginx-html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply it:

```bash
kubectl apply -f nginx-cm-nodeport.yaml
kubectl get pods -l app=nginx -w
```

When the Pod is `Running`, check the Service:

```bash
kubectl get svc nginx-nodeport
```

---

## 6) Access the page from localhost

Because we mapped **NodePort 30080** to **host port 30080** in the KinD config, you can hit:

```bash
curl -i http://localhost:30080
```

Or open in a browser:

- http://localhost:30080

---

## 7) Quick troubleshooting spells (if something acts like Scar)

### `kind create cluster` can’t find Docker / can’t talk to the daemon
Make sure the Podman socket is enabled and `DOCKER_HOST` is set (rootless Podman).

```bash
systemctl --user status podman.socket
echo "$DOCKER_HOST"
```

### NodePort reachable from inside cluster but not from localhost
You must create the cluster **with the port mapping** (you can’t add `extraPortMappings` after the fact).
If needed:

```bash
kind delete cluster --name lab
kind create cluster --name lab --config kind-cluster.yaml
```

### Rootless requirements
KinD supports rootless Podman and expects cgroup v2 hosts for that mode.

---

## 8) Cleanup

```bash
kubectl delete -f nginx-cm-nodeport.yaml
kind delete cluster --name lab
```

---

## Next steps (optional)
If you want, your next “level up” after this is:
- `kubectl port-forward` (quick dev loop)
- Ingress (NGINX Ingress Controller) instead of NodePort
- building images locally and loading into KinD (`kind load docker-image ...`)
