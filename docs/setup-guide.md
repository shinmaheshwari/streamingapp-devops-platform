# SETUP.md — Full Setup Guide

Complete, detailed walkthrough of everything needed to stand up this project from scratch, based on what was actually run and debugged end-to-end.

**Account/region used:** AWS Account `562904760755`, region `ap-south-1` (Mumbai)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Git — Fork & Clone](#2-git--fork--clone)
3. [Application Setup](#3-application-setup)
4. [Docker — Dockerfiles & Local Testing](#4-docker--dockerfiles--local-testing)
5. [AWS CLI & ECR](#5-aws-cli--ecr)
6. [Jenkins — Install & Configure](#6-jenkins--install--configure)
7. [Jenkins — Credentials & Plugins](#7-jenkins--credentials--plugins)
8. [Jenkinsfile — CI/CD Pipeline](#8-jenkinsfile--cicd-pipeline)
9. [Kubernetes — EKS Cluster](#9-kubernetes--eks-cluster)
10. [Helm — Deployment](#10-helm--deployment)
11. [Monitoring — CloudWatch](#11-monitoring--cloudwatch)
12. [ChatOps — SNS + Slack](#12-chatops--sns--slack)
13. [Validation & Testing](#13-validation--testing)
14. [Jenkins Maintenance — Disk Space](#14-jenkins-maintenance--disk-space)
15. [Teardown / Cost Cleanup](#15-teardown--cost-cleanup)
16. [Full Troubleshooting Log](#16-full-troubleshooting-log)

---

## 1. Prerequisites

- AWS account with CLI access
- Docker Desktop (with Buildx support — included by default in recent versions)
- `kubectl`, `eksctl`, `helm` (installed in later steps)
- A GitHub account, with the repo forked
- An EC2 instance for Jenkins (`t3.medium` recommended minimum — `t3.micro`/`t2.micro` will hit memory and disk issues under Docker build load)
- A MongoDB Atlas cluster with a database user

---

## 2. Git — Fork & Clone

```bash
git clone https://github.com/shinmaheshwari/streamingapp-devops-platform.git
cd streamingapp-devops-platform

git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
git remote -v
```

To pull updates from the original repo into your fork:
```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## 3. Application Setup

Actual repo structure:
```
backend/helloService/    (port 3001)
backend/profileService/  (port 3002, MongoDB)
frontend/                (React)
```

Create local `.env` files (never committed — add to `.gitignore`):

`backend/helloService/.env`
```
PORT=3001
```

`backend/profileService/.env`
```
PORT=3002
MONGO_URL="mongodb+srv://<user>:<password>@streaming-cluster.kzpnixw.mongodb.net/streamingdb?appName=streaming-cluster"
```

> **Note on the Mongo password:** avoid special characters like `@ : / ? # %` in the password — they break the connection string parser unless URL-encoded. Use a plain alphanumeric password to sidestep this entirely.

Install and test locally:
```bash
cd backend/helloService && npm install && npm start &
cd backend/profileService && npm install && npm start &
cd frontend && npm install && npm start
```

---

## 4. Docker — Dockerfiles & Local Testing

**`backend/helloService/Dockerfile`**
```dockerfile
FROM node:18-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .

FROM node:18-slim
WORKDIR /app
COPY --from=build /app .
EXPOSE 3001
CMD ["node", "index.js"]
```

**`backend/profileService/Dockerfile`** — same pattern, port `3002`.

> **Why `node:18-slim` instead of `node:18-alpine`:** Alpine's musl/OpenSSL build has a known TLS handshake incompatibility with MongoDB Atlas (`SSL alert number 80` errors). `slim` avoids this.

**`frontend/Dockerfile`**
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Build and test locally** (from repo root):
```bash
docker build -t streamingapp-helloservice:latest ./backend/helloService
docker build -t streamingapp-profileservice:latest ./backend/profileService
docker build -t streamingapp-frontend:latest ./frontend

docker run -d -p 3001:3001 --name hello-test streamingapp-helloservice:latest
docker run -d -p 3002:3002 -e MONGO_URL='<your-connection-string>' --name profile-test streamingapp-profileservice:latest
docker run -d -p 8080:80 --name frontend-test streamingapp-frontend:latest

docker ps
docker logs profile-test    # confirm clean Mongo connection, no errors
```

Visit `http://localhost:8080` to confirm the frontend loads.

---

## 5. AWS CLI & ECR

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

aws configure
# Region: ap-south-1
aws sts get-caller-identity
```

Create ECR repositories:
```bash
aws ecr create-repository --repository-name streamingapp-helloservice --region ap-south-1
aws ecr create-repository --repository-name streamingapp-profileservice --region ap-south-1
aws ecr create-repository --repository-name streamingapp-frontend --region ap-south-1
```

Authenticate and push (first manual push, to confirm access works):
```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 562904760755.dkr.ecr.ap-south-1.amazonaws.com
```

**Build for the correct architecture.** EKS worker nodes run `linux/amd64`; if you're building on Apple Silicon (or any ARM machine), you must cross-build, and disable Buildx's default attestation/provenance wrapping (which some containerd versions can't resolve):

```bash
docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
  -t 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-helloservice:latest \
  --push ./backend/helloService

docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
  -t 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-profileservice:latest \
  --push ./backend/profileService

docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
  -t 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-frontend:latest \
  --push ./frontend
```

---

## 6. Jenkins — Install & Configure

EC2 instance: Ubuntu, `t3.medium`, security group open on ports `22` and `8080`.

> **Note:** the standard `apt install jenkins` route hit unresolved GPG signature verification errors on this Ubuntu version. The working approach was installing Jenkins directly via its `.war` file, run as a proper `systemd` service.

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk   # Jenkins requires Java 21+
sudo update-alternatives --config java   # select java-21 if multiple versions present

sudo mkdir -p /opt/jenkins
curl -fsSL -o /opt/jenkins/jenkins.war https://get.jenkins.io/war-stable/latest/jenkins.war

sudo useradd -r -m -d /var/lib/jenkins -s /bin/false jenkins 2>/dev/null || true
sudo mkdir -p /var/lib/jenkins
sudo chown -R jenkins:jenkins /var/lib/jenkins /opt/jenkins
```

Create the systemd service:
```bash
sudo tee /etc/systemd/system/jenkins.service > /dev/null << 'EOF'
[Unit]
Description=Jenkins Continuous Integration Server
After=network.target

[Service]
Type=simple
User=jenkins
Environment="JENKINS_HOME=/var/lib/jenkins"
ExecStart=/usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar /opt/jenkins/jenkins.war --httpPort=8080
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Get the initial admin password and complete setup at `http://<ec2-ip>:8080`:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install Docker, AWS CLI, `kubectl`, and Helm on this same box (the pipeline builds/deploys directly from here):
```bash
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart jenkins

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Confirm the `jenkins` user specifically (not just `ubuntu`) can use all of these, since that's who runs pipeline steps:
```bash
sudo -u jenkins docker --version
sudo -u jenkins aws --version
sudo -u jenkins kubectl version --client
sudo -u jenkins helm version
```

**If you attach an Elastic IP** (recommended — otherwise the public IP changes every stop/start):
- EC2 Console → Elastic IPs → Allocate → Associate with this instance

---

## 7. Jenkins — Credentials & Plugins

**Plugins** (Manage Jenkins → Plugins → Available):
- Docker Pipeline
- Amazon ECR
- Pipeline: AWS Steps
- Git
- GitHub Integration / Webhook

**Credentials** (Manage Jenkins → Credentials → System → Global credentials):

| ID | Kind | Value |
|---|---|---|
| `aws-creds` | AWS Credentials | Access Key + Secret Key |
| `github-creds` | SSH Username with private key | Username: `git`, private key generated below |
| `mongo-url` | Secret text | Mongo connection string |

**Generate an SSH key for GitHub access** (as the `jenkins` user):
```bash
sudo -u jenkins ssh-keygen -t ed25519 -C "jenkins-streamingapp" -f /var/lib/jenkins/.ssh/id_ed25519 -N ""
sudo cat /var/lib/jenkins/.ssh/id_ed25519.pub
```
Add the public key to GitHub → Settings → SSH and GPG keys. Add the private key content to the `github-creds` Jenkins credential.

Test:
```bash
sudo -u jenkins ssh -T git@github.com
```

**GitHub webhook** (repo → Settings → Webhooks):
- Payload URL: `http://<jenkins-ip>:8080/github-webhook/`
- Content type: `application/json`
- Trigger: just the push event

**Give the `jenkins` user EKS access** (after the cluster exists — see Section 9):
```bash
sudo -u jenkins aws eks update-kubeconfig --name streamingapp-cluster --region ap-south-1
sudo -u jenkins kubectl get nodes
```

---

## 8. Jenkinsfile — CI/CD Pipeline

The full `Jenkinsfile` lives at the repo root. Key points:
- `Cleanup Old Images` stage runs first, pruning Docker images older than 24h to prevent disk space issues
- `Checkout` uses `retry(conditions: [nonresumable()], count: 2)` to survive Jenkins restarts mid-build
- `Build & Push Images` uses `docker buildx build --platform linux/amd64 --provenance=false --sbom=false --push`, tagging both the build number and `latest`
- `Deploy to EKS` runs `helm upgrade --install` with the new image tags
- `post { always }` prunes dangling images after every run
- `post { success/failure }` publishes to SNS in AWS Chatbot's required JSON schema (see Section 12)

See the actual [Jenkinsfile](../Jenkinsfile) for the complete, current version.

---

## 9. Kubernetes — EKS Cluster

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

eksctl create cluster \
  --name streamingapp-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

aws eks update-kubeconfig --name streamingapp-cluster --region ap-south-1
kubectl get nodes
```

Attach required IAM policies to the node group's role:
```bash
aws eks describe-nodegroup --cluster-name streamingapp-cluster --nodegroup-name standard-workers --query "nodegroup.nodeRole" --region ap-south-1

aws iam attach-role-policy \
  --role-name <role-name-from-above> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```
(`AmazonEC2ContainerRegistryReadOnly` is attached by default via `eksctl`.)

---

## 10. Helm — Deployment

```bash
cd streamingapp-devops-platform
helm create streamingapp-chart

# Remove unused scaffold templates that reference undefined values
rm -f streamingapp-chart/templates/hpa.yaml \
      streamingapp-chart/templates/ingress.yaml \
      streamingapp-chart/templates/httproute.yaml \
      streamingapp-chart/templates/serviceaccount.yaml \
      streamingapp-chart/templates/deployment.yaml \
      streamingapp-chart/templates/service.yaml
rm -rf streamingapp-chart/templates/tests
```

`streamingapp-chart/values.yaml`:
```yaml
helloService:
  image: 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-helloservice
  tag: latest
  port: 3001
  replicas: 2

profileService:
  image: 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-profileservice
  tag: latest
  port: 3002
  replicas: 2

frontend:
  image: 562904760755.dkr.ecr.ap-south-1.amazonaws.com/streamingapp-frontend
  tag: latest
  port: 80
  replicas: 2

service:
  type: LoadBalancer
```

Create `templates/hello-deployment.yaml`, `templates/profile-deployment.yaml`, `templates/frontend-deployment.yaml` — each with a Deployment + Service pair (see the chart directory for full content). `profile-deployment.yaml` reads `MONGO_URL` from a Secret, never a literal value:

```bash
kubectl create secret generic mongo-secret \
  --from-literal=MONGO_URL='<your-mongo-connection-string>'
```

Deploy:
```bash
helm install streamingapp ./streamingapp-chart
kubectl get pods
kubectl get svc
```

Update on new builds:
```bash
helm upgrade streamingapp ./streamingapp-chart \
  --set helloService.tag=<build-number> \
  --set profileService.tag=<build-number> \
  --set frontend.tag=<build-number>
```

---

## 11. Monitoring — CloudWatch

```bash
ClusterName=streamingapp-cluster
RegionName=ap-south-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
FluentBitReadFromTail='On'
FluentBitHttpServer='On'

curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | \
kubectl apply -f -

kubectl get pods -n amazon-cloudwatch
```

Verify in AWS Console: CloudWatch → Container Insights, and CloudWatch → Log groups → `/aws/containerinsights/streamingapp-cluster/...`.

If pods are `CrashLoopBackOff` or no log groups appear, confirm the node role has `CloudWatchAgentServerPolicy` (Section 9), then:
```bash
kubectl delete pods -n amazon-cloudwatch --all
```

---

## 12. ChatOps — SNS + Slack

```bash
aws sns create-topic --name streamingapp-deploy-notifications --region ap-south-1
```

**AWS Chatbot setup:**
1. AWS Console → Chatbot → Configure new client → Slack
2. Authorize your Slack workspace (create one if needed — office Slack sessions can interfere with OAuth, use an incognito window if unsure which workspace is being authorized)
3. Pick your notifications channel
4. Under SNS topics: select `ap-south-1` → `streamingapp-deploy-notifications`
5. Save

**Critical: message format.** AWS Chatbot silently drops plain-text SNS messages from custom sources. Messages must use this JSON schema:
```json
{
  "version": "1.0",
  "source": "custom",
  "content": {
    "textType": "client-markdown",
    "title": "Deployment SUCCESS",
    "description": "Build #12 succeeded."
  }
}
```
This is already built into the Jenkinsfile's `post` blocks.

Test directly:
```bash
aws sns publish --topic-arn arn:aws:sns:ap-south-1:562904760755:streamingapp-deploy-notifications \
  --message '{"version":"1.0","source":"custom","content":{"textType":"client-markdown","title":"Test","description":"Testing Slack integration"}}' \
  --region ap-south-1
```

---

## 13. Validation & Testing

```bash
kubectl get pods -o wide          # all 6 pods 1/1 Running
kubectl get svc                   # frontend has an EXTERNAL-IP

kubectl run debug --image=curlimages/curl -it --rm -- sh
# inside:
curl http://hello-service:3001
curl http://profile-service:3002
```

Open the frontend LoadBalancer URL in a browser and confirm the app loads and talks to both backend services.

```bash
kubectl logs deployment/profile-service   # confirm clean Mongo connection
kubectl top pods                          # if metrics-server installed
```

Test the full CI/CD loop:
```bash
git commit --allow-empty -m "test: trigger pipeline"
git push origin main
```
Confirm Jenkins auto-triggers, builds, pushes, deploys, and the Slack notification arrives.

---

## 14. Jenkins Maintenance — Disk Space

Small EC2 root volumes (6-8GB) fill up fast under repeated Docker builds + Jenkins build history. Symptoms: node shows **offline** in Manage Jenkins → Nodes, builds stuck at "Waiting for next available executor."

```bash
df -h
docker system df
docker system prune -a -f --volumes

# Also check for leftover/duplicate Jenkins homes from earlier manual runs
du -sh /home/*/.jenkins 2>/dev/null

# Old build history
du -sh /var/lib/jenkins/jobs/*/builds/* 2>/dev/null | sort -rh | head -10
```

Bring the node back online: **Manage Jenkins → Nodes → Built-In Node → "Bring this node back online"**, or `sudo systemctl restart jenkins`.

**Durable fix:** resize the EBS volume (EC2 Console → Volumes → Modify Volume → increase size), then:
```bash
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/root
```

---

## 15. Teardown / Cost Cleanup

```bash
# 1. Remove the LoadBalancer before deleting the cluster (avoids an orphaned ELB)
kubectl delete svc frontend
helm uninstall streamingapp

# 2. Delete the EKS cluster (also removes the NAT Gateway/VPC eksctl created)
eksctl delete cluster --name streamingapp-cluster --region ap-south-1

# 3. Terminate the Jenkins EC2 instance
aws ec2 terminate-instances --instance-ids <instance-id> --region ap-south-1

# 4. Release any Elastic IP
aws ec2 release-address --allocation-id <allocation-id> --region ap-south-1
```

ECR repos, the SNS topic, and the Chatbot config are left in place (negligible cost) to preserve submission evidence.

---

## 16. Full Troubleshooting Log

| Issue | Fix |
|---|---|
| `path "./helloService" not found` | Corrected build paths to `./backend/helloService` etc. |
| `MongoParseError: Invalid scheme...` | Password had an unescaped `@`; URL-encoded, then rotated |
| `SSL alert number 80` connecting to Atlas | Switched base image from `node:18-alpine` to `node:18-slim` |
| Jenkins apt install: GPG signature errors | Installed via `.war` + custom systemd service instead of apt |
| Jenkins: `Supported Java versions are [21, 25]` | Installed `openjdk-21-jdk`, set as default |
| Docker builds hanging indefinitely | EC2 memory-starved (`t3.micro`); added swap, recommended `t3.medium`+ |
| `aws: not found` in pipeline | Installed AWS CLI on the Jenkins EC2 box |
| Helm: `nil pointer evaluating .Values.httpRoute.enabled` | Deleted unused `helm create` scaffold templates |
| `ImagePullBackOff`: image not found | `values.yaml` had wrong region (`us-east-1` vs actual `ap-south-1`) |
| `ImagePullBackOff`: no match for platform in manifest | Rebuilt with `--platform linux/amd64 --provenance=false --sbom=false` |
| `403 Forbidden` pushing to ECR | ECR auth token expired; re-ran `docker login` |
| `SynchronousResumeNotSupportedException` | Jenkins restarted mid-build; wrapped checkout in `retry(nonresumable())` |
| SNS published successfully, nothing in Slack | Chatbot requires a specific JSON message schema for custom sources |
| Build stuck: "Waiting for next available executor" | Node auto-offline from low disk space; pruned Docker + old Jenkins data |

