# 3-Tier Application Deployment on AWS EKS

> A complete, reproducible walkthrough of deploying a **3-tier web application** (React frontend, Flask/Python backend, PostgreSQL database) on **Amazon EKS (Kubernetes 1.31)**, exposed to the internet through an **internet-facing AWS Application Load Balancer** provisioned by the **AWS Load Balancer Controller**, with the database on a **private Amazon RDS for PostgreSQL** instance.
>
> Every step below is backed by a real screenshot captured during this build, so you (or anyone) can follow it end-to-end and reproduce the exact result.

Adapted from the original reference project:
**[NotHarshhaa/DevOps-Projects — DevOps-Project-36](https://github.com/NotHarshhaa/DevOps-Projects/tree/master/DevOps-Project-36)**

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Prerequisites & Environment Setup](#3-prerequisites--environment-setup)
4. [Step-by-Step Deployment Guide](#4-step-by-step-deployment-guide)
   - [4.1 Create the EKS Cluster & Configure kubeconfig](#41-create-the-eks-cluster--configure-kubeconfig)
   - [4.2 Provision RDS PostgreSQL & Security Groups](#42-provision-rds-postgresql--security-groups)
   - [4.3 Namespace, ConfigMap, Secrets & DB Service](#43-namespace-configmap-secrets--db-service)
   - [4.4 Run the Database Migration Job](#44-run-the-database-migration-job)
   - [4.5 Deploy Backend & Frontend + Verify](#45-deploy-backend--frontend--verify)
   - [4.6 Install the AWS Load Balancer Controller & Ingress](#46-install-the-aws-load-balancer-controller--ingress)
   - [4.7 Access the Running Application via the ALB](#47-access-the-running-application-via-the-alb)
5. [Troubleshooting & Solved Issues](#5-troubleshooting--solved-issues)
6. [Resource Teardown & Cleanup](#6-resource-teardown--cleanup)

---

## 1. Project Overview

This project deploys a **DevOps Learning Platform** (a quiz app) as a 3-tier architecture on AWS EKS:

| Tier | Technology | Runs On |
|------|------------|---------|
| **Presentation (Frontend)** | React (served via Nginx) | EKS pods — `Deployment` + `ClusterIP` Service (port `80`) |
| **Application (Backend)** | Flask / Python | EKS pods — `Deployment` + `ClusterIP` Service (port `8000`) |
| **Data (Database)** | Amazon RDS for PostgreSQL 15 | Private DB subnet group (`db.t3.small`) |

Key platform components (exactly as built):

- **Compute:** EKS cluster `sushanka-cluster` (`us-east-1`, Kubernetes `v1.31`), managed node group `standard-workers` with 2× `t3.medium` nodes.
- **Ingress:** AWS Load Balancer Controller provisions an **internet-facing ALB** from a Kubernetes `Ingress`.
- **Database:** RDS PostgreSQL `sushanka-postgres` (`db.t3.small`) in the private DB subnet group `sushanka-postgres-private-subnet-group`. Pods reach it via a Kubernetes `ExternalName` service `postgres-db`, secured by a security group allowing port `5432` only from the EKS node SG.
- **Security & Secrets:** base64-encoded Kubernetes `Secret` for DB credentials/connection string, an **IAM OIDC provider** (IRSA) for the controller's pod identity, and a custom IAM policy for the ALB controller.

> **Scope note:** This guide intentionally **ends at exposing the app over the internet-facing ALB DNS name** and verifying it works in the browser. Custom domain routing, Route 53 / NS delegation, and TLS/ACM are **out of scope** and deliberately omitted.

---

## 2. Architecture

![3-Tier EKS Architecture](./screenshots/architecture-diagram.png)

**Traffic flow (right → left in the diagram):**

1. A user reaches the **internet-facing ALB** over the internet (via the Internet Gateway).
2. The ALB is created automatically by the **AWS Load Balancer Controller** in response to the `Ingress`. It sits in the **public subnets** (tagged `kubernetes.io/role/elb=1`).
3. The ALB routes by path:
   - `/api/*` → **backend** `ClusterIP` service (port `8000`)
   - `/*` → **frontend** `ClusterIP` service (port `80`)
4. **Backend pods** (on private-subnet worker nodes) resolve `postgres-db.3-tier-app-eks.svc.cluster.local` — a Kubernetes `ExternalName` service — which points to the **RDS PostgreSQL endpoint** in the private DB subnets.
5. RDS accepts connections on `5432` **only** from the EKS node security group.

**Networking summary:** public subnets host NAT gateways + the ALB; private subnets host the EKS worker nodes; private DB subnets host RDS with `--no-publicly-accessible`.

---

## 3. Prerequisites & Environment Setup

### Required tooling

| Tool | Purpose |
|------|---------|
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) | Interact with AWS APIs |
| [eksctl](https://eksctl.io/) | Create/manage the EKS cluster & IAM service accounts |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Manage Kubernetes resources |
| [Helm 3](https://helm.sh/docs/intro/install/) | Install the AWS Load Balancer Controller |
| [Docker](https://www.docker.com/) | (Optional) build/push container images |

### AWS account requirements

An IAM user/role with permissions for **EKS, EC2, RDS, IAM, CloudFormation, and ELB**, with credentials configured:

```bash
aws configure
aws sts get-caller-identity   # verify you are the expected account/identity
```

### Environment variables used throughout

```bash
export CLUSTER_NAME=sushanka-cluster
export AWS_REGION=us-east-1
export DB_IDENTIFIER=sushanka-postgres
export DB_SUBNET_GROUP=sushanka-postgres-private-subnet-group
```

> **Windows users:** this entire build was done in **Git Bash (MINGW64)**. Use Git Bash or WSL so `$VARIABLE` expansion, heredocs and JSON quoting behave as shown. PowerShell quoting differs — see [Troubleshooting](#5-troubleshooting--solved-issues).

---

## 4. Step-by-Step Deployment Guide

### 4.1 Create the EKS Cluster & Configure kubeconfig

Create the cluster with a managed node group (Kubernetes 1.31):

```bash
eksctl create cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --version 1.31 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

`eksctl` builds everything via CloudFormation (VPC, subnets, NAT gateways, control plane, node group). Expect **15–20 minutes**. You'll see the control-plane stack build first:

![eksctl creating the cluster control plane](./screenshots/01-eksctl-create-cluster.png)

When it finishes, the nodegroup is ready and the cluster is reported as created:

![eksctl cluster and nodegroup ready](./screenshots/02-eksctl-cluster-ready.png)

Confirm the cluster exists, then wire up `kubectl`:

```bash
aws eks list-clusters --region "$AWS_REGION"
```

![aws eks list-clusters shows sushanka-cluster](./screenshots/03-list-clusters.png)

```bash
aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
kubectl config current-context
kubectl get namespaces
```

![update-kubeconfig and current-context](./screenshots/04-update-kubeconfig.png)

Verify nodes are `Ready` and the system pods are running:

```bash
kubectl get node
kubectl get pod -A
```

![kubectl get node and get pod -A — nodes Ready](./screenshots/05-get-nodes-pods.png)

Both worker nodes show `Ready` on `v1.31`, and `kube-system` pods (`aws-node`, `coredns`, `kube-proxy`, `metrics-server`) are `Running`. The baseline cluster services look like this:

![kubectl get services -A](./screenshots/06-get-services-system.png)

---

### 4.2 Provision RDS PostgreSQL & Security Groups

#### 4.2.1 Discover the VPC and private subnets

```bash
export VPC_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)
echo "VPC_ID=$VPC_ID"

# Private subnets (no auto-assigned public IP) — these host RDS
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" \
  --output text --region "$AWS_REGION"
```

> **Important:** the DB subnet group must contain **only private subnets**. Subnets with `MapPublicIpOnLaunch=true` are public and must be excluded — see [Troubleshooting #2](#issue-2-rds-security-group--connectivity-failures).

#### 4.2.2 Create the private DB subnet group, then the RDS security group

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --db-subnet-group-description "Private subnet group for sushanka PostgreSQL RDS" \
  --subnet-ids subnet-xxxx subnet-yyyy subnet-zzzz \
  --region "$AWS_REGION"

aws ec2 create-security-group \
  --group-name postgressg \
  --description "SG for RDS" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"

export RDS_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=postgressg" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text --region "$AWS_REGION")
echo "RDS_SG_ID=$RDS_SG_ID"
```

The subnet group reaches `Complete` and the security group is created (note the returned `GroupId`):

![DB subnet group complete and RDS security group created](./screenshots/07-db-subnet-group-and-sg.png)

#### 4.2.3 Authorize EKS nodes → RDS on port 5432

This is the single most important rule — without it the migration/backend cannot reach the database.

```bash
export NODE_SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$RDS_SG_ID" \
  --protocol tcp --port 5432 \
  --source-group "$NODE_SG" \
  --region "$AWS_REGION"
```

The rule is created with `FromPort/ToPort 5432` and the node SG as the referenced source group:

![authorize-security-group-ingress on 5432 from the node SG](./screenshots/08-authorize-sg-ingress-5432.png)

#### 4.2.4 Create the PostgreSQL instance (private)

```bash
aws rds create-db-instance \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --db-instance-class db.t3.small \
  --engine postgres \
  --engine-version 15 \
  --allocated-storage 20 \
  --master-username postgresadmin \
  --master-user-password 'YourStrongPassword123!' \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --storage-type gp2 \
  --region "$AWS_REGION"
```

The instance starts in `creating` status, attached to the `postgressg` SG and the private subnet group:

![aws rds create-db-instance output](./screenshots/09-create-db-instance.png)

Wait for it to become available, then grab the endpoint:

```bash
aws rds wait db-instance-available --db-instance-identifier "$DB_IDENTIFIER" --region "$AWS_REGION"
aws rds describe-db-instances --db-instance-identifier "$DB_IDENTIFIER" --region "$AWS_REGION" \
  --query "DBInstances[0].Endpoint.Address" --output text
```

You can confirm it in the RDS console (status `Available`, `db.t3.small`, PostgreSQL):

![RDS console showing sushanka-postgres](./screenshots/10-rds-console.png)

> Save the endpoint (e.g. `sushanka-postgres.xxxxxxxx.us-east-1.rds.amazonaws.com`) — it goes into the `ExternalName` service and the `DATABASE_URL` secret.

---

### 4.3 Namespace, ConfigMap, Secrets & DB Service

All manifests live in [`3-tier-app-eks/k8s/`](./3-tier-app-eks/k8s). Change into that folder first:

```bash
cd 3-tier-app-eks/k8s
```

#### 4.3.1 Create the namespace

```bash
kubectl apply -f namespace.yaml   # creates namespace "3-tier-app-eks"
```

#### 4.3.2 ConfigMap (`configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: 3-tier-app-eks
data:
  DB_HOST: "postgres-db.3-tier-app-eks.svc.cluster.local"
  DB_NAME: "postgres"
  DB_PORT: "5432"
  FLASK_DEBUG: "0"
```

```bash
kubectl apply -f configmap.yaml
```

#### 4.3.3 Secrets (`secrets.yaml`)

Credentials and the full connection string are **base64-encoded**. Generate your own values:

```bash
echo -n 'postgresadmin' | base64
echo -n 'YourStrongPassword123!' | base64
echo -n 'postgresql://postgresadmin:YourStrongPassword123!@postgres-db.3-tier-app-eks.svc.cluster.local:5432/postgres' | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: 3-tier-app-eks
type: Opaque
data:
  DB_USERNAME: <base64-username>
  DB_PASSWORD: <base64-password>
  SECRET_KEY:  <base64-secret-key>
  DATABASE_URL: <base64-database-url>
```

```bash
kubectl apply -f secrets.yaml
```

#### 4.3.4 Database `ExternalName` service (`database-service.yaml`)

This maps an in-cluster DNS name to the RDS endpoint so pods never hardcode the RDS hostname. **Edit `externalName` to your RDS endpoint:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: 3-tier-app-eks
  labels:
    service: database
spec:
  type: ExternalName
  externalName: sushanka-postgres.xxxxxxxx.us-east-1.rds.amazonaws.com
  ports:
    - port: 5432
```

```bash
kubectl apply -f database-service.yaml
```

![kubectl apply -f database-service.yaml — service/postgres-db created](./screenshots/11-apply-database-service.png)

**Verify in-cluster DNS resolves the ExternalName to the RDS private IP** (proves the service and name resolution work before the app boots):

```bash
kubectl run -it --rm --restart=Never dns-test \
  --image=busybox:1.28 -n 3-tier-app-eks -- \
  nslookup postgres-db.3-tier-app-eks.svc.cluster.local
```

![nslookup resolves postgres-db to the RDS private IP](./screenshots/12-dns-test-nslookup.png)

The name resolves to a private `192.168.x.x` address inside the VPC — exactly what we want for a private DB.

---

### 4.4 Run the Database Migration Job

The migration `Job` (`migration_job.yaml`) runs `./migrate.sh` against RDS to create the schema and seed data **before** the app starts.

```bash
kubectl apply -f migration_job.yaml
kubectl get job -A
```

![migration job created and Running](./screenshots/13-migration-job-created.png)

Follow the logs — first it initializes the Alembic migrations directory:

```bash
kubectl logs -l job-name=database-migration -n 3-tier-app-eks
```

![migration logs — initializing migrations](./screenshots/14-migration-logs-init.png)

A successful run ends with the schema applied and seed data loaded:

![migration success — "Database setup completed successfully!"](./screenshots/15-migration-success.png)

> If the pod hangs or times out connecting to Postgres here, the SG rule from [4.2.3](#423-authorize-eks-nodes--rds-on-port-5432) is almost always the cause — see [Troubleshooting #2](#issue-2-rds-security-group--connectivity-failures).

---

### 4.5 Deploy Backend & Frontend + Verify

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
# optional autoscaling
kubectl apply -f hpa.yaml
```

![backend and frontend deployments + services created](./screenshots/16-apply-backend-frontend.png)

Check the rollout — both deployments should be `2/2`, all pods `Running`, and the `postgres-db` `ExternalName` service pointing at RDS:

```bash
kubectl get deployment -n 3-tier-app-eks
kubectl get svc        -n 3-tier-app-eks
kubectl get po         -n 3-tier-app-eks
```

![deployments 2/2, pods Running, postgres-db ExternalName](./screenshots/17-get-deploy-svc-pods.png)

> The backend `Deployment` uses an **init container** (`wait-for-db`) that blocks until `postgres-db.3-tier-app-eks.svc.cluster.local` resolves — guaranteeing the DB service exists before the app boots.

**Verify the app locally (before exposing it publicly) with port-forwarding:**

```bash
# Backend API
kubectl port-forward -n 3-tier-app-eks svc/backend 8000:8000
# in another terminal:
curl http://localhost:8000/api/topics
```

The API returns the seeded topics (Docker, Kubernetes, Jenkins) — proving the backend ↔ RDS path works end-to-end:

![/api/topics returns seeded data](./screenshots/18-api-topics-response.png)

```bash
# Frontend
kubectl port-forward -n 3-tier-app-eks svc/frontend 8080:80
# then open http://localhost:8080
```

---

### 4.6 Install the AWS Load Balancer Controller & Ingress

The controller watches `Ingress` objects and provisions a real AWS ALB. It needs an **IAM OIDC provider** (for IRSA) and an **IAM policy**.

#### 4.6.1 Associate the IAM OIDC provider

```bash
oidc_id=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo "$oidc_id"

# Create only if it doesn't already exist
aws iam list-open-id-connect-providers | grep "$oidc_id" \
  || eksctl utils associate-iam-oidc-provider --cluster "$CLUSTER_NAME" --approve --region "$AWS_REGION"
```

![eksctl associate-iam-oidc-provider — created](./screenshots/19-oidc-associate.png)

Verify the provider now exists for the cluster:

![OIDC provider verified via list-open-id-connect-providers](./screenshots/20-oidc-verify.png)

#### 4.6.2 Create the IAM policy and the IRSA service account

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster="$CLUSTER_NAME" \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn="arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy" \
  --region "$AWS_REGION" \
  --approve
```

> A copy of this policy is committed at [`k8s/iam_policy.json`](./3-tier-app-eks/k8s/iam_policy.json).

#### 4.6.3 Tag public subnets for ALB discovery

The controller auto-discovers public subnets by the `kubernetes.io/role/elb=1` tag. Without it, the ALB **cannot provision** — see [Troubleshooting #3](#issue-3-aws-load-balancer-controller--subnet-tagging-issues).

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=map-public-ip-on-launch,Values=true" \
  --query "Subnets[*].{SubnetId:SubnetId,AvailabilityZone:AvailabilityZone,PublicIp:MapPublicIpOnLaunch}"
```

![public subnets query (map-public-ip-on-launch=true)](./screenshots/21-public-subnets-query.png)

```bash
# Tag the public subnet IDs returned above
aws ec2 create-tags \
  --resources subnet-xxxx subnet-yyyy \
  --tags Key=kubernetes.io/role/elb,Value=1 \
  --region "$AWS_REGION"
```

#### 4.6.4 Install the controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID"
```

> `--set serviceAccount.create=false` is **required** — `eksctl` already created the IRSA-annotated service account. Letting Helm create its own would strip the IAM role binding.

#### 4.6.5 Apply the Ingress

`ingress.yaml` requests an **internet-facing, IP-target** ALB and routes `/api` → backend, `/` → frontend:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: 3-tier-app-ingress
  namespace: 3-tier-app-eks
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: backend, port: { number: 8000 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: frontend, port: { number: 80 } }
```

```bash
kubectl apply -f ingress.yaml
```

Confirm the controller pods are `Running` and the ALB is reported as **internet-facing / active** by `elbv2`:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl get ingress -n 3-tier-app-eks

aws elbv2 describe-load-balancers --region "$AWS_REGION" \
  --query "LoadBalancers[?DNSName=='<your-alb-dns>'].{State:State.Code,Scheme:Scheme,VPC:VpcId}" \
  --output table
```

![ALB controller pods running + elbv2 shows internet-facing/active](./screenshots/22-alb-controller-pods-elbv2.png)

Inspect the Ingress — it gets an ALB `Address`, the `/api`→backend and `/`→frontend rules are wired to pod IPs, and the event log shows `SuccessfullyReconciled`:

```bash
kubectl describe ingress 3-tier-app-ingress -n 3-tier-app-eks
```

![kubectl describe ingress — ALB address, rules, SuccessfullyReconciled](./screenshots/23-ingress-describe.png)

---

### 4.7 Access the Running Application via the ALB

Grab the ALB DNS name:

```bash
export ALB_DNS=$(kubectl get ingress 3-tier-app-ingress -n 3-tier-app-eks \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "App URL: http://$ALB_DNS"

curl -I "http://$ALB_DNS/"
curl "http://$ALB_DNS/api/topics"
```

In the EC2 → Load Balancers console the ALB is **Active** and **Internet-facing**:

![ALB active and internet-facing in the console](./screenshots/24-alb-console-active.png)

Open `http://<ALB_DNS>` in a browser — the React frontend loads and renders the topics it fetched from the backend `/api`, which in turn queried RDS. **The full 3-tier stack is live:**

![DevOps Learning Platform home page served via the ALB](./screenshots/25-app-home-browser.png)

Navigating into a quiz confirms the backend API is serving dynamic data (questions pulled from PostgreSQL):

![Kubernetes quiz page served via the ALB](./screenshots/26-app-quiz-browser.png)

> **That's the end of the deployment scope.** The app is reachable on the internet via the ALB DNS name. (Custom domains, Route 53 / NS delegation, and TLS are intentionally not covered.)

---

## 5. Troubleshooting & Solved Issues

These are the real issues hit during this build, with the exact fixes applied.

### Issue 1: Shell Quoting & Variable Expansion Errors (Git Bash / Windows)

**Symptom:** Passing JSON payloads (e.g. inline `--change-batch` / `--tags` JSON) to the AWS CLI from **Git Bash / MINGW64** produced `parse error` / `Invalid JSON`, or a literal string like `'$ALB_DNS'` was sent instead of its value.

**Root cause:** In Bash, single quotes (`'...'`) prevent `$VAR` expansion, and unescaped double quotes inside a double-quoted string terminate it early. Git Bash additionally rewrites some paths/quote characters.

**Fix:** use **escaped double quotes** (`\"`) inside a double-quoted shell string so variables expand, or avoid inline JSON entirely with a file:

```bash
# BAD — single quotes, $ALB_DNS NOT expanded → literal '$ALB_DNS'
--change-batch '{ "DNSName": "'$ALB_DNS'" }'

# GOOD — escaped double quotes, variable expands
--change-batch "{ \"DNSName\": \"$ALB_DNS\" }"

# BEST — no inline JSON at all
cat > payload.json <<EOF
{ "DNSName": "$ALB_DNS" }
EOF
aws ... --change-batch file://payload.json
```

For the subnet query, escape the backticks too:

```bash
--query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId"
```

---

### Issue 2: RDS Security Group & Connectivity Failures

**Symptom:** The `database-migration` pod hangs on `wait-for-db` or times out connecting to PostgreSQL (`could not connect to server: Connection timed out`), even though DNS resolves.

**Root cause(s):**
1. The RDS security group had **no inbound rule** allowing the EKS worker nodes on `5432`.
2. The DB subnet group accidentally included **public subnets** (`MapPublicIpOnLaunch=true`).

**Fix — (1) authorize the node SG → RDS SG on 5432:**

```bash
export NODE_SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$RDS_SG_ID" \
  --protocol tcp --port 5432 \
  --source-group "$NODE_SG" \
  --region "$AWS_REGION"
```

**Fix — (2) rebuild the DB subnet group using only private subnets:**

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" --output text --region "$AWS_REGION"
```

**Verify** from inside the cluster (should resolve to a private IP and connect):

```bash
kubectl run -it --rm --restart=Never dns-test --image=busybox:1.28 -n 3-tier-app-eks -- \
  nslookup postgres-db.3-tier-app-eks.svc.cluster.local

kubectl run debug-pod --rm -it --image=postgres -n 3-tier-app-eks -- bash
# inside the pod:
PGPASSWORD=YourStrongPassword123! psql \
  -h postgres-db.3-tier-app-eks.svc.cluster.local -U postgresadmin -d postgres -c '\l'
```

---

### Issue 3: AWS Load Balancer Controller & Subnet Tagging Issues

**Symptom:** The `Ingress` is created but **no ALB appears** (address stays empty), or the controller logs show `failed to discover subnets` / `AccessDenied` / `could not find any subnets for ALB`.

**Root cause(s):**
1. No **IAM OIDC provider** associated → the controller's service account has no usable IAM role (IRSA broken).
2. Public subnets were **not tagged** `kubernetes.io/role/elb=1`, so the controller couldn't find where to place an internet-facing ALB.
3. Helm created its own service account (overwriting the IRSA-annotated one), or `vpcId` wasn't passed.

**Fix — (a) create the OIDC provider:**

```bash
eksctl utils associate-iam-oidc-provider --cluster "$CLUSTER_NAME" --approve --region "$AWS_REGION"
```

**Fix — (b) tag the public subnets:**

```bash
aws ec2 create-tags --resources subnet-xxxx subnet-yyyy \
  --tags Key=kubernetes.io/role/elb,Value=1 --region "$AWS_REGION"
```

**Fix — (c) install reusing the eksctl service account and passing the VPC explicitly:**

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID"
```

Then force a clean re-reconcile if the Ingress was stuck:

```bash
kubectl delete ingress 3-tier-app-ingress -n 3-tier-app-eks --grace-period=0 --force
kubectl apply -f ingress.yaml
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -f
```

---

### Issue 4: Local DNS Flush Command Syntax on Windows (Git Bash)

**Symptom:** `ipconfig /flushdns` inside **Git Bash (MINGW64)** fails because MINGW rewrites `/flushdns` into a Windows path (`C:/...`).

**Fix:** use a **double slash** to escape MINGW path translation, or shell out to `cmd.exe`:

```bash
ipconfig //flushdns                 # double-slash escapes MINGW path mangling
cmd.exe /c "ipconfig /flushdns"     # or run through cmd.exe explicitly
```

Useful when re-testing the ALB DNS name locally after the load balancer finishes provisioning.

---

## 6. Resource Teardown & Cleanup

Delete in reverse order of creation to avoid dependency errors (e.g. ENIs held by the ALB blocking VPC deletion). Run from `3-tier-app-eks/k8s`.

### Step 1 — Delete the Ingress (removes the ALB)

```bash
kubectl delete -f ingress.yaml
# wait until the ALB disappears from EC2 -> Load Balancers before deleting the cluster
```

### Step 2 — Delete the application workloads & config

```bash
kubectl delete -f hpa.yaml --ignore-not-found
kubectl delete -f frontend.yaml
kubectl delete -f backend.yaml
kubectl delete -f migration_job.yaml --ignore-not-found
kubectl delete -f database-service.yaml
kubectl delete -f secrets.yaml
kubectl delete -f configmap.yaml
kubectl delete -f namespace.yaml
```

### Step 3 — Uninstall the AWS Load Balancer Controller (Helm)

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

### Step 4 — Delete the IAM service account & policy

```bash
eksctl delete iamserviceaccount \
  --cluster "$CLUSTER_NAME" --namespace kube-system \
  --name aws-load-balancer-controller --region "$AWS_REGION"

export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam delete-policy \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"
```

### Step 5 — Delete the RDS instance

```bash
aws rds delete-db-instance \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --skip-final-snapshot --delete-automated-backups \
  --region "$AWS_REGION"

aws rds wait db-instance-deleted --db-instance-identifier "$DB_IDENTIFIER" --region "$AWS_REGION"
```

### Step 6 — Delete the DB subnet group & RDS security group

```bash
aws rds delete-db-subnet-group --db-subnet-group-name "$DB_SUBNET_GROUP" --region "$AWS_REGION"
aws ec2 delete-security-group --group-id "$RDS_SG_ID" --region "$AWS_REGION"
```

### Step 7 — Delete the EKS cluster

Tears down the node group, VPC, NAT gateways, and the `eksctl` CloudFormation stacks:

```bash
eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
```

### Step 8 — (Optional) Remove the IAM OIDC provider

```bash
oidc_id=$(aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn, '$CLUSTER_NAME')].Arn" --output text)
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn "$oidc_id"
```

### Final verification

```bash
aws eks list-clusters --region "$AWS_REGION"
aws rds describe-db-instances --region "$AWS_REGION" --query "DBInstances[].DBInstanceIdentifier"
aws elbv2 describe-load-balancers --region "$AWS_REGION" --query "LoadBalancers[].LoadBalancerName"
```

All three should return empty (or no longer list this project's resources), confirming a clean teardown.

---

## Repository Structure

```
3-tier-app-eks/
├── backend/            # Flask/Python API (Dockerfile, migrate.sh, seed_data.py)
├── frontend/           # React app (Dockerfile, nginx.conf)
├── infra/              # Terraform (optional IaC: eks.tf, rds.tf, network.tf, oidc.tf)
├── k8s/                # Kubernetes manifests
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── database-service.yaml   # ExternalName -> RDS
│   ├── migration_job.yaml
│   ├── backend.yaml
│   ├── frontend.yaml
│   ├── ingress.yaml            # internet-facing ALB
│   ├── hpa.yaml
│   └── iam_policy.json         # ALB controller IAM policy
├── docker-compose.yml  # Local dev (frontend + backend + postgres)
└── README.md
screenshots/            # Build evidence referenced throughout this README
```

---

## Credits

Adapted from **[NotHarshhaa/DevOps-Projects — DevOps-Project-36](https://github.com/NotHarshhaa/DevOps-Projects/tree/master/DevOps-Project-36)**. Deployment naming (`sushanka-cluster`, `sushanka-postgres`) reflects this specific build.
