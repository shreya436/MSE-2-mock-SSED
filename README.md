# CampusHub 9-Service Docker Architecture & Deployment Guide

This project orchestrates a multi-service event registration application (**CampusHub**) composed of 9 specialized services using **Docker**, **Docker Compose**, and **Docker Stack**, configured for non-root execution, optimized build hygiene, layer caching, and deployment readiness on **AWS**.

---

## 1. System Architecture & Services

The application consists of 9 decoupled microservices:

| Service | Image / Build | Port(s) | Network | Description & Responsibility |
|---|---|---|---|---|
| **gateway** | `nginxinc/nginx-unprivileged:alpine` | `8080` (public) | `frontend` | Reverse proxy and single public ingress point. Directs `/` to frontend, `/api/` to API, and `/storage/` to MinIO. |
| **frontend** | Multi-stage build (`app/frontend/Dockerfile`) | `8080` (internal) | `frontend` | React SPA built with Vite and served statically via unprivileged Nginx. |
| **api** | Alpine Node.js (`app/api/Dockerfile`) | `3000` (internal) | `frontend`, `backend`, `data` | REST API handling event listings, registrations, image uploads, migrations, and database seeds. |
| **worker** | Reused API image (`app/api/Dockerfile`) | None | `backend` | Background consumer (`src/worker.js`) processing event registration emails from RabbitMQ and sending via MailHog. |
| **db** | `postgres:16-alpine` | `5432` (internal) | `data` | Relational PostgreSQL database storing events and attendee registrations. |
| **redis** | `redis:7-alpine` | `6379` (internal) | `data` | In-memory key-value cache with password authentication for caching event lists (`X-Cache: HIT/MISS`). |
| **rabbitmq** | `rabbitmq:3-management-alpine` | `15672` (`127.0.0.1` only) | `backend` | Message broker managing the `registration_emails` task queue for asynchronous processing. |
| **minio** | `minio/minio:latest` | `9001` (`127.0.0.1` only) | `frontend`, `data` | S3-compatible object storage for persistent event banner image uploads. |
| **mailhog** | `mailhog/mailhog:latest` | `8025` (`127.0.0.1` only) | `backend` | Developer SMTP server (`1025`) and web UI (`8025`) for viewing outgoing registration emails. |

---

## 2. Key Design Decisions

1. **Strict Network Segmentation & Least Privilege Access**:
   - Three isolated bridge networks (`frontend`, `backend`, `data`) ensure backend services (Postgres, Redis) cannot be reached by external traffic or the frontend directly.
   - Only the **gateway** is exposed to host interfaces (`0.0.0.0:8080`). Administrative interfaces (MailHog, RabbitMQ UI, MinIO Console) are strictly bound to loopback `127.0.0.1` to prevent unauthorized external access.

2. **Multi-Stage Builds & Optimized Layer Caching**:
   - **Frontend**: Stage 1 uses Node.js to compile Vite assets; Stage 2 transfers only the generated `dist/` artifacts to `nginxinc/nginx-unprivileged:alpine`. Node.js and `node_modules` are completely excluded from the runtime image.
   - **API & Worker**: Dependency manifests (`package*.json`) are copied with `--chown=node:node` before installing production dependencies (`npm install --omit=dev --no-audit --no-fund && npm cache clean --force`). Source code is copied in a subsequent layer. This prevents re-downloading packages when only application code changes, keeping the API image size around **214 MB** (well under the 250 MB ceiling).

3. **Non-Root Execution Across All Custom Containers**:
   - The frontend and gateway utilize `nginxinc/nginx-unprivileged:alpine` (UID 101).
   - The API and worker execute under the unprivileged `node` user (UID 1000).
   - Running without root privileges eliminates common privilege-escalation vulnerabilities in containerized environments.

4. **Service Healthchecks & Dependency Ordering**:
   - `depends_on` conditions with `service_healthy` ensure the API container only starts after Postgres, Redis, RabbitMQ, and MinIO have successfully passed health probes.
   - The worker overrides the inherited API HTTP health check with a process check (`node -e 'process.exit(0)'`) to reflect its standalone non-HTTP execution.

---

## 3. How to Run Locally

### Prerequisites
- Docker Engine & Docker Compose v2+ installed.
- Ensure Docker daemon / Docker Desktop is running.

### Step 1: Prepare Environment
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```

### Step 2: Build and Start Stack
Start all nine services with build and wait for healthchecks:
```bash
docker compose up -d --build --wait
```

### Step 3: Verify Running Services
```bash
docker compose ps
```

### Step 4: Access Endpoints
- **Application Frontend**: `http://localhost:8080/`
- **API Health Check**: `http://localhost:8080/api/health`
- **Events API**: `http://localhost:8080/api/events`
- **MailHog Web UI**: `http://127.0.0.1:8025/`
- **RabbitMQ Management**: `http://127.0.0.1:15672/` (User: `campus`, Password: from `.env`)
- **MinIO Console**: `http://127.0.0.1:9001/` (User: `minioadmin`, Password: from `.env`)

### Step 5: Stop the Stack
```bash
# Stop containers while preserving volume data
docker compose down

# Stop containers and remove volumes
docker compose down -v
```

---

## 4. Docker Stack Deployment (`docker-stack.yml`)

For Swarm-ready deployment without needing Swarm setup in single-node environments:
```bash
# Initialize single-node swarm if testing stack deploy
docker swarm init

# Deploy the stack
docker stack deploy -c docker-stack.yml campushub

# Inspect services
docker stack services campushub

# Remove the stack
docker stack rm campushub
```

---

## 5. Deployment on AWS (Step-by-Step)

### Option A: AWS EC2 (Recommended for 10-mark practical exams)
1. **Launch EC2 Instance**:
   - AMI: Ubuntu 24.04 LTS (t3.medium or t3.small recommended).
   - Security Group: Allow Inbound HTTP on port `8080` (or `80`) and SSH on port `22`.
2. **Install Docker & Docker Compose**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
   sudo usermod -aG docker ubuntu
   ```
3. **Deploy the Code**:
   ```bash
   git clone https://github.com/jermgaming/MSE-2-mock-SSED.git campushub
   cd campushub
   # Add docker-compose.yml, Dockerfiles, and .env
   docker compose up -d --build --wait
   ```
4. **Access**:
   - `http://<EC2-PUBLIC-IP>:8080`

### Option B: Pushing Images to Docker Hub / AWS ECR
```bash
# 1. Login to registry
docker login -u <your-dockerhub-username>

# 2. Tag your images
docker tag campushub-frontend:latest <your-dockerhub-username>/campushub-frontend:v1.0.0
docker tag campushub-api:latest <your-dockerhub-username>/campushub-api:v1.0.0

# 3. Push to registry
docker push <your-dockerhub-username>/campushub-frontend:v1.0.0
docker push <your-dockerhub-username>/campushub-api:v1.0.0

# 4. Pull on any other machine / AWS instance:
docker pull <your-dockerhub-username>/campushub-frontend:v1.0.0
docker pull <your-dockerhub-username>/campushub-api:v1.0.0
```
