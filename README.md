# 🐱🐶 Cats vs Dogs on AWS EKS: a multi-service voting app with CI/CD

A multi-stack voting application deployed on **Amazon EKS (Kubernetes)** in **us-east-1**, exposed through an **NGINX Ingress**, and delivered automatically by a **GitHub Actions** pipeline.

This is my fork of [Pokfinner/ironhack-project-1](https://github.com/Pokfinner/ironhack-project-1), completed as a DevOps project by **Bhaumik**. Every cloud and Kubernetes resource I created carries `-bhaumik` in its name so it is unique.

---

## 🧭 Architecture

```
                        ┌───────────────────────────── AWS (us-east-1) ─────────────────────────────┐
                        │                      EKS cluster: voting-app-cluster-bhaumik              │
 User ──► AWS Load      │   NGINX Ingress                                                           │
          Balancer ─────┼──► /vote/   ──► vote-service-bhaumik   ──► vote (Python Flask) ──┐        │
                        │    /result/ ──► result-service-bhaumik ──► result (Node.js)      │        │
                        │                                              ▲                   ▼        │
                        │                                              │            redis-bhaumik   │
                        │                                      db-bhaumik (Postgres)       │        │
                        │                                              ▲                   ▼        │
                        │                                              └──────── worker (.NET) ◄────┘
                        └───────────────────────────────────────────────────────────────────────────┘
```

**Flow of one vote:** vote page → Redis queue → worker → Postgres → result page (live updates through socket.io).

| Service | Tech | Role |
|---|---|---|
| `vote` | Python Flask | Web page where users vote Cats or Dogs |
| `redis` | Redis (alpine) | In-memory queue that buffers new votes |
| `worker` | .NET 7 | Moves votes from Redis into Postgres |
| `db` | PostgreSQL 15 (alpine) | Permanent storage of votes |
| `result` | Node.js / Express | Live scoreboard |

---

## 🛠️ Tech stack and why each piece is used

| Tool | Why |
|---|---|
| **Docker** | Packages each service with its dependencies so it runs identically everywhere |
| **Docker Hub** | Registry from which the cluster pulls the images |
| **Amazon EKS** | Managed Kubernetes: AWS runs the control plane, I manage the worker nodes |
| **eksctl** | Creates the cluster, VPC, subnets and node group from one config file |
| **Kubernetes** | Self-healing, scaling, service discovery and rolling updates |
| **NGINX Ingress (Helm)** | One load balancer routing URL paths to the right service |
| **Kubernetes Secret** | Keeps database credentials out of Git |
| **GitHub Actions** | Builds, publishes and deploys on every push to `main` |

---

## 📁 Repository structure

```
.
├── vote/                  # Python Flask app + Dockerfile
├── result/                # Node.js app + Dockerfile
├── worker/                # .NET worker + Dockerfile
├── healthchecks/          # health check scripts (from the original repo)
├── docker-compose.yml     # local development stack
├── cluster.yaml           # eksctl cluster definition
├── K8s/
│   ├── redis.yaml         # Redis Deployment + Service
│   ├── postgres.yaml      # Postgres Deployment + Service
│   ├── vote.yaml          # Vote Deployment (2 replicas) + Service
│   ├── result.yaml        # Result Deployment + Service
│   ├── worker.yaml        # Worker Deployment (no Service needed)
│   └── ingress.yaml       # Ingress rules (3 resources)
└── .github/workflows/
    └── ci-cd-pipeline.yml # build, push and deploy pipeline
```

---

## ✅ Prerequisites

- AWS account, an IAM user with CLI access keys, region `us-east-1`
- Installed: AWS CLI v2, `eksctl`, `kubectl`, `helm`, Docker, `git`
- Docker Hub account and an access token
- A fork of this repository

---

## 🚀 Deployment guide

### 1. Run locally first (optional sanity check)

```bash
docker compose up
# vote:   http://localhost:8080
# result: http://localhost:8081
```

### 2. Create the EKS cluster

```bash
aws configure                      # region: us-east-1
eksctl create cluster -f cluster.yaml
kubectl get nodes                  # expect 2 nodes in Ready state
```

`cluster.yaml` creates `voting-app-cluster-bhaumik` with the managed node group `voting-nodes-bhaumik` (2 × `t3.medium`).

### 3. Build and push the images

The pipeline does this automatically; to do it by hand:

```bash
docker login
docker buildx build --platform linux/amd64 -t <DOCKERHUB_USER>/vote-bhaumik:latest   --push ./vote
docker buildx build --platform linux/amd64 -t <DOCKERHUB_USER>/result-bhaumik:latest --push ./result
docker buildx build --platform linux/amd64 -t <DOCKERHUB_USER>/worker-bhaumik:latest --push ./worker
```

Use `--platform linux/amd64`: the EKS nodes are x86, and images built on Apple Silicon would fail with `exec format error`.

### 4. Create the database secret

The password is **not** stored in the repository. Create it once in the cluster:

```bash
kubectl create secret generic db-credentials-bhaumik \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD='<your-password>'
```

### 5. Deploy the application

```bash
kubectl apply -f K8s/
kubectl get pods
```

The apps read their configuration from environment variables:

| App | Variables |
|---|---|
| vote | `REDIS_HOST` |
| worker | `REDIS_HOST`, `DB_HOST`, `DB_USERNAME`, `DB_PASSWORD` |
| result | `PG_HOST`, `PG_USER`, `PG_PASSWORD` |

### 6. Install the NGINX Ingress controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-nginx-bhaumik ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

kubectl apply -f K8s/ingress.yaml
kubectl get svc -n ingress-nginx     # copy the EXTERNAL-IP hostname
```

### 7. Open the app

```
http://<LOAD_BALANCER_HOSTNAME>/vote/
http://<LOAD_BALANCER_HOSTNAME>/result/
```

Keep the trailing slash. DNS for a new load balancer can take a few minutes to propagate.

---

## 🌐 Ingress routing

| Path | Goes to | Why |
|---|---|---|
| `/vote/` | vote service | Prefix is stripped before forwarding |
| `/result/` | result service | Prefix is stripped before forwarding |
| `/vote/static/...` | vote service | The vote app serves its own static files under this path, so no rewrite |
| `/result/socket.io/...` | result service | The result app uses a custom socket.io path, so no rewrite; long timeouts keep the live connection open |

---

## 🤖 CI/CD pipeline

File: `.github/workflows/ci-cd-pipeline.yml`. It runs on every push to `main`.

1. **build-and-push**: builds `vote`, `result` and `worker` in parallel and pushes each image to Docker Hub with two tags: `latest` and the commit SHA.
2. **deploy**: configures AWS credentials, runs `aws eks update-kubeconfig`, applies `K8s/`, sets each Deployment to the image of the current commit, and waits for the rollout to finish.

Using the commit SHA as the tag guarantees that Kubernetes rolls out new pods for every change and makes it clear which code is running.

**GitHub repository secrets required** (Settings → Secrets and variables → Actions):

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AWS_ACCESS_KEY_ID` | Access key of the IAM user that created the cluster |
| `AWS_SECRET_ACCESS_KEY` | Matching secret key |

---

## 🔎 Verifying the deployment

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl logs deploy/worker-bhaumik --tail=10    # shows "Processing vote for ..."
kubectl logs deploy/result-bhaumik --tail=10    # shows "Connected to db"
```

To prove the pipeline works: change the heading text in `vote/`, push to `main`, watch the Actions run turn green, then hard-refresh `/vote/`.

---

## 🐛 Problems I solved

| Problem | Symptom | Cause and fix |
|---|---|---|
| Invalid image name | `InvalidImageName` on pods | A `<DOCKERHUB_USER>` placeholder was left in the YAML; replaced it with the real username |
| Wrong environment variables | `getaddrinfo ENOTFOUND db` in the result logs | The apps read `PG_HOST` and `DB_HOST`, not the names I first guessed; found the right names by reading the code |
| Missing static files | Vote page unstyled | The app serves files under `/vote/static`; added an Ingress rule without the path rewrite |
| Blank result page | `404` on `/socket.io` | The app uses the custom path `/result/socket.io`; added a dedicated Ingress rule |
| Startup order | `relation "votes" does not exist` | The worker creates the table; the result app recovered once the worker had connected |
| Kerberos warning | `libgssapi_krb5.so.2` message in worker logs | Harmless: the .NET driver falls back to password authentication |

---

## ⚠️ Known limitations

- **Postgres has no persistent volume**: votes are lost if the database pod restarts.
- The application code falls back to default credentials if variables are not set; the manifests always supply them from the Secret.
- The pipeline uses long-lived AWS access keys.
- The site is served over HTTP only.

## 🚀 Future improvements

- Persistent storage for Postgres (StatefulSet with an EBS volume)
- HTTPS with cert-manager, Let’s Encrypt and Route 53
- GitHub OIDC for AWS authentication instead of access keys
- Horizontal Pod Autoscaler, plus readiness and liveness probes
- Infrastructure as code with Terraform
- Monitoring and alerting with Prometheus and Grafana

---

## 🧹 Cleanup (avoid AWS charges)

Remove the Ingress and controller **before** deleting the cluster, so AWS removes the load balancer:

```bash
kubectl delete -f K8s/ingress.yaml
helm uninstall my-nginx-bhaumik -n ingress-nginx
eksctl delete cluster -f cluster.yaml
```

Then check EC2 → Load Balancers, VPC and CloudFormation in `us-east-1` for leftovers, delete the IAM access key used for the project, and remove the AWS secrets from GitHub.

---

## 📜 License and credits

Original project: [Pokfinner/ironhack-project-1](https://github.com/Pokfinner/ironhack-project-1), licensed under Apache-2.0. Kubernetes manifests, cluster configuration, CI/CD pipeline and documentation by Bhaumik.
