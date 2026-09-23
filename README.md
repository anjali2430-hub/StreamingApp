# StreamingApp — Container Orchestration & Scaling

> **HeroVired Graded Project** | DevOps Track | Multicloud Architecture in DevOps

Stream premium video content, host live watch parties, and manage your catalogue with a modern microservice architecture deployed on Kubernetes via Helm.

---

## 🐳 Docker Hub Images

| Service | Image | Tag |
|---|---|---|
| authService | [anjali2430/streaming-auth](https://hub.docker.com/r/anjali2430/streaming-auth) | 1.0.0 |
| streamingService | [anjali2430/streaming-stream](https://hub.docker.com/r/anjali2430/streaming-stream) | 1.0.0 |
| adminService | [anjali2430/streaming-admin](https://hub.docker.com/r/anjali2430/streaming-admin) | 1.0.0 |
| chatService | [anjali2430/streaming-chat](https://hub.docker.com/r/anjali2430/streaming-chat) | 1.0.0 |
| frontend | [anjali2430/streaming-frontend](https://hub.docker.com/r/anjali2430/streaming-frontend) | 1.0.0 |

---

## 🏗️ Architecture

Five services and a shared MongoDB database deployed on Kubernetes:

```
┌─────────────────────────────────────────────────────┐
│              React Frontend (Nginx) :80              │
└──────┬──────────┬──────────┬──────────┬────────────┘
       │          │          │          │
  :3001/api  :3002/api  :3003/api  :3004/api (WS)
       │          │          │          │
  authService  streaming  adminSvc  chatService
       │          │          │          │
       └──────────┴──────────┴──────────┘
                        │
                   MongoDB :27017
```

| Service | Port | Description | Docker Hub |
|---|---|---|---|
| `authService` | 3001 | User authentication, registration, JWT issuance | anjali2430/streaming-auth:1.0.0 |
| `streamingService` | 3002 | Video catalogue, S3 playback endpoints, public APIs | anjali2430/streaming-stream:1.0.0 |
| `adminService` | 3003 | Dedicated admin microservice for asset management and uploads | anjali2430/streaming-admin:1.0.0 |
| `chatService` | 3004 | WebSocket + REST chat for live watch parties | anjali2430/streaming-chat:1.0.0 |
| `frontend` | 80 | React SPA served via Nginx | anjali2430/streaming-frontend:1.0.0 |
| `mongo` | 27017 | Shared MongoDB instance (StatefulSet + PVC) | mongo:7 |

---

## ☸️ Kubernetes Deployment with Helm

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [Helm 3](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

### Step 1 — Start Minikube

```bash
minikube start --driver=docker
minikube addons enable ingress
```

### Step 2 — Add host entry

```bash
echo "127.0.0.1 streamingapp.local" | sudo tee -a /etc/hosts
```

### Step 3 — Install with Helm

```bash
helm install streamingapp ./streamingapp
```

### Step 4 — Verify all pods are running

```bash
kubectl get pods,svc,ingress -A
```

### Step 5 — Access the app

Open a new terminal and run:
```bash
minikube tunnel
```

Then open your browser at:
```
http://streamingapp.local
```

Or use the direct service URL:
```bash
minikube service frontend-svc --url
```

---

## 📁 Helm Chart Structure

```
streamingapp/
├── Chart.yaml                    # Chart metadata
├── values.yaml                   # Configurable values
└── templates/
    ├── configmap.yaml            # Non-secret env vars
    ├── secret.yaml               # JWT & AWS secrets
    ├── mongo-statefulset.yaml    # MongoDB StatefulSet + PVC
    ├── auth-deployment.yaml      # Auth service + ClusterIP
    ├── streaming-deployment.yaml # Streaming service + ClusterIP
    ├── admin-deployment.yaml     # Admin service + ClusterIP
    ├── chat-deployment.yaml      # Chat service + ClusterIP
    ├── frontend-deployment.yaml  # Frontend + ClusterIP
    └── ingress.yaml              # Nginx Ingress routing
```

---

## 🔀 Ingress Routing

All traffic enters through a single Nginx Ingress at `streamingapp.local`:

| Path | Backend Service | Port |
|---|---|---|
| `/` | frontend-svc | 80 |
| `/api/auth` | auth-svc | 3001 |
| `/api/streaming` | streaming-svc | 3002 |
| `/api/admin` | admin-svc | 3003 |
| `/api/chat` | chat-svc | 3004 |

---

## 📈 Scaling

Scale any service up or down:

```bash
# Scale streaming to 4 replicas
kubectl scale deploy/streaming --replicas=4

# Verify
kubectl get pods
```

Or via Helm:
```bash
helm upgrade streamingapp ./streamingapp --set services.streaming.replicas=4
```

---

## 🔄 Rolling Updates

Update a service image tag with zero downtime:

```bash
helm upgrade streamingapp ./streamingapp --set services.auth.tag=1.0.1
kubectl rollout status deploy/auth
```

Every Deployment uses `maxUnavailable: 0` and `maxSurge: 1` to ensure zero downtime during updates.

---

## 🩺 Health Checks

Every Deployment includes readiness and liveness probes:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3001
  initialDelaySeconds: 20
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /health
    port: 3001
  initialDelaySeconds: 30
  periodSeconds: 15
```

---

## 🐋 Building Docker Images Locally

```bash
export DOCKERUSER=anjali2430

docker build -t $DOCKERUSER/streaming-auth:1.0.0 backend/authService
docker build -t $DOCKERUSER/streaming-stream:1.0.0 -f backend/streamingService/Dockerfile backend
docker build -t $DOCKERUSER/streaming-admin:1.0.0 -f backend/adminService/Dockerfile backend
docker build -t $DOCKERUSER/streaming-chat:1.0.0 -f backend/chatService/Dockerfile backend
docker build -t $DOCKERUSER/streaming-frontend:1.0.0 frontend

docker push $DOCKERUSER/streaming-auth:1.0.0
docker push $DOCKERUSER/streaming-stream:1.0.0
docker push $DOCKERUSER/streaming-admin:1.0.0
docker push $DOCKERUSER/streaming-chat:1.0.0
docker push $DOCKERUSER/streaming-frontend:1.0.0
```

---

## 💻 Local Development (without Docker)

### Prerequisites
- Node.js 18
- MongoDB running locally

### Install dependencies

```bash
cd backend/authService && npm install
cd ../streamingService && npm install
cd ../adminService && npm install
cd ../chatService && npm install
cd ../../frontend && npm install
```

### Create .env files

**`backend/authService/.env`**
```ini
PORT=3001
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=supersecretkey123
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

**`backend/streamingService/.env`**
```ini
PORT=3002
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=supersecretkey123
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=
STREAMING_PUBLIC_URL=http://localhost:3002
```

**`backend/adminService/.env`**
```ini
PORT=3003
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=supersecretkey123
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

**`backend/chatService/.env`**
```ini
PORT=3004
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=supersecretkey123
CLIENT_URLS=http://localhost:3000
```

**`frontend/.env`**
```ini
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

### Run services (5 separate terminals)

```bash
cd backend/authService && npm run dev       # Terminal 1
cd backend/streamingService && npm run dev  # Terminal 2
cd backend/adminService && npm run dev      # Terminal 3
cd backend/chatService && npm run dev       # Terminal 4
cd frontend && npm start                    # Terminal 5
```

Visit `http://localhost:3000`

---

## 🐳 Running with Docker Compose

```bash
docker-compose up --build
```

Visit `http://localhost:3000`

---

## ✅ Smoke Tests

1. Register a new account at `/register`
2. Log in at `/login`
3. Browse the video catalogue
4. Upload a video via the admin dashboard (requires AWS S3 credentials)
5. Watch a video and send chat messages across two browser tabs

---

## 🚀 Production Considerations

For a production cluster the following changes would be made:

- **Namespaces** — dedicated namespaces per environment (`dev`, `staging`, `prod`) to isolate workloads and apply RBAC policies
- **TLS** — cert-manager with Let's Encrypt certificates on the Ingress for HTTPS termination
- **HPA** — Horizontal Pod Autoscalers on `auth` and `streaming` services based on CPU/memory metrics to handle traffic spikes automatically
- **Resource limits** — explicit `requests` and `limits` on every container to prevent noisy-neighbour issues
- **Managed MongoDB** — replace the StatefulSet with MongoDB Atlas or AWS DocumentDB for better durability, automated backups, and point-in-time recovery
- **Secrets management** — use AWS Secrets Manager or HashiCorp Vault instead of Kubernetes Secrets for better security
- **CI/CD** — Jenkins pipeline to automatically build, test, and push new images to ECR on every commit

---

## 📸 Screenshots

All submission screenshots are in the [`screenshots/`](./screenshots/) folder:

| File | Description |
|---|---|
| Screenshot 1 | `kubectl get pods,svc,ingress -A` — all services running |
| Screenshot 2 | Pod self-heal — deleted pod auto-recreated |
| Screenshot 3 | Docker Hub — all 5 images pushed |
| Screenshot 4 | Scaling — 4 streaming replicas running |
| Screenshot 5 | GitHub Helm chart folder structure |
| Screenshot 5A | GitHub repo root |
| Screenshot 6 | App running in browser on Kubernetes |
| Screenshot 7 | Rolling update — `deployment successfully rolled out` |

---

## 📝 Submission

- **GitHub Repository:** https://github.com/anjali2430-hub/StreamingApp
- **Docker Hub:** https://hub.docker.com/u/anjali2430
- **Helm Chart:** `./streamingapp/`

---

## 🌟 Feature Highlights

- **S3-backed adaptive streaming** with secure signed uploads for admins
- **Dedicated admin microservice** for video ingestion, metadata management, and featured curation
- **Real-time chat** overlay in the player (Socket.IO + persistent message history)
- **Modern React experience** featuring cinematic hero sections, dynamic carousels, and responsive design
- **Role-aware access control** across frontend routes and backend microservices
- **Kubernetes-native** deployment with health probes, rolling updates, and auto-scaling

---

## 📄 License

MIT © StreamFlix Team
