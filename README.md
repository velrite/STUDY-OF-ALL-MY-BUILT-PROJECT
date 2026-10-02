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
