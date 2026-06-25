# Installing Kubernetes on AWS with kops

**kops** (Kubernetes Operations) is the official tool for deploying production-grade
Kubernetes clusters on AWS. It provisions EC2 instances, networking, IAM roles,
autoscaling groups, and the full control plane automatically.

> kops manages the full lifecycle of a cluster — create, upgrade, scale, and delete.
> Think of it as `kubectl` but for entire clusters.

---

## Architecture Overview

```
AWS Account
├── S3 Bucket              ← kops stores cluster state here
├── Route53 Hosted Zone    ← DNS for the cluster (e.g. k8s.yourdomain.com)
├── VPC
│   ├── Control Plane (Master nodes)   ← EC2 instances managed by kops
│   └── Worker Nodes                   ← EC2 instances in Auto Scaling Groups
├── IAM Role               ← kops needs permissions to create all of the above
└── ELB                    ← API server load balancer (auto-created)
```

---

## Prerequisites

- AWS account with billing enabled
- A domain name (or subdomain) you control — kops uses Route53 for DNS
- AWS CLI installed and configured
- A local Linux/macOS/WSL machine to run kops from

---

## Step 1 — Install Required Tools

### Install AWS CLI

> Download the AWS CLI v2 installer

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

> Unzip the installer

```bash
unzip awscliv2.zip
```

> Run the installer

```bash
sudo ./aws/install
```

> Verify AWS CLI is installed

```bash
aws --version
```

### Install kubectl

> Download the latest kubectl binary

```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

> Make it executable and move to PATH

```bash
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

> Verify kubectl

```bash
kubectl version --client
```

### Install kops

> Download the latest kops binary

```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
```

> Make it executable and move to PATH

```bash
chmod +x kops && sudo mv kops /usr/local/bin/
```

> Verify kops

```bash
kops version
```

---

## Step 2 — Configure AWS CLI

> Configure AWS credentials — you'll be prompted for Access Key, Secret Key, region, and output format

```bash
aws configure
```

> Verify your identity is correct

```bash
aws sts get-caller-identity
```

---

## Step 3 — Create IAM User for kops

kops needs broad AWS permissions to provision infrastructure.

> Create a dedicated IAM group for kops

```bash
aws iam create-group --group-name kops
```

> Attach the required policies to the group

```bash
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess --group-name kops
```

> Create the kops IAM user

```bash
aws iam create-user --user-name kops
```

> Add the user to the kops group

```bash
aws iam add-user-to-group --user-name kops --group-name kops
```

> Create access keys for the kops user and save the output — you'll need these

```bash
aws iam create-access-key --user-name kops
```

> Export the kops user credentials as environment variables

```bash
export AWS_ACCESS_KEY_ID=<kops-user-access-key-id>
export AWS_SECRET_ACCESS_KEY=<kops-user-secret-access-key>
```

---

## Step 4 — Configure Route53 DNS

kops requires a DNS zone to assign hostnames to the cluster API and nodes.

### Option A — You own a domain (recommended for production)

> Create a hosted zone for your subdomain in Route53

```bash
aws route53 create-hosted-zone \
  --name k8s.yourdomain.com \
  --caller-reference $(date +%s)
```

> Get the NS records — add these to your domain registrar's DNS settings

```bash
aws route53 list-hosted-zones | grep -A 5 "k8s.yourdomain.com"
```

### Option B — Gossip-based DNS (no domain required, good for testing)

> kops supports gossip DNS — just name your cluster ending in `.k8s.local` and no Route53 setup is needed

```bash
export NAME=mycluster.k8s.local
```

> Skip to Step 5 if using gossip DNS.

---

## Step 5 — Create S3 Bucket for Cluster State

kops stores all cluster configuration and state in an S3 bucket.

> Create the S3 bucket — use a unique name

```bash
aws s3api create-bucket \
  --bucket my-kops-state-store \
  --region us-east-1
```

> Enable versioning on the bucket — allows rollback of cluster config changes

```bash
aws s3api put-bucket-versioning \
  --bucket my-kops-state-store \
  --versioning-configuration Status=Enabled
```

> Enable server-side encryption on the bucket

```bash
aws s3api put-bucket-encryption \
  --bucket my-kops-state-store \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

> Export the bucket name as an environment variable — kops reads this automatically

```bash
export KOPS_STATE_STORE=s3://my-kops-state-store
```

---

## Step 6 — Generate SSH Key Pair

kops uses SSH to access nodes for provisioning and debugging.

> Generate an SSH key pair (skip if you already have one at ~/.ssh/id_rsa)

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

---

## Step 7 — Create the Cluster

> Set your cluster name as an environment variable

```bash
export NAME=mycluster.k8s.yourdomain.com
```

> Create the cluster definition — this does NOT provision anything yet, it only writes config to S3

```bash
kops create cluster \
  --name=${NAME} \
  --state=${KOPS_STATE_STORE} \
  --cloud=aws \
  --zones=us-east-1a,us-east-1b,us-east-1c \
  --control-plane-count=3 \
  --control-plane-size=t3.medium \
  --node-count=3 \
  --node-size=t3.medium \
  --dns-zone=k8s.yourdomain.com \
  --ssh-public-key=~/.ssh/id_rsa.pub \
  --kubernetes-version=1.29.0
```

> Review the full cluster configuration before applying

```bash
kops edit cluster --name=${NAME}
```

> Edit the worker node instance group if needed

```bash
kops edit ig nodes --name=${NAME}
```

> Edit the control plane instance group if needed

```bash
kops edit ig control-plane-us-east-1a --name=${NAME}
```

---

## Step 8 — Apply and Build the Cluster

> Preview what kops will create before applying — dry run

```bash
kops update cluster --name=${NAME} --yes --dry-run
```

> Apply the cluster — this provisions all AWS resources (takes 5–10 minutes)

```bash
kops update cluster --name=${NAME} --yes
```

> Wait for the cluster to be fully ready — rerun until output shows "is ready"

```bash
kops validate cluster --name=${NAME} --wait 10m
```

> Verify all nodes are online

```bash
kubectl get nodes
```

---

## Step 9 — Verify the Cluster

> Check all system pods are running

```bash
kubectl get pods -n kube-system
```

> Check cluster info

```bash
kubectl cluster-info
```

> Deploy a test nginx pod to confirm the cluster works

```bash
kubectl create deployment nginx --image=nginx --replicas=2
```

> Expose it and get the external URL

```bash
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get svc nginx
```

> Clean up the test deployment

```bash
kubectl delete deployment nginx
kubectl delete svc nginx
```

---

## Cluster Management

### Upgrade Kubernetes version

> Edit the cluster config to change the Kubernetes version

```bash
kops edit cluster --name=${NAME}
```

> Apply the upgrade — rolling update replaces nodes one at a time

```bash
kops update cluster --name=${NAME} --yes
kops rolling-update cluster --name=${NAME} --yes
```

### Scale worker nodes

> Edit the worker node instance group

```bash
kops edit ig nodes --name=${NAME}
```

> Change `minSize` and `maxSize` values in the editor, then apply

```bash
kops update cluster --name=${NAME} --yes
kops rolling-update cluster --name=${NAME} --yes
```

### Export kubeconfig

> Export the kubeconfig for the cluster to use with kubectl

```bash
kops export kubeconfig --name=${NAME} --admin
```

---

## Common Commands

> List all clusters in your state store

```bash
kops get clusters
```

> Validate cluster health

```bash
kops validate cluster --name=${NAME}
```

> List instance groups

```bash
kops get ig --name=${NAME}
```

> Get cluster details

```bash
kops get cluster --name=${NAME} -o yaml
```

> SSH into a node

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<node-public-ip>
```

> Delete the cluster and all AWS resources

```bash
kops delete cluster --name=${NAME} --yes
```

---

## Persist Environment Variables

> Add these to your `~/.bashrc` or `~/.zshrc` so you don't need to export them every session

```bash
echo "export NAME=mycluster.k8s.yourdomain.com" >> ~/.bashrc
echo "export KOPS_STATE_STORE=s3://my-kops-state-store" >> ~/.bashrc
echo "export AWS_ACCESS_KEY_ID=<your-key>" >> ~/.bashrc
echo "export AWS_SECRET_ACCESS_KEY=<your-secret>" >> ~/.bashrc
source ~/.bashrc
```

---

## Estimated AWS Cost

| Resource | Type | Estimated Cost |
|----------|------|----------------|
| Control plane x3 | t3.medium | ~$90/month |
| Worker nodes x3 | t3.medium | ~$90/month |
| S3 state bucket | Standard | ~$1/month |
| ELB (API server) | Classic | ~$20/month |
| Route53 hosted zone | — | ~$0.50/month |
| **Total** | | **~$200/month** |

> Stop costs immediately by deleting the cluster when not in use — kops can recreate it from the S3 state in minutes.

---

## Troubleshooting

### kops validate shows nodes not ready

> Check if DNS has propagated — can take up to 5 minutes after cluster creation

```bash
nslookup api.mycluster.k8s.yourdomain.com
```

> Force revalidate after waiting

```bash
kops validate cluster --name=${NAME} --wait 10m
```

### kubectl cannot connect to cluster

> Re-export the kubeconfig

```bash
kops export kubeconfig --name=${NAME} --admin
```

> Verify the correct context is set

```bash
kubectl config current-context
```

### Nodes not joining the cluster

> Check the cluster is fully updated

```bash
kops update cluster --name=${NAME} --yes
```

> Trigger a rolling update to replace broken nodes

```bash
kops rolling-update cluster --name=${NAME} --yes
```

### AWS permissions error during cluster creation

> Verify the kops IAM user has all required policies attached

```bash
aws iam list-attached-user-policies --user-name kops
```

> Confirm you are using the kops user credentials, not your personal ones

```bash
aws sts get-caller-identity
```