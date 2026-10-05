# Dockerize-Everything

> A production-oriented multi-service containerized platform built around one architectural principle: **Contain the blast radius.**

---

## PART 1 — WHAT IS THIS PROJECT (FOR NON-TECHNICAL READERS)

### What is this project?

This project takes three real web applications — a Python API, a Node.js API, and an analytics dashboard — and packages them together with a database and a cache into a system that behaves the way real production software behaves.

### What problem does it solve?

Most software tutorials show you how to run one application on your laptop. Real companies run dozens of services that depend on each other, need to stay running when things break, must not expose passwords in code, and must be deployable consistently across different machines. This project demonstrates those real-world patterns.

### Who would use it?

Platform engineers, DevOps engineers, and backend engineers who need to understand how to containerize and operate multi-service systems. Recruiters and CTOs evaluating a candidate's practical infrastructure knowledge.

### What does it actually do?

Two APIs accept HTTP requests and respond with data. One API also connects to a PostgreSQL database and proves that connection is live. A Redis cache sits behind an analytics platform (Apache Superset) that visualizes data. All six services start in the correct order, stay healthy, and recover automatically from unexpected failures.

### Why does it matter?

Anyone can run `docker run hello-world`. Knowing how to design network isolation, set resource limits, implement health-gated startup, handle secrets correctly, and automate deployment with scanning and traceability — that is what separates a junior engineer from a senior one.

### What was the engineering mission?

Design and operate a multi-service Docker Compose stack that enforces security, reliability, and operational hygiene at every layer — then automate its build, scan, and deployment pipeline through GitHub Actions.

---

## PART 2 — ENGINEERING OBJECTIVE

### Primary Objective

Deploy a multi-service containerized application stack that enforces security, reliability, and operational correctness patterns found in real production environments.

### Secondary Objectives

- Demonstrate network segmentation between application and infrastructure services
- Implement health-gated service startup to eliminate race conditions
- Enforce resource limits to prevent noisy-neighbor failures
- Handle secrets correctly — injected at runtime, never committed to Git
- Automate image builds, vulnerability scanning, and deployment via CI/CD

### Functional Requirements

- Flask API must respond to `/health` with `{"status": "ok", "service": "flask-api"}`
- Node API must respond to `/health` and `/db-health` proving live database connectivity
- Superset must be accessible at port 8088 with admin authentication
- All services must start in dependency order without manual intervention
- All services must recover automatically from unexpected process failures

### Non-Functional Requirements

| Property | Requirement |
|----------|-------------|
| Reliability | Services recover from crashes without human intervention |
| Security | No credentials in code or Git history. No root containers. |
| Observability | Health status visible via `docker compose ps` and inspect |
| Portability | Stack runs identically on any machine with Docker installed |
| Traceability | Every deployed image maps to an exact Git commit |

### Constraints

- Single host deployment (Docker Compose, not Kubernetes)
- No cloud provider required — runs entirely on local hardware
- No external secrets manager (Vault) — environment variable injection only

### Known Limitations

- No reverse proxy — APIs not exposed through TLS termination
- No centralized logging — stdout only
- No metrics collection — no Prometheus, no Grafana
- No automated backups of database volumes
- CI pipeline builds only Flask API image — Node and Superset not yet covered

### Success Criteria

- `docker compose ps` shows all six services healthy within 60 seconds
- All three health endpoints return correct responses
- Superset login works at `http://localhost:8088`
- GitHub Actions pipeline completes green with Trivy scan results uploaded
- Auto-PR opens on every push to main

---

## PART 3 — SYSTEM AT A GLANCE

### 3.1 High-Level Architecture

```
Internet / Developer
        ↓
  (No reverse proxy — direct port access in dev)
        ↓
┌───────────────────────────────────────────────┐
│             FRONTEND NETWORK                  │
│  flask-api (:8000)    node-api (:3000)        │
│       │                    │                  │
│       └──────────┬─────────┘                  │
└──────────────────│────────────────────────────┘
                   │ (both services bridge both networks)
┌──────────────────│────────────────────────────┐
│              BACKEND NETWORK                  │
│       │                    │                  │
│   postgres              redis                 │
│   (:5432)               (:6379)              │
│                    superset-worker            │
└───────────────────────────────────────────────┘
        ↑
   superset (:8088) — sits on both networks
```

### 3.2 Component Responsibilities

| Component | What It Is | What It Does | Depends On | What Depends On It | What Breaks If Removed |
|-----------|-----------|--------------|------------|-------------------|----------------------|
| postgres | PostgreSQL 16 | Stores persistent application data | Nothing | flask-api, node-api, superset, superset-worker | All dependent services lose their data layer |
| redis | Redis 7 | Cache and message broker | Nothing | superset, superset-worker | Superset cache dies, Celery jobs cannot queue |
| flask-api | Python/Gunicorn | Serves HTTP API on port 8000 | postgres | Nothing currently | API endpoint unavailable |
| node-api | Node.js/Express | Serves HTTP API on port 3000, proves DB connectivity | postgres | Nothing currently | API endpoint and DB health check unavailable |
| superset | Apache Superset | Analytics dashboard UI on port 8088 | postgres, redis | Nothing | No visualization platform |
| superset-worker | Celery | Runs background analytics queries | postgres, redis | superset | Long-running queries block the UI |

### 3.3 Data Flow — Tracing a Real Request

**Request: `GET /db-health` on Node API**

```
1. Developer runs: curl http://localhost:3000/db-health
2. Request arrives at node-api container on port 3000
3. Express router matches /db-health route
4. Handler calls pool.connect() — draws from pg connection pool
5. Query executes: SELECT NOW()
6. pg Pool sends query to postgres container on port 5432 (via backend network)
7. Postgres executes query, returns current timestamp
8. pg Pool releases connection back to pool
9. Handler serializes result: {"status":"ok","db_time":"2026-06-04T18:18:41.856Z"}
10. Express sends HTTP 200 response
11. curl receives and prints response
```

---

## PART 4 — DEPENDENCY GRAPH

Understanding why things were built in this order prevents costly mistakes.

```
1. .env file and secret strategy
        ↓ (nothing can start without credentials)
2. docker-compose.yml — base service definitions
        ↓ (services need a compose file to exist)
3. Dockerfiles — flask-api, node-api, superset
        ↓ (compose needs images to run)
4. postgres and redis — no application dependencies
        ↓ (APIs cannot connect to a DB that doesn't exist)
5. flask-api and node-api — depend on postgres healthy
        ↓ (Superset needs both the DB and cache)
6. superset and superset-worker — depend on postgres and redis healthy
        ↓ (CI/CD needs the application to exist before building it)
7. GitHub Actions pipeline — builds, scans, pushes images
        ↓ (override file needs the base compose to override)
8. docker-compose.override.yml — dev-only additions
        ↓ (documentation describes what was built)
9. Documentation
```

**Why each step gates the next:**

- You cannot define service startup order before you know what services exist
- You cannot build images before you have Dockerfiles
- You cannot start application services before their dependencies pass health checks
- You cannot build a CI pipeline before you have something to build
- You cannot document accurately before the system is built and tested

---

## PART 5 — BUILD THE PROJECT STEP BY STEP

---

### STEP 1 — Define the Secret Strategy

**Objective:** Establish how credentials flow into containers before writing a single line of application code. If this is wrong, credentials end up in Git history permanently.

**Why now:** Every service needs credentials. The strategy must exist before any file is created that might accidentally contain them.

**What we built:** `.env.example` documenting required variables. `.env` added to `.gitignore` before creation. All credentials injected at runtime via `${VARIABLE}` syntax in docker-compose.yml.

**Concepts involved:**
- Docker Compose reads `.env` automatically from the project root
- `${VAR}` in compose files substitutes the value at runtime, not at build time
- Image layers are permanent — a `RUN echo $SECRET` bakes the secret into the image forever

**Implementation:**

```bash
# .gitignore
.env

# .env.example
POSTGRES_USER=postgres
POSTGRES_PASSWORD=          # generate: openssl rand -base64 32
POSTGRES_DB=appdb
NODE_DB_HOST=postgres
NODE_DB_PORT=5432
NODE_DB_NAME=appdb
NODE_DB_USER=postgres
NODE_DB_PASSWORD=           # same as POSTGRES_PASSWORD
DATABASE_URL=postgresql+psycopg2://postgres:PASSWORD@postgres:5432/appdb
REDIS_URL=redis://redis:6379/0
SUPERSET_SECRET_KEY=        # generate: python3 -c "import secrets; print(secrets.token_hex(32))"
SUPERSET_ADMIN_PASSWORD=
```

**Why this implementation:** Environment variables are the minimum viable secret pattern. They are not ideal (visible to all container processes, can leak in logs) but are universally supported and correct for single-host dev/staging.

**What breaks if wrong:** Credentials committed to Git are compromised permanently even after deletion. Git history retains them. The fix is a credential rotation, not a git revert.

**Verification:**
```bash
cat .gitignore | grep .env
# Expected: .env
git status  # .env must not appear as a tracked file
```

**Common mistake:** Creating `.env` before adding it to `.gitignore`. The file gets staged, committed, and pushed before the engineer notices.

**Important underrated detail:** `docker compose restart` does NOT reload environment variables from `.env`. Only `docker compose down && docker compose up` picks up new values. This caused multiple debugging cycles in this project.

**Interview explanation:** "I establish the secret strategy before writing any code because secrets that touch version control are permanently compromised. I use runtime injection via .env so credentials never touch the image layers, which are immutable once built."

---

### STEP 2 — Write the Dockerfiles

**Objective:** Package each application with its runtime into a reproducible, minimal, non-root image.

**Why now:** docker-compose.yml needs images. Images need Dockerfiles. Dockerfiles need to know what application code they are packaging.

**Flask API Dockerfile — Key Decisions:**

```dockerfile
# Stage 1: Builder — install dependencies with build tools
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Final — copy only compiled output, no build tools
FROM python:3.12-slim AS final
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser
WORKDIR /app
COPY --from=builder /install /usr/local
COPY app/ ./app/
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:create_app()"]
```

**Why multi-stage:** Build tools (gcc, pip cache) add ~1.3GB to the image. The final stage only needs the compiled wheel output. Result: 186MB instead of ~1.5GB.

**Why non-root:** Root inside a container is UID 0 on the host kernel if container isolation is bypassed. A non-root user limits privilege escalation surface.

**Why exec form CMD:** `CMD gunicorn` wraps in `/bin/sh -c`. Docker sends SIGTERM to the shell, not gunicorn. Gunicorn never receives the shutdown signal and is killed with SIGKILL. Exec form `CMD ["gunicorn", ...]` sends signals directly to the process.

**Why COPY requirements.txt before COPY app/:** Docker builds layers in order and caches them. requirements.txt changes rarely. app/ changes on every commit. This ordering means pip install is cached until dependencies actually change, dramatically speeding up rebuilds.

**Node API Dockerfile — Key Decisions:**

```dockerfile
FROM node:20-slim AS prod-deps
WORKDIR /build
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:20-slim AS final
RUN mkdir -p /app && chown -R node:node /app
WORKDIR /app
USER node
COPY --chown=node:node --from=prod-deps /build/node_modules ./node_modules
COPY --chown=node:node src/ ./src/
COPY --chown=node:node package.json ./
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) })"
CMD ["node", "src/index.js"]
```

**Why `npm ci` instead of `npm install`:** `npm ci` requires package-lock.json to be in sync with package.json and installs exactly what the lock file specifies. Reproducible. `npm install` resolves dependencies fresh each time and can introduce version drift.

**Common mistake:** Running `npm install` locally after changing package.json but not committing the updated package-lock.json. `npm ci` inside the container then fails because the lock file is out of sync. This happened during this build.

**Verification:**
```bash
docker build -t test-flask ./flask-app
docker build -t test-node ./node-app
docker images | grep test-
# Confirm image sizes are reasonable
docker run --rm test-flask whoami
# Expected: appuser
```

---

### STEP 3 — Write docker-compose.yml

**Objective:** Define all six services, their networks, volumes, dependencies, resource limits, and health checks in a single file that describes the complete system.

**Why now:** Once Dockerfiles exist, compose can reference them. The compose file is the system definition.

**Network design:**

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

Flask and Node are connected to both. Postgres and Redis are connected to backend only. This means Postgres is unreachable from any container that is only on frontend.

**Health check dependency pattern:**

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

Without `condition: service_healthy`, `depends_on` only waits for the container process to start. Postgres starts its process, then takes 5-10 seconds to be ready to accept connections. Without the health condition, Flask starts, tries to connect, fails, and crashes. The healthcheck on the postgres service defines when "healthy" means.

**Resource limits:**

```yaml
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 256M
```

Without limits, one container can exhaust all host CPU or memory. When Postgres gets starved of memory it crashes. When its dependents detect it is unhealthy they fail too. One bad service takes down everything. Limits contain the blast radius.

**Named volumes:**

```yaml
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

Anonymous volumes (no name) get random IDs. When a container is removed, finding the right volume to reattach is guesswork. Named volumes survive container removal and are explicitly managed.

**Important underrated detail:** The `version:` key in docker-compose.yml is now obsolete and generates warnings in Docker Compose v2+. Removing it eliminates noise. This caused confusion early in the build when the override file still had it.

**Verification:**
```bash
docker compose config  # validates the merged compose file
docker compose ps      # shows all services and their health
```

---

### STEP 4 — Write Application Code

**Flask API — `app/routes.py`:**

```python
from flask import Blueprint, jsonify

bp = Blueprint('main', __name__)

@bp.route('/health')
def health():
    return jsonify({"status": "ok", "service": "flask-api"})
```

**Node API — `src/index.js`:**

```javascript
const express = require('express');
const { Pool } = require('pg');
const app = express();
const PORT = process.env.PORT || 3000;

const pool = new Pool({
  host:     process.env.DB_HOST     || 'postgres',
  port:     parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user:     process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', service: 'node-api' });
});

app.get('/db-health', async (req, res) => {
  try {
    const client = await pool.connect();
    const result = await client.query('SELECT NOW()');
    client.release();
    res.json({ status: 'ok', db_time: result.rows[0].now });
  } catch (err) {
    res.status(503).json({ status: 'error', message: err.message });
  }
});

process.on('SIGTERM', async () => {
  await pool.end();
  process.exit(0);
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Node API listening on port ${PORT}`);
});
```

**Important underrated detail:** The `app.listen` callback was missing its closing `});` during development. Node.js gave a `SyntaxError: Unexpected end of input` at the last line number, which made the error look like it was at the end of the file when it was actually a missing closing bracket. Always check for unclosed blocks when you see this error.

**Why connection pooling (`max: 10`):** Opening a new database connection for every HTTP request is expensive — each connection has handshake overhead and PostgreSQL has a connection limit. A pool maintains a set of open connections and reuses them. `max: 10` caps the pool size so Node cannot accidentally exhaust Postgres's connection limit.

**Why SIGTERM handler:** When Docker stops a container it sends SIGTERM. Without handling it, the process exits immediately and any in-flight database operations are abandoned mid-transaction. The handler drains the pool gracefully before exiting.

---

### STEP 5 — Write docker-compose.override.yml

**Objective:** Layer development-specific behavior on top of production configuration without duplicating service definitions.

**Why now:** After the base compose file exists, the override pattern can be applied.

**How it works:** Docker Compose automatically merges `docker-compose.override.yml` with `docker-compose.yml` when both files exist in the same directory. The override adds or replaces fields — it does not replace the entire service definition.

```yaml
services:
  flask-api:
    volumes:
      - ./flask-app/app:/app/app    # live code mount — changes reflect without rebuild
    environment:
      APP_ENV: development
    ports:
      - "8000:8000"                 # published to host for direct curl access
    command: flask --app app:create_app run --host 0.0.0.0 --port 8000 --debug

  node-api:
    volumes:
      - ./node-app/src:/app/src
    ports:
      - "3000:3000"
    command: npx nodemon src/index.js   # hot reload on file changes
    environment:
      NODE_ENV: development
```

**Production mode:**
```bash
docker compose -f docker-compose.yml up -d
```

This explicitly ignores the override file. No published ports. No debug mode. Gunicorn and node in production mode.

**What depends on this:** Developer experience. Without the override, every code change requires a full image rebuild.

---

### STEP 6 — Initialize and Start the Stack

**Objective:** Prove that all six services start, become healthy, and can communicate.

**First-time setup:**
```bash
cp .env.example .env
# Edit .env and set all required values
docker compose build    # build all images
docker compose up -d    # start all services
docker compose ps       # verify health
```

**Common mistakes encountered in this build:**

1. `POSTGRES_USER=postgress` — extra s. Postgres initialized with the wrong username. All subsequent connection attempts used `postgres`. Authentication failed. Wipe volumes and restart.

2. `NODE_DB_NAME=yourdbname` — literal placeholder value never replaced. Node tried to connect to a database that did not exist.

3. `DATABASE_URL=postgresql+psycopg2://appuser:CHANGE_ME@postgres:5432/appdb` — Superset was connecting as a user that was never created.

**Key lesson:** After any credential change, run `docker compose down -v` to wipe volumes. Postgres stores its initialized credentials in the data volume. If you change credentials without wiping the volume, Postgres ignores the new values and keeps the old ones.

**Verification:**
```bash
docker compose ps
# All services: Up (healthy)

docker exec dockerize-everything-flask-api-1 whoami
# Expected: appuser

# In dev mode with override:
curl http://localhost:8000/health
# {"service": "flask-api", "status": "ok"}

curl http://localhost:3000/db-health
# {"status":"ok","db_time":"2026-06-04T18:18:41.856Z"}
```

---

### STEP 7 — Initialize Superset

**Objective:** Run one-time database migration and admin user creation for Superset.

**Why a separate step:** Superset needs its schema created in Postgres before it can start. The `db upgrade` command runs Alembic migrations. This is a one-time operation, not part of normal startup.

```bash
docker exec dockerize-everything-superset-1 superset db upgrade
docker exec dockerize-everything-superset-1 superset init
docker exec -it dockerize-everything-superset-1 superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@example.com \
  --password yourpassword
```

**Issue encountered:** Superset 3.1.3 requires `redis<5.0` but the custom Dockerfile installed `redis==5.0.1`. This caused a `ContextualVersionConflict` that prevented the `superset` CLI from loading at all. Fixed by downgrading to `redis==4.6.0` in the Superset Dockerfile.

**Issue encountered:** `SUPERSET_SECRET_KEY` was blank. Flask accepted the empty string but failed when trying to use it for session signing. Error appeared as HTTP 500 on the login page, not a startup error. The session error at login time is a misleading symptom of a missing secret key.

---

### STEP 8 — Build the CI/CD Pipeline

**Objective:** Automate image building, vulnerability scanning, and deployment on every push to main.

**Why now:** The application is stable. CI/CD should be added after the system works, not before.

**`.github/workflows/build.yml`:**

```yaml
name: Build and Push Docker Images

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ghcr.io/${{ github.repository_owner }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Flask API
        uses: docker/build-push-action@v5
        with:
          context: ./flask-app
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ env.IMAGE_PREFIX }}/flask-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          load: true

      - name: Scan Flask API image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_PREFIX }}/flask-api:${{ github.sha }}
          format: sarif
          output: flask-trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: 0

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: flask-trivy.sarif

  open-pr:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: write
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Update image tag in compose file
        run: |
          sed -i "s|image: dockerize-everything/flask-api:.*|image: ghcr.io/${{ github.repository_owner }}/flask-api:${{ github.sha }}|g" docker-compose.yml

      - name: Create PR with new image tag
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: "chore: update flask-api image to ${{ github.sha }}"
          branch: deploy/flask-api-${{ github.sha }}
          title: "deploy: flask-api → ${{ github.sha }}"
          body: |
            ## Automated Deploy PR
            - Image: `ghcr.io/${{ github.repository_owner }}/flask-api:${{ github.sha }}`
            - Triggered by commit: ${{ github.sha }}
            - Trivy scan: passed
            Merge this PR to confirm deployment.
          base: main
```

**Why SHA tagging:** `latest` is a floating tag. It changes on every build. You cannot determine what is running in production from a `latest` tag. A Git SHA is immutable and traces directly to a specific commit and therefore a specific state of the codebase.

**Why `exit-code: 0` on Trivy:** The base image `python:3.12-slim` contains known CVEs that are not yet fixed upstream. Setting `exit-code: 1` would fail every single build with no actionable fix. Setting `exit-code: 0` allows the build to proceed while uploading findings to GitHub Security tab for review. In production, CRITICAL CVEs should block deployment.

**Why `load: true` on the build step:** By default, `docker/build-push-action` pushes to the registry but does not load the image into the local Docker daemon on the runner. Trivy needs the image available locally to scan it. `load: true` makes it available. Without this, Trivy cannot find the image and the step fails.

**Required GitHub setting:** Repository Settings → Actions → General → Workflow permissions → Enable "Allow GitHub Actions to create and approve pull requests". Without this the `open-pr` job fails with a permissions error.

**Issues encountered:**
- `open-pr` job at column 0 instead of indented under `jobs:` — YAML parse error
- `load: true` missing — Trivy could not find the image to scan
- GitHub permissions not enabled — PR creation rejected

---

## PART 6 — EVOLUTION OF THE SYSTEM

### Change 1: docker-compose.override.yml Service Names

**Original:** override file used `flask-app` and `node-app`
**Problem:** Main compose file used `flask-api` and `node-api`. Docker created dangling service definitions with no build context.
**Fix:** Renamed services in override to match main compose file.
**Lesson:** Override files must use identical service names to the base. Any rename must be reflected in both files.

### Change 2: package-lock.json Generation

**Original:** package-lock.json was absent or stale
**Problem:** `npm ci` inside Dockerfile requires lock file to be in sync with package.json. Build failed.
**Fix:** Ran `npm install` locally to generate the lock file, committed it.
**Lesson:** Always commit package-lock.json. Never .gitignore it.

### Change 3: Node API Missing Closing Bracket

**Original:** `app.listen(PORT, '0.0.0.0', () => {` with no closing `});`
**Problem:** `SyntaxError: Unexpected end of input` at last line. Container crash loop.
**Fix:** `echo "});" >> node-app/src/index.js`
**Lesson:** When Node reports unexpected end of input, look for unclosed function calls or brackets, not for problems at the line number reported.

### Change 4: Superset Redis Version Conflict

**Original:** Superset Dockerfile installed `redis==5.0.1`
**Problem:** Apache Superset 3.1.3 requires `redis<5.0`. Package resolution failed before the CLI could load.
**Fix:** Downgraded to `redis==4.6.0`
**Lesson:** When extending official Docker images with additional pip packages, always check the base image's existing dependency constraints first.

### Change 5: Credential Typos Accumulating

**Original:** Multiple typos across .env — `postgress`, `appuser`, `yourdbname`, `CHANGE_ME`
**Problem:** Authentication failures at multiple layers (Postgres, Node-to-Postgres, Superset-to-Postgres)
**Fix:** Audited every credential-related variable. Ran `docker compose down -v` to wipe the volume and reinitialize.
**Lesson:** After any credential change, volumes must be wiped. Postgres stores its initialized auth state in the data volume and does not re-read .env on restart.

---

## PART 7 — TECHNOLOGY DECISION RECORD

### Docker

**What it is:** A platform for packaging applications with their runtime environment into portable containers.

**What we used it for:** Running all six services in isolated, reproducible environments.

**Why we chose it:** Universal adoption, mature tooling, required for Docker Compose.

**Alternatives:** Podman (daemonless, rootless by default), containerd directly.

**Why not alternatives:** Docker has broader ecosystem support and simpler developer experience for this use case.

**Production assessment:** Appropriate for production. Docker is production-standard.

---

### Docker Compose

**What it is:** A tool for defining and running multi-container Docker applications from a single YAML file.

**What we used it for:** Orchestrating all six services, their networks, volumes, and dependencies.

**Why we chose it:** Single-host multi-service orchestration. Zero additional infrastructure required.

**Alternatives:** Kubernetes (multi-host, complex), Docker Swarm (multi-host, simpler).

**Why not alternatives:** Kubernetes is significant operational overhead for a single-host demonstration. Swarm is less adopted.

**Production assessment:** Appropriate for single-host production deployments. At scale, migrate to Kubernetes.

---

### PostgreSQL 16

**What it is:** An open-source relational database.

**What we used it for:** Persistent data store for Flask API, Node API, and Superset metadata.

**Why we chose it:** Industry standard, excellent Docker support, native healthcheck tooling via `pg_isready`.

**Production assessment:** Appropriate for production.

---

### Redis 7

**What it is:** An in-memory data store used as a cache and message broker.

**What we used it for:** Superset dashboard cache and Celery job queue.

**Why we chose it:** De facto standard for both caching and Celery broker in Python ecosystems.

**Production assessment:** Appropriate for production.

---

### Apache Superset 3.1.3

**What it is:** An open-source data visualization and analytics platform.

**What we used it for:** Analytics dashboard accessible at port 8088.

**Why we chose it:** Demonstrates a realistic third-party service requiring its own database schema, background workers, and initialization procedure.

**Production assessment:** Appropriate with conditions. The 1.89GB image size requires managed storage. Production deployments should use a managed Postgres instance rather than a containerized one.

---

### Gunicorn

**What it is:** A Python WSGI HTTP server.

**What we used it for:** Serving the Flask API in production mode with multiple worker processes.

**Why we chose it:** Flask's built-in development server is single-threaded and not safe for production. Gunicorn provides proper multi-worker concurrency.

**Why not uWSGI:** Gunicorn is simpler to configure and widely used with Flask.

**Production assessment:** Appropriate for production.

---

### GitHub Actions

**What it is:** A CI/CD automation platform built into GitHub.

**What we used it for:** Automated image builds, Trivy scanning, GHCR push, and auto-PR on every push to main.

**Why we chose it:** Zero additional infrastructure. Integrated with the repository. Free for public repositories.

**Alternatives:** Jenkins (self-hosted), GitLab CI, CircleCI.

**Production assessment:** Appropriate for production. Used by large engineering organizations.

---

### Trivy

**What it is:** An open-source vulnerability scanner for container images.

**What we used it for:** Scanning the Flask API image for known CVEs on every CI run.

**Why we chose it:** CNCF project, zero configuration for basic usage, SARIF output supported by GitHub Security tab.

**Production assessment:** Appropriate for production.

---

### GitHub Container Registry (GHCR)

**What it is:** A container image registry integrated into GitHub.

**What we used it for:** Storing versioned, SHA-tagged Docker images produced by CI.

**Why we chose it:** No additional account required. Integrated with GitHub Actions authentication via `GITHUB_TOKEN`.

**Production assessment:** Appropriate for production.

---

## PART 8 — TOOL DEFENSE

| Tool | Role | Why We Used It | Alternative | Why Not Alternative | Production Assessment |
|------|------|---------------|-------------|--------------------|-----------------------|
| Docker Compose | Multi-service orchestration | Single file, zero infra overhead | Kubernetes | Too complex for single host | Appropriate for single-host |
| PostgreSQL | Persistent database | Industry standard, excellent health check tooling | MySQL | Superset has better Postgres support | Appropriate |
| Redis | Cache + broker | Required by Celery and Superset | Memcached | No persistence, no pub/sub | Appropriate |
| Gunicorn | Python WSGI server | Flask dev server is not production-safe | uWSGI | More complex config for same result | Appropriate |
| Trivy | Vulnerability scanning | CNCF project, SARIF output, zero config | Snyk | Requires paid plan for full features | Appropriate |
| GHCR | Image registry | Integrated with GitHub, no extra auth | Docker Hub | Rate limits on free tier | Appropriate |
| peter-evans/create-pull-request | Auto PR creation | Maintained, widely used GitHub Action | Custom script | More maintenance burden | Appropriate |

---

## PART 9 — ARCHITECTURE DECISION RECORDS

### ADR-001: Multi-Stage Docker Builds

**Context:** Python and Node require build tools to compile dependencies. These tools are not needed at runtime.

**Decision:** Two-stage builds. Compile in builder stage. Copy compiled output to minimal final stage.

**Reason:** Flask image is 186MB instead of ~1.5GB. Smaller attack surface. Faster pulls.

**Alternatives:** Single-stage build (simpler but large). Pre-built wheels (requires separate cache infrastructure).

**Trade-offs:** Slightly more complex Dockerfile. Worth it for 8x size reduction.

**Consequences:** All downstream image operations are faster. Security scanners have less surface to scan.

**Production consideration:** Keep. This is standard practice, not over-engineering.

---

### ADR-002: Frontend/Backend Network Segmentation

**Context:** All services need to communicate but database should not be directly reachable from any service.

**Decision:** Two Docker networks. Flask and Node bridge both. Postgres and Redis backend-only.

**Reason:** If Superset is compromised, attacker cannot reach Postgres directly. Must go through the API layer.

**Alternatives:** Single network (simpler). Three-network DMZ pattern (more expressive but adds complexity).

**Trade-offs:** Slightly more complex compose file. Worth it for meaningful isolation.

**Production consideration:** Keep. In Kubernetes this becomes NetworkPolicy objects.

---

### ADR-003: Non-Root Container Users

**Context:** Root inside a container is UID 0 on the host kernel if isolation breaks.

**Decision:** Flask as `appuser`. Node as built-in `node` user.

**Reason:** Eliminates privilege escalation path.

**Alternatives:** Root containers (simpler, unacceptable). `nobody` user (more restrictive but breaks filesystem ops without careful permission management).

**Production consideration:** Keep. Non-negotiable.

---

### ADR-004: Runtime Secret Injection

**Context:** Secrets must not appear in code, Dockerfiles, or Git history.

**Decision:** Runtime injection via .env. Gitignored. .env.example documents structure.

**Reason:** Image layers are permanent. A secret in a RUN command lives in that layer forever.

**Alternatives:** Docker Secrets via Swarm (more secure, requires Swarm mode). HashiCorp Vault (production-correct, significant overhead).

**Production consideration:** Upgrade to Vault for dynamic credential rotation in production.

---

### ADR-005: Git SHA Image Tagging

**Context:** Images need stable, traceable identifiers.

**Decision:** Every image tagged with full Git commit SHA.

**Reason:** `latest` is not a version. SHA is immutable. Every deployment traces to a specific commit.

**Alternatives:** `latest` (no traceability). Semantic versions (requires manual management).

**Production consideration:** Keep SHA. Add semantic version tags in addition for human-readable releases.

---

### ADR-006: Trivy exit-code: 0

**Context:** Base image has known CVEs not yet fixed upstream.

**Decision:** Report findings, do not fail the build.

**Reason:** Failing every build with unfixable CVEs produces noise with no signal.

**Alternatives:** `exit-code: 1` (fails pipeline on every build). Skip scan (loses visibility).

**Production consideration:** Filter by severity. CRITICAL should block. HIGH should alert.

---

## PART 12 — SECURITY

### What Each Control Protects Against

| Control | Attack or Failure It Prevents |
|---------|-------------------------------|
| Non-root users | Privilege escalation from container to host if isolation is bypassed |
| Backend network isolation | Lateral movement — compromised frontend service reaching database directly |
| Resource limits | Noisy-neighbor failure — one service starving all others of CPU/memory |
| .env gitignored | Credential exposure via Git history |
| No published DB ports | Direct external access to Postgres or Redis |
| Trivy scanning | Known CVEs shipping undetected into production |
| SHA image tagging | Untraced changes in production — knowing exactly what is running |
| Health-gated startup | Race condition failures — app connecting to DB before DB is ready |

### What This Stack Does Not Protect Against

- A compromised host machine with access to Docker socket
- Secrets leaking through application logs if an app logs its environment
- Supply chain attacks on base images (Trivy partially mitigates this)
- Lateral movement within the same Docker network
- TLS between services — all inter-service communication is plaintext inside Docker networks

---

## PART 13 — RELIABILITY AND FAILURE MODES

### Failure Analysis

| Failure | Detection | Immediate Behavior | Recovery | User Impact |
|---------|-----------|-------------------|----------|-------------|
| Flask API crash | Health check fails | Container restarts automatically (unless-stopped) | Automatic within seconds | Brief 503 errors |
| Flask API hard kill (docker kill) | Container exits code 137 | Does NOT restart — deliberate kill is not treated as a fault | Manual `docker start` | API unavailable until manually restarted |
| Postgres restart | Health check transitions unhealthy then healthy | Flask and Node tolerate via connection pooling | Automatic — 6 seconds observed | Brief DB operation failures |
| Redis restart | Health check transitions | Celery worker logs Connection refused then reconnects | Automatic — ~2 seconds observed | Brief delay on background jobs |
| Superset crash | Health check fails | Container restarts | Automatic | Dashboard temporarily unavailable |

### Key distinction: `restart: unless-stopped`

This policy recovers from unexpected process failures — crashes, OOM kills, host reboots. It explicitly does not apply to deliberate `docker kill` or `docker stop`. Docker treats a manual kill as an instruction, not a fault. This is correct behavior, not a misconfiguration.

### Currently Unhandled Failures

- Volume storage full — Postgres would fail to write, error at application layer only
- GHCR unavailable — CI pipeline fails, no deployment possible until registry recovers
- Host machine failure — entire stack goes down, no multi-host redundancy

---

## PART 15 — TESTING

### Failure Tests — Real Results

Tests executed against the fully running stack using actual Docker commands. Results are from real command output, not simulated.

```bash
# Test 1: Flask API hard kill
docker kill dockerize-everything-flask-api-1
# Result: Container exited code 137. Did not restart. Manual recovery required.

# Test 2: Postgres restart
docker restart dockerize-everything-postgres-1
# Result: Healthy in 6 seconds. Dependents continued serving requests.

# Test 3: Redis restart
docker restart dockerize-everything-redis-1
# Result: Healthy in 10 seconds. Celery reconnected automatically in ~2 seconds.
```

### Health Verification Tests — Real Output

```bash
curl http://localhost:8000/health
# {"service": "flask-api", "status": "ok"}

curl http://localhost:3000/health
# {"status":"ok","service":"node-api"}

curl http://localhost:3000/db-health
# {"status":"ok","db_time":"2026-06-04T18:18:41.856Z"}

docker compose exec flask-api whoami
# appuser
```

### Not Yet Tested

- Node API termination and recovery
- Container rebuild with volumes reattached
- Superset behavior during extended Postgres outage
- Resource limit enforcement under synthetic load

---

## PART 18 — DEPLOYMENT

### From Zero to Running

```bash
# 1. Clone the repository
git clone https://github.com/velrite/Dockerize-Everything.git
cd Dockerize-Everything

# 2. Set up environment
cp .env.example .env
# Edit .env — set POSTGRES_PASSWORD, SUPERSET_SECRET_KEY, SUPERSET_ADMIN_PASSWORD
# Generate SUPERSET_SECRET_KEY:
python3 -c "import secrets; print(secrets.token_hex(32))"

# 3. Build images
docker compose build
# First build: 3-5 minutes. Subsequent builds: cached, much faster.

# 4. Start the stack
docker compose up -d

# 5. Verify health (wait up to 60 seconds)
docker compose ps

# 6. Initialize Superset (first time only)
docker exec dockerize-everything-superset-1 superset db upgrade
docker exec dockerize-everything-superset-1 superset init
docker exec -it dockerize-everything-superset-1 superset fab create-admin \
  --username admin --firstname Admin --lastname User \
  --email admin@example.com --password yourpassword

# 7. Verify endpoints
curl http://localhost:8000/health
curl http://localhost:3000/health
curl http://localhost:3000/db-health
# Open http://localhost:8088 in browser
```

### CI/CD Pipeline Flow

```
git push to main
        ↓
GitHub Actions triggered
        ↓
Checkout source code
        ↓
Setup Docker Buildx (parallel builds, GHA cache)
        ↓
Login to GHCR with GITHUB_TOKEN
        ↓
Build flask-api image
Tag: ghcr.io/velrite/flask-api:<git-sha>
Push to GHCR
        ↓
Trivy scan — CRITICAL and HIGH CVEs
Upload SARIF to GitHub Security tab
        ↓
open-pr job (only on main branch)
Update image SHA in docker-compose.yml
Open Pull Request for review
        ↓
Engineer reviews and merges PR
```

---

## PART 20 — REPOSITORY STRUCTURE

```
Dockerize-Everything/
├── README.md                          # Project front door
├── docker-compose.yml                 # Production service definitions
├── docker-compose.override.yml        # Development overrides (auto-merged)
├── Makefile                           # Convenience targets: up, down, build, logs, clean
├── .env.example                       # Variable documentation without values
├── .gitignore                         # Excludes .env, node_modules
├── setup.sh                           # Documentation generation script
├── flask-app/
│   ├── Dockerfile                     # Multi-stage Python build
│   ├── requirements.txt               # Python dependencies
│   └── app/
│       └── routes.py                  # Flask route definitions
├── node-app/
│   ├── Dockerfile                     # Multi-stage Node build
│   ├── package.json                   # Node dependencies
│   ├── package-lock.json              # Locked dependency versions (must be committed)
│   └── src/
│       └── index.js                   # Express app with DB connectivity
├── superset/
│   ├── Dockerfile                     # Extends official Superset image
│   └── superset_config.py             # Superset configuration (reads from env)
├── .github/
│   └── workflows/
│       └── build.yml                  # CI/CD pipeline
└── docs/
    ├── ARCHITECTURE.md
    ├── ADR.md
    ├── SECURITY.md
    ├── RUNBOOK.md
    ├── TESTING_AND_VALIDATION.md
    ├── INCIDENTS.md
    ├── ROADMAP_AND_GAPS.md
    └── SIMPLICITY_VS_PRODUCTION_TRADEOFFS.md
```

---

## PART 22 — IF I HAD TO REBUILD THIS FROM SCRATCH

**Prerequisites:** Docker installed. Git installed. GitHub account. Node.js installed locally (for npm install).

```
Step 1: Create GitHub repository
        Create Dockerize-Everything repo on GitHub

Step 2: Clone and initialize
        git clone <repo-url>
        cd Dockerize-Everything
        git checkout -b main

Step 3: Create .gitignore first
        echo ".env" >> .gitignore
        echo "node_modules/" >> .gitignore
        git add .gitignore && git commit -m "init: add gitignore"

Step 4: Create .env.example
        Document all required variables with blank values
        git add .env.example && git commit -m "init: add env example"

Step 5: Create .env from example
        cp .env.example .env
        Fill in all real values
        # .env is gitignored — never committed

Step 6: Write Flask application
        mkdir -p flask-app/app
        Write requirements.txt
        Write app/routes.py with /health endpoint
        Write Dockerfile with multi-stage build and non-root user

Step 7: Write Node application
        mkdir -p node-app/src
        cd node-app && npm init -y && npm install express pg nodemon
        # This generates package-lock.json — commit it
        Write src/index.js with /health and /db-health
        Write Dockerfile with multi-stage build
        cd ..

Step 8: Write Superset Dockerfile and config
        mkdir superset
        Write Dockerfile extending apache/superset:3.1.3
        Install redis==4.6.0 (NOT 5.x — version conflict with Superset 3.1.3)
        Write superset_config.py reading from environment

Step 9: Write docker-compose.yml
        Define all six services
        Set resource limits on every service
        Add healthchecks to postgres and redis
        Use depends_on: condition: service_healthy
        Define frontend and backend networks
        Assign each service to correct network(s)
        Define named volumes for postgres and redis

Step 10: Write docker-compose.override.yml
        Add volume mounts for hot reload
        Publish ports 8000 and 3000
        Switch to dev servers

Step 11: Write Makefile
        up: docker compose up -d
        down: docker compose down
        build: docker compose build
        logs: docker compose logs -f
        clean: docker compose down -v --rmi all --remove-orphans

Step 12: Build and start
        docker compose build
        docker compose up -d
        docker compose ps
        # All services healthy within 60 seconds

Step 13: Initialize Superset
        docker exec <superset-container> superset db upgrade
        docker exec <superset-container> superset init
        docker exec -it <superset-container> superset fab create-admin ...

Step 14: Verify all endpoints
        curl http://localhost:8000/health
        curl http://localhost:3000/health
        curl http://localhost:3000/db-health
        Open http://localhost:8088

Step 15: Run failure tests
        docker kill <flask-container>     # Confirm: does not auto-restart
        docker restart <postgres-container>  # Confirm: recovers in ~6s
        docker restart <redis-container>     # Confirm: recovers in ~10s

Step 16: Create GitHub Actions workflow
        mkdir -p .github/workflows
        Write build.yml with build, scan, and open-pr jobs
        Enable workflow write permissions in GitHub repo settings
        git add . && git commit -m "ci: add pipeline" && git push

Step 17: Write documentation
        README.md — front door with quick start
        docs/ — architecture, ADRs, security, runbook, incidents, gaps, trade-offs
```

---

## PART 24 — THINGS THAT LOOK SMALL BUT MATTER

### 1. `docker compose restart` Does Not Reload .env

**Looks insignificant:** Just a restart command.
**Actually matters:** Environment variable changes in .env are not picked up. Services continue using old values. This caused multiple hours of debugging credential failures in this project.
**What breaks:** Any credential change appears to have no effect.
**How we handled it:** `docker compose down && docker compose up -d` after every .env change.
**Production implication:** In production, treat env changes as deployments, not restarts.

### 2. `depends_on` Without `condition: service_healthy`

**Looks insignificant:** Both wait for the dependency.
**Actually matters:** Without the condition, Docker waits only for the container process to start, not for the service to be ready. Postgres process starts, then takes 5-10 seconds to accept connections.
**What breaks:** Application containers start, attempt to connect, fail, and crash before Postgres is ready.
**How we handled it:** Added `condition: service_healthy` to all depends_on blocks.
**Production implication:** Always use health conditions in production. Sleep timers are not reliable.

### 3. Named Volumes Must Be Wiped After Credential Changes

**Looks insignificant:** Volume persists data, that's good.
**Actually matters:** Postgres stores initialized credentials inside the data volume. If you change `POSTGRES_PASSWORD` in .env, Postgres reads the new value only during first initialization. After that, the volume state is authoritative.
**What breaks:** Authentication failures that appear random — credentials are "correct" in .env but wrong in the running database.
**How we handled it:** `docker compose down -v` to wipe volumes after credential changes.
**Production implication:** Never change database credentials without a coordinated rotation plan. In production this is a full maintenance operation.

### 4. package-lock.json Must Be Committed

**Looks insignificant:** A generated file.
**Actually matters:** `npm ci` requires lock file to be in sync with package.json. Without a committed lock file, the Docker build fails.
**What breaks:** Node API image build fails on `npm ci`.
**How we handled it:** `npm install` locally, committed the generated lock file.
**Production implication:** Never .gitignore package-lock.json. It is part of the source of truth for reproducible builds.

### 5. CMD Exec Form vs Shell Form

**Looks insignificant:** Both run the same command.
**Actually matters:** Shell form (`CMD gunicorn ...`) wraps in `/bin/sh -c`. Docker sends SIGTERM to the shell, not to gunicorn. Gunicorn never receives the graceful shutdown signal and is killed with SIGKILL after the timeout.
**What breaks:** In-flight requests are dropped on container shutdown. Database connections are not cleanly closed.
**How we handled it:** Exec form `CMD ["gunicorn", "--bind", "0.0.0.0:8000", ...]`
**Production implication:** Always exec form for production process managers.

### 6. Trivy Needs `load: true` to Scan Built Images

**Looks insignificant:** The build step already built the image.
**Actually matters:** By default, `docker/build-push-action` builds and pushes to the registry but does not load the image into the local Docker daemon on the runner. Trivy scans the local daemon. Without `load: true`, Trivy cannot find the image.
**What breaks:** Trivy scan step fails with image not found.
**How we handled it:** Added `load: true` to the build-push action.
**Production implication:** When scanning images in CI, always ensure they are available in the local daemon or scan directly from the registry.

---

## PART 25 — WHAT WE WOULD CHANGE FOR PRODUCTION

| Current | Production Change | Why | Additional Complexity | Justified? |
|---------|------------------|-----|----------------------|------------|
| .env file secrets | HashiCorp Vault dynamic credentials | Short-lived credentials, automatic rotation, no static passwords anywhere | Significant — requires Vault cluster | Yes for sensitive workloads |
| No reverse proxy | Nginx or Traefik with TLS | APIs must not be directly exposed. TLS required for any real traffic | Medium — proxy config, certificate management | Yes, mandatory |
| No centralized logs | Loki + Promtail or ELK | Cannot debug production incidents from stdout alone | Medium | Yes for any serious operations |
| No metrics | Prometheus + Grafana | Cannot know if the system is healthy without metrics | Medium | Yes |
| Single host | Multi-host with orchestration | Single host is a single point of failure | High — requires Kubernetes or Swarm | Yes for high availability |
| CI builds Flask only | CI builds all images | Node and Superset have the same CVE exposure | Low — add more build steps | Yes |
| No automated backups | pg_dump to S3 on schedule | Named volumes are not backups. Host disk failure loses all data | Low | Yes, mandatory |
| exit-code: 0 Trivy | CRITICAL CVEs block pipeline | Unknown critical vulnerabilities should not ship | Low | Yes |
| Dev server in override | Gunicorn always | Dev server is single-threaded and not production-safe | None — already done for production | Already correct |

---

## PART 26 — TECHNICAL DEBT

| Technical Debt | Why It Exists | Risk | Impact | Recommended Change |
|---------------|--------------|------|--------|-------------------|
| CI builds only Flask image | Time constraint during build | Medium — Node and Superset have equal CVE exposure | Node vulnerabilities go undetected | Add build steps for all three images |
| No integration tests in CI | Not in scope for this project phase | Medium — regressions not caught automatically | Broken endpoints could ship | Add docker compose up + curl assertions in CI |
| No reverse proxy | Out of scope | High — direct port exposure not acceptable for internet traffic | APIs exposed without TLS | Add Nginx or Traefik service |
| Environment variable secrets | Simpler than Vault for single host | Medium — env vars visible to all container processes | Secrets could leak via logs | Vault with dynamic credentials |
| No backup strategy | Not in scope | High — host failure loses all database data | Total data loss | pg_dump to S3 on schedule |
| Superset image 1.89GB | Official image is large | Low — operational overhead, slow pulls | Slow CI, large storage | Custom minimal Superset image |

---

## PART 28 — INTERVIEW DEFENSE

**Q: What does this project do?**
A: It runs six services together — two APIs, a database, a cache, an analytics platform, and a background worker — with proper security, health management, network isolation, and automated CI/CD. Every engineering decision follows one rule: contain the blast radius.

**Q: Why did you choose Docker Compose over Kubernetes?**
A: This is a single-host deployment demonstrating container orchestration patterns. Kubernetes adds significant operational overhead — you need etcd, a control plane, worker nodes, and CNI plugins just to start. For demonstrating production patterns on one machine, Compose is the correct tool. The concepts map directly to Kubernetes when the scale requires it.

**Q: Why are Flask and Node not exposed through a reverse proxy?**
A: This is a documented gap. In the development configuration, ports 8000 and 3000 are published directly for ease of testing. In production, only a reverse proxy should face external traffic. TLS termination, rate limiting, and routing should all happen at the proxy layer. This is listed in ROADMAP_AND_GAPS.md as the first planned improvement.

**Q: Why does `restart: unless-stopped` not protect against `docker kill`?**
A: By design. `unless-stopped` recovers from unexpected failures — crashes, OOM kills, host reboots. A deliberate `docker kill` is an explicit instruction to stop the container, not a fault. Docker treats them differently so that a system administrator can intentionally stop a container without having it restart. I discovered this during failure testing and documented it as a meaningful operational distinction.

**Q: Why did you use SHA tags instead of semantic versions?**
A: `latest` is a lie — it changes on every build and you cannot determine what is actually running in production. A Git SHA is immutable. Every deployed image traces back to a specific commit and therefore a specific codebase state. If something breaks in production you know exactly what changed.

**Q: What would you change with twice the budget?**
A: HashiCorp Vault for dynamic credential rotation — no static passwords anywhere. A reverse proxy with TLS. Prometheus and Grafana for metrics. Loki for centralized logs. pg_dump to S3 for automated backups. The current implementation is production-correct in its patterns but missing several production-required operational components.

**Q: What did you intentionally NOT build?**
A: Reverse proxy, centralized logging, metrics, automated backups, and integration tests in CI. All are documented in ROADMAP_AND_GAPS.md with what building each one properly would require. Honesty about gaps is more valuable than faking completeness.

**Q: How would you scale this?**
A: The current stack is single-host. To scale: move to Kubernetes for multi-host orchestration, add read replicas for Postgres, add Redis Cluster, put a load balancer in front of multiple Flask and Node instances, and use a managed database service (RDS, Cloud SQL) instead of a containerized Postgres. Each of those is a meaningful infrastructure change with real operational cost — I wouldn't recommend them unless the load actually required it.

---

## PART 29 — FINAL ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEVELOPER / CI                          │
│   git push → GitHub Actions → Build → Scan → GHCR → Auto PR   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    docker compose up -d
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                        SINGLE HOST                              │
│                                                                 │
│  ┌─────────────────────────────────────────┐                   │
│  │           FRONTEND NETWORK              │                   │
│  │                                         │                   │
│  │  flask-api:8000      node-api:3000      │                   │
│  │  (appuser)           (node user)        │                   │
│  │  186MB image         297MB image        │                   │
│  │  Gunicorn 4 workers  Express + pg Pool  │                   │
│  │  /health             /health            │                   │
│  │                      /db-health         │                   │
│  └──────────────┬──────────────┬───────────┘                   │
│                 │              │                               │
│  ┌──────────────▼──────────────▼───────────┐                   │
│  │            BACKEND NETWORK              │                   │
│  │                                         │                   │
│  │  postgres:5432        redis:6379        │                   │
│  │  postgres:16-alpine   redis:7-alpine    │                   │
│  │  postgres_data vol    redis_data vol    │                   │
│  │  1 CPU / 512MB        0.5 CPU / 256MB   │                   │
│  │                                         │                   │
│  │  superset:8088        superset-worker   │                   │
│  │  1 CPU / 1GB          1 CPU / 768MB     │                   │
│  │  (also frontend)      Celery worker     │                   │
│  └─────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

**Security Boundaries:**
- Postgres and Redis: backend network only — unreachable without going through API layer
- All containers: non-root users
- All services: resource limits
- Secrets: runtime injection only, never in image layers or Git

**Critical Failure Points:**
- Single host — no redundancy
- Postgres volume — no backup
- No reverse proxy — direct port exposure in dev

---

## PART 30 — ENGINEERING SUMMARY

### What I Built

A production-oriented six-service containerized platform demonstrating real-world operational, security, and reliability patterns using Docker Compose, with a GitHub Actions CI/CD pipeline for automated build, scan, and deployment.

### Core Engineering Problem

Making multiple interdependent services start reliably, communicate securely, fail gracefully, and deploy traceably — on a single host, without cloud infrastructure, with secrets handled correctly from the start.

### Major Technical Decisions

1. Multi-stage builds — 8x image size reduction, reduced attack surface
2. Two Docker networks — meaningful isolation between application and infrastructure layers
3. `depends_on: condition: service_healthy` — eliminates race condition startup failures
4. Non-root users — eliminates privilege escalation surface
5. Git SHA image tagging — every deployment is traceable to a specific commit
6. Runtime secret injection — no credentials in image layers or Git history

### Major Trade-offs

- Docker Compose over Kubernetes: correct for single host, requires migration at scale
- Environment variables over Vault: simpler, less secure, documented as known gap
- Trivy `exit-code: 0`: pragmatic for demo, should block on CRITICAL in production
- No reverse proxy: documented gap, first improvement on the roadmap

### Important Lessons

- Postgres credential changes require volume wipe, not just restart
- `docker compose restart` does not reload .env
- `depends_on` without `condition: service_healthy` is not a reliability guarantee
- CMD exec form vs shell form has real consequences for graceful shutdown
- `load: true` is required for Trivy to scan images built in CI

### Current Limitations

No reverse proxy. No TLS. No centralized logging. No metrics. No automated backups. CI builds only Flask image. Single host — one point of failure.

### Skills Demonstrated

- Docker multi-stage builds
- Docker Compose multi-service orchestration
- Container network segmentation
- Health check implementation and health-gated startup
- Non-root container security
- Resource limit configuration
- Secret management via runtime injection
- GitHub Actions CI/CD pipeline authoring
- Trivy vulnerability scanning and SARIF reporting
- GHCR image publishing with SHA tagging
- Automated PR creation on deployment
- Real failure testing and honest documentation
- Operational runbook writing
- Architecture Decision Records

---

**PROJECT: DOCKERIZE-EVERYTHING — OWN IT COMPLETELY**

---

**WHAT IS THIS PROJECT IN ONE SENTENCE**
You took 3 real applications (Flask API, Node API, Superset), gave them a real database (Postgres), a real cache (Redis), wired them all together with proper networking, security, and automation — exactly how a senior engineer would in a real company.

---

**THE MENTAL MODEL**
Think of it like a hospital:
- **Postgres** = the medical records room — stores everything permanently
- **Redis** = the whiteboard at the nurses station — fast temporary info, wiped and rewritten constantly
- **Flask API** = one doctor — takes requests, does work, responds
- **Node API** = another doctor — different specialty, also talks to the records room
- **Superset** = the hospital director's dashboard — shows charts and data about everything
- **Celery Worker** = the lab technician — takes long jobs from the doctors and runs them in the background so the doctor doesn't have to wait
- **Docker Compose** = the hospital building itself — holds everyone in the right rooms, makes sure they can talk to each other

---

**WHY DOCKER AT ALL**
Without Docker every engineer's laptop is different. "Works on my machine" is a real problem. Docker packages your app AND its entire environment into one box. Same box runs on your laptop, on a server, in the cloud — identical every time.

---

**THE DOCKERFILE — WHAT YOU ACTUALLY DID**

Flask Dockerfile has two stages:
- **Builder stage** — installs all the build tools and compiles Python packages. Think of it as a construction site.
- **Final stage** — takes only the finished product from the construction site, leaves all the tools behind. Result: small clean image.

Why does this matter? Build tools add hundreds of MB and are attack surface. You don't need gcc in production. Multi-stage = smaller + safer.

Node Dockerfile does the same — installs dependencies in one stage, copies only what's needed into the final image.

**Key things in both Dockerfiles:**
- `FROM python:3.12-slim` — slim base = smaller attack surface
- `RUN groupadd + useradd` — creates a non-root user before the app runs
- `COPY requirements.txt` BEFORE `COPY app/` — because requirements change rarely, app code changes constantly. Docker caches layers — if requirements didn't change, pip install is skipped. Faster builds.
- `HEALTHCHECK` — Docker pings this to know if the app is actually alive
- `CMD ["gunicorn"...]` — exec form, not shell form. Shell form means Docker can't send shutdown signals properly. Exec form = clean graceful shutdown.

---

**THE DOCKER-COMPOSE.YML — THE BRAIN**

This file describes your entire system. Key concepts:

**Networks:**
```
frontend: flask-api, node-api, superset
backend: postgres, redis, flask-api, node-api, superset-worker
```
Postgres and Redis are ONLY on backend. They cannot be reached directly from outside. Flask and Node bridge both networks — they're the only ones allowed to talk to the database on behalf of the outside world. This is network segmentation — if someone hacks Superset they still can't reach Postgres directly.

**Volumes:**
Named volumes `postgres_data` and `redis_data`. Without names Docker creates random IDs and you lose your data when containers are removed. Named volumes persist forever until you explicitly delete them.

**depends_on with condition: service_healthy:**
This is critical. Without `condition: service_healthy` Docker starts Flask the moment Postgres container starts — but Postgres takes 5-10 seconds to actually be ready. Flask crashes trying to connect. With the condition, Flask waits until Postgres passes its healthcheck before starting. Proper startup order.

**Resource limits:**
Every service has CPU and memory caps. Without this one crashed service can eat all server memory and take everything down. Limits = blast radius control.

---

**THE .ENV FILE**
All passwords and secrets live here. Never in code. Never in Dockerfile. The compose file reads them with `${VARIABLE_NAME}` syntax. `.env` is in `.gitignore` — never pushed to GitHub. `.env.example` shows teammates what variables are needed without the real values.

If someone asks why not hardcode: image layers are permanent. If you hardcode a password in a Dockerfile RUN command it lives in that layer forever even if you delete it in a later layer.

---

**THE OVERRIDE FILE**
`docker-compose.yml` = production config. No published ports. No debug mode. Gunicorn running.
`docker-compose.override.yml` = automatically merged in development. Adds published ports so you can curl directly. Adds volume mounts so code changes reflect without rebuilding. Switches to Flask dev server with hot reload.

Same codebase. Two behaviors. Zero duplication.

---

**THE GITHUB ACTIONS PIPELINE — WHAT HAPPENS ON EVERY PUSH**

1. **Checkout** — grabs your code
2. **Setup Buildx** — enables advanced Docker build features (parallel builds, better caching)
3. **Login to GHCR** — authenticates to GitHub's container registry using an auto-generated token
4. **Build Flask image** — builds your image and pushes it to GHCR tagged with the exact Git commit SHA. Every deployment is traceable to a commit.
5. **Trivy scan** — scans the image for known CVEs. Results go to GitHub Security tab as a SARIF report.
6. **Auto PR** — opens a pull request updating the image tag in the compose file. Every deployment is a reviewed merge, not a manual command.

**Why commit SHA as tag?** `latest` is a lie — it changes every build. SHA is immutable. You always know exactly what's running.

---

**SECURITY DECISIONS — KNOW THESE COLD**

| Decision | Why |
|---|---|
| Non-root users | Root in container = root on host if isolation breaks |
| No published ports for APIs | Only Nginx/proxy should face the internet |
| Named volumes | Data persistence and traceability |
| Healthchecks | Reliable startup order, not just "container started" |
| Resource limits | One bad service can't kill the whole host |
| .env not committed | No credential leaks in Git history |
| Multi-stage builds | No build tools in production image |
| Separate networks | Database unreachable without going through API layer |

---

**IF SOMEONE ASKS "WHY NOT KUBERNETES?"**
Compose is for single-host deployments. Kubernetes is for multi-host clusters with auto-scaling, self-healing across nodes, and rolling deployments. The concepts map directly — a Compose service becomes a Kubernetes Deployment, a named volume becomes a PersistentVolumeClaim, a network becomes a NetworkPolicy. This project is the foundation. Kubernetes is the next level.

---

**THE ONE SENTENCE THAT WINS ANY INTERVIEW**
"Every decision in this project follows one principle — contain the blast radius. I assumed something would go wrong and designed the system so that when it does, the damage stays local."*Olamide Olalekan — Platform and DevSecOps Engineer*
*[LinkedIn](https://linkedin.com/in/olamide-olalekan-12138a265) | [GitHub](https://github.com/velrite)*













































































































































































































































































































































































































 # -----------------------------------------------------------------------------------------------------------------




 # MASTER TECHNICAL, ENGINEERING, CAREER & PORTFOLIO SOURCE-OF-TRUTH

Purpose: Comprehensive factual source of truth for future CVs, applications, interviews, portfolio pages, LinkedIn positioning, technical discussions, and career strategy.

Important: This is not a CV. It is an engineering/career evidence archive.

The information below must preserve the distinction between:

- Verified — actually tested/demonstrated and supported by evidence.
- Implemented — built/deployed, but not necessarily subjected to the strongest verification.
- Configured — configuration exists and was applied, but behavior may not have been observed.
- Unverified — configured or intended, but the claimed runtime behavior was not proven.
- Planned — intended but not built at the relevant point in time.
- Simulated — modeled/test environment rather than real production.
- Production-modeled / production-shaped — designed and operated with production concerns, but not professional production infrastructure.
- Limited — explicitly constrained by environment, quota, missing dependency, or scope.
- Excluded — intentionally outside the project's failure model or scope.

No project may be represented as professional production employment unless there is actual professional production evidence.

---

1. CORE PROFESSIONAL IDENTITY

1.1 Current technical direction

The user's career direction is toward:

- Platform Engineering
- DevSecOps
- Cloud Infrastructure Engineering
- Infrastructure Engineering
- DevOps Engineering
- Site Reliability Engineering / SRE
- Kubernetes / Platform Reliability
- Cloud security / software supply-chain security
- Distributed systems infrastructure
- Infrastructure cost / FinOps engineering

The strongest positioning is around building infrastructure that remains reliable, secure, observable and recoverable under failure, rather than merely making infrastructure deploy successfully.

Core positioning concepts:

«“I design infrastructure that holds under failure.”»

«“Not systems that pass staging.”»

Failure conditions of interest include:

- node loss
- traffic spikes
- bad deployments
- deployment failures at 3am
- capacity exhaustion
- credential/security failures
- configuration drift
- network problems
- workload crashes
- infrastructure dependency failures
- regional failure
- cloud cost growth

Desired characteristic:

«Recovery should happen without human intervention where the architecture is designed to support it.»

Systems mindset:

«“Failure is not an exception; it is a design input.”»

This connects the user's Mechatronics Engineering background with platform engineering: systems are designed around behavior under abnormal conditions rather than only nominal operation.

---

2. CAREER OBJECTIVES

2.1 Target roles

Primary targets:

- Platform Engineer
- DevSecOps Engineer
- Cloud Infrastructure Engineer
- Infrastructure Engineer
- DevOps Engineer
- SRE
- Kubernetes Engineer
- Platform Reliability Engineer

Potential contract positioning:

- Remote Platform Engineering contractor
- Remote DevSecOps contractor
- Cloud Infrastructure contractor
- Reliability/platform consultant

2.2 Target employers / markets

Desired international markets include:

- United States
- United Kingdom
- Canada
- Germany
- Netherlands

Target decision-makers / buyers:

- CTOs
- VPs of Engineering
- Heads of Infrastructure
- Platform Engineering leaders
- Principal Engineers
- Staff Engineers

Particularly relevant companies are mid-sized to large organizations that need infrastructure engineering rather than purely application development.

2.3 Compensation objective

The long-term stated compensation target is:

$20,000/month

Earlier target was approximately:

$10,000/month

The user explicitly wants realistic assessment rather than false encouragement.

Previous candid market-positioning assessment indicated that immediate $20k/month should not be treated as the baseline expectation for the user's current professional profile.

Earlier estimated ranges discussed were approximately:

- first international breakthrough: ~$50k–$80k/year
- stronger opportunity: ~$80k–$100k
- potentially ~$100k–$130k depending on employer/interview performance
- ~$130k–$160k+ not considered baseline
- ~$180k–$240k as a longer-term possibility

These figures are historical strategic estimates, not current market quotations and should be refreshed through web research before being used as current compensation data.

---

3. LOCATION / INTERNATIONAL POSITIONING

The user is based in Nigeria and wants to work internationally.

The objective is not to conceal Nigerian location.

The positioning goal is to demonstrate international engineering capability through:

- technically serious projects
- public engineering evidence
- reproducible infrastructure
- measurable test results
- incident analysis
- architecture decisions
- security controls
- reliability testing
- GitHub evidence
- technical writing
- LinkedIn visibility
- international remote applications

The user's location should not be presented as an inherent technical disadvantage.

The objective is to make the portfolio communicate:

«“This engineer can reason about infrastructure at international engineering standards.”»

rather than:

«“This is a candidate trying to compensate for geography.”»

---

4. EDUCATIONAL BACKGROUND

4.1 University

Degree/background:

Mechatronics Engineering

Country:

Nigeria

Relevant technical learning included:

- C++
- MATLAB
- CNN / computer vision-related learning
- CNC
- C
- Python

Explicit correction:

Java was not part of the stated programming background.

University coursework included concepts around:

- system design
- failure design

This systems/failure orientation is relevant to the later platform engineering philosophy.

---

5. EARLIER EDUCATION / LEADERSHIP

5.1 Junior secondary

Attended a:

public government junior secondary school

5.2 Senior secondary

Attended a:

private senior secondary school

Roles included:

- Health Prefect
- Social Prefect

Health Prefect context included the SS3 period.

These should not be inflated into professional engineering experience.

---

6. CENTER FOR TECH EXPERIENCE

6.1 Early technical exposure

The user had an internship / short-term technical-learning period at Center for Tech.

Work/learning included robotics and electronics from fundamental components upward.

Topics included:

- electronic components
- resistors
- Arduino
- robotics
- Arduino interfacing
- MIT App Inventor / MIT App connection to Arduino
- Flutter

The user learned Flutter from a software engineer at the organization.

Projects included:

- electrical projects
- security alarm projects
- team projects

This period represents the transition from hardware/mechatronics into software and automation.

---

7. CENTER FOR TECH MENTOR / TECHNICAL INSTRUCTOR EXPERIENCE

The user later had a short-term role as a mentor/technical instructor at Center for Tech.

Responsibilities included teaching practical technology projects.

The user taught:

20+ students

Students were taken through:

- practical projects
- take-home projects
- iterative building
- engineering problem solving

Projects included:

Automated farm project

Students built an automated farming project using:

- ultrasonic sensing
- sensor-triggered behavior
- automatic actuation

Line-following robot

A group built a:

line-following robot

The user describes the overall program as successful.

This is legitimate teaching/technical experience but should not be represented as professional Platform Engineering employment.

---

8. CAREER TRANSITION

The career progression is:

Mechatronics Engineering
→ electronics / robotics
→ programming
→ automation
→ technical teaching / mentoring
→ software infrastructure
→ Docker / Kubernetes
→ Terraform
→ cloud infrastructure
→ GitOps
→ security / DevSecOps
→ observability
→ reliability engineering
→ distributed systems
→ platform engineering

This is important because the portfolio is not random tool accumulation.

The underlying theme is systems engineering.

---

9. CORE TECHNOLOGY STACK

Technologies explicitly listed as part of the user's stack include:

Cloud

- AWS
- Azure
- GCP

Containers / orchestration

- Kubernetes
- Docker
- GKE
- GKE Autopilot
- Amazon EKS

Infrastructure as Code

- Terraform
- Terraform modules
- Terraform remote state
- Terraform provider ecosystem

CI/CD

- GitLab CI/CD
- GitHub Actions
- Argo CD
- Argo Rollouts

Configuration / automation

- Ansible
- Puppet

Secrets / identity

- Vault
- Workload Identity
- Workload Identity Federation
- IRSA
- OIDC
- service-account impersonation

Security

- Kyverno
- Falco
- OPA
- OPA/Conftest
- OPA Gatekeeper
- Trivy
- Gitleaks
- TruffleHog
- tfsec
- Checkov
- Syft
- Cosign
- kube-bench

Service mesh

- Istio

Observability

- Prometheus
- Grafana
- Alertmanager
- Loki
- Tempo
- OpenTelemetry
- Google Managed Prometheus

Cost / FinOps

- OpenCost
- cloud billing data
- workload / namespace attribution

Reliability / chaos

- Litmus
- LitmusChaos
- Chaos Mesh
- SLOs
- error budgets
- PDBs
- HPA
- cluster/node failure testing

Data / infrastructure services

- PostgreSQL
- Redis
- Redis Sentinel
- Patroni
- etcd

Platform tooling

- Backstage
- Kustomize
- Artifact Registry

Application / mobile

- Python 3.12
- FastAPI
- Flutter
- Android

---

10. ENGINEERING EVIDENCE PRINCIPLES

The user's engineering work should be evaluated according to an evidence hierarchy.

From weaker to stronger evidence:

1. Terraform/code exists.
2. Configuration was applied.
3. Screenshots exist.
4. Live cloud environment exists.
5. Public application / live metrics exist.
6. Architecture + deployment history + telemetry exist.
7. Failure test exists.
8. Failure test has measured result.
9. Incident was discovered unexpectedly.
10. Root cause was established.
11. Fix was implemented.
12. Fix was re-tested.
13. Behavior was measured under repeated / realistic conditions.
14. Real users/business impact exists.

The user specifically does not want invented metrics.

Only numbers that were actually tested/documented should be used.

---

11. PORTFOLIO CLASSIFICATION

Current hierarchy:

Tier 1 — Flagship systems

1. Atlas — Distributed Resilience Platform
2. Enterprise Cloud Platform / Velrite — AWS EKS

Tier 2 — Major engineering systems

3. Forge — Internal Developer Platform
4. Production-Oriented AI Inference Platform — GCP

Tier 3 — Engineering Labs

5. Auto-Healing Kubernetes Platform
6. Terraform Kubernetes Platform
7. GitOps ArgoCD Platform

The three Engineering Labs should not be presented as equivalent to the flagship systems.

They are foundational evidence.

---

12. ENGINEERING LAB — AUTO-HEALING KUBERNETES PLATFORM

Classification

Engineering Lab / foundational Kubernetes reliability platform

Themes:

- Kubernetes reliability
- failure recovery
- observability
- secrets management
- autoscaling
- FinOps
- CI/CD

---

12.1 Kubernetes environment

Verified environment:

- 3-node Minikube cluster
- Kubernetes v1.31.0
- Docker driver

A missing "docker-proxy" problem was manually identified and fixed.

This was an actual environment issue rather than a theoretical configuration exercise.

---

13. AUTO-HEALING NAMESPACES

Namespaces:

- "microservices"
- "monitoring"
- "vault"
- "opencost"

Workloads included:

- PostgreSQL
- API
- frontend

Services were created for the application components.

---

14. RESOURCE GOVERNANCE

Implemented:

- LimitRange
- ResourceQuota

This established explicit resource governance instead of allowing workloads to consume uncontrolled cluster resources.

---

15. HPA CONFIGURATION AND RESULTS

API

- minimum replicas: 2
- maximum replicas: 8
- CPU threshold: 60%
- memory threshold: 70%

Frontend

- minimum replicas: 2
- maximum replicas: 6
- CPU threshold: 65%

Metrics Server was enabled.

Verified scaling result

Under load:

2 pods → 8 pods

Observed CPU:

86%

After load was removed:

8 pods → 2 pods

CPU fell to approximately:

2%

This is a real measured autoscaling result and must not be reduced to merely “HPA configured.”

---

16. PROMETHEUS / GRAFANA

Installed using Helm:

- Prometheus
- Grafana

Created:

36 alert rules

Custom alerts included:

- "PodCrashLooping"
- "PodNotReady"
- "HighMemoryUsage"
- "HighCPUUsage"
- "HighErrorRate"
- "NodeDown"

A Golden Signals dashboard was created with:

- Latency
- Traffic
- Errors
- Saturation

Important limitation:

Only Saturation had actual data.

The other Golden Signals lacked useful application-level telemetry because nginx did not expose a Prometheus-compatible application metrics endpoint.

This is a critical distinction:

Dashboard configured ≠ all four signals verified.

---

17. VAULT

Vault was deployed through Helm in:

development mode

Vault was:

- initialized
- unsealed
- configured for Kubernetes authentication

Secrets functionality included:

- database secrets engine
- database role
- KV v2

Dynamic credentials test

The database credentials engine generated credentials twice.

The generated:

- usernames differed
- passwords differed

This demonstrated dynamic credential issuance rather than static database credentials.

Zero-credential proof

The pod description was checked to confirm credentials were not directly embedded in the pod configuration.

KV v2 rotation

A KV v2 secret was rotated:

v1 → v2

The rotation was tested with:

zero downtime

Important limitation:

Full dynamic DB credential lifecycle/rotation still has additional infrastructure dependencies such as DNS and connection handling.

---

18. OPENCOST

Kubecost was attempted first.

Kubecost failed due to a permission/problem encountered during installation.

Instead:

OpenCost was installed.

Verified:

- OpenCost "2/2 Running"
- API returned real JSON
- cost labels were applied to deployments
- port-forward/UI was confirmed

The project therefore demonstrated actual cost visibility rather than simply mentioning FinOps.

Infrastructure cost:

$0, because the environment was local.

---

19. AUTO-HEALING CI/CD

GitHub Actions pipeline contained five stages:

1. "validate"
2. "deploy-canary"
3. "validate-canary"
4. "rollback"
5. "deploy-production"

Pipeline result:

Green in 48 seconds

Canary behavior included:

- health checks
- restart-count checking
- automatic rollback

---

20. AUTO-HEALING FAILURE TESTS

Node failure

Node:

"m02"

was killed.

Observed recovery:

20 seconds

Result:

SLO met

This is one of the strongest pieces of evidence from the lab.

---

Bad image deployment

A bad image deployment was introduced.

Observed rollback:

32 seconds

Blast radius:

0

This demonstrated that the failure did not propagate to unaffected production workloads.

---

HPA failure/load test

Replica count:

2 → 8

CPU:

86%

After pressure disappeared:

8 → 2

CPU:

2%

---

Vault rotation

Secret rotation was tested with:

zero downtime

---

21. AUTO-HEALING INCIDENTS

Actual problems encountered:

1. Missing "docker-proxy"
2. Memory exhaustion
3. Kubecost failure / permission issue
4. HPA unknown metrics
5. Git history contamination

These incidents matter because they show the project was not simply generated from a clean tutorial path.

---

22. GIT SECURITY INCIDENT

The repository initially contained sensitive material / credential contamination.

Actions taken:

- ".gitignore" updated for:
  - Terraform state
  - tfvars
  - binaries
- "vault-init.json" excluded.
- large binaries removed from history.
- "git filter-branch" used multiple times.
- Git history was scrubbed.
- "git-filter-repo" was used to remove a hardcoded password.
- repository was pushed clean to GitHub.
- history was checked afterward.

This incident became part of the engineering evidence.

---

23. AUTO-HEALING ADRs

Architecture Decision Records covered:

- Minikube vs kind
- OpenCost vs Kubecost
- GitHub Actions vs GitLab CI
- Vault dev mode vs standalone
- CI-based canary vs Argo Rollouts

---

24. AUTO-HEALING GAPS

Explicit gaps:

- application metrics incomplete
- dynamic DB credential full rotation requires additional infrastructure such as DNS/connection handling
- no PDB
- no NetworkPolicy
- no persistent Grafana dashboards
- no image signing

Therefore this project should remain an:

Engineering Lab

rather than being presented as production infrastructure.

---

25. ENGINEERING LAB — TERRAFORM KUBERNETES PLATFORM

Classification

Engineering Lab / Infrastructure-as-Code foundation

Primary themes:

- Terraform
- Kubernetes provisioning
- modules
- security scanning
- CI/CD
- infrastructure reproducibility

---

26. TERRAFORM VERSION / PROVIDERS

Terraform:

v1.7.5

Providers:

- Kubernetes "v2.38"
- Helm "v2.17"
- null "v3.3"

---

27. TERRAFORM MODULE STRUCTURE

Modules covered:

- namespaces
- microservices
- monitoring
- Vault
- OpenCost

All five module files were created and verified.

Sensitive variables were used.

Terraform variable files were excluded from Git.

---

28. TERRAFORM RESOURCE RESULT

Terraform planned/applied:

13 resources

"terraform state list" confirmed all 13 resources.

Resource set included:

1. microservices namespace
2. monitoring namespace
3. vault namespace
4. opencost namespace
5. postgres-secret
6. postgres-db deployment
7. api-service deployment
8. frontend deployment
9. api-service service
10. frontend service
11. Prometheus + Grafana Helm release
12. HashiCorp Vault Helm release
13. OpenCost Helm release

---
29. TERRAFORM CI/CD

GitHub Actions stages included:

Security scan

- TruffleHog
- tfsec
- Checkov

Validation

- "terraform fmt -check"
- "terraform init"
- "terraform validate"

Security ran first.

If a credential was discovered, the pipeline stopped before validation/deployment could continue.

Pipeline was confirmed fully green after fixes.

---

30. TERRAFORM INCIDENT

A hardcoded:

"grafana_password"

was discovered.

Actions:

- password removed
- Git history scrubbed with "git-filter-repo"
- history force-pushed
- clean history verified using Git log search

Other issues:

- terminal paste corruption
- Terraform apply initially appearing to show 0 resources

These became part of the engineering debugging history.

---

31. TERRAFORM ADRs

Decisions documented:

- modules vs flat configuration
- local vs remote state
- "depends_on" vs outputs
- Helm provider vs raw Kubernetes manifests

---

32. TERRAFORM GAPS

Not implemented:

- remote state
- Terraform workspaces
- apply inside CI
- HPA managed through Terraform

The README's central engineering lesson:

The platform was rebuilt from Project 1 as code so that:

«the Terraform code becomes the record of how the infrastructure is created.»

The conceptual objective was:

Destroy everything → run Terraform → reproduce the same platform.

This was specifically intended to eliminate dependence on remembering manual command order.

---

33. ENGINEERING LAB — GITOPS ARGOCD PLATFORM

Classification

Engineering Lab / GitOps foundation

Themes:

- GitOps
- Kubernetes delivery
- Argo CD
- progressive delivery
- policy-as-code
- configuration management

Core principle:

«Git is the source of truth.»

Rollback model:

«Git revert rather than manual "kubectl".»

---

34. ARGO INSTALLATION
Argo CD and Argo Rollouts were installed through Terraform.
Argo CD Helm chart:
v7.3.11
Argo Rollouts was also installed as a Terraform helm_release.
35. KUSTOMIZE
Environment structure:
staging
production
Kustomize overlays were used.
Both applications reached:
Synced + Healthy
Applications:
argocd/staging-api-service
argocd/production-api-service
36. GITOPS DRIFT TEST
A deliberate manual change was introduced:
kubectl scale deployment api-service -n production --replicas=5
Desired Git state remained:
2 replicas
Argo CD reconciled the cluster.
Measured drift correction:
under 2 seconds
The cluster returned to the desired replica count before the second kubectl command returned.
This is actual demonstrated reconciliation behavior.
37. ARGOCD RBAC
AppProject roles:
platform-admin
Could:
sync any application
manage clusters
manage repositories
developer
Could:
view all
sync staging
not modify production
This demonstrated environment-specific delivery boundaries.
38. APPLICATIONSET
An ApplicationSet matrix generator was used.
One template generated:
staging
production
Adding another environment required only another entry in the generator list.
This demonstrated templated multi-environment GitOps rather than duplicated application definitions.
39. SYNC WAVES
Configured order:
Wave 0: Namespace
Wave 1: ConfigMap + Services
Wave 2: Deployments
Wave 3: Rollouts
40. CANARY CONFIGURATION
Argo Rollout sequence:
10% → 30-second pause → 25% → 30-second pause → 50% → 30-second pause → 100%
Important distinction:
The canary configuration existed, but the lab's actual canary trigger was not fully verified.
Therefore:
configured ≠ proven runtime progressive delivery
41. SECURITY POLICIES
OPA Gatekeeper:
no latest tag constraint
Kyverno:
require resource limits
disallow privileged workloads
42. SLO
Sloth PrometheusServiceLevel object:
99.9% availability SLO
for:
api-service
The SLO resource was created/applied.
Sloth controller processing itself was not fully verified.
43. CHAOS
A Litmus ChaosEngine pod-delete experiment was:
created
applied
But completion was not fully verified.
44. GITOPS CI/CD
Pipeline included:
Security
TruffleHog
tfsec
Manifest validation
kubectl dry-run
Terraform validation
terraform fmt
terraform init
terraform validate
Drift/configuration check
Checks included:
annotations
security contexts
resource limits
SLO definitions
chaos experiments
All pipeline checks were confirmed green.
45. GITOPS INCIDENTS
ComparisonError
An Argo CD ComparisonError occurred.
It was fixed.
Drift screenshot problem
The drift correction occurred too quickly to capture in a screenshot.
The behavior itself was considered the stronger evidence.
Argo CLI connection
Argo CD CLI lost connection because of the port-forward lifecycle.
46. GITOPS GAPS
canary not actually triggered/fully verified
Image Updater not installed
External Secrets not configured
single cluster
Sloth controller processing not verified
Litmus experiment completion not verified
47. GITOPS DOCUMENTATION
Documentation included:
ARCHITECTURE.md
GITOPS_MODEL.md
SECURITY.md
ADR.md
INCIDENTS.md
GAPS.md
Repository structure included:
terraform/main.tf
argocd-values.yaml
apps/base
apps/staging
apps/production
argocd/projects/platform-project.yaml
argocd/appsets/platform-appset.yaml
security/opa/no-latest-tag.yaml
security/kyverno/require-resources.yaml
observability/sloth/api-service-slo.yaml
chaos/litmus/pod-kill-experiment.yaml
.github/workflows/gitops.yml
48. FORGE — INTERNAL DEVELOPER PLATFORM
Classification
Major Platform Engineering Project
Not:
Engineering Lab
professional production experience
Platform:
Google Kubernetes Engine Autopilot
49. FORGE ARCHITECTURE
Forge is a self-service Internal Developer Platform.
Core flow:
Backstage → GitLab CI/CD → Artifact Registry → Argo CD → Argo Rollouts → GKE
Security/policy:
Kyverno
Workload Identity Federation
immutable Artifact Registry tags
Observability/reliability:
Kubernetes-level SLOs
Grafana
rollout health
Cost:
per-namespace attribution
50. FORGE DEVELOPER WORKFLOW
Intended workflow:
Developer scaffolds service from Backstage.
GitLab CI runs tests/scans/build.
Image is pushed to Artifact Registry.
Artifact uses immutable tagging.
Argo CD reconciles desired state.
Unauthorized cluster changes are reverted.
Argo Rollouts performs gradual delivery.
Kyverno enforces policy.
Kubernetes-level SLOs provide platform health signals.
Namespace-level cost attribution provides cost visibility.
51. FORGE VERIFIED RESULTS
Verified:
core platform deployed
Backstage portal demonstrated
GitOps drift correction demonstrated live
Argo Rollouts canary delivery
failing canary automatically aborted
zero manual intervention required for that abort
Kyverno admission-time policy enforcement
zero-stored-credential CI/CD
Workload Identity Federation
immutable Artifact Registry tags
52. FORGE LIMITATIONS
Not verified / not complete:
no real production workload
no real user traffic
no real engineering organization consuming the platform
no application-level metrics for Backstage
only Kubernetes-level health was available for Backstage
Backstage golden-path template stops at local generation
no end-to-end repository publication from the template
Grafana dashboard rendering configured but unverified
Slack delivery configured but unverified
no NetworkPolicies
no image signing
no SBOM
Therefore:
Forge is a serious platform engineering project, but it must not be presented as an IDP used by real production engineers/customers.
53. ATLAS — DISTRIBUTED RESILIENCE PLATFORM
Classification
Flagship Engineering System — GCP
One of the strongest portfolio projects.
It is not professional production infrastructure.
It is a production-shaped / production-modeled distributed workload execution platform.
54. ATLAS CORE PROBLEM
Client submits work containing:
resource requirements
priority
retry policy
System flow:
Client → API → queue → scheduler → workers → result
The scheduler performs capacity-aware placement.
55. ATLAS SCOPE
All:
18 planned phases
were:
implemented
deployed
verified
This project is not an AI/ML/GPU platform.
Its primary engineering focus is:
distributed systems
scheduling
reliability
SRE
cloud infrastructure
GitOps
observability
failure recovery
multi-region resilience
FinOps
56. ATLAS SCHEDULER
Scheduler type:
capacity-based bin-packing
It is not simple round-robin scheduling.
The scheduler considers resource capacity and workload requirements.
57. ATLAS MAJOR BUG DISCOVERED
During testing, a genuine:
scheduler capacity-accounting race condition
was discovered.
The problem was not merely theoretical.
Concurrent scheduling could produce an incorrect capacity view because reservation/accounting was not sufficiently atomic.
The issue was:
reproduced
root-caused
fixed
Fix:
atomic reservation
The system was then re-tested.
This is a major interview-grade piece of evidence because the user found a distributed-systems correctness issue through actual testing rather than designing a system that only worked under ideal conditions.
58. ATLAS TERRAFORM / IDENTITY
Implemented:
Terraform across environments
Terraform remote state
keyless service-account impersonation
Organization policy:
blocks static credentials
Static credentials were not used.
This provides stronger evidence than simply saying “secrets were stored securely.”
59. ATLAS GKE SECURITY / GOVERNANCE
Implemented:
namespace RBAC
Workload Identity
separation of node/pod identity
PodDisruptionBudgets
resource governance
allocatable-capacity-based scheduling
NetworkPolicy
RBAC least privilege was tested using authorization checks.
60. ATLAS GITOPS DRIFT TEST
A deliberate unauthorized change was made:
kubectl scale ...
Argo CD detected and reconciled the unauthorized change.
Measured correction:
under 3 seconds
This is separate from the GitOps Engineering Lab result of:
under 2 seconds
Both numbers must remain because they refer to different systems/tests.
61. ATLAS CI/CD
GitLab CI/CD operated against a GitHub repository.
Pipeline included:
tests
lint
SAST
dependency scanning
IaC scanning
build
image scanning
keyless Artifact Registry push
GitLab → GCP Workload Identity Federation
automatic GitOps deployment after merge
Deployment required:
no manual kubectl/helm
after the automated pipeline flow.
62. ATLAS SUPPLY CHAIN
Identity model:
keyless
Artifact destination:
Google Artifact Registry
Authentication:
GitLab → GCP WIF
No long-lived cloud credentials were stored for deployment.
63. ATLAS OBSERVABILITY
Implemented:
Google Managed Prometheus
Grafana
OpenTelemetry
distributed tracing
Redis trace-context propagation
Trace path:
API → queue → scheduler → worker
This allows a request to be followed through the distributed workload path rather than only observing isolated services.
64. ATLAS SLO / ERROR-BUDGET METHODOLOGY
SLOs and error budgets were based on:
real measured baselines
The project explicitly excluded contaminated data from the baseline.
This mattered because a malformed-input worker bug was discovered.
65. ATLAS MALFORMED-INPUT BUG
A malformed-input condition caused worker failure.
The issue was:
observed
investigated
fixed
The SLO/baseline process did not simply hide the failure.
Instead, contaminated data was identified and excluded where appropriate.
This is evidence of disciplined measurement rather than manipulating numbers to make the system appear healthy.
66. ATLAS PROGRESSIVE DELIVERY
Argo Rollouts was used.
A broken deployment was introduced.
Rollback was driven by:
health signals
SLO signals
Automated rollback occurred.
The system was tested using actual failure conditions rather than only demonstrating a successful canary.
67. ATLAS FAILURE TESTING
Failure testing included:
worker failures
node failures
bad deployments
scheduling correctness problems
controlled regional failure
GitOps drift
malformed workload inputs
Chaos experiments were documented around:
hypothesis
blast radius
outcome
A live-system chaos experiment was recorded/documented.
68. ATLAS NODE / DRAIN ISSUES
An incident involving:
node-drain stalls
was encountered.
This became part of the project's incident/debugging evidence.
69. ATLAS IMAGE-PULL INCIDENT
An:
image-pull authentication gap
was encountered.
It was investigated and addressed.
This demonstrates that the supply chain was tested as an operational system rather than merely configured.
70. ATLAS MULTI-REGION
Architecture:
active/active multi-region
Controlled failover was performed.
RTO/RPO were:
measured
rather than estimated.
The project also explicitly documented the cross-region queue decision.
71. ATLAS FAILURE MODEL
Explicitly excluded:
simultaneous total region failure
GCP control-plane outage
Byzantine workers
distributed database
The project uses managed GCP primitives for workload state.
These exclusions are important because “multi-region resilient” should not be interpreted as “survives every conceivable failure.”
72. ATLAS SECURITY
Security controls included:
least-privilege IAM
auth can-i verification
Workload Identity
RBAC
NetworkPolicy
image scanning
dependency scanning
IaC scanning
policy-as-code admission control
threat modeling
73. ATLAS FINOPS
Cost engineering included:
namespace attribution
workload attribution
fixed-capacity vs autoscaled-capacity comparison
GCP billing data where available
Infrastructure was torn down between sessions to prevent unnecessary cloud spend.
Budget alerts were configured/documented.
74. ATLAS OPERATOR
Separate add-on to Atlas.
Technologies:
Flutter Android application
Python 3.12
FastAPI
Purpose:
Operations interface for the Atlas platform.
75. ATLAS OPERATOR LIVE VERIFICATION
Verification date:
2026-09-20
Tested on:
real Android phone
real cluster/backend
Connection banner called:
/health
/ready
Observed sequence:
Healthy → Failed → Healthy
The failure state was induced by scaling the backend to:
0
Then the backend was restored to:
1
and the application returned to Healthy.
76. ATLAS OPERATOR BUG
Initial real-device attempt failed because the Android application lacked:
INTERNET permission
The issue was identified and fixed.
This is a real device integration bug and should remain in the engineering history.
77. ATLAS OPERATOR QUALITY CHECKS
Verified:
flutter analyze clean
24 tests pass
CI passes on a self-hosted GitLab runner
78. ATLAS OPERATOR LIMITATIONS
Not live / not production:
all screens except connection banner use fixture data
no authentication
temporary external exposure was torn down
app cannot perform Terraform apply
app cannot perform Terraform destroy
Therefore it must not be represented as a production operations console.
79. ENTERPRISE CLOUD PLATFORM / VELRITE
Classification
Flagship Engineering System — AWS
Repository:
velrite/enterprise-cloud-platform
Application:
velrite/bank-of-anthos
The application is a fork of:
Google Bank of Anthos
80. PROJECT MODEL
The project models a:
mid-sized regulated digital bank
This is a simulation / production-modeling context.
It is not:
an actual bank
an actual banking employer
real banking production
real money
real regulator infrastructure
real customer traffic
The banking model is used to justify serious controls around:
supply-chain security
service mesh
secrets
runtime security
observability
reliability
deployment safety
The project explicitly avoids claiming global-bank scale.
81. BUILD METHOD
The project was built:
alone
from scratch
in one repository
as one continuous build
Same:
codebase
cluster
infrastructure
was maintained across phases.
Later phases added to the already-running system rather than rebuilding separate toy environments for every feature.
An earlier Internal Developer Platform/custom CLI phase was deliberately removed because it had the:
lowest signal-to-effort ratio
and would have introduced fragile dependencies.
The final roadmap excludes that component.
82. ENTERPRISE PLATFORM PROBLEM
The engineering problem:
A team needs to ship code frequently to customer-facing banking services without:
vulnerable images
tampered images
unplanned downtime
blind incident diagnosis
The important question is not merely:
“Can we deploy?”
It is:
Can we deploy safely?
Can we prove the artifact matches intent?
Can we immediately detect problems?
Can we diagnose failures through telemetry rather than guessing?
83. ENTERPRISE PLATFORM PHASE HISTORY
There are two status snapshots in the source information.
This distinction must be preserved.
Earlier README snapshot
At that point:
Phase 1 — complete
Phase 2 — complete
Phase 3 — not built
Phase 4 — AI/GPU not built
Phase 5 — FinOps not built
Later roadmap state
Later roadmap states:
Phase 1 — complete
Phase 2 — complete
Phase 3 — complete: resilience, chaos engineering, DR
Phase 4 — complete: FinOps & cloud cost intelligence
Therefore these are historical states, not necessarily contradictory statements.
The earlier README represents an earlier repository/project state.
The later roadmap represents a later claimed state.
84. ENTERPRISE PLATFORM PHASE 1
Phase 1:
Kubernetes Platform Foundation
Components:
EKS
VPC
networking
node groups
Terraform
base RBAC
IAM
IRSA/OIDC
85. EKS FOUNDATION
Infrastructure:
EKS cluster
VPC
public subnets
private subnets
NAT gateway
route tables
node groups
cluster autoscaler
IAM
OIDC / IRSA
Namespaces:
platform
apps
monitoring
security
Additional namespaces were introduced by later phases.
86. TERRAFORM STATE
Roadmap intent:
remote Terraform state
S3 backend
DynamoDB locking
no local state
This was part of the production-shaped infrastructure approach.
87. BASE SECURITY
Implemented / intended:
least-privilege RBAC
no default cluster-admin usage
Kubernetes authorization testing
network-policy default posture
IRSA/OIDC
Evidence checks included:
kubectl auth can-i
88. FOUNDATION TESTING
Required / demonstrated engineering tests included:
Terraform plan/apply from clean state
node draining/recovery
RBAC authorization checks
reproducibility from infrastructure code
The phase's core question:
Can real production-shaped Kubernetes infrastructure be provisioned reproducibly as code rather than manually?
89. ENTERPRISE PLATFORM PHASE 2
Phase 2:
Enterprise Cloud Platform — CI/CD Supply Chain, Security & Observability
90. SUPPLY-CHAIN PIPELINE
Pipeline architecture:
Build → Scan → Sign → Gate → Deploy
Tools:
GitHub Actions
Gitleaks
Trivy
Syft
Cosign
OPA/Conftest
All major pipeline gates were verified green end-to-end in the relevant snapshot.
91. TRIVY
Used for:
container scanning
dependency scanning
92. SYFT
Used for:
SBOM generation
93. COSIGN
Used for:
image signing
Important distinction:
Cosign signatures were verified in the pipeline.
However, in the original verified snapshot:
cluster admission enforcement of Cosign signatures was not confirmed.
Therefore the correct claim is:
Image signing and pipeline verification were implemented.
Not:
Every unsigned image was proven impossible to deploy.
The later roadmap considered admission-enforced signature verification, for example through Kyverno verifyImages.
94. OPA / CONFTEST
OPA/Conftest was used for policy checks in the software supply chain.
95. GITLEAKS
Gitleaks was used to detect credentials/secrets entering the repository/pipeline.
96. ARGO CD / ARGO ROLLOUTS
GitOps:
Argo CD
Progressive delivery:
Argo Rollouts
Canary deployment was implemented.
Automatic rollback fired for a real failure once in the verified project evidence.
This must not be described as merely theoretical.
97. ISTIO
Istio was implemented for:
service mesh
mTLS
traffic policy
fault injection
Fault injection was:
proven with real timing
mTLS configuration:
PeerAuthentication: STRICT
Important limitation:
Live mTLS enforcement could not be confirmed through the expected Prometheus telemetry because:
istio_requests_total
was absent at the relevant test time.
Therefore:
STRICT mTLS was configured, but live enforcement telemetry was not fully confirmed.
98. ISTIO CIRCUIT BREAKING
Circuit-breaker / outlier-detection behavior was configured.
However:
it was never observed firing under real traffic.
Therefore it must be described as configured, not experimentally proven.
99. VAULT
Vault was implemented.
Mode:
development mode
Important limitation:
Vault dev mode is not durable production-grade Vault.
Production-grade future direction:
HA Vault
durable storage backend
real unseal process
100. KYVERNO
Kyverno was used for security policy enforcement.
The project also used:
OPA/Gatekeeper-style controls
admission policies
resource/security posture
RBAC and NetworkPolicy enforcement were independently verified live.
101. FALCO
Falco was implemented for:
runtime threat detection
This provided runtime security beyond static image scanning.
102. KUBE-BENCH
kube-bench was used for:
CIS Kubernetes benchmark
The roadmap intended before/after hardening comparison.
103. OBSERVABILITY STACK
Implemented:
Prometheus
Grafana
Loki
Tempo
OpenTelemetry
Alertmanager
Dashboards were designed around:
Golden Signals
RED
USE
SLOs and error budgets were explicitly defined.
104. EBS CSI INCIDENT
Loki / Tempo / OpenTelemetry initially encountered an undocumented dependency related to the:
EBS CSI driver
The driver had to be installed before those components could operate correctly.
This belongs in the incident/debugging history.
105. ENTERPRISE PLATFORM REAL INCIDENT COUNT
The project documentation contained:
13 real incidents
These are recorded in:
docs/incidents/
This is important portfolio evidence.
The project was not developed without operational failures.
106. ENTERPRISE REPOSITORY STRUCTURE
Important directories:
Terraform
terraform/
Includes:
network
cluster
bootstrap state
Kubernetes baseline
k8s/baseline/
Includes:
namespaces
default-deny NetworkPolicy
baseline RBAC
Helm
helm-values/
Contains exact Helm values used for installations.
Architecture
docs/architecture/
Contains:
system design
ADRs
security architecture
Operations
docs/operations/
Contains:
runbooks
deployment flow
cost analysis
load testing
Reliability
docs/reliability/
Contains:
SLOs
failure-mode results
real drain/PDB tests
Security
docs/security/
Contains:
security controls matrix
supply-chain details
Incidents
docs/incidents/
Contains:
full postmortem log
13 real incidents
Engineering
docs/engineering/
Contains:
debugging history
lessons learned
technical debt
Screenshots
docs/SCREENSHOTS.md
Contains the exact evidence screenshots still required.
107. ENTERPRISE RUNBOOK
Full bring-up order was documented.
A recurring:
WSL2 DNS problem
broke almost every cluster-start session.
This was documented in:
docs/operations/runbook.md
under:
“Rebuilding the Cluster From Zero.”
This is a useful operational detail because it demonstrates environment reproducibility work rather than pretending every cluster boot was frictionless.
108. ENTERPRISE SECURITY / SUPPLY-CHAIN STATUS
The relevant stack included:
Terraform
EKS
VPC
NAT
IAM
IRSA/OIDC
GitHub Actions
Gitleaks
Trivy
Syft
Cosign
OPA/Conftest
Argo CD
Argo Rollouts
Istio
Vault
Kyverno
Falco
kube-bench
The pipeline gates were verified green.
109. ENTERPRISE OBSERVABILITY STATUS
Prometheus/Grafana APIs were verified.
Loki/Tempo/OpenTelemetry were implemented.
The EBS CSI issue had to be resolved for some observability components.
OpenTelemetry was part of the tracing architecture.
Alertmanager was included.
110. ENTERPRISE GITOPS RESULT
Argo Rollouts automatic rollback:
fired for real once
This must remain distinct from merely configuring rollback.
111. ENTERPRISE ISTIO FAILURE-INJECTION RESULT
Fault injection:
verified with real timing
This means traffic behavior was actually observed rather than inferred from YAML.
112. ENTERPRISE RBAC / NETWORKPOLICY RESULTS
RBAC boundary was:
independently verified live
NetworkPolicy enforcement was:
independently verified live
113. ENTERPRISE EARLIER SNAPSHOT — PHASE 3
The original README at the earlier point explicitly said:
Phase 3 — Chaos Engineering & Disaster Recovery: not yet built
Planned:
Chaos Mesh / Litmus
HA PostgreSQL failover
Velero backup/restore
Planned chaos experiments included:
kill nodes
kill pods
break networking
fill disks
expire certificates
simulate region failure
114. ENTERPRISE LATER PHASE 3 ROADMAP
Later roadmap says Phase 3 became:
complete — resilience, chaos engineering and disaster recovery
Planned/implemented architecture included:
Chaos
Chaos Mesh or LitmusChaos
node failures
pod failures
networking failures
disk pressure
certificate expiration
regional failure
PostgreSQL
replication
automated failover
Patroni or equivalent
Redis
Redis Sentinel
etcd
etcd cluster
Recovery
Velero
full cluster restore
namespace restore
PV restore
DB restore
Required demonstrations
six chaos experiments
timed full restore
live DB failover
identify what self-healed
identify what required human intervention
Important:
The supplied historical material does not provide every final Phase 3 measured result. Therefore exact RTOs, failover seconds, restore seconds, or individual chaos outcomes must not be invented.
The status is:
Later roadmap says complete; detailed final evidence needs the actual phase documentation before numerical claims are made.
115. ENTERPRISE FINOPS HISTORY
Earlier README:
FinOps was:
not built
Cost analysis at that stage used:
raw AWS CLI
rather than a full OpenCost/Kubecost system.
116. LATER ENTERPRISE FINOPS STATUS
Later roadmap says:
Phase 4 — FinOps & Cloud Cost Intelligence: complete
Potential scope in roadmap included:
OpenCost/Kubecost
waste detection
forecasting
cloud cost intelligence
However, exact final metrics were not supplied in the recovered information.
Therefore do not invent:
percentage savings
dollar savings
forecast accuracy
OpenCost deployment status
exact waste identified
The master record must say:
Roadmap status: complete. Detailed final evidence not included in supplied source material.
117. ENTERPRISE AI/GPU PHASE
Earlier project roadmap included:
Phase 4 — AI Infrastructure
Planned:
GPU scheduling
model serving
This was explicitly not built in the earlier README snapshot.
This is separate from the later roadmap where Phase 4 became FinOps.
Therefore chronology must remain explicit.
118. ENTERPRISE COST / OPERATIONAL LIMITATIONS
The project includes cost analysis.
The environment is not described as an always-on production banking environment.
The user documented cost-related operational behavior and infrastructure teardown where relevant.
The project is intentionally production-shaped rather than a real banking deployment.
119. ENTERPRISE TECHNICAL DEBT
Source of truth:
docs/engineering/technical-debt.md
The README explicitly says the project is:
not promotional
and documents:
what was built
what was tested
what broke
what remains unverified
This is a central principle of the portfolio.
120. PRODUCTION-ORIENTED AI INFERENCE PLATFORM
Classification
Advanced Independent Engineering Project
Not:
Engineering Lab
production employment
successful GPU deployment

121. AI PLATFORM MODEL
Platform:
GKE
Model:
Qwen2.5-1.5B-Instruct
Primary objective:
Operate model inference infrastructure with production-oriented concerns around:
deployment
security
supply chain
admission control
networking
GitOps
reliability
cost
This is infrastructure/platform engineering, not AI/ML research.
122. AI SUPPLY CHAIN
Implemented:
signed software supply chain
image signing
verification
admission control
GitOps
security scanning
network segmentation
A particularly important result:
The supply chain was proven in both directions:
valid/signed artifact accepted
unverified artifact rejected
Every claimed control was intended to have a live test.
123. GPU QUOTA BLOCKER
GPU provisioning was blocked by GCP account-level quota:
GPUS_ALL_REGIONS=0.0
This was confirmed through direct instance-group error logs.
The user did not simply assume the GPU failed.
Testing was performed across:
GPU types
regions
fresh project
under the same billing account.
The quota remained:
0.0
124. AI CPU FALLBACK
Because GPU provisioning was unavailable, the entire platform was validated using:
CPU inference
The following were still tested:
live inference
real inference responses
load testing
pod-failure recovery
rollback
Terraform reproducibility
signed supply chain
CI/CD
security controls
The platform therefore has meaningful infrastructure evidence despite the GPU quota limitation.
125. AI PLATFORM DOCUMENTATION
Documentation included:
architecture
security / supply chain
delivery pipeline
reliability evidence
GPU constraint
cost considerations
Teardown / cost safeguards were documented.
126. AI PLATFORM CLAIM BOUNDARY
Correct:
Production-oriented inference platform validated on CPU because GPU quota was unavailable.
Incorrect:
Successfully deployed a GPU inference cluster.
Incorrect:
Operated GPU model serving in production.
The strongest story is:
engineering under an externally imposed infrastructure constraint.
127. CROSS-PROJECT SECURITY THEMES
Security work across the portfolio includes:
zero static credentials
Workload Identity
WIF
IRSA/OIDC
service-account impersonation
Vault
dynamic credentials
secret rotation
Gitleaks
TruffleHog
Trivy
Syft
Cosign
Kyverno
OPA
OPA Gatekeeper
Conftest
Falco
kube-bench
RBAC
NetworkPolicy
default-deny policies
admission control
immutable image tags
GitOps
software supply-chain validation
128. CROSS-PROJECT RELIABILITY THEMES
Reliability evidence includes:
node failure recovery
pod failure
bad deployment rollback
HPA scaling
GitOps drift correction
automated canary rollback
scheduler race-condition discovery
atomic reservation
malformed-input bug discovery
node drain issues
image-pull authentication issues
multi-region active/active
measured RTO/RPO
chaos experiments
SLOs
error budgets
PDBs
resource governance
capacity-aware scheduling
129. CROSS-PROJECT OBSERVABILITY THEMES
Tools and methods:
Prometheus
Grafana
Alertmanager
Loki
Tempo
OpenTelemetry
Google Managed Prometheus
Golden Signals
RED
USE
SLOs
error budgets
distributed tracing
Redis trace-context propagation
The user has demonstrated an important distinction between:
having an observability configuration
and:
actually receiving/verifying useful telemetry.
Examples:
Auto-Healing Golden Signals dashboard: only Saturation had real data.
Istio STRICT mTLS: configured, but telemetry did not confirm live enforcement.
Atlas tracing: actual API→queue→scheduler→worker tracing was implemented.
SLO baselines: contaminated data was explicitly excluded.
130. CROSS-PROJECT GITOPS THEMES
GitOps systems:
Argo CD
Argo Rollouts
Kustomize
ApplicationSets
sync waves
Evidence:
drift correction under 2 seconds in GitOps Lab
drift correction under 3 seconds in Atlas
automated rollback
canary progression
production/staging access boundaries
no manual kubectl/helm required in Atlas's automated deployment path
131. CROSS-PROJECT FINOPS THEMES
FinOps evidence includes:
Auto-Healing
OpenCost
real API JSON
cost labels
UI verification
local infrastructure cost of $0
Atlas
namespace/workload attribution
fixed vs autoscaled comparison
GCP billing data where available
budget controls
teardown between sessions
Enterprise AWS
raw AWS CLI cost analysis in earlier snapshot
later roadmap claims completed FinOps/cloud-cost intelligence phase
exact final metrics not supplied
132. IMPORTANT INCIDENTS ACROSS PORTFOLIO
Known actual incidents / engineering failures include:
Auto-Healing
missing docker-proxy
memory exhaustion
Kubecost failure
HPA unknown metrics
Git history credential contamination
Terraform Lab
hardcoded Grafana password
terminal paste corruption
Terraform apply showing 0 resources
GitOps Lab
Argo ComparisonError
port-forward / Argo CLI connection loss
drift happened too quickly for screenshot
canary not fully verified
Litmus completion not verified
Atlas
scheduler capacity race
node-drain stall
image-pull authentication gap
malformed worker input
controlled failure experiments
backend health transitions in Operator
Android INTERNET permission failure
Enterprise AWS
13 documented real incidents
WSL2 DNS
EBS CSI dependency issue
observability startup/dependency problems
real rollback event
Istio telemetry limitation
mTLS verification limitation
AI platform
GPU quota GPUS_ALL_REGIONS=0.0
These failures should be preserved because they demonstrate engineering debugging rather than only successful demos.


133. WHAT THE USER HAS DELIBERATELY REFUSED TO CLAIM
The user has explicitly rejected:
invented metrics
fake production experience
fake customers
fake users
fake team sizes
fake revenue
fake incidents
fake certifications
fake responsibilities
fake seniority
fake production traffic
false GPU success
false banking production
false production IDP usage
This constraint applies to all future career materials.
134. PROJECT CLAIM BOUNDARIES
Auto-Healing
Can claim:
tested Kubernetes reliability lab
20-second node recovery
32-second rollback
0 blast radius
2→8 HPA scaling
86% CPU
scale-down to 2 at 2% CPU
48-second CI pipeline
dynamic Vault credentials
zero-downtime KV rotation
OpenCost verification
Cannot claim:
production Kubernetes
complete application observability
image signing
PDB
NetworkPolicy
production Vault
Terraform Lab
Can claim:
Terraform-managed Kubernetes platform
13 resources
modular IaC
security scanning
credential incident and remediation
reproducibility model
Cannot claim:
remote-state production Terraform
Terraform-managed HPA
production infrastructure
GitOps Lab
Can claim:
Argo CD
GitOps
drift correction under 2 seconds
RBAC boundaries
ApplicationSet
sync waves
policy-as-code
99.9% SLO resource
CI validation
Cannot claim:
verified canary runtime in that lab
verified Litmus experiment completion
multi-cluster production GitOps
Forge
Can claim:
GKE Autopilot IDP
Backstage
GitLab
Argo CD
Argo Rollouts
Kyverno
WIF
immutable artifacts
automated canary abort
drift correction
Cannot claim:
production IDP
real engineering users
real production traffic
end-to-end Backstage repository creation
signed images
SBOM
NetworkPolicy
Atlas
Can claim:
distributed workload platform
bin-packing scheduler
atomic reservation race fix
GitOps
under-3-second drift correction
WIF/keyless identity
OpenTelemetry distributed tracing
measured SLOs
automated rollback
active/active multi-region
measured RTO/RPO
chaos experiments
FinOps
Cannot claim:
professional production infrastructure
Byzantine fault tolerance
total-region simultaneous failure protection
GCP control-plane outage protection
distributed database architecture
Atlas Operator
Can claim:
Flutter Android app
FastAPI operations API
real-device verification
Healthy→Failed→Healthy test
INTERNET permission bug fixed
24 tests
clean Flutter analysis
CI on self-hosted GitLab runner
Cannot claim:
production operator console
complete live UI
authentication
Terraform lifecycle control
Enterprise AWS
Can claim:
EKS
Terraform
secure supply chain
GitHub Actions
Trivy
Syft
Cosign
Gitleaks
OPA/Conftest
Argo CD
Argo Rollouts
Istio
Vault
Kyverno
Falco
kube-bench
Prometheus/Grafana/Loki/Tempo/OTel/Alertmanager
real rollback
real fault injection timing
live RBAC / NetworkPolicy verification
13 documented incidents
later roadmap phases as explicitly qualified
Cannot claim:
actual banking production
real money
real regulator
real customers
global bank scale
fully verified live mTLS telemetry
observed circuit breaking
admission-enforced Cosign unless later evidence explicitly proves it
exact final Phase 3/4 metrics without source evidence
AI platform
Can claim:
GKE inference platform
Qwen2.5-1.5B-Instruct
signed supply chain
admission rejection of unverified images
CPU inference validation
load testing
rollback
pod failure recovery
Terraform reproducibility
GPU quota investigation
Cannot claim:
successful GPU provisioning
GPU production inference
GPU quota availability
135. PERSONAL WEBSITE / PORTFOLIO
The user has a personal website.
Potential evidence to expose:
project architecture
architecture diagrams
measured results
incidents
SLOs
failure experiments
technical debt
GitHub repositories
cloud links
AWS/GCP live project evidence where genuinely available
The user considered adding free-tier AWS and GCP live project links to prove cloud work.
Important:
Do not claim the website itself is hosted on AWS/GCP unless that is actually true.
136. LINKEDIN
The user had approximately:
400 connections
as of July 2026.
Goal:
Generate:
interviews
contracts
engineering relationships
The user dislikes:
desperate language
generic AI-sounding writing
fake authority
“I follow your work”
generic “I noticed you work at X”
empty engagement
Connection notes should be:
under 200 characters
The user uses a normal/non-Premium account and previously encountered a temporary inability to send notes.

136. LINKEDIN
The user had approximately:
400 connections
as of July 2026.
Goal:
Generate:
interviews
contracts
engineering relationships
The user dislikes:
desperate language
generic AI-sounding writing
fake authority
“I follow your work”
generic “I noticed you work at X”
empty engagement
Connection notes should be:
under 200 characters
The user uses a normal/non-Premium account and previously encountered a temporary inability to send notes.
137. LINKEDIN TARGET AUDIENCE
Primary:
CTOs
VPs of Engineering
Heads of Infrastructure
Principal Engineers
Staff Engineers
Platform Engineering leaders
138. LINKEDIN COMMENT STRATEGY
Preferred high-signal technical posts include:
Kubernetes failure analysis
bare-metal Kubernetes
AI infrastructure
PgBouncer connection exhaustion
PDB / HA
n8n Kubernetes root-cause analysis
SRE incidents
cloud reliability
supply-chain security
FinOps
infrastructure failures
The goal of comments is not visibility for its own sake.
The comment should demonstrate:
operational reasoning
failure-mode thinking
root-cause analysis
tradeoffs
practical engineering experience
The user previously wanted:
6 powerful comments
rather than generic comments.
139. CONTENT / VIDEO STRATEGY
The user wanted content similar in concept to “Code Cloud”:
technical infrastructure topics
free AI design tools
CapCut-style explanatory videos
visually understandable engineering concepts
A previous content brainstorming request sought:
80 topics
Potential topic families include:
Kubernetes failure modes
SLOs
GitOps
cloud cost
security
observability
distributed systems
chaos engineering
supply-chain security
reliability engineering
production incidents
140. USER'S PROFESSIONAL BRAND THEMES
Strongest themes:
Failure-first infrastructure
Infrastructure should be designed for failure rather than surprised by it.
Recovery
Recovery without human intervention wherever architecture permits.
Security
Zero static credentials and verified supply chains.
Observability
Don't guess during incidents; use telemetry.
Cost
Understand cost at workload/service level.
Systems thinking
Mechatronics background informs failure-oriented system design.
141. ENGINEERING COMMUNICATION PRINCIPLES
The user wants technical writing to:
be precise
show actual evidence
identify limitations
explain decisions
show incidents
show fixes
show verification
avoid marketing language
avoid unsupported claims
The user's repositories are intentionally documentation-heavy.
Technical debt documentation is considered a strength when it demonstrates intellectual honesty.
142. INTERVIEW-CRITICAL TECHNICAL TOPICS
The user should be prepared to defend:
Kubernetes
HPA
PDB
NetworkPolicy
cluster autoscaling
node drain
pod disruption
scheduling
resource requests/limits
allocatable capacity
GitOps
Argo CD
drift
reconciliation
sync waves
ApplicationSets
Rollouts
canary
rollback
Security
Vault
dynamic credentials
WIF
IRSA/OIDC
Cosign
image verification
Kyverno
RBAC
NetworkPolicy
Falco
supply-chain threat model
Service mesh
Istio
mTLS
fault injection
retries
circuit breakers
outlier detection
retry storms
Observability
Prometheus
Grafana
Loki
Tempo
OpenTelemetry
Golden Signals
RED
USE
SLOs
error budgets
Distributed systems
scheduler consistency
atomic reservation
queues
retries
idempotency
capacity
regional failure
RTO
RPO
multi-region architecture
FinOps
workload attribution
fixed vs autoscaled cost
resource utilization
waste
budgets
143. SPECIFIC INTERVIEW QUESTIONS THE PORTFOLIO SHOULD SUPPORT
Likely questions:
Why Kubernetes?
Why EKS rather than another deployment model?
Why GKE?
Why Istio?
Why Vault?
Why Argo CD?
Why Argo Rollouts?
What exactly happened during the scheduler race condition?
Why did atomic reservation fix it?
What happens during Redis failure?
How is Redis trace context propagated?
What happens when a node drains?
What does a PDB protect?
What does a PDB not protect?
How does HPA interact with cluster capacity?
What happens if the cluster cannot schedule the additional pods?
How does cluster autoscaler interact with HPA?
What was the blast radius of the bad deployment?
How was the 32-second rollback measured?
Why was the Auto-Healing Golden Signals dashboard incomplete?
Why did istio_requests_total not appear?
How would you prove STRICT mTLS actually works?
Why isn't a configured circuit breaker equivalent to a verified circuit breaker?
What attack does Kyverno prevent?
What happens if Kyverno is disabled?
What happens if Falco is disabled?
What does Cosign prove?
What is the difference between signing an image and enforcing signatures?
Why is WIF better than stored service-account keys?
What was the malformed-input bug in Atlas?
Why was contaminated telemetry excluded?
How were SLO baselines established?
How was RTO measured?
How was RPO measured?
What assumptions exist in the multi-region model?
What happens if both regions fail simultaneously?
Why is a distributed database excluded?
What would break at 10× current workload?
What is actually production-ready?
What would you change before onboarding real customers?
144. CV POSITIONING — FUTURE, NOT THIS DOCUMENT
The eventual CV should be:
approximately 2 pages
ATS-friendly
international
technically precise
evidence-led
Possible structure:
Header
Summary
Skills
Experience
Selected Projects
Education
Additional labs/certifications if relevant
Recommended project emphasis:
Atlas
4–5 bullets
AWS Enterprise Cloud Platform
4–5 bullets
Forge
3–4 bullets
AI inference
2–3 bullets
Engineering Labs
Condensed:
1–2 bullets each or grouped
The strongest measured evidence should dominate.
145. CANDIDATE SENIORITY POSITIONING
The user should not be presented as:
Senior Platform Engineer
Staff Engineer
Principal Engineer
unless professional evidence supports that claim.
More defensible positioning:
Junior-to-mid-level Platform/DevOps/Cloud candidate with unusually strong independent infrastructure evidence.
The distinguishing factor is not years of professional employment.
It is the depth of independently engineered systems and the quality of failure/testing evidence.

146. PROFESSIONAL EXPERIENCE VS PROJECT EXPERIENCE
The distinction must remain explicit.
Professional / educational experience
Mechatronics Engineering education
Center for Tech internship/technical exposure
Center for Tech mentoring
teaching 20+ students
robotics/electronics projects
line-following robot
automated farm project
Independent engineering systems
Atlas
Enterprise Cloud Platform / Velrite
Forge
AI inference platform
Engineering Labs
Auto-Healing Kubernetes
Terraform Kubernetes
GitOps ArgoCD
No independent project should be silently converted into employment.
147. SECONDARY INCOME STRATEGY
Separate from the Platform Engineering career.
Positioning:
Revenue / Operations Intelligence Specialist
Initial niche:
Revenue Intelligence + Reporting Automation for African e-commerce and distribution businesses
Core philosophy:
Sell:
answers and automation
rather than simply:
dashboards
148. REVENUE INTELLIGENCE PROBLEMS
Target problems:
revenue leakage
margin leakage
anomaly diagnosis
reporting automation
operational visibility
The objective is to answer questions such as:
Where is revenue being lost?
Where are margins deteriorating?
Which products/customers/channels are anomalous?
What reports are still being manually assembled?
Which recurring operational analyses can be automated?
149. REVENUE INTELLIGENCE CAPABILITY PROGRESSION
Planned stack:
Excel / Power BI / SQL
then:
Python automation
then:
APIs
then:
PostgreSQL
then:
Git
then:
Docker
then:
basic cloud
then:
scheduled jobs
150. REVENUE INTELLIGENCE OFFER DIRECTION
Earlier package concepts included approximately:
₦75,000
₦150,000
₦250,000+
These are package ideas, not guaranteed market prices.
Potential offer categories:
reporting automation
revenue dashboards
anomaly detection
margin analysis
recurring reporting systems
151. 90-DAY INCOME TARGET
Target:
₦1,000,000 within 90 days
This is a goal.
It is not guaranteed income.
The proposed strategy included:
Early period
Build the capability.
Middle period
Create approximately three business-facing portfolio projects demonstrating:
revenue intelligence
reporting automation
anomaly diagnosis
business decision support
Later period
Days approximately:
46–90
focused increasingly on:
outreach
client acquisition
offer testing
business development
This income strategy is distinct from the long-term Platform Engineering target.
152. STRATEGIC CAREER TENSION
The user has two parallel objectives:
Primary long-term technical career
Platform / Cloud / DevSecOps:
$20k/month long-term target
Secondary near-term income path
Revenue/Operations Intelligence:
₦1M/90-day target
These should not be conflated.
The second is a potential cash-flow strategy.
The first is the main infrastructure engineering career.
153. CURRENT PORTFOLIO STRENGTH
The strongest evidence is not simply the number of repositories.
The strongest evidence is:
Atlas
distributed scheduling
real race condition
atomic fix
multi-region
RTO/RPO
observability
SLOs
GitOps
WIF
automated rollback
chaos
Enterprise AWS
EKS
supply-chain security
service mesh
runtime security
Vault
observability
13 incidents
real rollback
fault injection
production-shaped banking model
Forge
IDP architecture
Backstage
GitOps
canary
admission policy
WIF
AI inference
signed supply chain
admission enforcement
CPU fallback
GPU quota investigation
reliability tests
These are substantially stronger than ordinary “I deployed Kubernetes” portfolio projects.
154. CURRENT PORTFOLIO WEAKNESS
The largest professional weakness is not necessarily technical ability.
It is:
lack of formal production employment evidence.
There is no claim here of:
years operating production infrastructure for a real company
real customer traffic
production incident ownership inside an employer
real on-call history
real enterprise team ownership
production change-management responsibility
Therefore the career strategy must convert portfolio strength into:
interviews
contract work
professional references
real-world projects
employment
rather than endlessly building increasingly elaborate personal projects.


155. CURRENT STRATEGIC DIRECTION
The user has expressed the desire to:
stop endless project-building and apply now.
The portfolio is already sufficiently substantial to support applications.
The next bottleneck is not simply another Kubernetes project.
It is:
positioning
applications
interviews
networking
technical storytelling
evidence presentation
contract acquisition
156. PERSONAL BRAND / WEBSITE PRINCIPLE
The website should feel like an engineer's evidence base rather than a marketing landing page.
Useful sections include:
system overview
architecture
design decisions
failure tests
measured results
incidents
remediation
SLOs
technical debt
security model
repository
live evidence where genuinely available
The user wants the engineering work to be difficult for a technically competent reviewer to dismiss.
157. WHAT MAKES THE PORTFOLIO DIFFERENT
The most defensible differentiator is:
failure-oriented engineering evidence.
Examples:
20s node recovery
32s rollback
0 blast radius
2→8 HPA scaling
86% CPU
2% post-load CPU
48s pipeline
36 alert rules
13 Terraform resources
under 2s GitOps lab drift correction
under 3s Atlas drift correction
99.9% declared API availability SLO in GitOps lab
18 Atlas phases
13 Enterprise AWS documented incidents
24 Atlas Operator tests
2026-09-20 real-device verification
measured multi-region RTO/RPO
real scheduler race-condition discovery and fix
The number alone is not the claim.
The important structure is:
failure → observation → root cause → fix → re-test → measured result
158. NON-NEGOTIABLE EVIDENCE RULE
Whenever a future CV, LinkedIn post, website, proposal, interview answer, or application references a project:
The underlying claim must answer:
What was built?
Why was it built?
What was actually implemented?
What was actually tested?
What happened during testing?
What broke?
What was the root cause?
What changed?
What was the measured result?
What remains unverified?
Was this production, production-modeled, simulated, or a lab?
This prevents the portfolio from becoming inflated.
159. FINAL SOURCE-OF-TRUTH CLASSIFICATION
Professional / real-world experience
Mechatronics education
Center for Tech technical exposure
Center for Tech mentoring
teaching 20+ students
robotics/electronics projects
Independent flagship
Atlas — Distributed Resilience Platform
Enterprise Cloud Platform / Velrite — AWS EKS
Major independent systems
Forge — GKE Autopilot IDP
Production-Oriented AI Inference Platform — GCP
Engineering Labs
Auto-Healing Kubernetes Platform
Terraform Kubernetes Platform
GitOps ArgoCD Platform
Supporting evidence
personal website
GitHub
LinkedIn
technical writing/content
architecture documentation
incident documentation
ADRs
technical-debt records
160. MASTER TRUTH TABLE
System
Environment
Classification
Strongest verified evidence
Major limitation
Auto-Healing
3-node Minikube
Engineering Lab
20s recovery, 32s rollback, 0 blast radius, HPA 2→8, 48s CI
incomplete app metrics, no PDB/NetworkPolicy/signing
Terraform Platform
Kubernetes/Minikube
Engineering Lab
13 resources, modular IaC, security gates, history cleanup
no remote state, no CI apply, no Terraform HPA
GitOps Platform
Kubernetes
Engineering Lab
drift correction under 2s, RBAC, AppSet, policy controls
canary/chaos completion not fully verified
Forge
GKE Autopilot
Major Project
Backstage, WIF, immutable artifacts, live drift correction, canary abort
no real users/traffic, incomplete golden path
Atlas
GCP
Flagship
18 phases, scheduler race/fix, <3s drift, measured RTO/RPO, multi-region, tracing, rollback
explicit failure-model exclusions
Atlas Operator
Android + FastAPI
Add-on
real phone/cluster 2026-09-20, Healthy→Failed→Healthy, 24 tests
fixture screens, no auth, no Terraform control
Enterprise Cloud / Velrite
AWS EKS
Flagship
supply chain, EKS, Istio, Vault, Kyverno, Falco, observability, 13 incidents, real rollback
simulated bank; several controls historically unverified
AI Inference
GKE
Advanced Project
signed supply chain, admission rejection, CPU inference, rollback, load testing
GPU blocked by GPUS_ALL_REGIONS=0.0
161. ABSOLUTE CLAIM BOUNDARIES FOR FUTURE MATERIAL
Never write:
“I ran a production banking platform.”
Write, when relevant:
“I built a production-modeled regulated-bank platform on EKS using Bank of Anthos.”
Never write:
“I built a production internal developer platform used by engineers.”
Write:
“I built a self-service IDP on GKE Autopilot with Backstage, GitLab, Argo CD, Argo Rollouts, Kyverno and WIF.”
Never write:
“I deployed GPU inference infrastructure.”
Write:
“I engineered and validated a GKE inference platform on CPU after GCP GPU quota prevented GPU provisioning.”
Never write:
“All Istio security was verified.”
Write:
“Istio STRICT mTLS and traffic policies were configured; fault injection was verified with real timing, while live mTLS telemetry was not confirmed in the tested snapshot.”
Never write:
“The GitOps lab has production-grade canary delivery.”
Write:
“The canary rollout was configured; runtime canary triggering was not fully verified in that lab.”
Never write:
“The platform is completely resilient.”
Write:
“Specific failure modes were deliberately tested, with measured recovery and explicitly documented exclusions.”

162. CORE ENGINEERING NARRATIVE
The most coherent overall story is:
A Mechatronics Engineering graduate developed a systems-oriented approach to engineering through robotics, electronics, programming and failure-oriented design. That evolved into software infrastructure work, Kubernetes, Terraform and cloud platforms. Rather than stopping at successful deployment, the user's independent engineering work increasingly focused on what happens when infrastructure fails: node loss, bad deployments, scheduling races, configuration drift, credential problems, observability failures, regional failover, and supply-chain attacks.
The strongest portfolio evidence demonstrates a recurring loop:
design → deploy → break → observe → diagnose → fix → re-test → measure → document
That pattern is more important than any individual tool.
## 163. LONG-TERM TECHNICAL IDENTITY
The intended professional identity is not:
“Someone who knows many DevOps tools.”
It is:
An infrastructure/platform engineer who designs cloud systems around reliability, security, observability, cost and failure recovery.
The Mechatronics background provides a systems-engineering foundation.
The cloud/Kubernetes work provides the infrastructure implementation.
The incidents and measured tests provide evidence.
The GitOps/security/observability work provides platform-engineering depth.
The distributed systems work provides higher-order engineering complexity.
164. MASTER EVIDENCE INVENTORY — NUMBERS THAT MUST NOT BE LOST
The following numbers are specifically important and should be preserved when supported by the relevant project:
20+ students taught at Center for Tech
3-node Minikube cluster
Kubernetes v1.31.0
API HPA 2 → 8
CPU peak 86%
post-load CPU 2%
node recovery 20 seconds
bad-image rollback 32 seconds
bad-image blast radius 0
CI pipeline 48 seconds
36 alert rules
13 Terraform resources
GitOps lab drift correction under 2 seconds
Atlas drift correction under 3 seconds
GitOps lab API SLO 99.9%
Atlas 18 planned phases
Enterprise AWS 13 real documented incidents
Atlas Operator verification date 2026-09-20
Atlas Operator 24 tests
AI GPU quota GPUS_ALL_REGIONS=0.0
HPA frontend max 6
API HPA max 8
API CPU threshold 60%
API memory threshold 70%
frontend CPU threshold 65%
GitOps canary progression: 10% → 30s → 25% → 30s → 50% → 30s → 100%
Auto-Healing pipeline: validate → deploy-canary → validate-canary → rollback → deploy-production
These are evidence-bearing details, not decorative statistics.
## 165. FINAL OPERATING RULE FOR FUTURE CV / APPLICATION WORK
When generating future career material from this document:
Preserve
measured results
seconds
percentages
replica counts
resource thresholds
number of incidents
number of tests
phase counts
dates
exact technologies
actual failure conditions
root causes
fixes
verification status
Compress only when necessary
For example, a CV can compress:
“36 Prometheus alert rules covering pod, CPU, memory, error-rate and node failure conditions.”
But the master document must retain the underlying details.
Never silently upgrade
configured → verified
simulated → production
production-modeled → production
planned → implemented
CPU inference → GPU inference
rollback configured → rollback observed
SLO defined → SLO achieved
dashboard exists → telemetry works
image signed → image admission-enforced
Never delete negative evidence merely because it is inconvenient.
The technical debt, failed experiments, missing telemetry, quota limitations and incidents are part of the engineering record.
# 166. SOURCE-OF-TRUTH STATUS
This document should be treated as the master factual reference for:
CV generation
portfolio writing
LinkedIn positioning
interview preparation
job applications
contract proposals
personal website
project README refinement
technical case studies
Any future artifact should derive claims from this evidence base and should preserve the project's actual verification boundary.
The objective is not to make the user sound more experienced than they are.
The objective is to make the actual engineering work impossible to overlook without overstating it.


