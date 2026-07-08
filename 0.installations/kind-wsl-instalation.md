# Installing KIND on WSL Ubuntu

**KIND** (Kubernetes IN Docker) runs Kubernetes clusters inside Docker containers.
Each "node" is a Docker container — making it perfect for simulating **multi-node clusters**
on your local machine without needing extra VMs or cloud resources.

> Use KIND when you need to practice multi-node scenarios, node failures,
> pod scheduling across nodes, or test cluster-level configs — things Minikube can't do.

---

## Prerequisites

You need **Docker** and **kubectl** already installed and working.

> Verify Docker is running

```bash
docker info
```

> Verify kubectl is installed

```bash
kubectl version --client
```

> If either is missing, follow the Minikube WSL README first — it covers Docker and kubectl installation in detail.

---

## Install KIND

> Download the latest KIND binary for Linux AMD64

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
```

> Make it executable

```bash
chmod +x ./kind
```

> Move it to your PATH

```bash
sudo mv ./kind /usr/local/bin/kind
```

> Verify installation

```bash
kind version
```

---

## Single-Node Cluster (Quick Start)

> Create a cluster with one control-plane node

```bash
kind create cluster --name my-cluster
```

> Check the cluster is running

```bash
kubectl cluster-info --context kind-my-cluster
```

> See the node

```bash
kubectl get nodes
```

> See it running as a Docker container

```bash
docker ps
```

---

## Multi-Node Cluster (The Real Power of KIND)

This is what separates KIND from Minikube — you can simulate a real cluster with multiple workers.

### 1. Create the cluster config file

> Save this as `kind-multi-node.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

### 2. Create the cluster from the config

> Create the multi-node cluster

```bash
kind create cluster --name prod-sim --config kind-multi-node.yaml
```

> Verify all nodes are ready — expected output shows 1 control-plane + 3 workers

```bash
kubectl get nodes
```

---

## High Availability Cluster (Multiple Control Planes)

Simulates a production HA setup with 3 control-plane nodes and 3 workers.

> Save this as `kind-ha-cluster.yaml` — quorum requires an odd number of control-plane nodes

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

> Create the HA cluster

```bash
kind create cluster --name ha-cluster --config kind-ha-cluster.yaml
```

> Verify — you should see 3 control-plane nodes and 3 workers

```bash
kubectl get nodes
```

---

## Expose Services Locally (Port Mapping)

By default, KIND clusters are not accessible from your browser.
Add port mappings to the control-plane node to reach services.

> Save this as `kind-with-ports.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
      - containerPort: 30443
        hostPort: 8443
        protocol: TCP
  - role: worker
  - role: worker
```

> Create the cluster with port mappings — services on NodePort 30080 are reachable at http://localhost:8080

```bash
kind create cluster --name dev --config kind-with-ports.yaml
```

---

## Install Ingress Controller (nginx)

KIND doesn't come with an ingress controller — install one manually.

> Apply the ingress-nginx manifest built for KIND

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

> Wait for the ingress controller pod to be ready

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

> Verify the ingress controller is running

```bash
kubectl get pods -n ingress-nginx
```

> For ingress to work, your cluster config must have port mappings with containerPort 80/443 mapped to hostPort 80/443.

---

## Load a Local Docker Image into KIND

KIND nodes are isolated containers — they can't pull from your local Docker daemon directly.
You need to explicitly load images into the cluster.

> Build your image locally

```bash
docker build -t my-app:latest .
```

> Load the image into the KIND cluster

```bash
kind load docker-image my-app:latest --name my-cluster
```

> Deploy using the loaded image — no registry needed

```bash
kubectl create deployment my-app --image=my-app:latest
```

---

## Managing Multiple Clusters

> List all KIND clusters

```bash
kind get clusters
```

> Switch kubectl context to a specific cluster

```bash
kubectl config use-context kind-my-cluster
```

> List all available contexts

```bash
kubectl config get-contexts
```

> Delete a specific cluster

```bash
kind delete cluster --name my-cluster
```

> Delete all clusters

```bash
kind delete clusters --all
```

---

## Common Commands

> Create a cluster

```bash
kind create cluster --name <name>
```

> Create from a config file

```bash
kind create cluster --name <name> --config <file.yaml>
```

> Delete a cluster

```bash
kind delete cluster --name <name>
```

> Get nodes in a cluster

```bash
kind get nodes --name <name>
```

> Export cluster logs for debugging

```bash
kind export logs --name <name> ./kind-logs
```

> Load a Docker image into the cluster

```bash
kind load docker-image <image:tag> --name <name>
```

---

## KIND vs Minikube — When to Use Which

| Feature | Minikube | KIND |
|---------|----------|------|
| Single-node cluster | ✅ | ✅ |
| Multi-node cluster | ❌ | ✅ |
| HA control-plane | ❌ | ✅ |
| Dashboard built-in | ✅ | ❌ (install manually) |
| Addons (ingress, metrics) | ✅ one command | ❌ manual install |
| Load local images | ✅ eval trick | ✅ kind load |
| Speed | Slower start | Fast |
| Best for | Beginners, quick tests | Multi-node, CI, advanced |

> **Recommended workflow:** Use **Minikube** to learn the basics.
> Switch to **KIND** when you need multi-node behavior or want to simulate
> closer to a real production cluster.

---

## Troubleshooting

### Cluster creation fails — Docker not running

> Start Docker without systemd

```bash
sudo service docker start
```

> Start Docker with systemd

```bash
sudo systemctl start docker
```

> Retry cluster creation

```bash
kind create cluster --name my-cluster
```

---

### kubectl commands go to wrong cluster

> Check which cluster is currently active

```bash
kubectl config current-context
```

> Switch to the correct KIND cluster

```bash
kubectl config use-context kind-<cluster-name>
```

---

### Nodes stuck in NotReady

> Watch node status live — KIND pulls images on first run, give it a minute

```bash
kubectl get nodes -w
```

> If still stuck, check the node container logs

```bash
docker logs <node-container-name>
```

> Get the container name

```bash
docker ps
```

---

### WSL — not enough memory

> KIND with 3+ nodes needs at least 8GB RAM available to WSL.
> Create or edit `%USERPROFILE%\.wslconfig` on Windows with the following content:

```ini
[wsl2]
memory=8GB
processors=4
```

> Restart WSL from PowerShell to apply

```powershell
wsl --shutdown
```