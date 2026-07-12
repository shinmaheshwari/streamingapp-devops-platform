# Setup Guide

Condensed record of the steps actually run to stand up this project end-to-end.

## 1. Git

```bash
git clone https://github.com/shinmaheshwari/streamingapp-devops-platform.git
cd streamingapp-devops-platform
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
```

To sync with upstream:
```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

## 2. Application Setup

Environment files (not committed — see `.gitignore`):

`backend/helloService/.env`
```
PORT=3001
```

`backend/profileService/.env`
```
PORT=3002
MONGO_URL="<mongodb+srv connection string>"
```

```bash
cd backend/helloService && npm install && cd ../..
cd backend/profileService && npm install && cd ../..
cd frontend && npm install && npm start   # dev server
```

## 3. Docker

Dockerfiles created for each of `backend/helloService`, `backend/profileService`, and `frontend` (multi-stage builds; Node base image with `slim` variant used for the backend services to avoid Alpine/OpenSSL TLS issues connecting to MongoDB Atlas).

```bash
docker build -t streamingapp-helloservice:latest ./backend/helloService
docker build -t streamingapp-profileservice:latest ./backend/profileService
docker build -t streamingapp-frontend:latest ./frontend
```

## 4. AWS CLI & ECR

```bash
aws configure   # region: ap-south-1

aws ecr create-repository --repository-name streamingapp-helloservice --region ap-south-1
aws ecr create-repository --repository-name streamingapp-profileservice --region ap-south-1
aws ecr create-repository --repository-name streamingapp-frontend --region ap-south-1

aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com
```

## 5. Jenkins

Installed on a dedicated EC2 instance (`t3.medium`, Ubuntu) as a systemd service running the Jenkins `.war` file directly under Java 21 (the `apt` repo route hit unresolved GPG signing issues, so the WAR + systemd approach was used instead — see `/etc/systemd/system/jenkins.service`).

Plugins: Docker Pipeline, Amazon ECR, Pipeline: AWS Steps, Git, GitHub Integration.

Credentials configured:
- `aws-creds` — AWS access key/secret
- `github-creds` — SSH deploy key (GitHub authenticates as `git@github.com`)
- `mongo-url` — Mongo connection string (Secret text)

AWS CLI, `kubectl`, and Helm were also installed directly on the Jenkins EC2 instance so the pipeline can build, push, and deploy in the same job.

GitHub webhook configured to trigger builds on push to `main`.

## 6. Kubernetes (EKS)

```bash
eksctl create cluster \
  --name streamingapp-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --managed

aws eks update-kubeconfig --name streamingapp-cluster --region ap-south-1
```

Node IAM role additionally granted `CloudWatchAgentServerPolicy` for monitoring.

## 7. Helm Deployment

Chart at `streamingapp-chart/`, `values.yaml` pointing at the three ECR image URIs. Three deployment+service template pairs under `templates/`: `hello-deployment.yaml`, `profile-deployment.yaml`, `frontend-deployment.yaml`.

Mongo secret created before first install:
```bash
kubectl create secret generic mongo-secret \
  --from-literal=MONGO_URL='<mongo-connection-string>'
```

```bash
helm install streamingapp ./streamingapp-chart
kubectl get pods
kubectl get svc
```

Subsequent deploys from Jenkins use:
```bash
helm upgrade --install streamingapp ./streamingapp-chart \
  --set helloService.tag=<build-number> \
  --set profileService.tag=<build-number> \
  --set frontend.tag=<build-number>
```

## 8. Monitoring

CloudWatch Container Insights deployed as a DaemonSet:
```bash
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
sed 's/{{cluster_name}}/streamingapp-cluster/;s/{{region_name}}/ap-south-1/;...' | \
kubectl apply -f -
```

Verified via:
```bash
kubectl get pods -n amazon-cloudwatch
```

Metrics and logs visible in the AWS Console under CloudWatch → Container Insights and CloudWatch → Log groups (`/aws/containerinsights/streamingapp-cluster/...`).

## 9. Bonus — SNS + Slack

```bash
aws sns create-topic --name streamingapp-deploy-notifications --region ap-south-1
```

Subscribed to a Slack channel via AWS Chatbot. Jenkinsfile `post { success / failure }` blocks publish deployment status to this topic.

## Known Issues Encountered & Resolutions

| Issue | Resolution |
|---|---|
| `unable to prepare context: path "./helloService" not found` | Corrected Docker build paths to `./backend/helloService` etc. (actual repo structure) |
| `MongoParseError: Invalid scheme...` | Mongo password contained an `@` character, breaking the connection string; URL-encoded it, then rotated password entirely |
| `SSL alert number 80` connecting to MongoDB Atlas | Switched backend Dockerfiles from `node:18-alpine` to `node:18-slim` (Alpine's OpenSSL had TLS handshake issues with Atlas) |
| Jenkins GPG signing errors on `apt install jenkins` | Switched to running Jenkins via the `.war` file directly under a custom systemd service |
| Jenkins required Java 21, only Java 17 installed | Installed `openjdk-21-jdk`, set as default via `update-alternatives` |
| Docker build hanging indefinitely | Jenkins EC2 (`t3.micro`, ~900MB RAM) was memory-starved; added a 2GB swap file as a stopgap, recommended resizing to `t3.medium`+ |
| `aws: not found` in Jenkins pipeline | AWS CLI wasn't installed on the (rebuilt) Jenkins EC2 instance; installed it |
| `helm template: nil pointer evaluating .Values.httpRoute.enabled` | Removed leftover default template file (`httproute.yaml`) generated by `helm create` that referenced unset values |
| No CloudWatch log groups appearing | Node IAM role was missing `CloudWatchAgentServerPolicy`; attached it and restarted the CloudWatch agent DaemonSet pods |
