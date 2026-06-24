# Installing Minikube on Ubuntu

Two environments covered:
- **WSL (Windows Subsystem for Linux)** — local development on Windows
- **AWS EC2 Ubuntu** — cloud-based Kubernetes playground or CI environment

---

## Table of Contents

- [Prerequisites — Both Environments](#prerequisites--both-environments)
- [Install on WSL Ubuntu](#install-on-wsl-ubuntu)
- [Install on AWS EC2 Ubuntu](#install-on-aws-ec2-ubuntu)
- [Post-Install — Both Environments](#post-install--both-environments)
- [Common Commands](#common-commands)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites — Both Environments

You need **kubectl** installed before using minikube.

>  Download the latest kubectl binary
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
> Make it executable and move to PATH
```bash
chmod +x kubectl
```
```bash
sudo mv kubectl /usr/local/bin/
```
> Verify
```bash
kubectl version --client
```

## Install on WSL Ubuntu

### 1. Check WSL Version (must be WSL 2)

Run this in **PowerShell on Windows**:

```powershell
wsl --list --verbose
```
> Look for VERSION 2 next to your Ubuntu distro
> If VERSION 1: 
```powershell
wsl --set-version Ubuntu 2
```

### 2. Install Docker in WSL (recommended driver) skip to stage 4 if you already have docker

Minikube on WSL works best with the **Docker driver** — no VM needed.

> Update packages
```bash
sudo apt update && sudo apt upgrade -y
```
> Install Docker dependencies
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```
> Add Docker's official GPG key
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
> Add Docker repository
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
> Install Docker Engine
```bash
sudo apt update
```
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
> Add your user to the docker group (avoids using sudo with docker)
```bash
sudo usermod -aG docker $USER
```
> Apply group change without logging out
```bash
newgrp docker
```
> Start Docker daemon (WSL doesn't use systemd by default on older versions)
```bash
sudo service docker start
```
> Verify Docker works
```bash
docker run hello-world
```

### 3. Enable systemd (Ubuntu 22.04+ on WSL 2)

> Check if systemd is enabled
```bash
cat /etc/wsl.conf
```
> If the file doesn't have systemd=true, add it:
```bash
sudo tee /etc/wsl.conf <<EOF
[boot]
systemd=true
EOF
```
> Restart WSL from PowerShell:
> Then reopen Ubuntu — Docker will now start automatically
```bash
wsl --shutdown
```

### 4. Install Minikube


> Download minikube binary
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```
> Install it
```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
> Clean up the downloaded file
```bash
rm minikube-linux-amd64
```
> Verify installation
```bash
minikube version
```

### 5. Start Minikube with Docker Driver

> Start minikube using Docker as the driver (no VM required in WSL)
```bash
minikube start --driver=docker
```
> Set Docker as the default driver permanently
```bash
minikube config set driver docker
```

> **Why Docker driver on WSL?**
> WSL 2 doesn't support VMs inside it (no KVM/VirtualBox), so Docker containers
> are used to simulate nodes. It's lightweight and works out of the box.

---

## Install on AWS EC2 Ubuntu

### Recommended EC2 Instance

| Setting | Value |
|---------|-------|
| AMI | Ubuntu 22.04 LTS |
| Instance type | `t3.medium` minimum (2 vCPU, 4GB RAM) |
| Storage | 20 GB+ |
| Security Group | Allow SSH (22), and optionally 8443 for kubectl API |

### 1. Connect and Update the Instance

# SSH into your instance
```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```
> Update all packages
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Docker

# Install Docker dependencies
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```
> Add Docker's GPG key
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
> Add Docker repo
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
> Install Docker
```bash
sudo apt update
```
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
> Add ubuntu user to docker group
```bash
sudo usermod -aG docker $USER
```
> Apply group change
```bash
newgrp docker
```
> Enable and start Docker with systemd
```bash
sudo systemctl enable docker
```
```bash
sudo systemctl start docker
```
> Verify
```bash
docker run hello-world
```

### 3. Install conntrack (required on EC2)

> conntrack is required by minikube on Linux bare-metal/VMs
```bash
sudo apt install -y conntrack
```

### 4. Install Minikube

> Download minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```
> Install
```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
> Clean up
```bash
rm minikube-linux-amd64
```
> Verify
```bash
minikube version
```

### 5. Start Minikube on EC2

> Start with Docker driver (no hypervisor needed — EC2 is already a VM)
```bash
minikube start --driver=docker
```
> If running as root (not recommended), add --force
> minikube start --driver=docker --force

> **Why Docker driver on EC2?**
> EC2 instances are already virtual machines — you can't nest another VM (KVM)
> inside them on most instance types. Docker driver runs Kubernetes inside
> containers instead, which works perfectly on EC2.

### 6. Access the Dashboard Remotely (optional)

> Start the dashboard — it binds to localhost by default
```bash
minikube dashboard --url
```
> To access it from your local machine, create an SSH tunnel:
> Run this on your LOCAL machine:
```bash
ssh -i your-key.pem -L 42000:localhost:42000 ubuntu@<EC2_PUBLIC_IP>
```
> Then open http://localhost:42000 in your browser


## Post-Install — Both Environments

### Verify the Cluster is Running

> Check minikube status
```bash
minikube status
```
> Should output:
> minikube: Running
> kubelet: Running
> apiserver: Running
> kubeconfig: Configured

> Check nodes with kubectl
```bash
kubectl get nodes
```
> Should show:
> NAME       STATUS   ROLES           AGE   VERSION
> minikube   Ready    control-plane   1m    v1.x.x

### Enable Useful Addons

> Enable ingress controller (nginx)
```bash
minikube addons enable ingress
```
> Enable metrics server (for kubectl top)
```bash
minikube addons enable metrics-server
```
> Enable Kubernetes dashboard
```bash
minikube addons enable dashboard
```
> List all available addons
```bash
minikube addons list
```

### Deploy a Test App

> Deploy a simple nginx app to verify everything works
```bash
kubectl create deployment nginx --image=nginx
```
> Expose it as a service
```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```
> Get the URL to access it
```bash
minikube service nginx --url
```
> Clean up
```bash
kubectl delete deployment nginx
```
```bash
kubectl delete service nginx
```

## Common Commands

> Start the cluster
```bash
minikube start
```
> Stop the cluster (preserves state)
```bash
minikube stop
```
> Delete the cluster entirely
```bash
minikube delete
```
> Pause cluster (frees CPU without deleting)
```bash
minikube pause
```
> Unpause
```bash
minikube unpause
```
> SSH into the minikube node
```bash
minikube ssh
```
> Point your local Docker CLI to minikube's Docker daemon
> (build images directly inside minikube — no push/pull needed)
```bash
eval $(minikube docker-env)
```
> Point back to local Docker
```bash
eval $(minikube docker-env --unset)
```
> Get minikube's IP
```bash
minikube ip
```
> Open the dashboard in browser
```bash
minikube dashboard
```

## Troubleshooting

### Docker permission denied

> If you get: permission denied while trying to connect to Docker socket
```bash
sudo usermod -aG docker $USER && newgrp docker
```

### Minikube fails to start — insufficient resources

> Set resource limits explicitly
```bash
minikube start --driver=docker --cpus=2 --memory=3900mb
```

### kubectl not connecting after restart

> Minikube updates kubeconfig automatically — just restart it
```bash
minikube stop
```
```bash
minikube start
```
> Verify kubeconfig points to minikube
> should output: minikube
```bash
kubectl config current-context 
```

### WSL — Docker daemon not running

> If systemd is not enabled, start Docker manually each session
> Or enable systemd (see Step 3 in WSL section) so it starts automatically
```bash
sudo service docker start
```

### EC2 — exited with error code 137 (out of memory)

```bash
> Upgrade your instance to t3.medium or larger
> Or increase swap:
```bah
sudo fallocate -l 4G /swapfile
```
```bash
sudo chmod 600 /swapfile
```
```bash
sudo mkswap /swapfile
```
```bash
sudo swapon /swapfile
```