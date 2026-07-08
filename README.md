# DevBoard — React + Go + Postgres + DevSecOps Pipeline

A full-stack project management dashboard with a production-grade DevSecOps CI/CD pipeline.

```
browser  →  frontend (React + Vite)  →  backend (Go + Gin)  →  database (Postgres)
```

- **frontend** — React 18 app with TailwindCSS, Tanstack Query, Kanban board, and task management UI. Forwards `/api` requests to the backend.
- **backend** — Go (Gin) REST API that reads/writes Postgres.
- **database** — Postgres 16, seeded with example projects and tasks on first start.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, TailwindCSS, Tanstack Query, React Router |
| Backend | Go 1.23, Gin, `lib/pq` |
| Database | PostgreSQL 16 |
| Testing | Vitest (frontend), `go test` (backend) |
| Containerization | Docker, Docker Compose |
| CI/CD | GitHub Actions (9-gate DevSecOps pipeline) |
| SAST | SonarQube, ESLint, `go vet`, `go fmt` |
| Secret Scan | Gitleaks |
| SCA | npm audit, Govulncheck |
| Container Scan | Trivy, Hadolint |
| DAST | OWASP ZAP |
| Registry | Docker Hub |

---

## What you need

- **Docker** with Docker Compose (comes with Docker Desktop). That's it — no Node, Go, or Postgres needed locally.

---

## Part 1 — Manual way (understand the wiring)

Run all commands from the project root.

### Step 1: Create a network

```bash
docker network create devboard-net
```

### Step 2: Build the images

```bash
docker build -t devboard-frontend ./frontend
docker build -t devboard-backend ./backend
```

### Step 3: Run the database

```bash
docker run -d --name postgres --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/init/postgres":/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

### Step 4: Run the backend

```bash
docker run -d --name backend --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-backend
```

### Step 5: Run the frontend

```bash
docker run -d --name frontend --network devboard-net \
  -p 8080:4173 \
  devboard-frontend
```

### Step 6: Open and verify

Open **http://localhost:8080** — you should see the DevBoard dashboard.

```bash
curl http://localhost:8081/health                       # backend health check
curl "http://localhost:8080/api/tasks?project_id=1"    # app → backend → database
```

### Step 7: Clean up

```bash
docker rm -f frontend backend postgres
docker network rm devboard-net
```

> The backend finds the database by the name `postgres` (via `POSTGRES_URL`). The frontend finds the backend by the name `backend` (via `frontend/vite.config.js`). Container names must match, and they only resolve because everything is on the same `devboard-net` network.

---

## Part 2 — Docker Compose (the easy way)

```bash
cp .env.example .env        # one time only
docker compose up --build
```

Open **http://localhost:8080**. Stop with `docker compose down`.

| Service  | URL | Notes |
|---|---|---|
| Frontend | http://localhost:8080 | React app; proxies `/api` to backend |
| Backend  | http://localhost:8081/health | Go API |
| Postgres | localhost:5432 | user/password: `devboard`/`devboard` |

---

## Part 3 — Makefile shortcuts

```bash
make           # list all commands
make setup     # create .env (first time only)
make up        # build and start everything
make down      # stop everything
make logs      # watch logs
make reset     # wipe database and start fresh
make smoke     # quick health check
```

`make up` creates `.env` automatically.

---

## Settings — `.env`

All configurable values live in `.env`. Copy the template once:

```bash
cp .env.example .env
```

| Variable | Default | Description |
|---|---|---|
| `POSTGRES_USER` | `devboard` | Postgres username |
| `POSTGRES_PASSWORD` | `devboard` | Postgres password |
| `POSTGRES_DB` | `devboard` | Database name |
| `BACKEND_PORT` | `8080` | Port Go app listens on inside container |
| `POSTGRES_HOST_PORT` | `5432` | Host port for Postgres |
| `BACKEND_HOST_PORT` | `8081` | Host port for backend |
| `FRONTEND_HOST_PORT` | `8080` | Host port for frontend |
| `IMAGE_TAG` | `latest` | Docker image tag (CI uses `github.sha`) |
| `DOCKERHUB_USERNAME` | — | Your Docker Hub username |

`.env` is gitignored. `.env.example` is the committed template.

---

## API Reference

The browser calls these as `/api/...`; the backend serves them at the root.

| Method | Path | Description |
|---|---|---|
| GET | `/projects` | List all projects |
| POST | `/projects` | Create a project |
| GET | `/tasks?project_id=N` | List tasks in a project |
| POST | `/tasks` | Create a task |
| PATCH | `/tasks/:id` | Update a task (status, priority, etc.) |
| GET | `/search?q=&project_id=N` | Search tasks by title |
| GET | `/health` | Health check |

---

## DevSecOps CI/CD Pipeline

Every `git push` to `main` triggers the full pipeline defined in [devsecops.yaml](.github/workflows/devsecops.yaml). It sequences **9 security gates** across modular reusable workflows before deploying.

| Gate | Workflow | Tool | What it does |
|:---:|---|---|---|
| 1 | `1-code-lint.yml` | ESLint, `go fmt`, `go vet` | Lints frontend JS/JSX and formats/vets Go code |
| 2 | `2-secret-scan.yml` | Gitleaks | Scans full Git history for leaked secrets |
| 3 | `3-dependency.yml` | npm audit, Govulncheck | Checks frontend and Go dependencies for known CVEs |
| 4 | `4-docker-checks.yml` | Hadolint, Trivy | Lints Dockerfiles; builds and scans images for CRITICAL CVEs |
| 5 | `5-sonar-qube.yml` | SonarQube | SAST scan across `backend/` and `frontend/` source |
| 6 | `6-code-test.yml` | Vitest, `go test` | Runs frontend unit tests and Go backend tests |
| 7 | `7-docker-push.yml` | Docker Hub | Builds and pushes `devboard-frontend` and `devboard-backend` images tagged with `github.sha` |
| 8 | `8-deploy.yml` | Docker Compose (self-hosted runner) | Pulls latest images and redeploys via `docker compose up -d` |
| 9 | `9-dast-scan.yml` | OWASP ZAP | Baseline DAST scan against the live deployed app |

Gates 1–7 run in parallel. Gate 8 (deploy) waits for all of them to pass. Gate 9 (DAST) runs after deploy.

All scan reports (dependency, Govulncheck, ZAP) are uploaded as **GitHub Actions Artifacts**.

---

## GitHub Actions Secrets & Variables

### Repository Secrets

| Secret | Description |
|---|---|
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token |
| `SONAR_TOKEN` | SonarQube authentication token |
| `SONAR_HOST_URL` | SonarQube server URL (e.g. `http://<ip>:9000`) |
| `EC2_HOST` | Public IP of the deployment EC2 instance |

### Repository Variables

| Variable | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |

---

## SonarQube Setup (Self-Hosted)

```bash
docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:community
```

- Open port `9000` in your EC2 Security Group.
- Access `http://<EC2_PUBLIC_IP>:9000` — default login: `admin` / `admin`.
- Go to **Profile → My Account → Security → Generate Token**.
- Add `SONAR_TOKEN` and `SONAR_HOST_URL` as GitHub repository secrets.

The SonarQube project is pre-configured in [sonar-project.properties](sonar-project.properties) to scan both `backend/` and `frontend/`.

---

## Deployment (Self-Hosted Runner)

The deploy job (`8-deploy.yml`) runs on a **self-hosted GitHub Actions runner** on your EC2 instance. It:

1. Checks out the repo.
2. Copies `.env.example` → `.env`.
3. Logs into Docker Hub.
4. Runs `docker compose pull && docker compose up -d` with `IMAGE_TAG=${{ github.sha }}`.

---

## Folder Layout

```
.
├── .github/workflows/
│   ├── devsecops.yaml        # main orchestrator — calls all 9 workflows
│   ├── 1-code-lint.yml       # ESLint + go fmt + go vet
│   ├── 2-secret-scan.yml     # Gitleaks
│   ├── 3-dependency.yml      # npm audit + Govulncheck
│   ├── 4-docker-checks.yml   # Hadolint + Docker build + Trivy
│   ├── 5-sonar-qube.yml      # SonarQube SAST
│   ├── 6-code-test.yml       # Vitest + go test
│   ├── 7-docker-push.yml     # Docker Hub push
│   ├── 8-deploy.yml          # Docker Compose deploy (self-hosted)
│   └── 9-dast-scan.yml       # OWASP ZAP DAST
├── backend/                  # Go + Gin REST API
├── frontend/                 # React + Vite + TailwindCSS
├── init/postgres/            # Schema + seed data (loaded on first start)
├── docker-compose.yml
├── Makefile
├── .env.example
└── sonar-project.properties
```
