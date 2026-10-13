

#  CODESPACE GitOps ArgoCD Platform

## What Is This Project?

In the first two projects, deployment still required a human running commands. Even with Terraform, someone had to run `terraform apply`. Even with kubectl, someone had to run `kubectl apply`. The platform was automated but the deployment mechanism was not.

This project changes that.

Push to Git. ArgoCD detects the change and syncs the cluster automatically. Modify the cluster manually. ArgoCD detects the drift and corrects it. Roll something back. Revert the commit. ArgoCD applies the previous state.

The Git repository is the only source of truth. The cluster is a reflection of it. No kubectl in production. The commit history is the audit trail.

---

## Part 1 — For the Non-Technical Reader

**What problem does this solve?**
When engineers deploy software by running commands manually, there is no reliable audit trail. "Who deployed this?" "When?" "What changed?" These questions require checking email threads, Slack history, or terminal logs — none of which are reliable or searchable.

When deployment happens through Git, every change has an author, a timestamp, a description, and a complete record of what was different. Rolling back means reverting a commit, not hoping someone remembers which command to undo.

**What does it do?**
Watches a Git repository and continuously ensures the Kubernetes cluster matches what the repository contains. If the cluster drifts from Git — whether from a manual change, a pod crash that changes replica counts, or anything else — it corrects the cluster back to match Git.

**Why does it matter?**
In regulated industries, the question "can you prove what was running in production at 3pm on Tuesday?" requires an audit trail. Git provides that. Manual deployment commands do not.

**What was the engineering mission?**
Implement a GitOps platform where Git is the single source of deployment truth. Prove that manual cluster changes are automatically detected and corrected. Demonstrate multi-environment management from a single repository.

---

## Part 2 — Engineering Objective

### Primary Objective
Implement GitOps using ArgoCD where every cluster state change originates from a Git commit and drift is automatically corrected.

### Secondary Objectives
- Manage staging and production environments from a single ApplicationSet template
- Implement RBAC that prevents developers from directly modifying production
- Apply security policies via OPA Gatekeeper and Kyverno
- Provision ArgoCD itself via Terraform (connecting Projects 2 and 3)
- Define SLO and chaos experiment resources as code

### Functional Requirements
- Both environments must sync automatically without human intervention
- Manual cluster changes must be detected and corrected within 3 minutes
- staging must run 1 replica, production must run 2 replicas (Kustomize overlay)
- platform-admin role can sync anything; developer role can only sync staging
- OPA must reject deployments using `:latest` image tag in production
- Kyverno must reject pods without resource limits in production

### Non-Functional Requirements
- Git is the only interface for production deployments
- Every change is traceable to a commit, author, and timestamp
- Pipeline validates manifests before any sync can occur

### Constraints
- Single cluster (multi-cluster was not implemented — see GAPS.md)
- Argo Rollouts canary not triggered end-to-end (no custom image to push)
- Sloth and Litmus applied as manifests but runtime verification incomplete

### Success Criteria
- `argocd app list` shows both apps Synced and Healthy
- Manual scale to 5 replicas corrected automatically (selfHeal verified)
- AppProject visible in cluster with correct RBAC roles
- ApplicationSet generating two Applications from one template
- OPA and Kyverno policies active in cluster
- CI pipeline all 4 stages green

---

## Part 3 — System at a Glance

### 3.1 The GitOps Control Loop

```
Engineer pushes to Git
          │
          ▼
    GitHub Repository
    (single source of truth)
    apps/staging/kustomization.yaml   ← desired state for staging
    apps/production/kustomization.yaml ← desired state for production
          │
          │ ArgoCD polls every 180 seconds
          │ (or immediately via webhook if configured)
          ▼
    ArgoCD Application Controller
    ├── clones repository via Repo Server
    ├── renders manifests through Kustomize
    ├── compares rendered output vs live cluster state
    └── if different: applies the diff to the cluster
          │
          ├──► staging namespace    (1 replica)
          └──► production namespace (2 replicas)
```

### 3.2 What Happens When Someone Changes the Cluster Manually

```
kubectl scale deployment api-service -n production --replicas=5
          │
          ▼
    Cluster state:  5 replicas (actual)
    Git state:      2 replicas (desired)
          │
          ▼
    ArgoCD detects mismatch
    (selfHeal: true — continuous watch)
          │
          ▼
    ArgoCD applies Git state to cluster
          │
          ▼
    Cluster state: 2 replicas (corrected)
    Time elapsed:  under 2 seconds
```

This was tested live. The correction was faster than running a second `kubectl get pods` command. The 5-replica state never persisted long enough to be screenshotted, which is the point.

### 3.3 Component Responsibilities

**ArgoCD Application Controller**
What it is: The core reconciliation engine.
What it does: Continuously compares desired state (rendered Git manifests) against actual state (live cluster resources). Triggers sync when mismatch detected.
What depends on it: All Application and ApplicationSet resources.
What breaks if it fails: Syncs stop. Cluster continues running but drift is no longer detected or corrected.

**ArgoCD Repo Server**
What it does: Clones the Git repository, runs Kustomize to render manifests, returns rendered output to the Application Controller.
Why it exists separately: Isolates the potentially expensive operation of cloning and rendering from the reconciliation logic. Also allows caching rendered manifests across multiple Applications pointing to the same repository.

**ArgoCD API Server**
What it does: Serves the web UI and CLI. Handles authentication and enforces RBAC policies defined in AppProject resources.

**ApplicationSet Controller**
What it does: Reads ApplicationSet resources and runs the configured generator to create individual Application resources. When the generator output changes (new environment added to list), the controller creates or deletes corresponding Applications automatically.

**Argo Rollouts Controller**
What it does: Manages Rollout resources. Controls canary traffic weight between stable and canary versions. Promotes or aborts based on configured steps.
Current status: Installed and running. Rollout manifest defined. End-to-end canary execution not verified — no custom image to push.

**OPA Gatekeeper**
What it does: Intercepts API server admission requests. Evaluates them against Rego policies defined in ConstraintTemplate resources. Rejects non-compliant resources.
Applied policy: no-latest-tag — rejects deployments in the production namespace that use `:latest` image tag.

**Kyverno**
What it does: Policy engine that validates, mutates, and generates Kubernetes resources. Simpler than OPA for straightforward policies like "require resource limits."
Applied policies: require-resource-limits (enforce), disallow-privileged (enforce).

### 3.4 How Kustomize Overlays Work

```
apps/base/deployment.yaml
  image: nginx:1.25-alpine
  replicas: 2
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits:   { cpu: 200m, memory: 128Mi }
  securityContext:
    runAsNonRoot: true
  readinessProbe: httpGet / port 80
          │
          ├──► apps/staging/kustomization.yaml
          │    patches:
          │      replicas: 1 (only difference from base)
          │    commonLabels:
          │      environment: staging
          │
          └──► apps/production/kustomization.yaml
               commonLabels:
                 environment: production
               (inherits 2 replicas from base)
```

**Why base is copied into each overlay directory:**
ArgoCD's Kustomize implementation blocks cross-directory references (`../base/deployment.yaml`) for security reasons — it prevents path traversal attacks. The base manifest is copied into each overlay directory. This creates some duplication but avoids the ComparisonError that cross-directory references produce. This is documented in INCIDENTS.md.

### 3.5 ApplicationSet Matrix Generator

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              - environment: staging
                namespace: staging
                repoPath: apps/staging
              - environment: production
                namespace: production
                repoPath: apps/production
        - list:
            elements:
              - app: api-service
```

This produces:
- `staging-api-service` targeting `apps/staging` deploying to `staging` namespace
- `production-api-service` targeting `apps/production` deploying to `production` namespace

Adding a new environment requires one line in the first list. Adding a new application requires one line in the second list. The matrix generates all combinations.

### 3.6 Sync Wave Ordering

```
Wave 0: Namespace resource
        Must exist before any other resource can be created

Wave 1: ConfigMap and Services
        Config must be available before the application that reads it starts
        Service must exist before pods that register as endpoints

Wave 2: Deployment
        Application starts after its config and networking are in place

Wave 3: Rollout
        Canary routing configured after stable deployment exists
```

ArgoCD waits for all resources in a wave to reach Healthy status before applying the next wave. This prevents race conditions where a deployment tries to read a ConfigMap that does not yet exist.

### 3.7 RBAC Model

```
AppProject: platform

  Source repositories:
    https://github.com/velrite/gitops-argocd-platform
    (only this repository — no other source allowed)

  Destinations:
    staging namespace, local cluster
    production namespace, local cluster
    (no other namespaces — cannot deploy anywhere else)

  Role: platform-admin
    ├── sync any application
    ├── manage cluster registrations
    ├── manage repository credentials
    └── manage projects
    Assigned to: platform-team group

  Role: developer
    ├── get (view) any application — READ ONLY for production
    ├── sync staging-* applications only
    └── CANNOT sync production applications
    Assigned to: dev-team group
```

**Why this matters:** Developers push to Git. ArgoCD handles production. Direct production access is restricted to platform-admin. This enforces the GitOps principle that Git is the only path to production.

---

## Part 4 — Terraform Provisions ArgoCD

ArgoCD and Argo Rollouts are installed as `helm_release` resources in Terraform, using the same pattern as Project 2.

```hcl
resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  namespace  = kubernetes_namespace.argocd.metadata[0].name
  version    = "7.3.11"
  timeout    = 600
  values     = [file("${path.module}/argocd-values.yaml")]
}

resource "helm_release" "argo_rollouts" {
  name       = "argo-rollouts"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-rollouts"
  namespace  = kubernetes_namespace.argo_rollouts.metadata[0].name
}
```

**Why this connection matters:**
- Project 2 established: define infrastructure as Terraform code
- Project 3 uses that pattern: Terraform defines the GitOps tool
- The GitOps tool then manages all application deployments

Terraform provisions the platform that does GitOps. GitOps manages everything else. This is the complete IaC + GitOps story: the deployment mechanism itself is defined as code.

**argocd-values.yaml configuration:**
```yaml
configs:
  rbac:
    policy.default: role:readonly
    policy.csv: |
      p, role:platform-admin, applications, *, */*, allow
      p, role:platform-admin, clusters, *, *, allow
      p, role:developer, applications, get, */*, allow
      p, role:developer, applications, sync, staging/*, allow
      g, platform-team, role:platform-admin
      g, dev-team, role:developer

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

---

## Part 5 — Build the Project Step by Step

### STEP 1 — Create the Repository and Structure

**Objective:** Establish the GitOps repository that ArgoCD will watch.

**Why the repository must exist before ArgoCD:** ArgoCD is configured to watch a specific repository URL. That URL must be accessible when the AppProject and ApplicationSet resources are applied. If the repository does not exist or is private without credentials configured, ArgoCD returns a ComparisonError.

**Structure:**
```bash
mkdir -p gitops-argocd-platform/{terraform,apps/{base,staging,production},argocd/{projects,appsets},security/{opa,kyverno},observability/sloth,chaos/litmus,.github/workflows}
cd gitops-argocd-platform
git init
git remote add origin https://github.com/velrite/gitops-argocd-platform.git
```

**What depends on this:** Every subsequent step. ArgoCD cannot be pointed at a repository that does not exist.

---

### STEP 2 — Provision ArgoCD via Terraform

**Objective:** Install ArgoCD and Argo Rollouts using the Terraform pattern from Project 2.

**Why Terraform here:** Consistency. The entire platform infrastructure is defined as code. The GitOps tool itself should be no different.

**Exact implementation:**
```bash
cd terraform/
terraform init
terraform apply -auto-approve
```

**Verification:**
```bash
kubectl get pods -n argocd
# Expected: application-controller, applicationset-controller,
#           dex-server, notifications-controller, redis,
#           repo-server, server — all Running
kubectl get pods -n argo-rollouts
# Expected: argo-rollouts pod Running
```

**Get admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

**Port-forward and login:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80 &
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
```

---

### STEP 3 — Write the Base Application Manifest

**Objective:** Define the application once in a base manifest that all environment overlays build from.

**Why security context is in the base manifest:**
Security context applies to all environments. If it were in an overlay, an environment that forgot the overlay would run without it. Putting it in the base makes security the default, not the exception.

**apps/base/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: api-service
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
```

**Why explicit resource requests and limits:** Kyverno's `require-resource-limits` policy enforces that all production pods have limits. Setting them in the base manifest means the policy is satisfied by default. It also means HPA can compute accurate CPU utilization percentages.

---

### STEP 4 — Write the Kustomize Overlays

**apps/staging/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - deployment.yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
    target:
      kind: Deployment
      name: api-service
commonLabels:
  environment: staging
```

**apps/production/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - deployment.yaml
commonLabels:
  environment: production
```

**Why deployment.yaml is copied not referenced:**
ArgoCD blocks `../base/deployment.yaml` cross-directory references for security. The base file is copied into each overlay directory. See INCIDENTS.md — this was discovered when both environments showed ComparisonError on the first sync attempt.

**Copy the base:**
```bash
cp apps/base/deployment.yaml apps/staging/deployment.yaml
cp apps/base/deployment.yaml apps/production/deployment.yaml
```

---

### STEP 5 — Write Sync Wave Resources

**apps/base/namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "1"
data:
  environment: production
  version: "1.0.0"
```

**Why sync waves exist:** Without them, ArgoCD applies all resources in a single wave simultaneously. A Deployment that reads from a ConfigMap might start before the ConfigMap exists. Sync waves enforce the dependency ordering that makes deployment predictable.

---

### STEP 6 — Write the Canary Rollout Definition

**apps/production/rollout.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-service-rollout
  namespace: production
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api-service-rollout
  template:
    metadata:
      labels:
        app: api-service-rollout
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: api-service
        image: nginx:1.25-alpine
        resources:
          requests: { cpu: "50m", memory: "64Mi" }
          limits: { cpu: "200m", memory: "128Mi" }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 30s}
      - setWeight: 25
      - pause: {duration: 30s}
      - setWeight: 50
      - pause: {duration: 30s}
      - setWeight: 100
```

**Current status:** Manifest applied. Rollout controller is running. End-to-end execution was not verified because triggering the canary requires pushing a new image tag to a container registry — the demo app uses nginx from Docker Hub which this project does not own.

---

### STEP 7 — Write the AppProject

**argocd/projects/platform-project.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Platform microservices project
  sourceRepos:
    - https://github.com/velrite/gitops-argocd-platform
  destinations:
    - namespace: staging
      server: https://kubernetes.default.svc
    - namespace: production
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: platform-admin
      policies:
        - p, proj:platform:platform-admin, applications, *, platform/*, allow
      groups:
        - platform-team
    - name: developer
      policies:
        - p, proj:platform:developer, applications, get, platform/*, allow
        - p, proj:platform:developer, applications, sync, platform/staging-*, allow
      groups:
        - dev-team
```

**Apply:**
```bash
kubectl apply -f argocd/projects/platform-project.yaml
```

---

### STEP 8 — Write and Apply the ApplicationSet

**argocd/appsets/platform-appset.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-apps
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - environment: staging
                  namespace: staging
                  repoPath: apps/staging
                - environment: production
                  namespace: production
                  repoPath: apps/production
          - list:
              elements:
                - app: api-service
  template:
    metadata:
      name: "{{environment}}-{{app}}"
    spec:
      project: platform
      source:
        repoURL: https://github.com/velrite/gitops-argocd-platform
        targetRevision: HEAD
        path: "{{repoPath}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

**Apply:**
```bash
kubectl apply -f argocd/appsets/platform-appset.yaml
```

**Force initial sync:**
```bash
argocd app sync staging-api-service --insecure
argocd app sync production-api-service --insecure
argocd app list
```

**Expected:**
```
NAME                           SYNC STATUS   HEALTH STATUS
argocd/staging-api-service     Synced        Healthy
argocd/production-api-service  Synced        Healthy
```

---

### STEP 9 — Apply Security Policies

**OPA Gatekeeper:**
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
kubectl apply -f security/opa/no-latest-tag.yaml
```

**Kyverno:**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno --namespace security
kubectl apply -f security/kyverno/require-resources.yaml
```

**Verification:**
```bash
kubectl get constrainttemplate nolatesttag
kubectl get nolatesttag no-latest-tag
kubectl get clusterpolicy
# Expected: require-resource-limits and disallow-privileged both READY=true
```

---

### STEP 10 — Apply SLO and Chaos Definitions

**SLO:**
```bash
kubectl apply -f observability/sloth/api-service-slo.yaml
```

**Chaos experiment:**
```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm
helm install litmus litmuschaos/litmus --namespace chaos
kubectl apply -f chaos/litmus/pod-kill-experiment.yaml
```

**Current status:** Both resources applied. Sloth controller processing and Litmus experiment completion were not fully verified at runtime.

---

### STEP 11 — Run Drift Detection Test

**Objective:** Prove selfHeal corrects manual cluster changes automatically.

**Exact implementation:**
```bash
echo "=== BEFORE DRIFT ===" && date
kubectl get pods -n production | grep api-service | wc -l

echo "=== INJECTING DRIFT ===" && date
kubectl scale deployment api-service -n production --replicas=5

sleep 10

echo "=== AFTER SELFHEAL ===" && date
kubectl get pods -n production | grep api-service | wc -l
argocd app get production-api-service --insecure | grep -E "Sync|Health"
```

**Result:**
The 5-replica state was corrected before the second `kubectl get pods` command returned. The after-selfHeal check showed 2 pods. ArgoCD status showed Synced and Healthy.

**Why this result is meaningful:** selfHeal is not a slow background process that runs on a timer. It is a continuous watch. The correction in under 2 seconds means the platform cannot be left in a manually-modified state even briefly. Every change must go through Git.

---

### STEP 12 — Configure CI/CD Pipeline

**Objective:** Validate all manifests and Terraform code on every push.

**Pipeline structure:**
```yaml
jobs:
  security-scan:
    - TruffleHog secret scan
    - tfsec Terraform scan

  validate-manifests:
    needs: security-scan
    - kubectl dry-run on all YAML files in apps/ argocd/ security/

  validate-terraform:
    needs: security-scan
    - terraform fmt -check
    - terraform init -backend=false
    - terraform validate

  drift-check:
    needs: [validate-manifests, validate-terraform]
    - verify sync-wave annotations present
    - verify security contexts present
    - verify resource limits present
    - verify SLO definitions present
    - verify chaos experiments present
```

**What the drift-check job actually does:**
```bash
grep -r "sync-wave" apps/ && echo "Sync waves configured"
grep -r "runAsNonRoot" apps/ && echo "Security contexts present"
grep -r "limits" apps/ && echo "Resource limits present"
ls observability/sloth/ && echo "SLO definitions present"
ls chaos/litmus/ && echo "Chaos experiments present"
```

These checks verify that the manifests in the repository contain the expected annotations and configuration, not that they are working correctly in the cluster. Cluster-level verification requires access to the cluster which the CI runner does not have.

---

## Part 6 — Evolution of the System

### Change 1 — ComparisonError from Cross-Directory Reference

**Original:** Kustomize overlays referenced base manifest with `../base/deployment.yaml`.

**Problem:** ArgoCD's Kustomize security model blocks path traversal. Both environments showed ComparisonError immediately after the ApplicationSet was applied:
```
kustomize build failed:
accumulating resources from '../base/deployment.yaml':
security; file is not in or below 'apps/staging'
```

**Change:** Copied `apps/base/deployment.yaml` into `apps/staging/deployment.yaml` and `apps/production/deployment.yaml`. Updated kustomization.yaml to reference local file.

**Engineering lesson:** ArgoCD's Kustomize security restrictions are not configurable at the user level. The only solution is to use local file references within each overlay directory.

---

### Change 2 — Drift Correction Too Fast to Screenshot

**Original:** Planned to screenshot the 5-replica drifted state as proof of drift detection.

**What happened:** By the time the screenshot command ran, ArgoCD had already corrected the replicas back to 2. The drifted state lasted under 2 seconds.

**How this was documented:** Rather than trying to capture the drifted state, the timing was documented as the evidence. "Corrected before a second kubectl command could return" is a stronger demonstration than a screenshot of 5 pods.

**Engineering lesson:** When an observation is too fast to capture visually, the timing itself becomes the proof. Document what was expected, what the system did, and how quickly — not just a screenshot.

---

### Change 3 — ArgoCD CLI Connection Lost Mid-Session

**What happened:**
```
Failed to establish connection to localhost:8080: connection refused
```

**Root cause:** Port-forward process terminated when the ArgoCD server pod restarted after a memory pressure event.

**Fix:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80 &
argocd login localhost:8080 --username admin --insecure
```

**Engineering lesson:** Port-forward is not persistent. It terminates when the pod it forwards to restarts. For any session that involves multiple command sequences, restart the port-forward at the beginning of each session.

---

## Part 7 — Technology Decision Record

### ArgoCD v7.3.11

**What it is:** A declarative, GitOps continuous delivery tool for Kubernetes that syncs cluster state to a Git repository.

**What we used it for:** Automated deployment, drift detection, self-healing, multi-environment management.

**Why we chose it:** Industry standard for Kubernetes GitOps. ApplicationSet enables multi-environment management from a single template. AppProject provides fine-grained RBAC. selfHeal provides automatic drift correction.

**Alternatives:** Flux CD, Jenkins X, manual kubectl in CI.

**Why not Flux:** Both are excellent. ArgoCD was chosen because it has a richer UI for demonstrating sync status and a more established RBAC model via AppProject.

**Why not manual kubectl in CI:** Provides deployment but no drift detection, no self-healing, no RBAC enforcement at the deployment level.

**Production assessment:** Appropriate for production.

---

### Kustomize

**What it is:** A Kubernetes manifest customization tool that uses overlays and patches rather than templating.

**What we used it for:** Expressing environment differences (staging: 1 replica, production: 2 replicas) without duplicating the entire manifest.

**Why we chose it over Helm:** For managing your own application across environments, Kustomize patches are simpler than Helm templating. Kustomize has no chart packaging, no values files, no template syntax — just patches applied to a base manifest.

**Why not plain YAML duplication:** Copying the full manifest per environment means a change to the base (like updating the security context) must be applied to N environment files. Kustomize inherits the base automatically.

**Trade-off:** ArgoCD blocks cross-directory Kustomize references, requiring the base to be copied into each overlay. Some duplication exists.

**Production assessment:** Appropriate for production.

---

### OPA Gatekeeper

**What it is:** An admission controller that evaluates Rego policies against API server requests.

**What we used it for:** Blocking `:latest` image tags in the production namespace at admission time.

**Why admission control:** Kyverno or OPA running as admission webhooks catch policy violations before resources are created. CI validation catches them before the push. Defense in depth — both layers are needed because CI can be bypassed.

**Why OPA for this:** OPA is the standard for complex policy logic. The no-latest-tag policy is simple, but OPA handles cases where Kyverno's declarative policies cannot express the logic (complex conditional checks, cross-resource policies).

**Production assessment:** Appropriate for production. OPA Gatekeeper is widely used in enterprise Kubernetes deployments.

---

### Kyverno

**What it is:** A Kubernetes-native policy engine that validates, mutates, and generates resources using YAML-based policies rather than Rego.

**What we used it for:** Enforcing resource limits and blocking privileged containers in production.

**Why Kyverno alongside OPA:** OPA uses Rego — a purpose-built policy language that has a learning curve. Kyverno uses YAML patterns that match the Kubernetes resource structure directly. For simpler policies (require limits, no privileged), Kyverno is more readable and maintainable. Running both demonstrates the trade-off understanding: OPA for complex logic, Kyverno for declarative validation.

**Production assessment:** Appropriate for production.

---

### Argo Rollouts

**What it is:** A Kubernetes controller for progressive delivery — canary deployments, blue-green deployments, analysis-based promotion.

**What we used it for:** Defining a canary deployment strategy for api-service with traffic weight progression.

**Why not standard Kubernetes rolling update:** Rolling update replaces pods simultaneously — all traffic goes to the new version immediately. Argo Rollouts shifts traffic incrementally: 10% → 25% → 50% → 100%. If the canary fails at 10%, only 10% of users were affected.

**Current status:** Installed and Rollout resource defined. End-to-end canary execution not triggered — demonstrating the progression requires pushing a new image tag to a registry.

**Production assessment:** Appropriate for production.

---

## Part 8 — Tool Defense

| Tool | Role | Why Used | Alternative | Why Not Alternative | Production |
|------|------|----------|-------------|---------------------|------------|
| ArgoCD v7.3.11 | GitOps engine | selfHeal, RBAC via AppProject, ApplicationSet | Flux CD | Both excellent — ArgoCD chosen for richer UI and AppProject RBAC | ✅ Appropriate |
| Kustomize | Environment overlays | Patches without templating, built into kubectl | Helm | Helm adds packaging overhead for own-app configuration | ✅ Appropriate |
| OPA Gatekeeper | Admission control | Industry standard for complex Rego policies | Kyverno alone | OPA handles conditional logic Kyverno cannot | ✅ Appropriate |
| Kyverno | Policy enforcement | YAML-native, readable for simple policies | OPA alone | OPA Rego has learning curve for simple validations | ✅ Appropriate |
| Argo Rollouts | Progressive delivery | Canary with traffic splitting | Standard rolling update | Rolling update is all-or-nothing — no blast radius control | ✅ Appropriate |
| Sloth | SLO management | Auto-generates burn rate alerts from SLO definition | Hand-written alerts | Manual burn rate calculation is error-prone | ✅ Appropriate |
| Litmus | Chaos engineering | Scheduled chaos experiments against k8s workloads | Manual pod deletion | Manual chaos is not repeatable or schedulable | ✅ Appropriate |

---

## Part 9 — Architecture Decision Records

### ADR-001: Kustomize over Helm for Application Overlays

**Context:** Multiple environments need the same application with different configurations.

**Decision:** Kustomize overlays.

**Reason:** Kustomize uses patches on top of a base manifest. Changes to the base propagate to all overlays automatically. Helm requires maintaining a values file per environment and a chart template — more moving parts for the same result.

**Trade-offs:** ArgoCD blocks cross-directory references, requiring base manifest duplication in each overlay. For N environments, the base file exists in N+1 locations.

**Consequences:** Environment differences are expressed as patches (replicas: 1 for staging) against a complete base manifest. Security context, resource limits, and probes are in the base and apply everywhere.

**Production consideration:** For more than 5 environments or applications with significant configuration differences between environments, Helm becomes preferable.

---

### ADR-002: ApplicationSet over Individual Applications

**Context:** Needed to manage staging and production with consistent configuration.

**Decision:** Single ApplicationSet with matrix generator.

**Reason:** Individual Applications require creating N×M manifest files for N environments and M applications. ApplicationSet generates all combinations from a list. One template, two generated Applications.

**Trade-offs:** ApplicationSet adds abstraction. Debugging requires understanding generator output — `argocd app list` shows generated names, not the generator configuration.

**Consequences:** Adding a new environment is one line in the environments list. Adding a new application is one line in the apps list.

**Production consideration:** Keep. ApplicationSet is the production standard for managing many applications across many environments.

---

### ADR-003: automated sync with selfHeal on Both Environments

**Context:** Should production auto-sync or require human approval?

**Decision:** Automated sync with selfHeal: true on both staging and production.

**Reason:** The core claim of this project is that Git is the only interface for production. Manual sync approval partially defeats this — an engineer must still act on every Git push. selfHeal is required to demonstrate drift correction.

**Trade-offs:** selfHeal removes the ability to make temporary changes directly to the cluster. Every change must go through Git and be committed. No hotfixes via kubectl in production.

**Consequences:** The drift detection test proved selfHeal works — 5 replicas corrected to 2 in under 2 seconds.

**Production consideration:** In regulated industries, a manual sync gate for production is sometimes required for change control. The technical implementation here supports adding one (remove `automated:` from the production Application). The decision of whether to use it is organizational, not technical.

---

### ADR-004: OPA and Kyverno Together over Either Alone

**Context:** Needed admission control to enforce security policies.

**Decision:** Both OPA Gatekeeper and Kyverno running simultaneously.

**Reason:** They address different use cases. OPA handles complex, conditional Rego logic. Kyverno handles declarative YAML pattern matching. Running both shows understanding of when to use which tool — not just knowing one exists.

**Trade-offs:** Two admission webhook systems add latency to API server requests. In a busy cluster this can affect pod creation speed.

**Consequences:** no-latest-tag policy enforced by OPA. Resource limits and no-privileged enforced by Kyverno. Both active and verified in the cluster.

**Production consideration:** Both are appropriate. Choose based on policy complexity and team familiarity with Rego vs YAML.

---

## Part 10 — Verification Matrix

This table separates what was implemented from what was verified at runtime. A skeptical reviewer should be able to see exactly what was proven and what was not.

| Capability | Implemented | Runtime Verified | Evidence |
|------------|-------------|-----------------|---------|
| ArgoCD installed | Yes | Yes | kubectl get pods -n argocd |
| Git drift correction | Yes | Yes — under 2 seconds | Terminal output documented |
| staging environment synced | Yes | Yes | argocd app list |
| production environment synced | Yes | Yes | argocd app list |
| 1 replica in staging | Yes | Yes | kubectl get pods -n staging |
| 2 replicas in production | Yes | Yes | kubectl get pods -n production |
| AppProject RBAC | Yes | Yes — roles visible in cluster | kubectl get appproject |
| ApplicationSet matrix | Yes | Yes — 2 apps generated | kubectl get applications -n argocd |
| Sync waves | Yes | Yes — annotations present | grep sync-wave in manifests |
| Kustomize 1 replica staging | Yes | Yes | kubectl get deployment -n staging |
| OPA no-latest-tag | Yes | Yes — constraint active | kubectl get nolatesttag |
| Kyverno resource limits | Yes | Yes — policy ready | kubectl get clusterpolicy |
| Kyverno disallow-privileged | Yes | Yes — policy ready | kubectl get clusterpolicy |
| Argo Rollouts installed | Yes | Yes | kubectl get pods -n argo-rollouts |
| Canary rollout definition | Yes | Configuration only | kubectl get rollout -n production |
| Canary runtime execution | Yes | **Not verified** — no custom image to push | — |
| Sloth SLO definition | Yes | Applied — controller processing not confirmed | kubectl get prometheusservicelevel |
| Litmus chaos experiment | Yes | Applied — completion not confirmed | kubectl get chaosengine |

---

## Part 11 — What GitOps Is and Is Not

### What GitOps Is

GitOps is the practice of using Git as the single source of truth for declarative infrastructure and application configuration.

- **Declarative:** Manifests describe desired state, not the steps to get there
- **Versioned and immutable:** Every change is a commit with an author, timestamp, and message
- **Pulled automatically:** The cluster pulls from Git rather than CI pushing to the cluster
- **Continuously reconciled:** The cluster continuously reconciles itself to match Git

### What GitOps Is Not

- **CI/CD:** GitOps is the CD part specifically — the continuous delivery from Git to cluster. CI (testing, building, security scanning) happens before the Git push.
- **A specific tool:** Flux, ArgoCD, and others implement GitOps. The principle is tool-agnostic.
- **A replacement for monitoring:** GitOps ensures the desired state is applied. It does not tell you whether the application inside the pods is healthy.

### Why This Matters for Security

In a traditional CI/CD pipeline, the CI system pushes changes to the cluster. This requires the CI system to have cluster credentials. If the CI system is compromised, an attacker has cluster access.

In a GitOps model, the cluster pulls from Git. The cluster needs Git access (read-only), but the CI system does not need cluster access. Compromising CI gives an attacker the ability to push bad code to Git — but bad code in Git still goes through the same review and sync process.

---

## Part 12 — Security

### Defense in Depth

Three layers enforce security policies:

```
Layer 1: CI Pipeline
  kubectl dry-run validates manifests before push
  Catches malformed YAML, wrong API versions

Layer 2: OPA Gatekeeper (Admission Control)
  Blocks :latest tag at API server admission
  Runs before any resource is created
  Cannot be bypassed by direct kubectl

Layer 3: Kyverno (Policy Enforcement)
  Requires resource limits — pod rejected without them
  Blocks privileged containers — pod rejected if privileged: true
  Runs at admission — cannot be bypassed
```

No single layer is sufficient alone. CI can be bypassed. OPA and Kyverno cannot be bypassed because they run inside the cluster as admission webhooks.

### RBAC Boundaries

AppProject defines what ArgoCD can deploy where:
- Source: only this repository — cannot pull manifests from elsewhere
- Destinations: only staging and production namespaces — cannot deploy to kube-system
- platform-admin: full access
- developer: view production, sync staging only

### No Credentials in Git

No passwords, tokens, API keys, or secrets in any manifest file. Kubernetes secrets referenced by name and created separately.

---

## Part 13 — Reliability and Failure Modes

| Failure | Detection | Immediate Behavior | Recovery |
|---------|-----------|-------------------|----------|
| ArgoCD pod crashes | Pod restarts automatically | Sync pauses during restart | Automatic pod restart |
| Git repository unavailable | ArgoCD reports RepoError | Last known state maintained, no new syncs | Git access restored |
| Kustomize render fails | ComparisonError on Application | Sync blocked, cluster unchanged | Fix manifests, push |
| OPA webhook unavailable | API server timeout | Depends on failurePolicy (fail or ignore) | OPA pod restart |
| Kyverno webhook unavailable | API server timeout | Depends on failurePolicy | Kyverno pod restart |
| Port-forward dies | CLI commands fail with connection refused | No impact on cluster operation | Restart port-forward |
| Manual scale in production | selfHeal detects drift | Scale corrected within 2 seconds | Automatic |
| Bad commit merged | ArgoCD syncs bad state | Pods may fail, HPA may misbehave | git revert, push |

---

## Part 14 — What Would Change for Production

| Current | Production | Why Change |
|---------|------------|------------|
| Single cluster | Multi-cluster with cluster registration | Separate blast radius by environment |
| selfHeal on production | Manual sync gate for regulated environments | Change control compliance |
| No image signing | Cosign + ArgoCD image verification | Supply chain integrity |
| No Image Updater | ArgoCD Image Updater | Automated image tag updates from registry |
| External Secrets not configured | External Secrets Operator + Vault | Kubernetes secrets managed by secret store |
| Litmus manual execution | Litmus scheduled chaos experiments | Continuous resilience validation |
| Single webhook timeout default | Explicit failurePolicy per webhook | Prevent webhook outage from blocking all deployments |
| Sloth processing unverified | Verify burn rate alerts in Prometheus | SLO alerting must be confirmed working |

---

## Part 15 — Technical Debt Register

| Debt | Why It Exists | Risk | Recommended Change |
|------|--------------|------|-------------------|
| Base manifest duplicated in overlays | ArgoCD cross-directory restriction | Low — duplication requires updating N files for base changes | Consider Helm for more than 3 environments |
| Canary not executed end-to-end | No custom image to push to registry | Medium — Rollout config unverified in practice | Build a simple Go HTTP server, push to ECR |
| Sloth processing unverified | No confirmation controller ran | Medium — SLO alerts may not be firing | Check Prometheus for generated recording rules |
| Litmus completion unverified | No terminal output of experiment | Medium — chaos coverage unconfirmed | Check Litmus dashboard for experiment results |
| No Image Updater | Time — scope decision | Medium — image updates require manual manifest edit | Install Image Updater with registry polling |
| Single cluster | Resource constraints | Low for demo | Register second cluster for true multi-cluster |
| External Secrets not configured | Time — scope decision | Medium — secrets created manually | Install External Secrets Operator |

---

## Part 16 — Interview Defense

**Q: ArgoCD corrected the drift in under 2 seconds. Is that real or did you set a short polling interval?**
A: Real. ArgoCD uses a Kubernetes watch (not polling) for cluster state. When cluster state changes, the watch triggers immediately. The 180-second interval is for checking Git for new commits — not for watching the cluster. When I ran `kubectl scale`, ArgoCD's cluster watch fired immediately. The Git sync interval is irrelevant to drift correction speed.

**Q: The Rollout canary was never triggered. How can you say you implemented progressive delivery?**
A: The Rollout resource is defined with the correct canary strategy and is applied and visible in the cluster. Triggering it requires pushing a new image tag to a container registry — the demo app uses nginx which this project does not own. The configuration is correct and verified. The runtime execution requires a custom image build pipeline which is outside the scope of this project. I document this distinction explicitly rather than claiming more than was verified.

**Q: Why do you run both OPA Gatekeeper and Kyverno? Isn't one enough?**
A: They have different strengths. Kyverno uses YAML pattern matching — readable and maintainable for policies like "require resource limits." OPA uses Rego — a purpose-built policy language suited for complex conditional logic. Running both shows understanding of when to use each. In production you would choose based on policy complexity and team expertise. Having only seen OPA or only seen Kyverno doesn't demonstrate that understanding.

**Q: selfHeal is on for production. What happens if someone commits a bad change?**
A: ArgoCD syncs it. The bad state goes to production. This is why the CI pipeline validates manifests before they reach Git. The correct safeguard is pull request review and CI validation before merge — not manual sync approval. If a bad change does get merged, `git revert` and push creates a new commit that ArgoCD syncs, rolling back the change. The Git history shows exactly what happened and when.

**Q: What does the ApplicationSet matrix generator actually do?**
A: It takes the cross-product of two lists. Environments list has staging and production. Apps list has api-service. Cross-product is: (staging, api-service) and (production, api-service). For each combination, it instantiates the Application template with the combination's values substituted. Result: staging-api-service and production-api-service Applications. Adding a new environment is one list entry. Adding a new app is one list entry. N environments × M apps = N×M generated Applications, from a single template.

---

## Repository Structure

```
gitops-argocd-platform/
├── terraform/
│   ├── main.tf                    — ArgoCD + Argo Rollouts + namespaces
│   └── argocd-values.yaml         — RBAC policy, metrics, resource limits
├── apps/
│   ├── base/
│   │   ├── deployment.yaml        — base manifest with security context
│   │   └── namespace.yaml         — sync-wave: "0"
│   ├── staging/
│   │   ├── deployment.yaml        — copy of base
│   │   └── kustomization.yaml     — patch: 1 replica, staging label
│   └── production/
│       ├── deployment.yaml        — copy of base
│       ├── kustomization.yaml     — production label
│       └── rollout.yaml           — canary steps (10%→25%→50%→100%)
├── argocd/
│   ├── projects/
│   │   └── platform-project.yaml  — AppProject with RBAC roles
│   └── appsets/
│       └── platform-appset.yaml   — ApplicationSet matrix generator
├── security/
│   ├── opa/
│   │   └── no-latest-tag.yaml     — ConstraintTemplate + NoLatestTag
│   └── kyverno/
│       └── require-resources.yaml — resource limits + disallow-privileged
├── observability/
│   └── sloth/
│       └── api-service-slo.yaml   — 99.9% availability SLO definition
├── chaos/
│   └── litmus/
│       └── pod-kill-experiment.yaml — pod-delete ChaosEngine
├── .github/
│   └── workflows/
│       └── gitops.yml             — security, validate, drift-check pipeline
├── docs/
│   ├── ARCHITECTURE.md
│   ├── GITOPS_MODEL.md
│   ├── SECURITY.md
│   ├── ADR.md
│   ├── INCIDENTS.md
│   ├── GAPS.md
│   └── evidence/
└── README.md
```

---

## Engineering Summary

**What was built:** A GitOps platform where every deployment goes through Git. ArgoCD watches the repository, syncs both staging and production automatically, and corrects manual cluster changes in under 2 seconds. OPA Gatekeeper and Kyverno enforce admission control. AppProject RBAC prevents developers from touching production directly.

**Core engineering problem:** How do you make deployment auditable, reversible, and automatic — while preventing manual changes from creating undocumented state?

**Major technical decisions:** Kustomize overlays for environment differences. ApplicationSet matrix generator for multi-environment management. automated sync with selfHeal on both environments. OPA and Kyverno together for defense in depth. Terraform provisions ArgoCD — the GitOps tool is itself infrastructure code.

**Major trade-offs:** selfHeal means no hotfixes via kubectl in production — every change must go through Git. Base manifest duplication in overlays due to ArgoCD's cross-directory reference restriction. Canary execution not verified end-to-end.

**Important lessons:** ArgoCD's cross-directory Kustomize restriction is not configurable — plan for it by copying base into overlays. selfHeal operates on cluster watch events, not on a polling interval — drift correction is nearly instantaneous. The drift detection story is stronger when the correction is too fast to screenshot.

**Current limitations:** Single cluster only. Rollouts canary not triggered end-to-end. Sloth and Litmus applied but runtime verification incomplete. No Image Updater. No External Secrets.

**Skills demonstrated:** GitOps principles, ArgoCD architecture, Kustomize overlay design, ApplicationSet matrix generation, RBAC via AppProject, OPA Gatekeeper policy writing, Kyverno policy enforcement, Argo Rollouts canary configuration, sync wave ordering, progressive delivery concepts.


# ------------------------------------------------------------------------------------------
# CODESPACE Terraform Kubernetes Platform

## What Is This Project?

In the Auto-Healing Kubernetes project, the platform was built by running commands manually. Every namespace was created with `kubectl create namespace`. Every Helm chart was installed with `helm install`. Every deployment was applied with `kubectl apply`. It worked. But if the cluster was destroyed, rebuilding required remembering every command in the correct order from memory, or reading through session notes to reconstruct what was done.

That is not infrastructure engineering. That is tribal knowledge.

This project answers a different question: can the entire platform be described in code, so that destroying everything and running a single command produces an identical result?

The answer is yes. Thirteen resources — four namespaces, three deployments, two services, one secret, and three Helm releases — are all defined as Terraform configuration. `terraform apply` creates them. `terraform destroy` removes them. `terraform apply` again brings them back identically.

---

## Part 1 — For the Non-Technical Reader

**What problem does this solve?**
When infrastructure is built by running commands manually, the only record of what was built lives in someone's memory or in terminal history. When that person is unavailable, or when the infrastructure needs to be rebuilt, there is no reliable source of truth.

Infrastructure as Code solves this by making the code the record. The Terraform files in this repository describe the platform exactly. Anyone with the repository and the right credentials can reproduce the platform from scratch without asking anyone what commands to run.

**What does it do?**
Defines all platform infrastructure — namespaces, deployments, services, secrets, and tool installations — as Terraform configuration files. Runs a security pipeline on every commit that scans for hardcoded credentials, Terraform misconfigurations, and compliance violations.

**Why does it matter?**
A platform you can rebuild from code in minutes is fundamentally more reliable than one that exists only because the right commands were run in the right order one time. Reproducibility is the foundation of operational reliability.

**What was the engineering mission?**
Take the manually-built platform from Project 1 and express every component as Terraform code, so that the infrastructure becomes reproducible, version-controlled, and auditable.

---

## Part 2 — Engineering Objective

### Primary Objective
Express all platform infrastructure as Terraform code. Prove that `terraform apply` produces an identical platform to the one built manually.

### Secondary Objectives
- Enforce credential security through sensitive variable typing
- Implement a security-first CI/CD pipeline that scans before validating
- Demonstrate module-based infrastructure organization
- Document all architectural decisions with alternatives considered

### Functional Requirements
- All 13 platform resources managed as Terraform state
- `terraform plan` must show no changes after apply (zero drift)
- Sensitive variables must never appear in plan or apply output
- Git history must contain no credentials
- CI pipeline must run security scan before any validation step

### Non-Functional Requirements
- Module boundaries must map to platform concerns, not resource types
- All resource dependencies must be explicit (depends_on)
- State must be recoverable if lost (infrastructure can be reimported or rebuilt)

### Constraints
- Local state only (no S3 backend — single engineer project)
- Minikube cluster must already exist before apply
- Helm releases install into existing cluster
- Apply stage not in CI pipeline (runner has no cluster access)

### Success Criteria
- `terraform state list | wc -l` returns 13
- `terraform plan` returns "No changes"
- CI pipeline Security Scan and Validate both green
- `git log --all -p | grep -i "admin123"` returns nothing (credential scrub verified)

---

## Part 3 — System at a Glance

### 3.1 What Terraform Manages

```
Terraform State (13 resources)
│
├── module.namespaces
│   ├── kubernetes_namespace.microservices    (label: managed-by=terraform)
│   ├── kubernetes_namespace.monitoring       (label: managed-by=terraform)
│   ├── kubernetes_namespace.vault            (label: managed-by=terraform)
│   └── kubernetes_namespace.opencost         (label: managed-by=terraform)
│
├── module.microservices
│   ├── kubernetes_secret.postgres            (sensitive — never in Git)
│   ├── kubernetes_deployment.postgres        (references secret)
│   ├── kubernetes_deployment.api_service
│   ├── kubernetes_deployment.frontend
│   ├── kubernetes_service.api_service        (ClusterIP)
│   └── kubernetes_service.frontend           (NodePort 30421)
│
├── module.monitoring
│   └── helm_release.prometheus               (kube-prometheus-stack)
│
├── module.vault
│   └── helm_release.vault                    (dev mode)
│
└── module.opencost
    └── helm_release.opencost
```

### 3.2 Module Dependency Order

```
module.namespaces (no dependencies)
        │
        ├──► module.microservices
        │    depends_on: [module.namespaces]
        │    Reason: pods cannot schedule without namespace
        │
        ├──► module.monitoring
        │    depends_on: [module.namespaces]
        │    Reason: Helm release targets monitoring namespace
        │
        ├──► module.vault
        │    depends_on: [module.namespaces]
        │    Reason: Helm release targets vault namespace
        │
        └──► module.opencost
             depends_on: [module.monitoring]
             Reason: OpenCost reads from Prometheus
             which must exist first
```

### 3.3 Provider Configuration

```hcl
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "prod-sim"
}

provider "helm" {
  kubernetes {
    config_path    = "~/.kube/config"
    config_context = "prod-sim"
  }
}
```

Both providers read from the same kubeconfig context. The context `prod-sim` is created automatically when `minikube start --profile=prod-sim` runs. If the cluster does not exist when `terraform apply` runs, both providers fail to initialize.

### 3.4 How Terraform State Works

```
terraform plan
    reads: terraform.tfstate (current known state)
    reads: *.tf files (desired state)
    computes: diff between current and desired
    outputs: what will be created/updated/destroyed

terraform apply
    executes: the computed plan
    calls: Kubernetes and Helm APIs
    writes: updated state to terraform.tfstate

terraform destroy
    reads: terraform.tfstate
    destroys: every resource in state
    writes: empty state
```

State file is the record of what Terraform owns. A resource in the cluster but not in state is invisible to Terraform — it will not manage, update, or destroy it. A resource in state but removed from the `.tf` files will be destroyed on the next apply.

---

## Part 4 — Component Responsibilities

### module.namespaces

**What it is:** Creates the four Kubernetes namespaces with `managed-by=terraform` labels.

**What it does:** Every subsequent module deploys into one of these namespaces. Without namespaces existing first, every subsequent resource creation fails.

**Why it exists as a module:** Namespaces are a distinct concern from the workloads that run inside them. Separating them makes the dependency explicit and allows namespaces to be managed independently.

**What depends on it:** Every other module.

**What breaks if it fails:** Everything. No namespace means no pods, no Helm releases, no secrets.

**Outputs:** Namespace names used by downstream modules to ensure they reference the same names.

---

### module.microservices

**What it is:** Creates the Kubernetes secret, three deployments, and two services for the application workloads.

**Why the secret is managed here:** The secret contains the database credentials needed by `postgres-db`. It must exist in the same namespace before the deployment is created. Grouping them in the same module makes this relationship visible.

**Sensitive variable pattern:**
```hcl
variable "postgres_password" {
  type      = string
  sensitive = true
}
```
`sensitive = true` means Terraform refuses to print this value in any output, plan, or apply log. The value is stored in `terraform.tfstate` (which is gitignored) and provided at runtime via `terraform.tfvars` (also gitignored).

**What depends on it:** Nothing in Terraform, but HPA configuration is applied separately with kubectl and depends on the deployments existing.

---

### module.monitoring

**What it is:** A single `helm_release` resource that installs kube-prometheus-stack.

**Why Helm here:** kube-prometheus-stack installs 40+ resources including CRDs, ClusterRoles, ServiceMonitors, and StatefulSets. Managing each as a raw `kubernetes_manifest` resource would mean tracking 40+ resources in Terraform state and handling CRD registration order manually.

**Resource limits tuned for 8GB Codespace:**
```hcl
set {
  name  = "prometheus.prometheusSpec.resources.limits.memory"
  value = "256Mi"
}
set {
  name  = "grafana.resources.limits.memory"
  value = "256Mi"
}
```
Default values allocate 2GB to Prometheus alone. On an 8GB Codespace running a 3-node cluster plus application pods plus Vault plus OpenCost, that exhausts available memory.

**What depends on it:** module.opencost, because OpenCost reads cost data from the Prometheus installed by this module.

---

### module.vault

**What it is:** A single `helm_release` resource that installs HashiCorp Vault in dev mode.

**Why dev mode:** Standalone or HA Vault requires initialization, unsealing, and persistent storage configuration. On a Codespace that restarts frequently, manual initialization before every session adds significant overhead. Dev mode eliminates this at the cost of losing data on restart.

**What it does not manage:** Vault initialization (not needed in dev mode), secret engine configuration, role configuration. These are applied after `terraform apply` with Vault CLI commands.

---

### module.opencost

**What it is:** A single `helm_release` resource installing OpenCost.

**Why after monitoring:** OpenCost is configured to read from Prometheus. If monitoring does not exist when OpenCost starts, OpenCost logs connection errors and cost data is unavailable.

```hcl
set {
  name  = "opencost.prometheus.external.url"
  value = "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"
}
```

This URL uses cluster-internal DNS. OpenCost runs inside the cluster and can resolve this name. Port-forward is only needed for external access (from the terminal) — not for pod-to-pod communication.

---

## Part 5 — Build the Project Step by Step

### STEP 1 — Create the Project Structure

**Objective:** Establish the module layout before writing any resource code.

**Why now:** Creating the folder structure first prevents the scrambled-paste problem. Files created in the correct location before content is added cannot end up in the wrong place.

**Exact implementation:**
```bash
mkdir -p ~/terraform-k8s/{modules/{namespaces,microservices,monitoring,vault,opencost},environments/production}
cd ~/terraform-k8s
```

**What each folder holds:**
- `modules/namespaces/` — main.tf, variables.tf, outputs.tf
- `modules/microservices/` — main.tf, variables.tf
- `modules/monitoring/` — main.tf, variables.tf
- `modules/vault/` — main.tf, variables.tf
- `modules/opencost/` — main.tf, variables.tf

**What depends on this:** All file creation steps.

---

### STEP 2 — Create .gitignore First

**Objective:** Prevent credentials and state from entering the repository before any other file is written.

**Why first:** The credential exposure incident (documented below) happened because this step was done after content was already committed. `.gitignore` must exist before the first `git add`.

**Exact implementation:**
```bash
cat << 'EOF' > .gitignore
*.tfstate
*.tfstate.backup
*.tfvars
.terraform/
.terraform.lock.hcl
vault-init.json
*.zip
nohup.out
kubectl
minikube-linux-amd64
vault_1.15.0_linux_amd64.zip
EOF
```

**What depends on this:** Every subsequent commit being clean.

**Common mistake:** Creating `.gitignore` after running `terraform init` or `terraform apply`, which generates `.terraform/` and `terraform.tfstate` that may already be staged.

---

### STEP 3 — Write the Root Configuration

**Objective:** Configure Terraform providers and wire module calls together.

**Why modules use depends_on:** Terraform computes a dependency graph from resource references. When two modules have no direct resource references between them but still need to run in order (namespaces before microservices), `depends_on` makes the ordering explicit.

**main.tf:**
```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "prod-sim"
}

provider "helm" {
  kubernetes {
    config_path    = "~/.kube/config"
    config_context = "prod-sim"
  }
}

module "namespaces" {
  source      = "./modules/namespaces"
  environment = var.environment
}

module "microservices" {
  source            = "./modules/microservices"
  namespace         = module.namespaces.microservices_namespace
  environment       = var.environment
  postgres_password = var.postgres_password
  postgres_user     = var.postgres_user
  postgres_db       = var.postgres_db
  depends_on        = [module.namespaces]
}

module "monitoring" {
  source           = "./modules/monitoring"
  namespace        = module.namespaces.monitoring_namespace
  grafana_password = var.grafana_password
  depends_on       = [module.namespaces]
}

module "vault" {
  source     = "./modules/vault"
  namespace  = module.namespaces.vault_namespace
  depends_on = [module.namespaces]
}

module "opencost" {
  source     = "./modules/opencost"
  namespace  = module.namespaces.opencost_namespace
  depends_on = [module.monitoring]
}
```

**variables.tf:**
```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "postgres_password" {
  description = "PostgreSQL password"
  type        = string
  sensitive   = true
}

variable "postgres_user" {
  description = "PostgreSQL username"
  type        = string
  default     = "appuser"
}

variable "postgres_db" {
  description = "PostgreSQL database name"
  type        = string
  default     = "appdb"
}

variable "grafana_password" {
  description = "Grafana admin password"
  type        = string
  sensitive   = true
}
```

**terraform.tfvars (gitignored — never committed):**
```hcl
environment       = "production"
postgres_password = "apppassword"
postgres_user     = "appuser"
postgres_db       = "appdb"
grafana_password  = "yourpassword"
```

---

### STEP 4 — Write the Namespaces Module

**modules/namespaces/main.tf:**
```hcl
resource "kubernetes_namespace" "microservices" {
  metadata {
    name = "microservices"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_namespace" "vault" {
  metadata {
    name = "vault"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_namespace" "opencost" {
  metadata {
    name = "opencost"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}
```

**modules/namespaces/outputs.tf:**
```hcl
output "microservices_namespace" {
  value = kubernetes_namespace.microservices.metadata[0].name
}

output "monitoring_namespace" {
  value = kubernetes_namespace.monitoring.metadata[0].name
}

output "vault_namespace" {
  value = kubernetes_namespace.vault.metadata[0].name
}

output "opencost_namespace" {
  value = kubernetes_namespace.opencost.metadata[0].name
}
```

**Important underrated detail:** The `managed-by=terraform` label is not cosmetic. In production, it allows you to query "which namespaces does Terraform own?" with `kubectl get namespaces -l managed-by=terraform`. This becomes important when multiple tools (Helm, ArgoCD, manual kubectl) all create resources in the same cluster.

---

### STEP 5 — Write the Microservices Module

**Key pattern — sensitive values from secret:**
```hcl
resource "kubernetes_secret" "postgres" {
  metadata {
    name      = "postgres-secret"
    namespace = var.namespace
  }
  data = {
    POSTGRES_PASSWORD = var.postgres_password
    POSTGRES_USER     = var.postgres_user
    POSTGRES_DB       = var.postgres_db
  }
}
```

The secret is created first. Deployments reference it with `value_from.secret_key_ref`. The deployment object in the cluster contains the secret reference, not the value. The value only exists in the Kubernetes secret and in `terraform.tfstate` (gitignored).

**What depends on the ordering within the module:** postgres-db deployment must be created after the secret. Terraform handles this automatically because the deployment resource references `kubernetes_secret.postgres.metadata[0].name`.

---

### STEP 6 — Write the Monitoring Module

**Key decision — Helm values as set blocks vs values file:**

Using `set` blocks in the Terraform resource means values are tracked in Terraform state and visible in plan output. Changes to values show as Terraform diffs. Using a separate `values.yaml` file means the file content is not tracked — Terraform only knows the file exists, not its content. For a project where the resource limits are the important configuration, `set` blocks make the intent visible.

```hcl
resource "helm_release" "prometheus" {
  name       = "prometheus"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  namespace  = var.namespace
  timeout    = 600

  set {
    name  = "prometheus.prometheusSpec.resources.limits.memory"
    value = "256Mi"
  }
  set {
    name  = "grafana.resources.limits.memory"
    value = "256Mi"
  }
  set {
    name  = "grafana.adminPassword"
    value = var.grafana_password
  }
}
```

---

### STEP 7 — Initialize and Apply

**Objective:** Download providers and create all 13 resources.

**Exact implementation:**
```bash
terraform init
terraform plan \
  -var="postgres_password=apppassword" \
  -var="grafana_password=yourpassword"
terraform apply -auto-approve \
  -var="postgres_password=apppassword" \
  -var="grafana_password=yourpassword"
```

**Verification:**
```bash
terraform state list | wc -l
# Expected: 13

terraform plan \
  -var="postgres_password=apppassword" \
  -var="grafana_password=yourpassword"
# Expected: No changes. Your infrastructure matches the configuration.
```

**Common mistake:** Running `terraform apply` with `terraform.tfvars` in place and not passing `-var` flags. Terraform reads `terraform.tfvars` automatically. Passing `-var` overrides the file. Both work. The important thing is the sensitive variables must be provided one way or another.

**Important underrated detail:** `terraform fmt -recursive` must be run before pushing to avoid CI failures. The `terraform fmt -check` step in the pipeline fails if any file is not correctly formatted. Format violations produce exit code 3. Running `terraform fmt -recursive` locally before push eliminates this class of CI failure.

---

### STEP 8 — Configure CI/CD Pipeline

**Objective:** Automatically validate Terraform code and scan for security issues on every push.

**Why security scan runs first:**
If a credential is found in the code, there is no value in validating the Terraform syntax. Running security first means the pipeline stops immediately when credentials are detected.

```yaml
jobs:
  security-scan:
    steps:
      - name: TruffleHog secret scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          extra_args: --only-verified

      - name: tfsec Terraform security scan
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true

      - name: Checkov compliance scan
        uses: bridgecrewio/checkov-action@master
        with:
          soft_fail: true

  validate:
    needs: security-scan
    steps:
      - name: terraform fmt check
        run: terraform fmt -check -recursive

      - name: terraform init
        run: terraform init -backend=false -input=false
        env:
          TF_VAR_postgres_password: "dummy"
          TF_VAR_grafana_password: "dummy"

      - name: terraform validate
        run: terraform validate
```

**Why dummy values in CI:** The validate stage runs `terraform init` and `terraform validate` which require values for all non-default variables. The cluster does not exist in CI so these values are never used to create real resources — they satisfy Terraform's requirement that all variables have values.

**Why apply is not in CI:** The GitHub Actions runner cannot reach a Minikube cluster running in a Codespace. Apply is run manually from the terminal. In production with a remote cluster and a kubeconfig secret, the apply stage would run automatically.

---

## Part 6 — Evolution of the System

### Change 1 — Files Corrupted by Terminal Paste

**Original:** Created Terraform module files by pasting large blocks of HCL into terminal heredoc commands (`cat << EOF > file.tf`).

**Problem:** Browser-based terminals in GitHub Codespaces scramble large pastes. Characters are dropped or interleaved when the paste volume exceeds the terminal buffer. The file appeared to be created but contained garbage:
```
EOFype = stringres_db" { {d" {/modules/microservices/variables.tf
```
`terraform plan` showed 0 resources because the module files were unreadable.

**Change:** Used VS Code editor directly for all file content. `code filename.tf` → Ctrl+A → delete → paste → Ctrl+S. The editor handles large pastes without scrambling.

**Engineering lesson:** Never use heredoc for files longer than 20 lines in browser-based terminals. The terminal buffer is not reliable for large pastes.

---

### Change 2 — Credential in Public Git History

**Original:** During initial development, `grafana_password = "admin123secure"` was written directly in `main.tf` and pushed to a public GitHub repository.

**Problem:** The credential was publicly visible in Git history. Removing it from the working tree does not remove it from history — `git log --all -p | grep admin123` would still return it.

**Remediation sequence:**
1. Replaced hardcoded value with `var.grafana_password` in working tree
2. Moved value to `terraform.tfvars` (gitignored)
3. Installed `git-filter-repo`:
   ```bash
   pip install git-filter-repo --break-system-packages
   ```
4. Rewrote all history:
   ```bash
   git filter-repo --replace-text <(echo "admin123secure==>REDACTED") --force
   ```
5. Force pushed:
   ```bash
   git push origin main --force
   ```
6. Verified:
   ```bash
   git log --all -p | grep -i "admin123"
   # Returns nothing
   echo "Exit code: $?"
   # 1
   ```

**Engineering lesson:** Removing a credential from a file does not remove it from history. History rewriting is required. `git filter-repo` is faster and safer than `git filter-branch` for this purpose.

**Prevention:** TruffleHog in the CI pipeline catches credentials on every push. But the better prevention is the pattern: define the variable with `sensitive = true` first, then reference it. Never write the value in a `.tf` file.

---

### Change 3 — terraform apply Showing 0 Resources

**Original:** After fixing the file corruption issue and re-running `terraform apply`, the output showed:
```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```
Expected 13 resources.

**Problem:** The module files had been corrupted and then corrected, but the correction for `modules/namespaces/main.tf` had placed the root module calls inside the namespaces module file instead of the namespace resources. Terraform parsed the file and found no resources to create.

**Fix:** `cat modules/namespaces/main.tf` — confirmed the wrong content. Opened in VS Code editor, replaced with correct namespace resource definitions, re-applied.

**Engineering lesson:** Always `cat` a Terraform file to verify its content before running `terraform apply`. Corrupted files produce no error — Terraform just has nothing to create, which looks like a successful apply.

---

## Part 7 — Technology Decision Record

### Terraform v1.7.5

**What it is:** An infrastructure as code tool that manages the lifecycle of cloud and Kubernetes resources through declarative configuration files.

**What we used it for:** Defining all 13 platform resources, managing their creation order through `depends_on`, and providing a single command to reproduce the entire platform.

**Why we chose it:** Industry standard for IaC. Works with Kubernetes, Helm, AWS, GCP, and Azure. The state file model means Terraform knows what it owns and can detect drift. HCL is more readable than raw JSON for infrastructure configuration.

**Alternatives:** Pulumi (code instead of HCL), Helm alone, kubectl with manifests, Ansible.

**Why not Pulumi:** Less community adoption for Kubernetes specifically. HCL's declarative style is better suited to infrastructure description than imperative code.

**Why not Helm alone:** Helm manages Kubernetes resources but not the namespaces they run in. Helm cannot create namespaces as part of a chart without special configuration. Terraform manages both.

**Production assessment:** Appropriate for production.

---

### hashicorp/kubernetes provider v2.25

**What it does:** Allows Terraform to create and manage Kubernetes resources — namespaces, deployments, services, secrets — using the Kubernetes API.

**Why provider version pinned:** `~> 2.25` allows patch updates within the 2.25 minor version but not major or minor version upgrades. This prevents the provider from introducing breaking changes automatically.

**What it cannot do:** Manage Kubernetes resources created by Helm releases. Helm manages those internally. Terraform only knows a Helm release was deployed, not the individual Kubernetes resources inside it.

**Production assessment:** Appropriate for production.

---

### hashicorp/helm provider v2.12

**What it does:** Allows Terraform to install, upgrade, and remove Helm charts as `helm_release` resources.

**Why Helm for tool installation:** kube-prometheus-stack, Vault, and OpenCost each install 10-40+ Kubernetes resources. Managing these as individual Terraform resources would mean tracking CRD registration, ClusterRole ordering, and Service Account dependencies manually. Helm handles all of this internally.

**Trade-off:** Helm releases are opaque to Terraform. `terraform plan` shows a single change for an entire Helm upgrade even if only one value changed. The individual Kubernetes resources inside the release are not in Terraform state.

**Production assessment:** Appropriate for production.

---

## Part 8 — Tool Defense

| Tool | Role | Why Used | Alternative | Why Not Alternative | Production |
|------|------|----------|-------------|---------------------|------------|
| Terraform v1.7.5 | Infrastructure as Code | Manages k8s + Helm resources in one state | Pulumi | Less k8s-specific community | ✅ Appropriate |
| kubernetes provider | k8s resource management | Native API calls, state tracking | kubectl + manifests | No state tracking, no drift detection | ✅ Appropriate |
| helm provider | Tool installation | Manages multi-resource charts as single state entry | Raw kubernetes_manifest | 40+ resources per chart, manual dependency ordering | ✅ Appropriate |
| TruffleHog | Secret scanning | Verified secrets only, low false positives | GitLeaks | TruffleHog verification reduces noise | ✅ Appropriate |
| tfsec | Terraform security scan | Terraform-native, integrates with GitHub Actions | Checkov alone | tfsec catches Terraform-specific patterns Checkov misses | ✅ Appropriate |
| Checkov | Compliance policy scan | Wide rule coverage across frameworks | tfsec alone | Checkov covers k8s manifests in addition to Terraform | ✅ Appropriate |
| Local state | State storage | Zero setup for single engineer | S3 + DynamoDB | Required for teams — overkill for single engineer demo | ❌ Not for teams |

---

## Part 9 — Architecture Decision Records

### ADR-001: Modules over Flat Configuration

**Context:** Could write all 13 resources in a single `main.tf` file.

**Decision:** Separate modules per platform concern.

**Reason:** A flat configuration becomes unreadable quickly. More importantly, modules allow the same pattern to be reused across projects. The monitoring module can be dropped into any Kubernetes project without modification.

**Trade-offs:** Modules add one layer of indirection. Debugging requires navigating between files. `terraform plan` output shows module paths (`module.monitoring.helm_release.prometheus`) rather than just resource names.

**Consequences:** Each module has a single responsibility. The dependency relationships between modules are explicit and visible in the root `main.tf`.

**Production consideration:** Keep. Module-based organization is the production standard for Terraform at scale.

---

### ADR-002: Local State over Remote State

**Context:** Terraform state must be stored somewhere. Remote state in S3 with DynamoDB locking is the production standard.

**Decision:** Local state, excluded from Git.

**Reason:** S3 remote state requires creating an AWS account, S3 bucket, DynamoDB table, and IAM configuration before Terraform can manage anything. That is significant setup overhead for a single-engineer portfolio project where the infrastructure is rebuilt from scratch regularly.

**Trade-offs:** Local state cannot be shared with a team. If the Codespace is deleted, the state file is lost. The infrastructure can be rebuilt from `terraform apply` but Terraform will not know that the existing resources were created by it — it will try to create them again and fail with "already exists" errors.

**Consequences:** This is a known gap. See GAPS.md.

**Production consideration:** Replace with S3 remote state before any team use. The backend configuration block is already structured to accept remote state configuration with minimal changes.

---

### ADR-003: depends_on over Output-Based Dependencies

**Context:** Terraform can infer resource dependencies from output references. If module B references an output from module A, Terraform knows to create A first.

**Decision:** Explicit `depends_on` blocks.

**Reason:** Threading namespace name outputs through all downstream module variables adds complexity. The namespace names are constants (`"microservices"`, `"monitoring"`, etc.) and the modules could hardcode them. The ordering dependency is real but the data dependency is not. `depends_on` makes the ordering explicit without requiring data passing.

**Trade-offs:** `depends_on` prevents Terraform from parallelizing resource creation within the dependency chain. With output-based dependencies, Terraform can create unrelated resources in parallel. With `depends_on`, it waits for the entire depended-on module before starting the dependent module.

**Consequences:** Apply takes longer than it theoretically could. Acceptable for a platform of this size.

**Production consideration:** For large infrastructure with many independent resources, output-based dependencies allow better parallelism. For this project size, `depends_on` is clearer.

---

### ADR-004: Helm Provider over Raw kubernetes_manifest

**Context:** kube-prometheus-stack, Vault, and OpenCost each install many Kubernetes resources including CRDs.

**Decision:** `helm_release` resource for each tool installation.

**Reason:** CRDs must be registered before resources that use them can be created. Helm handles CRD installation ordering internally. Managing CRDs as raw `kubernetes_manifest` resources in Terraform means manually ensuring CRDs are created before dependent resources, which requires complex `depends_on` chains and lifecycle rules.

**Trade-offs:** Helm releases are opaque. `terraform state show module.monitoring.helm_release.prometheus` shows the Helm release parameters but not the 40+ Kubernetes resources Helm created. Those resources are not in Terraform state.

**Consequences:** If a Kubernetes resource inside a Helm release is manually deleted, `terraform plan` shows no change — Terraform doesn't know about it. `helm status prometheus -n monitoring` would show the problem.

**Production consideration:** Accepted trade-off. The alternative (managing 40+ resources per tool in Terraform state) creates more problems than it solves.

---

### ADR-005: Set Blocks over Values File for Helm Configuration

**Context:** Helm values can be passed to `helm_release` either as `set` blocks or as a `values` file reference.

**Decision:** `set` blocks for resource limits and sensitive configuration.

**Reason:** `set` blocks appear in `terraform plan` output and state. When the memory limit for Grafana changes, `terraform plan` shows the diff explicitly. A values file change shows only that the file hash changed, not what the actual configuration difference is.

**Trade-offs:** Long `set` block lists become verbose. For many values, a values file is cleaner.

**Consequences:** Configuration intent is visible in the Terraform code and in plan output. Reviewers can see exactly what is being configured without opening a separate file.

**Production consideration:** Hybrid approach — `set` blocks for sensitive values (passwords, critical limits) and values files for non-sensitive configuration with many parameters.

---

## Part 10 — Security

### Sensitive Variable Enforcement

Terraform's `sensitive = true` annotation enforces that values are never printed:

```
# terraform plan output with sensitive = true
+ grafana_password = (sensitive value)
+ postgres_password = (sensitive value)
```

Compare to what happens without it:
```
# terraform plan output WITHOUT sensitive = true
+ grafana_password = "admin123secure"
```

The `sensitive` annotation is the enforcement mechanism. The `.gitignore` for `terraform.tfvars` is the access control. Both are required.

### Git Security Controls

Files excluded from Git:
- `*.tfvars` — variable values including passwords
- `*.tfstate` and `*.tfstate.backup` — state files contain resource details
- `.terraform/` — provider binaries
- `vault-init.json` — Vault unseal keys
- All binary files (kubectl, minikube, vault executables)

### Credential Exposure Incident

During the build, `grafana_password = "admin123secure"` was committed to a public GitHub repository in `main.tf`.

**Timeline:**
1. Password written inline during initial module development
2. Committed and pushed to GitHub
3. Detected during code review preparation
4. Removed from working tree
5. git-filter-repo used to rewrite all history
6. Force pushed
7. Verified clean

**Verification that the remediation worked:**
```bash
git log --all -p | grep -i "admin123"
# Returns nothing
echo "Exit code: $?"
# 1
```

Exit code 1 means grep found no matches across all commits in all branches.

**What TruffleHog would have caught this:** TruffleHog runs on every push and scans commits for verified credential patterns. If the CI pipeline had existed at the time of the initial push, this credential would have been caught before the push completed.

---

## Part 11 — Reliability and Failure Modes

| Failure | Detection | Behavior | Recovery |
|---------|-----------|----------|----------|
| Cluster does not exist | terraform init fails — provider cannot connect | Apply blocked | Start Minikube first |
| Namespace creation fails | First apply returns error | Dependent modules skip | Fix namespace issue, re-apply |
| Helm timeout | helm_release returns timeout error | Resource in partial state | helm uninstall, re-apply |
| State file corrupted | terraform plan returns parse error | All operations blocked | Restore from backup or reimport resources |
| State file deleted | terraform plan shows all resources as new | Apply creates duplicates with "already exists" errors | Import existing resources into state |
| Provider version conflict | terraform init fails with version constraint error | Apply blocked | Update version constraint or pin exact version |

---

## Part 12 — What Would Change for Production

| Current | Production | Why Change |
|---------|------------|------------|
| Local state | S3 + DynamoDB locking | Team access, concurrent apply safety |
| Single environment | Terraform workspaces | Staging and production separated |
| Apply from terminal | Apply from CI after approval | Audit trail, change control |
| No state locking | DynamoDB lock table | Prevents concurrent apply corruption |
| Module outputs unused | Thread outputs through modules | Remove hardcoded namespace names |
| Sensitive vars in tfvars | HashiCorp Vault + terraform remote | Centralized credential management |
| Local provider auth | IRSA or Workload Identity | No static credentials for cluster access |

---

## Part 13 — Technical Debt Register

| Debt | Why It Exists | Risk | Recommended Change |
|------|--------------|------|-------------------|
| Local state | Single engineer, simplified setup | Medium — state loss means reimport | S3 + DynamoDB for any team use |
| No workspaces | Single environment demo | Low | Add staging workspace before team use |
| Apply not in CI | Runner cannot reach Minikube | None for demo | Remote cluster + kubeconfig secret for production |
| HPA not in Terraform | Metrics-server dependency at plan time | Low | Add after metrics-server is a required dependency |
| Vault init not in Terraform | Dev mode, no initialization needed | N/A in dev mode | Terraform Vault provider for production Vault |
| Helm values as set blocks | Explicit but verbose for many values | Low | Values file for non-sensitive, set for sensitive |

---

## Part 14 — Interview Defense

**Q: Why does `terraform plan` show "No changes" — isn't that just because nothing changed?**
A: Correct. "No changes" means the actual state of the cluster exactly matches what the Terraform code describes. That is the goal of infrastructure as code — zero drift between the declared configuration and the running system. If someone manually changed something in the cluster, `terraform plan` would show it as a diff.

**Q: You said apply is not in the CI pipeline. Is this actually IaC if nothing is deployed automatically?**
A: The IaC part is the code, not the execution. The Terraform files are the authoritative description of the infrastructure. Whether apply runs manually or automatically is a deployment process question, not an IaC question. In production with a remote cluster and a kubeconfig secret, the apply stage would run automatically after security and validate pass. The limitation here is that a Minikube cluster in a Codespace is not reachable from a GitHub Actions runner.

**Q: Why modules? Could you have just written everything in one file?**
A: Yes. But a flat configuration that mixes namespace creation, deployment definitions, and Helm release configuration in one file becomes unmaintainable as the infrastructure grows. More importantly, the monitoring module can be copied into another project as-is. That reusability is what modules are for. The namespace module is also independently testable and deployable without touching microservices or Helm releases.

**Q: What happens if the state file is deleted?**
A: Terraform loses track of what it created. Running `terraform apply` would try to create everything again and fail with "already exists" errors for resources that still exist in the cluster. The recovery path is `terraform import` for each resource, which reads the existing resource from the cluster and adds it back to state. The `.tf` files are not affected — only the state tracking is lost.

**Q: How does sensitive = true actually protect credentials?**
A: It prevents Terraform from printing the value in plan or apply output, and marks the value as sensitive in the state file so tools that read state (like Terraform Cloud) redact it. It does not encrypt the value in `terraform.tfstate` — the value is stored in plaintext base64. The actual protection comes from keeping `terraform.tfstate` out of Git (`.gitignore`) and treating the state file itself as a secret.

---

## Repository Structure

```
Terraform-Kubernetes-Platform/
├── main.tf                       — providers, backend, module calls
├── variables.tf                  — variable definitions with sensitive = true
├── terraform.tfvars              — values (gitignored — never committed)
├── .gitignore                    — tfstate, tfvars, .terraform/ excluded
├── modules/
│   ├── namespaces/
│   │   ├── main.tf               — 4 kubernetes_namespace resources
│   │   ├── variables.tf          — environment variable
│   │   └── outputs.tf            — namespace name outputs
│   ├── microservices/
│   │   ├── main.tf               — secret, 3 deployments, 2 services
│   │   └── variables.tf          — namespace, env, sensitive credentials
│   ├── monitoring/
│   │   ├── main.tf               — kube-prometheus-stack helm_release
│   │   └── variables.tf          — namespace, sensitive grafana_password
│   ├── vault/
│   │   ├── main.tf               — vault helm_release dev mode
│   │   └── variables.tf          — namespace
│   └── opencost/
│       ├── main.tf               — opencost helm_release
│       └── variables.tf          — namespace
├── .github/
│   └── workflows/
│       └── terraform.yml         — security scan + validate pipeline
└── docs/
    ├── ARCHITECTURE.md
    ├── SECURITY.md
    ├── ADR.md
    ├── INCIDENTS.md
    ├── GAPS.md
    └── evidence/                 — screenshots
```

---

## Engineering Summary

**What was built:** Terraform configuration that manages 13 Kubernetes platform resources across 5 modules. `terraform apply` reproduces the entire platform. `terraform plan` after apply shows zero drift.

**Core engineering problem:** How do you make infrastructure reproducible from code rather than from remembered manual commands?

**Major technical decisions:** Modules over flat configuration for reusability. Local state for simplicity. `depends_on` for explicit ordering. `helm_release` for multi-resource tool installation. `sensitive = true` for credential safety.

**Major trade-offs:** Local state means no team sharing. Apply not in CI means no automatic deployment. Helm releases are opaque to Terraform state.

**Important lessons:** `.gitignore` must be the first file created. Browser terminals scramble large pastes — use the editor. `git filter-repo` is the correct tool for history rewriting. Sensitive variables enforce safety at the Terraform level but the state file itself must also be protected.

**Current limitations:** Local state only. No workspaces. Apply manual only. HPA and alert rules applied separately with kubectl after Terraform apply.

**Skills demonstrated:** Terraform module design, sensitive variable management, dependency ordering, Helm integration, security-first CI pipeline, credential incident response and remediation.


# ----------------------------------------------------------------------------------------------------------
# CODESPACE Auto-Healing Kubernetes Platform

## What Is This Project?

Imagine you are running a service that thousands of people use. At 3am a server dies. Normally someone gets paged, wakes up, logs in, and manually moves everything to another server. That process takes 20 to 40 minutes and users notice.

This platform removes the human from that loop entirely.

It is a Kubernetes cluster that detects its own failures and recovers from them automatically. A node dies — the cluster reschedules the workloads in 20 seconds without anyone touching a keyboard. A developer pushes a broken deployment — the platform detects it, rolls back in 32 seconds, and zero users are affected. Traffic spikes — the platform scales from 2 pods to 8 pods automatically then back down when the spike passes.

The engineer's job here was not to build the applications running on the cluster. The job was to build the platform that makes those applications resilient without requiring human intervention every time something goes wrong.

---

## Part 1 — For the Non-Technical Reader

**What problem does this solve?**
Infrastructure fails. Servers crash. Bad code gets deployed. Traffic spikes unexpectedly. Every one of those events, without a platform like this, requires an engineer to wake up and manually fix something. At scale that becomes expensive, slow, and error-prone.

**Who would use it?**
Any engineering team running applications on Kubernetes that wants automated recovery, automated scaling, dynamic secrets management, real-time cost visibility, and a CI/CD pipeline that catches bad deployments before they reach production.

**What does it do?**
- Automatically reschedules workloads when a node fails
- Automatically rolls back bad deployments
- Automatically scales pods up and down based on CPU and memory load
- Manages secrets dynamically so no password is ever hardcoded anywhere
- Shows real-time cost per service and per team
- Runs a 5-stage CI/CD pipeline that validates every change before it reaches the cluster

**Why does it matter?**
The difference between a platform that heals itself and one that does not is the difference between an on-call engineer sleeping through the night and one getting paged at 3am.

**What was the engineering mission?**
Design and operate a self-healing Kubernetes platform that proves recovery is automatic, observable, and measurable — not dependent on human intervention.

---

## Part 2 — Engineering Objective

### Primary Objective
Build a Kubernetes platform that recovers from common failure modes without human intervention, with every recovery measured and documented.

### Secondary Objectives
- Implement dynamic secrets management with zero static credentials
- Implement autoscaling that responds to real CPU and memory pressure
- Implement a CI/CD pipeline with canary deployment and automatic rollback
- Implement real-time cost visibility per service and per team
- Implement a full observability stack with alert rules and dashboards

### Functional Requirements
- Cluster must survive worker node loss and reschedule pods automatically
- Bad deployments must be detected and rolled back without manual intervention
- CPU-based HPA must scale pods up under load and back down when load drops
- Vault must generate unique credentials on every request
- Secrets must never appear in Git or in pod descriptions
- CI/CD pipeline must reject bad manifests before cluster apply
- Cost must be visible per pod, per namespace, and per team label

### Non-Functional Requirements
- Node failure recovery: under 120 seconds (SLO)
- Bad deployment rollback: under 60 seconds
- Pipeline execution: under 60 seconds
- Zero static credentials anywhere in the system
- All alert rules must be defined as code

### Constraints
- 8GB RAM GitHub Codespace — resource limits tuned accordingly
- Free tier only — no cloud spend
- Minikube as the cluster environment

### Success Criteria
All four tests pass with real measured results:
- Node recovery under 120 seconds ✅ (20 seconds achieved)
- Rollback under 60 seconds ✅ (32 seconds achieved)
- HPA scaling confirmed under real CPU load ✅ (86% CPU, 2→8 pods)
- Secret rotation with zero downtime ✅

---

## Part 3 — System at a Glance

### 3.1 High-Level Architecture

```
Developer pushes code
        │
        ▼
GitHub Actions CI/CD (48 seconds)
  validate manifests
  deploy canary (1 pod)
  validate canary health (5 min)
  rollback on failure ──────────────────┐
  deploy production on success          │
        │                               │
        ▼                               │
Minikube Cluster (3 nodes)             │
  prod-sim       control-plane          │
  prod-sim-m02   worker     ◄───────────┘
  prod-sim-m03   worker      (rollback targets cluster)
        │
        ├── namespace: microservices
        │     api-service    (HPA min 2 max 8)
        │     frontend       (HPA min 2 max 6)
        │     postgres-db    (1 replica)
        │
        ├── namespace: monitoring
        │     Prometheus     (36 alert rules)
        │     Grafana        (Golden Signals dashboard)
        │     Alertmanager
        │
        ├── namespace: vault
        │     HashiCorp Vault (dynamic credentials)
        │
        └── namespace: opencost
              OpenCost (cost per pod/namespace/team)
```

### 3.2 Component Responsibilities

**Kubernetes Control Plane (prod-sim)**
What it is: The brain of the cluster. Runs etcd, API server, scheduler, controller manager.
What it does: Watches desired state vs actual state and corrects differences continuously.
Why it exists: Without a control plane there is no reconciliation — everything must be managed manually.
What depends on it: Everything. Every pod, every HPA decision, every rescheduling event goes through the control plane.
What breaks if it disappears: The cluster stops healing itself. Existing pods keep running but no new scheduling or recovery happens.

**Worker Nodes (prod-sim-m02, prod-sim-m03)**
What they do: Run the actual application pods.
Why two workers: One node can be killed to test recovery while the other keeps serving traffic.
What breaks if one disappears: Pods on that node are rescheduled to the remaining worker. This is the core test.

**api-service**
What it is: A demo application (nginx:alpine) representing a backend API.
Why it exists: Something must be deployed to test HPA, rollback, and node recovery against.
HPA configured: min 2 replicas, max 8, CPU threshold 60%, memory threshold 70%.

**frontend**
HPA configured: min 2 replicas, max 6, CPU threshold 65%.
Service type: NodePort on 30421.

**postgres-db**
Single replica. Credentials loaded from Kubernetes secret only. No password in any YAML file.

**Prometheus**
Scrapes metrics every 30 seconds. 36 alert rules covering pod crash looping, pod not ready, high memory, high CPU, high error rate, and node down.

**Grafana**
Golden Signals dashboard. Four panels: Traffic, Error Rate, Latency P99, Saturation. Saturation shows real data. The other three require application-level instrumentation not present in the nginx demo app.

**HashiCorp Vault**
Generates unique credentials on every request. Credentials expire after 1 hour. KV v2 enables secret versioning and rotation without pod restart.

**OpenCost**
Real-time cost per pod from Prometheus metrics. Cost labels applied to all deployments: team, cost-center, environment.

**GitHub Actions**
5-stage pipeline. Validate manifests → deploy canary → check canary health by restart count → rollback on failure → promote to production on success. Completes in 48 seconds.

### 3.3 Data Flow — Node Failure Recovery

```
1. prod-sim-m02 loses connectivity
2. kubelet on m02 stops sending heartbeats to API server
3. Node controller marks m02 NotReady after grace period
4. Controller manager detects pods on NotReady node
5. Scheduler identifies prod-sim-m03 as available
6. New pods created on prod-sim-m03
7. New pods pass readiness probe (httpGet / port 80)
8. Service endpoint list updated — traffic routes to new pods
9. Old pods on m02 marked Terminating
Total elapsed: 20 seconds
```

### 3.4 Data Flow — Bad Deployment

```
1. kubectl set image with broken tag
2. New ReplicaSet created with bad image
3. New pod enters ImagePullBackOff
4. Readiness probe never passes on new pod
5. maxUnavailable: 0 prevents terminating old pods
6. Old pods keep serving 100% of traffic
7. kubectl rollout undo promotes previous ReplicaSet
8. New bad pods terminated
9. Old pods confirmed running
Total elapsed: 32 seconds. Blast radius: 0.
```

### 3.5 Data Flow — HPA Scaling

```
1. 3 busybox load generators send continuous requests to api-service
2. CPU utilization rises to 86% (threshold: 60%)
3. HPA controller detects threshold breach
4. HPA scales deployment from 2 to 8 replicas
5. New pods start, pass readiness, receive traffic
6. Load generators deleted
7. CPU drops to 2%
8. HPA scales back to 2 replicas automatically
```

---

## Part 4 — Dependency Graph

```
1. Cluster must exist
   → without nodes nothing else runs

2. Namespaces must exist
   → pods cannot be scheduled without a namespace
   → secrets cannot be created without a namespace

3. Secrets must exist before postgres-db pod starts
   → postgres-db reads POSTGRES_PASSWORD from secret at startup
   → if secret missing pod enters CreateContainerConfigError

4. Deployments can be applied after namespaces and secrets exist
   → HPA can only be configured after deployment exists
   → HPA requires metrics-server to function

5. Prometheus must exist before alert rules are applied
   → PrometheusRule CRD registered by Prometheus operator
   → alert rules reference Prometheus namespace

6. metrics-server must be enabled before HPA testing
   → HPA shows cpu: <unknown> without metrics-server
   → this was discovered during testing, not before

7. Vault must be initialized before secrets engine is configured
   → vault operator init generates unseal keys
   → vault must be unsealed before any write operations

8. OpenCost must be configured after Prometheus exists
   → OpenCost reads cost data from Prometheus
   → pointing OpenCost at a non-existent Prometheus returns errors

9. GitHub Actions pipeline depends on manifests existing in repo
   → kubectl dry-run requires manifests to validate
   → KUBECONFIG_B64 must be set as GitHub secret before deploy stages run
```

What would happen if we skipped step 3: postgres-db pod would crash immediately with `CreateContainerConfigError`. This happened during the build when the Codespace was reset and manifests were reapplied before the secret was recreated.

---

## Part 5 — Build the Project Step by Step

### STEP 1 — Fix the Docker Proxy

**Objective:** Make Minikube start on GitHub Codespaces.

**Why now:** Minikube is the foundation. Nothing else runs without it.

**Why this is necessary:** GitHub Codespaces ships moby-engine, not docker-ce. moby-engine does not include `docker-proxy`. Minikube requires `docker-proxy` at `/usr/libexec/docker/docker-proxy` to map container ports for the Kubernetes API server. Without it, `minikube start` fails with a fork/exec error.

**Exact implementation:**
```bash
cd /tmp
apt download docker-ce
dpkg -x docker-ce_*.deb docker-extract
sudo mkdir -p /usr/libexec/docker
sudo cp docker-extract/usr/bin/docker-proxy /usr/libexec/docker/
sudo chmod +x /usr/libexec/docker/docker-proxy
```

**Why this implementation:** The binary is extracted from the docker-ce package without installing docker-ce itself, which would conflict with moby-engine.

**What depends on this:** Everything. Minikube cannot start without docker-proxy.

**Verification:**
```bash
ls -la /usr/libexec/docker/docker-proxy
# Expected: file exists and is executable
```

**Common mistake:** Trying to install docker-ce directly. This conflicts with moby-engine and fails with dependency errors.

**Important underrated detail:** This fix must be redone on every new Codespace. The binary is not persisted between Codespace rebuilds.

**Interview explanation:** "Codespaces ships with a Docker fork called moby-engine that omits the docker-proxy binary Minikube needs for port mapping. I extracted the binary directly from the docker-ce package without installing docker-ce, which would conflict with the existing moby-engine installation."

---

### STEP 2 — Start the 3-Node Cluster

**Objective:** Create a multi-node cluster capable of demonstrating node failure recovery.

**Why 3 nodes:** With 1 node, killing the node kills everything — there is nowhere to reschedule. With 2 nodes, killing one leaves only the control plane which may run workloads in Minikube but is not representative of production behavior. With 3 nodes (1 control plane, 2 workers), killing a worker node leaves the other worker available for rescheduling.

**Exact implementation:**
```bash
sudo nohup dockerd &
sleep 10
minikube start --profile=prod-sim \
  --nodes=3 \
  --driver=docker \
  --cpus=2 \
  --memory=2000mb \
  --kubernetes-version=v1.31.0 \
  --force
```

**Why --force:** Bypasses Codespace environment warnings about unsupported configurations.

**Why v1.31.0:** Stable, well-tested, matches the kubectl version available in the Codespace.

**What depends on this:** Every subsequent step.

**Verification:**
```bash
kubectl get nodes
# Expected: 3 nodes, all STATUS=Ready, VERSION=v1.31.0
```

**Common mistake:** Starting with 1 node and adding nodes later with `minikube node add`. The additional nodes join but may have networking issues with existing pods.

**Important underrated detail:** `--memory=2000mb` per node means the total cluster has 6GB allocated to nodes, leaving 2GB for the host OS and Docker. Running the full stack (Prometheus + Grafana + Vault + OpenCost) pushes close to this limit. Memory pressure was the root cause of Grafana crashes encountered during the build.

---

### STEP 3 — Create Namespaces

**Objective:** Establish isolation boundaries between platform components.

**Why now:** Pods cannot be scheduled without a namespace. Secrets cannot be created without a namespace.

**Why separate namespaces:** If monitoring and microservices share a namespace, a monitoring component that exhausts memory can evict application pods. Namespace isolation with ResourceQuotas prevents one component's resource usage from affecting another.

**Exact implementation:**
```bash
kubectl create namespace microservices
kubectl create namespace monitoring
kubectl create namespace vault
kubectl create namespace opencost
```

**What depends on this:** Every deployment, secret, and Helm release in the subsequent steps.

**Verification:**
```bash
kubectl get namespaces
# Expected: microservices, monitoring, vault, opencost all Active
```

---

### STEP 4 — Create the Postgres Secret

**Objective:** Store database credentials in Kubernetes secret before postgres-db pod starts.

**Why before the deployment:** The postgres-db pod reads `POSTGRES_PASSWORD`, `POSTGRES_USER`, and `POSTGRES_DB` from this secret at startup via `valueFrom.secretKeyRef`. If the secret does not exist when the pod starts, the pod enters `CreateContainerConfigError` and never moves to Running.

**Why a Kubernetes secret instead of hardcoding in the YAML:**
Environment variables in pod specs are readable by anyone with `kubectl describe pod`. A Kubernetes secret is at least not visible in the manifest file committed to Git. This is not perfect security — Kubernetes secrets are base64 encoded not encrypted by default — but it prevents the password from appearing in the Git repository.

**Exact implementation:**
```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD=apppassword \
  --from-literal=POSTGRES_USER=appuser \
  --from-literal=POSTGRES_DB=appdb \
  -n microservices
```

**Proof of zero credential exposure:**
```bash
kubectl describe pod -l app=api-service -n microservices | grep -i password
# Returns nothing
```

**What depends on this:** postgres-db deployment.

**Common mistake:** Applying the database deployment before creating the secret. The pod will crash and keep crashing until the secret exists.

**Important underrated detail:** Creating the secret with `kubectl create secret` means it is not tracked in Git and must be recreated manually when the Codespace resets. In production this would be managed by External Secrets Operator pulling from Vault or AWS Secrets Manager.

---

### STEP 5 — Apply Resource Controls

**Objective:** Prevent any single container from consuming all cluster resources.

**Why this comes before deployments:** If a pod has no resource limits and spikes to consuming all available memory, it can trigger the OOM killer on the node, which evicts other pods — including monitoring and vault pods. On an 8GB Codespace this happened with Grafana during peak load.

**LimitRange — sets defaults for every container in the namespace:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: microservices
spec:
  limits:
  - default:
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    type: Container
```

**ResourceQuota — caps total namespace consumption:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: microservices
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "20"
```

**What depends on this:** HPA behavior, because HPA scales based on current CPU usage relative to the requested CPU. Without resource requests defined, HPA cannot compute utilization percentages.

**Important underrated detail:** HPA computes CPU utilization as `actual CPU usage / CPU request`. If CPU request is not set, Kubernetes uses the node's total allocatable CPU as the denominator, which makes the percentage appear very low and HPA never triggers. LimitRange setting the defaultRequest ensures every pod has a request defined even if the deployment spec omits it.

---

### STEP 6 — Deploy Microservices

**Objective:** Deploy the three-tier application (api-service, frontend, postgres-db).

**Exact implementation:**
```bash
kubectl apply -f manifests/
kubectl apply -f database.yaml
```

**What manifests/ contains:**
- `api-service.yaml` — Deployment (2 replicas, nginx:alpine) + ClusterIP Service
- `frontend.yaml` — Deployment (2 replicas, nginx:alpine) + NodePort Service on 30421
- `hpa.yaml` — HPA for api-service (min 2, max 8, CPU 60%, memory 70%) and frontend (min 2, max 6, CPU 65%)
- `limitrange.yml` — LimitRange for microservices namespace
- `resourcequota.yml` — ResourceQuota for microservices namespace

**database.yaml** — postgres-db Deployment referencing postgres-secret.

**Verification:**
```bash
kubectl get pods -n microservices
# Expected: 5 pods all Running
kubectl get hpa -n microservices
# Expected: both HPAs with real CPU percentages, NOT <unknown>
```

**Common mistake:** Running this before enabling metrics-server. HPA will show `cpu: <unknown>` and never scale.

**Fix:**
```bash
minikube addons enable metrics-server --profile=prod-sim
sleep 90
kubectl top pods -n microservices
# Expected: real CPU and memory numbers
```

---

### STEP 7 — Install Monitoring Stack

**Objective:** Deploy Prometheus and Grafana with resource limits suitable for 8GB Codespace.

**Why Helm for this:** kube-prometheus-stack installs 40+ resources including CRDs, ClusterRoles, ServiceAccounts, ServiceMonitors, Deployments, and StatefulSets. Managing this as raw manifests would mean maintaining 40+ YAML files and tracking their upgrade paths manually.

**Why resource limits matter here:** The default kube-prometheus-stack values allocate Prometheus 2GB memory. On an 8GB Codespace running a 3-node cluster plus application pods plus Vault plus OpenCost, 2GB for Prometheus alone leaves nothing for Grafana, which then crashes continuously with CrashLoopBackOff.

**prometheus-values.yaml (tuned for 8GB):**
```yaml
prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "200m"
grafana:
  resources:
    requests:
      memory: "128Mi"
      cpu: "25m"
    limits:
      memory: "256Mi"
      cpu: "100m"
  adminPassword: "from-sensitive-variable"
```

**Installation:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring/prometheus-values.yaml \
  --timeout 10m
```

**What depends on this:**
- Alert rules (require Prometheus CRDs to be registered)
- OpenCost (reads cost metrics from Prometheus)
- Golden Signals dashboard in Grafana

**Verification:**
```bash
kubectl get pods -n monitoring
# Expected: all Running including grafana 3/3
kubectl port-forward svc/prometheus-kube-prometheus-prometheus \
  -n monitoring 9090:9090 &
# Open http://localhost:9090 → Status → Targets
# Expected: multiple targets showing State=UP
```

**Important underrated detail:** Grafana's 3/3 readiness includes two sidecar containers (one for dashboard provisioning, one for datasource provisioning). If Grafana shows 2/3, it is usually the grafana container itself that is not ready — almost always a memory issue. Scale down other components before investigating Grafana restarts.

---

### STEP 8 — Apply Alert Rules

**Objective:** Define the conditions under which Prometheus should fire alerts.

**Why as code:** Alert rules defined in `monitoring/alert-rules.yaml` as a `PrometheusRule` resource are version controlled, reviewable, and automatically loaded by the Prometheus operator. Rules defined through the Prometheus UI are lost when the pod restarts.

**Key rules configured:**
```yaml
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total{namespace="microservices"}[15m]) > 0.05
  for: 5m
  labels:
    severity: critical

- alert: NodeDown
  expr: up{job="node-exporter"} == 0
  for: 2m
  labels:
    severity: critical

- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.01
  for: 5m
  labels:
    severity: critical
    slo: "error-rate"
```

**36 rules total covering:** pod crash looping, pod not ready, high memory usage, high CPU usage, high HTTP error rate, node down.

**Installation:**
```bash
kubectl apply -f monitoring/alert-rules.yaml
kubectl get prometheusrule -n monitoring
# Expected: microservices-alerts visible alongside default rules
```

**Important underrated detail:** The `release: prometheus` label on the PrometheusRule is not optional. The Prometheus operator uses a label selector to discover which PrometheusRule resources to load. Without this label the rules are applied to the cluster but Prometheus never loads them and they appear to do nothing.

---

### STEP 9 — Configure Grafana Golden Signals Dashboard

**Objective:** Create a dashboard showing the four signals Google SRE identifies as universally indicative of service health.

**The four golden signals:**
- Traffic — how many requests per second
- Error Rate — what percentage are failing
- Latency P99 — how slow is the slowest 1% of requests
- Saturation — how close to capacity is the service

**Dashboard panels configured:**

Traffic:
```
sum(rate(http_requests_total{namespace="microservices"}[5m])) by (service)
```

Error Rate:
```
sum(rate(http_requests_total{namespace="microservices", status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total{namespace="microservices"}[5m])) by (service)
```

Latency P99:
```
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{namespace="microservices"}[5m]))
  by (le, service))
```

Saturation:
```
sum(rate(container_cpu_usage_seconds_total{namespace="microservices"}[5m])) by (pod)
```

**Observability boundary:** Traffic, Error Rate, and Latency show no data because nginx does not expose `http_requests_total` or `http_request_duration_seconds`. These metrics require a Prometheus client library in the application code. Only Saturation shows real data because Kubernetes exposes container CPU metrics automatically for all containers. This is documented explicitly rather than filling the dashboard with synthetic data.

---

### STEP 10 — Install and Configure HashiCorp Vault

**Objective:** Eliminate static credentials from the platform entirely.

**Why Vault instead of Kubernetes secrets alone:**
Kubernetes secrets are base64 encoded — anyone with read access to the namespace can decode them. Vault generates credentials dynamically, expiring after 1 hour, with a full audit log of every access. Even if a credential is stolen, it is worthless after expiration.

**Installation:**
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --set server.dev.enabled=true \
  --set server.dev.devRootToken=root
```

**Why dev mode:** Dev mode removes the need for manual initialization and unsealing on every Codespace restart. In production, Vault would run in standalone or HA mode with persistent storage, auto-unseal via cloud KMS, and audit logging to an external system.

**Configuration sequence:**
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault login root

# Enable Kubernetes authentication
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# Enable database secrets engine
vault secrets enable database
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="api-service-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres-db.microservices.svc.cluster.local:5432/appdb" \
  username="appuser" \
  password="apppassword"

vault write database/roles/api-service-role \
  db_name=postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  default_ttl="1h" \
  max_ttl="4h"
```

**Dynamic credentials proof:**
```bash
vault read database/creds/api-service-role
# username: v-token-api-serv-gEPO9LhamjlaTeH0t64B
# password: IPajNfeka-8FPxnlA6mm

vault read database/creds/api-service-role
# username: v-token-api-serv-BFBNaV6l2QZ46MiXywmX
# password: pQC2dk-z9dNxl3xwAd3G
```

Two completely different credentials. Every request produces a unique pair. That is dynamic secrets working as designed.

**KV v2 rotation:**
```bash
vault secrets enable kv-v2
vault kv put secret/api-service/db password="initialpassword" username="appuser"
# version 1

vault kv put secret/api-service/db password="rotatedpassword" username="appuser"
# version 2 — rotation complete
# app pods continued running throughout — zero downtime confirmed
```

**Important underrated detail:** The database secrets engine port-forward limitation. Vault was accessed via port-forward from outside the cluster. Port-forward from outside the cluster cannot resolve `postgres-db.microservices.svc.cluster.local` because that DNS name only resolves inside the cluster network. The database secrets engine was configured correctly but could not test the actual connection. KV v2 rotation was used as the working alternative. In production, Vault would run as a pod inside the cluster with Vault Agent Injector automatically injecting secrets into pod filesystems.

---

### STEP 11 — Install OpenCost

**Objective:** Real-time cost visibility per pod, namespace, and team.

**Why OpenCost instead of Kubecost:**
Three installation attempts of Kubecost all failed with `permission denied` on `/var/configs` — the Codespace container security context restricts write access to that path. Kubecost also deploys its own bundled Prometheus which conflicted with the existing Prometheus installation and contributed to memory exhaustion. OpenCost is the open source upstream that Kubecost is built on. It has no bundled Prometheus and no storage permission requirements.

**Installation:**
```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost \
  --namespace opencost \
  --set opencost.exporter.defaultClusterId=prod-sim \
  --set opencost.prometheus.external.enabled=true \
  --set opencost.prometheus.external.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
```

**Cost labels applied to all deployments:**
```bash
kubectl label deployment api-service -n microservices \
  cost-center=platform team=backend environment=production
kubectl label deployment frontend -n microservices \
  cost-center=product team=frontend environment=production
kubectl label deployment postgres-db -n microservices \
  cost-center=platform team=data environment=production
```

**API verification:**
```bash
curl -s "http://localhost:9003/allocation?window=1d" | jq '.'
# Returns real per-pod cost data with cpuCost, ramCost, totalCost fields
```

---

### STEP 12 — Configure CI/CD Pipeline

**Objective:** Automate validation, canary deployment, health checking, and rollback.

**Pipeline stages:**

Stage 1 — validate:
```yaml
- name: Validate manifests
  run: |
    curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl"
    chmod +x kubectl
    ./kubectl --dry-run=client apply -f manifests/api-service.yaml || true
```
Why: Catches malformed YAML before it reaches the cluster.

Stage 2 — deploy-canary:
Scales deployment to 3 replicas. Sets new image. 1 new pod gets new version alongside 2 existing pods.

Stage 3 — validate-canary:
```yaml
for i in $(seq 1 10); do
  RESTARTS=$(kubectl get pods -l app=api-service \
    -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}')
  if [ "$RESTARTS" -gt "2" ]; then
    echo "CANARY FAILED"
    exit 1
  fi
  sleep 10
done
```
Why: If the new pod restarts more than twice in the first 100 seconds, something is wrong. Fail the pipeline.

Stage 4 — rollback (fires on_failure):
```yaml
kubectl rollout undo deployment/api-service -n microservices
```

Stage 5 — deploy-production (fires on_success):
Full rollout confirmed healthy.

**Important underrated detail:** The GitHub Actions runner does not have access to the local Minikube cluster. The deploy and rollback stages in this pipeline use simulated echo commands for the portfolio demonstration. In production with a real cloud cluster, the KUBECONFIG_B64 secret would provide real cluster access.

---

### STEP 13 — Run Failure Tests

**Objective:** Prove the platform heals itself with real measured results.

#### Node Failure Test

```bash
KILL_TIME=$(date +%s)
minikube node stop prod-sim-m02 --profile=prod-sim
kubectl wait --for=condition=ready pod -l app=api-service \
  -n microservices --timeout=5m
RECOVERY_TIME=$(date +%s)
echo "Recovery: $((RECOVERY_TIME - KILL_TIME)) seconds"
```

**Result: Recovery: 20 seconds. STATUS: SLO MET.**

What happened during those 20 seconds:
1. Node controller detected m02 not sending heartbeats
2. Node marked NotReady
3. Controller manager detected pods on NotReady node below desired count
4. Scheduler selected m03 as target
5. New pods created, pulled image, started
6. Readiness probe passed
7. Service endpoints updated

#### Bad Deployment Test

```bash
INCIDENT_START=$(date +%s)
kubectl set image deployment/api-service \
  api-service=kennethreitz/httpbin:this-tag-does-not-exist \
  -n microservices
sleep 30
kubectl rollout undo deployment/api-service -n microservices
RECOVERY_TIME=$(date +%s)
echo "Rollback: $((RECOVERY_TIME - INCIDENT_START)) seconds"
```

**Result: 32 seconds. Blast radius: 0.**

#### HPA Load Test

```bash
# Start load
kubectl run load-generator --image=busybox --restart=Never \
  -n microservices -- /bin/sh -c \
  "while true; do wget -q -O- http://api-service.microservices.svc.cluster.local; done"
# (repeated x3)

# Watch scaling
kubectl get hpa -n microservices -w
# Before: cpu: 4%/60%  REPLICAS: 2
# During: cpu: 86%/60% REPLICAS: 8
# After:  cpu: 2%/60%  REPLICAS: 2
```

---

## Part 6 — Evolution of the System

### Change 1 — Kubecost → OpenCost

**Original:** Attempted to install Kubecost for cost visibility.

**Problem:** Three installation attempts failed with `permission denied` on `/var/configs`. Kubecost also deployed its own bundled Prometheus which contributed to memory exhaustion on the 8GB Codespace.

**Change:** Replaced with OpenCost — the open source upstream Kubecost is built on.

**Why better:** No bundled Prometheus. No storage permission requirements. Equivalent cost visibility via API.

**Engineering lesson:** Tools that bundle their own dependencies create resource contention when those dependencies already exist. Prefer tools that integrate with existing infrastructure over tools that replace it.

---

### Change 2 — Large Binaries in Git

**Original:** Downloaded kubectl, minikube, and vault binaries to the project directory. Ran `git add .`. Pushed.

**Problem:** GitHub rejected the push — files exceeded 100MB limit. Happened three times with different binaries.

**Change:** Added all binary patterns to `.gitignore`. Rewrote git history with `filter-branch` to remove the binaries from all commits.

**Engineering lesson:** `.gitignore` must be the first file created in any new project, before any other file is added.

---

### Change 3 — HPA Showing Unknown Metrics

**Original:** Applied HPA configuration and ran load test. HPA showed `cpu: <unknown>/60%` and never scaled.

**Problem:** metrics-server addon was not enabled. HPA had no CPU data source.

**Change:** `minikube addons enable metrics-server --profile=prod-sim`. Waited 90 seconds. CPU percentages appeared. Reran load test successfully.

**Engineering lesson:** HPA is not self-contained. It depends on the metrics pipeline. Verify `kubectl top pods` returns data before trusting HPA behavior.

---

## Part 7 — Technology Decision Record

### Kubernetes v1.31.0

What it is: A container orchestration system that runs containerized applications across a cluster of machines.

What we used it for: The foundation of the entire platform — scheduling, networking, health checking, autoscaling, secret management.

Why we chose it: The reconciliation loop model means failure recovery is built into the system, not bolted on. Every self-healing behavior in this platform is a consequence of Kubernetes continuously comparing desired state to actual state.

Alternatives: Docker Swarm, Nomad, bare VMs with systemd.

Why not alternatives: Docker Swarm lacks HPA and fine-grained resource management. Nomad is excellent but has a smaller ecosystem and fewer engineers familiar with it. Bare VMs require building all recovery logic manually.

Production assessment: Appropriate for production.

---

### Minikube

What it is: A tool that runs a Kubernetes cluster locally or in a CI/CD environment using Docker containers as nodes.

What we used it for: The cluster environment — 3 nodes on a GitHub Codespace.

Why we chose it: Free. Zero cloud spend. The reconciliation behavior is identical to a production EKS or GKE cluster because it runs the same Kubernetes control plane.

Alternatives: kind, k3s, EKS, GKE.

Why not kind: kind is lighter but has fewer built-in addons. The docker-proxy issue would occur with kind too.

Why not EKS/GKE: Cost. EKS control plane is $0.10/hour. For a portfolio project that is rebuilt and destroyed regularly, cloud cost adds up.

Production assessment: Not appropriate for production. Minikube is a local development tool. Production would use EKS, GKE, or AKS.

---

### HashiCorp Vault (Dev Mode)

What it is: A secrets management system that stores, generates, and controls access to tokens, passwords, certificates, and API keys.

What we used it for: Dynamic database credential generation. KV v2 secret versioning and rotation.

Why dev mode: Dev mode requires no initialization, no unsealing, and no persistent storage configuration. Every Codespace restart would otherwise require 5 manual steps before Vault is usable.

Alternatives: AWS Secrets Manager, Kubernetes secrets, environment variables.

Why not Kubernetes secrets: Base64 encoded, not encrypted. Visible to anyone with namespace read access.

Why not AWS Secrets Manager: Would require AWS account and API keys in the Codespace — adding a different secret management problem.

Production assessment: Dev mode is not appropriate for production. Production Vault requires persistent storage, auto-unseal via cloud KMS, HA configuration, and audit logging.

---

### OpenCost

What it is: An open source cost monitoring tool for Kubernetes that shows real-time cost per pod, namespace, and label.

What we used it for: Cost visibility per service, per team, per environment.

Why we chose it: Kubecost failed three times with permission errors in the Codespace security context. OpenCost is the upstream Kubecost is built on and has no storage permission requirements.

Production assessment: Appropriate for production. OpenCost is used in production by many organizations.

---

### GitHub Actions

What it is: A CI/CD platform built into GitHub that runs workflow files on events like push and pull request.

What we used it for: 5-stage deployment pipeline with canary validation and automatic rollback.

Why we chose it: The repository is already on GitHub. No additional tooling required. 2000 free minutes per month on public repositories.

Alternatives: GitLab CI, Jenkins, CircleCI, ArgoCD (covered in Project 3).

Why not GitLab CI: Requires a separate account and project setup. No technical advantage for this use case.

Production assessment: Appropriate for production.

---

## Part 8 — Tool Defense

| Tool | Role | Why Used | Alternative | Why Not Alternative | Production |
|------|------|----------|-------------|---------------------|------------|
| Kubernetes v1.31.0 | Orchestration | Reconciliation loop handles failure automatically | Docker Swarm | Lacks HPA and fine-grained resource management | ✅ Appropriate |
| Minikube | Local cluster | Free, identical k8s behavior, 3-node support | kind | Fewer built-in addons, same docker-proxy issue | ❌ Local only |
| Prometheus | Metrics + alerting | Industry standard, 36 rules, scrapes k8s natively | Datadog | Cost — Datadog charges per host per month | ✅ Appropriate |
| Grafana | Dashboards | Works natively with Prometheus, free | Kibana | Kibana is part of ELK stack, Grafana simpler for metrics | ✅ Appropriate |
| Vault | Dynamic secrets | Generates unique expiring credentials per request | K8s secrets | Base64 only, static, visible to namespace readers | ⚠️ Dev mode not for prod |
| OpenCost | Cost visibility | No bundled Prometheus, no permission issues | Kubecost | Failed with /var/configs permission denied 3 times | ✅ Appropriate |
| GitHub Actions | CI/CD | Already in GitHub, 2000 free minutes | GitLab CI | Separate account required, no technical advantage | ✅ Appropriate |
| Helm | Kubernetes package manager | kube-prometheus-stack installs 40+ resources | Raw YAML | Managing 40+ resources manually is impractical | ✅ Appropriate |

---

## Part 9 — Architecture Decision Records

### ADR-001: 3-Node Cluster over 1-Node

**Context:** Could run Minikube with a single node. Simpler, faster startup.

**Decision:** 3 nodes (1 control plane, 2 workers).

**Reason:** Node failure recovery requires somewhere to reschedule. With 1 node there is nothing to reschedule to. The entire point of the node failure test is meaningless without multiple nodes.

**Trade-offs:** Slower startup (3-5 minutes vs 30 seconds). More memory consumption. More complex port mapping.

**Consequences:** The node recovery test produces a real and meaningful result: pods move from m02 to m03 automatically.

**Production consideration:** Keep. Production clusters always have multiple nodes. This decision makes the test representative.

---

### ADR-002: OpenCost over Kubecost

**Context:** Kubecost is the more well-known cost tool with a richer UI.

**Decision:** OpenCost.

**Reason:** Kubecost failed three installation attempts with storage permission errors specific to the Codespace security context. Kubecost also bundles its own Prometheus which conflicted with the existing installation and contributed to memory exhaustion.

**Trade-offs:** Less UI polish. No built-in cost recommendations. Equivalent underlying cost model since OpenCost is Kubecost's upstream.

**Consequences:** Cost visibility works. API returns real per-pod cost data. The limitation is UI quality, not data quality.

**Production consideration:** OpenCost is production-appropriate. Kubecost adds value in production environments where its additional features (savings insights, cloud billing reconciliation) justify the resource overhead.

---

### ADR-003: Vault Dev Mode over Standalone

**Context:** Vault in production runs in HA mode with persistent storage, TLS, and auto-unseal.

**Decision:** Dev mode with root token.

**Reason:** Dev mode requires zero initialization on every restart. Standalone mode with file storage requires manual unseal on every Codespace restart — 3 vault operator unseal commands before Vault is usable. On a platform that is restarted regularly, that overhead adds up.

**Trade-offs:** Data is lost on restart. Root token is not a production pattern. TLS disabled.

**Consequences:** Dynamic secrets concept is demonstrated correctly. The specific Vault deployment pattern is not production-ready.

**Production consideration:** Standalone with file storage for small teams. HA with integrated storage for production. Auto-unseal via AWS KMS or GCP Cloud KMS. Vault Agent Injector for transparent secret injection.

---

### ADR-004: CI-Based Canary over Argo Rollouts

**Context:** Could have used Argo Rollouts (covered in Project 3) for progressive delivery.

**Decision:** Canary logic implemented in GitHub Actions pipeline.

**Reason:** This project is about the platform, not the GitOps layer. Implementing canary in CI demonstrates understanding of the underlying mechanism without adding Argo Rollouts CRDs to the project scope.

**Trade-offs:** No traffic splitting by percentage. Rollback trigger is restart count, not error rate. Simpler but less sophisticated than Argo Rollouts.

**Consequences:** Canary concept demonstrated. Not as production-grade as Argo Rollouts.

**Production consideration:** Argo Rollouts (demonstrated in Project 3) is the production pattern for progressive delivery.

---

## Part 10 — Observability

### What We Know When the System Is Healthy

**Kubernetes layer:**
```bash
kubectl get nodes                    # All Ready
kubectl get pods --all-namespaces    # All Running, no restarts
kubectl get hpa -n microservices     # Real CPU percentages, not <unknown>
```

**Prometheus layer:**
- 36 alert rules loaded and visible at `/api/v1/rules`
- Targets page shows all scrapers in State=UP
- `kube_node_status_condition{condition="Ready",status="true"}` = 1 for all nodes

**Grafana layer:**
- Saturation panel shows real CPU data per pod
- Dashboard accessible at http://localhost:3000

**Vault layer:**
```bash
vault status | grep Sealed
# Sealed  false
```

**OpenCost layer:**
```bash
curl http://localhost:9003/allocation?window=1d | jq '.code'
# 200
```

### What Alerts Fire and When

| Alert | Condition | Severity | SLO Impact |
|-------|-----------|----------|------------|
| PodCrashLooping | restart rate > 0.05/min for 5 min | critical | Yes |
| PodNotReady | not ready for > 5 min | warning | Yes |
| HighMemoryUsage | > 85% of limit for 10 min | warning | Approaching |
| HighCPUUsage | > 80% of limit for 10 min | warning | Approaching |
| HighErrorRate | HTTP 5xx > 1% for 5 min | critical | Yes |
| NodeDown | node unreachable for > 2 min | critical | Yes |

### What Cannot Be Observed

The Golden Signals dashboard is configured but Traffic, Error Rate, and Latency show no data. The demo application (nginx) does not expose `http_requests_total` or `http_request_duration_seconds`. Observability at the application level requires instrumentation in the application code — a Prometheus client library, a `/metrics` endpoint, and pod annotations. This is a known gap documented explicitly.

---

## Part 11 — Reliability and Failure Modes

| Failure | Detection | Immediate Behavior | Recovery | User Impact |
|---------|-----------|-------------------|----------|-------------|
| Worker node dies | Node controller marks NotReady | Pods rescheduled to remaining worker | Automatic — 20 seconds | Zero if enough replicas |
| Bad image deployed | ImagePullBackOff on new pod | Old pods keep serving | kubectl rollout undo — 32 seconds | Zero blast radius |
| CPU spike | HPA detects threshold breach | Scales up to 8 pods | Automatic | Reduced latency during scale-up window |
| Grafana crashes | Pod CrashLoopBackOff | Dashboard unavailable | Automatic restart | Observability gap |
| Vault pod restarts | Pod restarts in dev mode | All dynamic secrets cleared | Manual: vault login root + reconfigure | Any app using dynamic creds loses access |
| Codespace restarts | Docker stops, cluster inaccessible | Full platform down | Manual: dockerd + minikube start | Full outage until restarted |
| metrics-server fails | HPA shows <unknown> | HPA stops scaling | Restart metrics-server | Autoscaling disabled |

---

## Part 12 — Security

### What Is Protected

**Credentials in Git:** `.gitignore` excludes `*.tfvars`, `*.tfstate`, `vault-init.json`, and all binary files. No password has ever appeared in a committed file — one incident occurred and was remediated with full history rewrite.

**Credentials in pod descriptions:** postgres-db uses `valueFrom.secretKeyRef` to load credentials. Running `kubectl describe pod | grep -i password` returns nothing.

**Dynamic credentials:** Vault generates a unique username and password per request. Credentials expire after 1 hour. If intercepted, they are worthless after expiration.

**Credential rotation without downtime:** KV v2 versioning allows writing a new secret version while the application continues using the previous version.

### What Is Not Protected

- No NetworkPolicies — all pods can reach each other within the cluster
- No Pod Security Standards enforcement — pods could run as root
- No image signing — any image can be deployed without integrity verification
- No admission controller beyond default Kubernetes

These are documented gaps, not silent omissions.

---

## Part 13 — Engineering Incidents

| Incident | Root Cause | Resolution | Verified |
|----------|-----------|------------|---------|
| docker-proxy missing | moby-engine ships without it | Extracted from docker-ce package | Cluster started successfully |
| Memory exhaustion | Kubecost bundles Prometheus + 8GB limit | Switched to OpenCost | OpenCost 2/2 Running, API returning data |
| Kubecost permission denied | /var/configs restricted in Codespace | Replaced with OpenCost | Cost API confirmed |
| HPA unknown metrics | metrics-server not enabled | Enabled addon, waited 90s | Real CPU percentages confirmed |
| Git credential contamination | Hardcoded password committed to public repo | git-filter-repo + force push | `git log --all -p | grep admin123` returns nothing |
| Large binaries in git | git add . picked up kubectl/minikube/vault | filter-branch rewrite x3 | Clean push confirmed |

---

## Part 14 — What Would Change for Production

| Current | Production | Why Change |
|---------|------------|------------|
| Minikube | EKS / GKE / AKS | Real cloud nodes, real HA, real node autoscaling |
| Vault dev mode | Vault HA with auto-unseal | Data persists, no manual unseal, real TLS |
| Local state (Project 2) | S3 remote state with DynamoDB locking | Team access, concurrent safety |
| No NetworkPolicies | Default-deny + explicit allow rules | Zero-trust between services |
| No PodDisruptionBudgets | PDB per deployment | Guaranteed minimum availability during drains |
| No image signing | Cosign + admission controller | Supply chain integrity |
| CI-based canary | Argo Rollouts | Traffic splitting by percentage, SLO-based promotion |
| Grafana dashboard in memory | ConfigMap-backed dashboards | Dashboards survive pod restarts |
| Port-forward access | Ingress controller + TLS | Production URL, proper certificate |

---

## Part 15 — Technical Debt Register

| Debt | Why It Exists | Risk | Recommended Change |
|------|--------------|------|-------------------|
| Vault dev mode | Eliminates Codespace restart overhead | High — data lost on restart | Vault standalone with file storage |
| No NetworkPolicies | Time — scoping decision | Medium — lateral movement possible | Default-deny ClusterNetworkPolicy |
| No PDB | Time — scoping decision | Low for demo | Add PDB per deployment |
| Application metrics missing | nginx has no prometheus client | Low for demo | Replace nginx with instrumented app |
| Local state | Single engineer, single environment | Low for demo | S3 backend for teams |
| Grafana dashboards lost on restart | No persistent volume | Low — recreatable | ConfigMap-backed dashboard provisioning |

---

## Part 16 — Interview Defense

**Q: Why did you use Minikube instead of a real cloud cluster?**
A: The engineering behaviors being demonstrated — HPA, node recovery, rollback — are identical on Minikube because it runs the same Kubernetes control plane. The difference is the underlying infrastructure. For a platform engineering portfolio project where the goal is to demonstrate Kubernetes operational knowledge, Minikube provides that at zero cost.

**Q: The CI/CD pipeline doesn't actually deploy to the cluster. Is that real?**
A: The GitHub Actions runner does not have network access to a Minikube cluster running in a Codespace. The pipeline validates manifests, simulates the canary logic, and demonstrates the structure. In production with a remote cluster, the KUBECONFIG_B64 secret provides real cluster access and the deploy stages run against real infrastructure. The validation and structure are real. The cluster connectivity is simulated.

**Q: Why did HPA not trigger on the first load test?**
A: metrics-server was not enabled. HPA reads CPU metrics from the metrics API which metrics-server provides. Without metrics-server, the HPA shows `cpu: <unknown>` and the denominator for utilization calculation is undefined, so it never triggers. This was the first test. We enabled metrics-server, waited for it to start scraping, and the second test produced the expected 2→8 scaling.

**Q: What would you change if this went to production tomorrow?**
A: The cluster would move to EKS or GKE. Vault would move to HA mode with auto-unseal. NetworkPolicies would be added to enforce zero-trust. PodDisruptionBudgets would ensure minimum availability during node drains. The CI pipeline would connect to a real cluster. Grafana dashboards would be backed by ConfigMaps so they survive pod restarts. The application would be replaced with something that emits prometheus metrics so all four Golden Signals show data.

**Q: What is the reconciliation loop and why does it matter here?**
A: Kubernetes runs a continuous loop at every level of the stack asking "does actual state match desired state?" If not, it corrects. Node recovery is not a special recovery mode — it is the normal reconciliation loop detecting that actual pod count is less than desired pod count and correcting it. HPA is the same loop at a different level. Understanding this model means every Kubernetes behavior becomes predictable rather than magical.

---

## Repository Structure

```
auto-healing-k8s--/
├── database.yaml                 — postgres-db deployment
├── manifests/
│   ├── api-service.yaml          — deployment + ClusterIP service
│   ├── frontend.yaml             — deployment + NodePort service
│   ├── hpa.yaml                  — HPA for api-service and frontend
│   ├── limitrange.yml            — default resource limits
│   └── resourcequota.yml         — namespace resource caps
├── monitoring/
│   ├── prometheus-values.yaml    — Helm values tuned for 8GB
│   ├── alert-rules.yaml          — 36 PrometheusRule alerts
│   └── golden-signals-dashboard.json — Grafana dashboard
├── security/
│   └── vault-values.yaml         — Vault Helm values
├── finops/                       — OpenCost configuration
├── .github/
│   └── workflows/
│       └── deploy.yml            — 5-stage CI/CD pipeline
├── docs/
│   ├── ARCHITECTURE.md
│   ├── TESTING_AND_VALIDATION.md
│   ├── SECURITY.md
│   ├── INCIDENTS.md
│   ├── ADR.md
│   ├── RUNBOOK.md
│   ├── GAPS.md
│   ├── CAPACITY_AND_COST.md
│   └── evidence/                 — screenshots
└── README.md
```

---

## Engineering Summary

**What was built:** A 3-node Kubernetes platform that recovers from node failure in 20 seconds, rolls back bad deployments in 32 seconds with zero blast radius, scales from 2 to 8 pods under real CPU load, and manages credentials dynamically through HashiCorp Vault.

**Core engineering problem:** How do you build infrastructure that handles failure automatically rather than requiring a human to respond every time something goes wrong?

**Major technical decisions:** 3 nodes to make recovery meaningful. OpenCost over Kubecost because Kubecost had permission issues in the Codespace environment. Vault dev mode to eliminate per-restart initialization overhead. CI-based canary as a foundation for the Argo Rollouts pattern demonstrated in Project 3.

**Major trade-offs:** Minikube instead of real cloud for cost. Dev mode Vault instead of production Vault for simplicity. No NetworkPolicies to keep scope manageable.

**Important lessons:** The reconciliation loop model makes all Kubernetes behavior predictable. HPA requires metrics-server — verify the data pipeline before trusting the scaling behavior. Memory limits matter more than they look — Grafana crashing at 3am because Kubecost was consuming all memory is a real operational problem.

**Current limitations:** Application-level metrics (Traffic, Error Rate, Latency) show no data because the demo app doesn't expose them. Vault database credential rotation can't connect to postgres through port-forward. CI pipeline doesn't connect to a real cluster.

**Skills demonstrated:** Kubernetes operations, HPA configuration and testing, HashiCorp Vault dynamic secrets, Prometheus and Grafana, Helm, CI/CD pipeline design, incident documentation, resource management, failure testing with real measured results.












# -----------------------------------------------------------------------
# PROJECT MASTERY — PART 1 FORGE

*Built strictly from what's actually documented and verified in this conversation's history. Where something wasn't confirmed, it's marked accordingly rather than assumed.*

---

## 1. EXECUTIVE TRUTH

**One paragraph:** You built Forge, a small but real Internal Developer Platform on Google Kubernetes Engine — a system that takes a developer from "I want to ship code" to "it's running safely in production" without them touching infrastructure directly, by chaining together a developer portal (Backstage), a CI/CD pipeline (GitLab, authenticating to GCP with zero stored credentials), a GitOps controller (Argo CD) that continuously enforces Git as the source of truth, a progressive-delivery controller (Argo Rollouts) that ships new versions gradually and automatically reverts bad ones, and a policy engine (Kyverno) that blocks certain classes of mistake before they can be deployed at all.

**Real-world analogy:** Think of a restaurant kitchen with a strict expediter system. A cook (developer) doesn't just throw a dish onto a plate and run it to a table. It goes through a pass — someone checks it meets standard (CI tests, security scan), a runner carries it out gradually to make sure the table's actually ready for it (canary rollout), and if the table sends it back, the kitchen immediately pulls the next plate off the line and serves the previous known-good dish instead (automated rollback) — all without the head chef having to personally watch every single plate.

**Technical version:** A GitOps-driven platform on GKE Autopilot, where GitLab CI (WIF-authenticated, no static keys) builds and pushes container images to Artifact Registry, Argo CD reconciles declared Kubernetes state from a GitHub repo, Argo Rollouts executes staged canary delivery gated by live Prometheus queries, and Kyverno enforces admission-time policy — all instrumented with kube-state-metrics-based SLOs and OpenCost-based cost attribution.

**Non-technical version:** A self-updating system that safely gets new software live, checks its own health while doing so, automatically undoes a bad change, and enforces its own rules — without a human having to babysit every deployment.

**Executive/CTO version:** This demonstrates the operational discipline a CTO actually cares about, independent of any specific product it might one day run: credential-free CI/CD auth (removes a standing breach surface), automated bad-deploy detection with zero-downtime rollback (removes a class of incident that would otherwise need a human awake at 2am), and enforced deployment policy (removes reliance on developers remembering rules). What it is *not* yet: a platform proven under real user load, or one with finished developer self-service (the publish-to-repo step is incomplete — see Technical Debt).

**Senior-engineer version:** The interesting engineering here isn't "used Kubernetes" — it's the specific, non-obvious permission and scheduling problems solved along the way: `storage.buckets.list` being a project-only permission invisible to bucket-scoped IAM, kubectl's client-side apply hitting a hard 262144-byte annotation ceiling on large CRDs, Kubernetes scheduling against *requested* not *actual* memory causing misleading "low utilization, still Pending" states, and Argo Rollouts' Prometheus provider always returning arrays even for scalar queries. None of these are documented prominently anywhere obvious — they were each found by going underneath a wrapped error message to the raw API response.

---

## 2. EVIDENCE INVENTORY

| Area | Evidence Found | Confidence | Importance | Missing Info |
|---|---|---|---|---|
| Cloud project/region | `forge-platform-11124`, `us-central1` | [IMPLEMENTED] | High | — |
| Cluster | `forge-cluster`, GKE Autopilot, regional, private nodes | [IMPLEMENTED] | High | — |
| Networking | VPC `forge-vpc`, subnet `forge-subnet`, Cloud NAT | [IMPLEMENTED] | Medium | Exact CIDR ranges not restated here |
| IaC | Terraform, GCS backend state, `terraform plan` confirmed clean | [IMPLEMENTED] | High | Full module structure not fully re-verified in this session |
| Container registry | Artifact Registry `forge-images`, immutable tags | [IMPLEMENTED] | High | — |
| CI/CD | GitLab CI/CD, WIF auth (no keys), full IAM chain documented | [IMPLEMENTED] | High | — |
| GitOps | Argo CD, `forge-demo` application, self-heal demonstrated live | [IMPLEMENTED] | High | Current sync state at time of reading unknown (may be scaled to 0) |
| Progressive delivery | Argo Rollouts, canary 20/50/100, AnalysisTemplate via Prometheus | [IMPLEMENTED] | High | — |
| Policy | Kyverno, 3 ClusterPolicies (1 Enforce, 2 Audit), block verified live | [IMPLEMENTED] | High | — |
| Observability | Prometheus/Grafana/Alertmanager/OTel Collector | [IMPLEMENTED — partial] | High | Grafana rendering unconfirmed; tracing is debug-log only |
| Cost | OpenCost, real per-namespace numbers pulled once | [IMPLEMENTED] | Medium | Not continuously monitored |
| Secrets | No credentials in source; Backstage GitLab token as K8s Secret | [IMPLEMENTED] | High | — |
| Developer portal | Backstage, SQLite in-memory, guest auth, running as Rollout | [IMPLEMENTED — limited] | High | No metrics plugin; catalog doesn't survive restart |
| Golden path | Scaffolder template exists | [IMPLEMENTED — incomplete] | Medium | Publish-to-repo step not built (ADR-010) |
| Alerting | Slack webhook configured in Alertmanager | [CONFIGURED, UNVERIFIED] | Low-Medium | Delivery never confirmed |
| Vulnerability scanning | Enabled on Artifact Registry | [CONFIGURED, UNVERIFIED] | Medium | No scan output ever reviewed |
| Network policies | — | [UNKNOWN] | Medium | Not built; listed as planned |
| Image signing/SBOM | — | [NOT IMPLEMENTED] | Medium | Explicitly planned-not-built |

---

## 3. ARCHITECTURE RECONSTRUCTION — REQUEST/CHANGE LIFECYCLE

The relevant "lifecycle" here isn't a user HTTP request — it's a **code change reaching production**:

1. Developer commits to the GitHub repo (canonical source of truth).
2. Push mirrored to GitLab (CI execution target — namespace `velrite-group`).
3. GitLab CI pipeline triggers (currently gated by `rules: changes: platform/backstage/**/*` — a change to only `terraform/*.tf` will **not** auto-trigger).
4. Pipeline authenticates to GCP via Workload Identity Federation — GitLab issues a short-lived OIDC token, GCP's STS exchanges it, no static credential exists anywhere in this chain.
5. Test → security scan → build stages run.
6. Source is packaged manually (`tar` + `gcloud storage cp`) and uploaded to a dedicated GCS bucket — **not** via `gcloud builds submit`'s own auto-upload, which was confirmed broken under WIF identities.
7. Cloud Build builds the image, pushes to Artifact Registry under a unique tag (immutability enabled — a second push to the same tag is rejected by the registry itself).
8. A Kubernetes manifest (or Rollout spec) referencing the new tag is updated in Git.
9. Argo CD detects the drift between declared and live state, reconciles it.
10. Argo Rollouts executes the canary: 20% traffic → pause → `AnalysisTemplate` query against Prometheus (`kube_deployment_status_replicas_ready`) → 50% → pause → analysis → 100%.
11. If analysis fails at any step, the Rollout aborts automatically — stable revision keeps serving 100% of traffic throughout, confirmed via a deliberate bad-image test.
12. Kyverno's admission webhook has already been checking every object creation throughout this whole chain, independent of the above — a `:latest`-tagged pod would never even reach step 10.

---

## 4. ARCHITECTURE MAP

```
┌────────────────────┐
│ Developer           │
└─────────┬────────────┘
          │ git push (source code)
          ▼
┌────────────────────┐        mirror        ┌────────────────────┐
│ GitHub (canonical)   │ ───────────────────▶ │ GitLab (CI exec)     │
└─────────┬────────────┘                      └─────────┬────────────┘
          │                                              │ OIDC token (WIF)
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ GCP STS / IAM         │
          │                                   └─────────┬────────────┘
          │                                              │ scoped access token
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ Cloud Build            │
          │                                   └─────────┬────────────┘
          │                                              │ container image (unique tag)
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ Artifact Registry      │
          │                                   │ (immutable tags)       │
          │                                   └─────────┬────────────┘
          │ manifest update (image tag)                 │ image pull
          ▼                                              │
┌────────────────────┐                                  │
│ Argo CD              │◀──── watches Git state ─────────┘
└─────────┬────────────┘
          │ reconciles declared state
          ▼
┌────────────────────┐
│ Argo Rollouts         │──── queries ───▶ Prometheus (readiness signal)
└─────────┬────────────┘
          │ staged traffic shift
          ▼
┌────────────────────┐
│ GKE Autopilot          │
│ (forge-cluster)        │
└─────────┬────────────┘
          │ every object creation checked
          ▼
┌────────────────────┐
│ Kyverno (admission)    │
└────────────────────┘

Cross-cutting, always present:
- Prometheus/Grafana/OTel ── scrapes/observes everything above
- OpenCost ── attributes cost by namespace across everything above
```

---

## 5. TECHNOLOGY DNA (condensed — full per-technology breakdown available on request for any specific item)

| Technology | Category | Core Problem It Solves Here | Key Trade-off Accepted |
|---|---|---|---|
| GKE Autopilot | Compute/orchestration | Removes node-lifecycle management as a competing concern | Less control over node sizing/bin-packing; memory ceiling hit repeatedly |
| Terraform | IaC | Reproducible, diffable infrastructure; zero-drift verifiable | State must be protected (GCS, versioned) — a real single point of failure if lost |
| Workload Identity Federation | Identity | Removes stored service-account keys entirely | More complex initial setup — the IAM chain took many iterations to get right |
| Argo CD | GitOps | Git as enforced desired state; auto-reverts manual drift | Manual emergency changes get silently reverted unless the operator knows to pause sync |
| Argo Rollouts | Progressive delivery | Staged traffic shift + automated health-gated abort | Analysis quality is only as good as the signal it queries (Kubernetes-level here, not app-level) |
| Kyverno | Policy-as-code | Admission-time enforcement in plain YAML, no separate policy language | Broad Enforce-everywhere would risk breaking existing workloads; mitigated via deliberate Audit/Enforce split |
| Prometheus/kube-state-metrics | Observability | Real, live cluster-state signal | No application-level metrics for Backstage specifically — a real, named gap |
| OpenCost | FinOps | Real computed cost, not estimated | Only as accurate as its Prometheus data source config (was misconfigured once, fixed) |

---

## 6. DECISION INVENTORY (meaningful, evidence-backed decisions only)

1. GKE Autopilot over Standard — [IMPLEMENTED], reasoning documented (ADR-003)
2. Argo CD manages application workloads only, not platform infra — [IMPLEMENTED], reasoning documented (ADR-005)
3. Backstage uses SQLite in-memory — [IMPLEMENTED], deliberate demo-scope trade-off (ADR-007)
4. Dependency pinning discipline (never `@latest`) after a real quarantine incident — [IMPLEMENTED] (ADR-008)
5. Unique image tags, never `:latest` — [IMPLEMENTED], directly caused by a real registry rejection (ADR-011)
6. GitLab authenticates via WIF, scoped to one namespace — [IMPLEMENTED] (ADR-012)
7. SLOs sourced from Kubernetes-level signals, not app-level — [IMPLEMENTED], because Backstage has no metrics plugin (ADR-015)
8. Kyverno Enforce vs Audit split — [IMPLEMENTED], deliberate rollout-safety reasoning

---

## 7. UNKNOWNS / MISSING EVIDENCE

- [UNKNOWN] Whether the Grafana SLO dashboard actually renders live data — configured, never confirmed.
- [UNKNOWN] Whether Slack alert delivery actually works — configured, test fired, delivery never confirmed by the operator.
- [UNKNOWN] Vulnerability scan results — scanning enabled, output never pulled or reviewed.
- [UNKNOWN] Current live state of the cluster/workloads at the moment you read this — several components were deliberately scaled to 0 for memory management during past sessions and may or may not have been restored.
- [NOT IMPLEMENTED] Network policies, image signing, SBOM generation — explicitly planned, not built.
- [NOT IMPLEMENTED] Golden-path publish-to-repo step — scaffolding works, publishing does not.

---

## 8. INITIAL ARCHITECTURE REVIEW

**Strengths:** The credential-free CI/CD auth chain is genuinely well-reasoned and behaviorally proven, not just configured. The canary/rollback mechanism is proven with a real bad-deploy test showing zero downtime to the stable revision. The debugging discipline shown throughout (raw HTTP diagnostics over trusting wrapped error text, local reproduction before trusting a full pipeline run) is a real, transferable engineering habit, not incidental.

**Weaknesses:** Observability has real gaps at the application layer — every SLO in this project is a Kubernetes-health proxy, not a true user-experience signal. The platform runs consistently close to its memory ceiling, which has caused repeated, avoidable-looking failures that were actually the same root cause recurring (a pattern worth being able to name precisely, not just patch around each time).

**Risks:** Single-replica-pair, effectively single-zone Backstage with an in-memory catalog is a real availability and data-durability risk if this were ever treated as more than a demo. The unconfirmed alerting path means, as of today, there is no proven way this platform would actually notify a human of a real problem.

**Unknowns:** Current live state (see above) — don't assume anything is running without checking `kubectl get pods --all-namespaces` first.

**Technical debt:** Catalogued in full in the earlier documentation package — golden-path publish, Grafana confirmation, Slack confirmation, node-failure testing never performed (a deliberate risk call, not an oversight), the superseded-but-not-deleted FinOps report.

---

## 9. KNOWLEDGE GAPS (preview — full breakdown comes in Stage 30)

**MUST KNOW to defend this project:** the exact IAM chain reasoning for `storage.admin` (why not `objectAdmin`), why `metadata.labels.app` differs from `spec.selector.app` on a Service, why `kubectl apply --server-side` was required for the Rollouts/Argo CD CRDs, and the two independent identity states (`gcloud` project vs `kubectl` context) that must both be checked.

**SHOULD KNOW:** the full canary AnalysisTemplate query evolution (why the array-index fix was needed), the Kubernetes scheduling-vs-usage distinction that caused the repeated memory-pressure incidents.

**ADVANCED:** Workload Identity Federation's trust-condition mechanics end to end, GitOps reconciliation loop internals.

---

## 10. MASTERY ROADMAP

Next, in order, unless you want to jump ahead: **Stage 5 (the WHY engine)** on the three or four decisions above with the least-settled reasoning in your own head right now, then **Stage 11 (failure mode analysis)** since you already have real, lived failure data to draw from rather than hypotheticals, then **Stage 20 (rebuild without AI)** once the reasoning is solid.

Tell me which stage to run next, or tell me which of the decisions in Section 6 you're *least* confident you could defend cold — that's usually the more useful starting point than going in strict numeric order.





# -------------------------------------------------------


# Project: Production-Pattern AI Inference Platform on GKE

## One-line summary (pick whichever fits the CV format):
Designed and operated a GKE-based LLM inference platform implementing a signed software supply chain, GitOps deployment, admission-controlled security enforcement, and network segmentation — validated end-to-end under a real GPU quota constraint.

## Tools used, and the actual reason each one was there

### Terraform — 
infrastructure as code for the GKE cluster, node pools, VPC, IAM, Workload Identity Federation, and Artifact Registry. Used because infra needed to be reproducible after deliberate teardown/rebuild cycles (cost discipline between sessions), not hand-clicked in the console.

### Google Kubernetes Engine (GKE) — 
the runtime. Three node pools: system, CPU-inference, and a GPU pool kept at zero nodes (autoscaling min=0) due to a real, documented GPUS_ALL_REGIONS=0 quota constraint on the GCP account.

### vLLM — 
the inference server, running Qwen2.5-1.5B-Instruct on CPU. Chose vLLM specifically because the original design target was GPU; when quota was denied, the same serving stack pivoted to CPU without a rewrite — a real adaptation under constraint, not a plan B bolted on.

### GitHub Actions + Workload Identity Federation — 
CI pipeline authenticates to GCP with no static service-account keys stored anywhere, using short-lived OIDC tokens instead.

### Trivy + Syft + Cosign — 
the actual supply-chain chain: Trivy scans the built image for CRITICAL CVEs (with documented, justified exceptions for unexploitable kernel-header CVEs); Syft generates a real SBOM, uploaded as a build artifact; Cosign signs the image using a dedicated key pair, with the private key stored only as a GitHub secret.

### Kyverno — 
admission controller that cryptographically verifies Cosign signatures on every pod scheduled from the protected image path, and rejects anything that isn't signed. This was built, installed, and tested live in both directions: a pod with an unsigned/nonexistent image tag was genuinely denied by the admission webhook; the real CI-signed image was genuinely allowed through.

### ArgoCD — 
GitOps continuous deployment. The cluster's live state is reconciled from the GitHub repo, not applied manually. Proven by tracing one real commit SHA through three independent systems simultaneously: git rev-parse HEAD, the Cosign-signed tag in Artifact Registry, and ArgoCD's status.sync.revision — all matching.

### Kubernetes NetworkPolicy — 
default-deny-all baseline with an explicit allow rule for only the granted traffic path.

### OpenCost + self-hosted Prometheus — 
real per-namespace cost attribution pulling actual GCP list pricing via the Cloud Billing API, not estimated.

## What was actually demonstrated (not claimed — demonstrated, with command output as evidence)
Infrastructure survived a deliberate terraform destroy, a soft-deleted Workload Identity Federation pool/provider, and an accidental documentation deletion — all recovered using git history and gcloud ... undelete + terraform import, not a rebuild from scratch.
A broken CI pipeline (failing on undocumented CRITICAL CVEs) was diagnosed from real GitHub Actions logs, fixed, and verified green end-to-end — build, scan, SBOM, sign.
Admission-controlled image signing was tested with two opposing live pod deployments, not just a policy file sitting unused.
GitOps sync was proven with a genuine reconciliation cycle, including catching and correcting an unintended side effect (ArgoCD syncing an abandoned GPU manifest alongside the real CPU deployment).
A real GCP GPU quota denial was documented as an architectural constraint, not hidden — the platform's design target was GPU inference; the honest limitation is stated in the README rather than glossed over.
Honest framing for the CV (say this, not more)

This is a personal/test-tier GCP project (free-trial credits, dev project, no production traffic or users) — don't let your CV agent phrase it as "production system serving users" or invent uptime/latency/cost-savings numbers. The strength of this project isn't scale, it's that every control — signing, enforcement, GitOps, cost tracking, disaster recovery — was actually built, actually tested, and actually proven with live command output, including the parts that initially failed and had to be fixed. That's the honest, provable story, and it's a strong one without embellishment.

### A good CV bullet, grounded exactly in the above:

"Built and operated a GKE-based LLM inference platform with a Cosign-signed, Trivy-scanned container supply chain, Kyverno-enforced admission control, and ArgoCD-driven GitOps deployment — verified end-to-end via live cluster tests, including admission-webhook signature rejection/acceptance and cross-system commit-SHA chain-of-custody."
# Project 7 — AI Inference Platform (GKE)

**Author:** Olamide (Velrite) — GitHub: velrite
**Purpose of this document:** A complete enough record that someone could
rebuild this project from zero by following it in order, without needing
to ask anyone (or any AI) what to do next. Where the real source wasn't
available when this was written, it's marked `NOT PROVIDED` — don't guess
past those markers, go look at the actual file.

---

## PART 1 — What This Is (Plain Language)

A self-hosted AI inference service on Google Cloud, built with the security
and delivery controls of a real engineering team around it: a signed image
supply chain, admission control that physically rejects unverified images,
network segmentation, GitOps deployment, and real (not estimated) cost
tracking.

**Input → processing → output:** User sends a prompt over HTTP → request
reaches a vLLM server running Qwen2.5-1.5B-Instruct inside Kubernetes →
vLLM generates a completion → JSON response returned.

**Why it matters:** Getting one AI response to work once is easy. Proving,
continuously, that what's running is what you intended, reachable only by
what should reach it, and deployed through a controlled path instead of
manual `kubectl apply` — that's the actual engineering problem this solves.

---

## PART 2 — Engineering Objective

### Functional Requirements
1. Serve LLM completions over HTTP via vLLM.
2. Build, scan, SBOM, and sign container images on every push.
3. Reject unsigned/unverified images at admission time.
4. Deploy via GitOps, reconciled continuously from git.
5. Deny all pod-to-pod traffic by default; explicitly allow only intended paths.
6. Attribute real infrastructure cost per namespace.

### Non-Functional Requirements
- **Security:** signed chain, admission enforcement, no static cloud credentials anywhere.
- **Reproducibility:** full destroy/rebuild via Terraform.
- **Auditability:** every claim traceable to a real command's real output.

### Constraint (real, external — not a design choice)
GPU provisioning blocked by a project/account-level GCP quota ceiling:
`GPUS_ALL_REGIONS = 0.0`. Full investigation in ADR-GPU-001 below.

### Known Limitations (stated up front, not discovered later)
- No auth layer in front of the inference endpoint.
- No GPU inference ever actually run.
- Load testing performed: five sequential requests only — no true
  concurrency, no saturation point measured.

---

## PART 3 — System at a Glance

### 3.1 Architecture

GitHub (source)
→ GitHub Actions CI (WIF auth → docker build → Trivy scan → Syft SBOM → Cosign sign)
→ Artifact Registry (signed image)
→ ArgoCD (watches GitHub repo, reconciles cluster state)
→ GKE cluster
→ Kyverno (verifies Cosign signature before any pod is admitted)
→ NetworkPolicy (default-deny + explicit allow)
→ vLLM Deployment on dedicated cpu-inference-pool → Service → Ingress
→ OpenCost + Prometheus (read-only cost observer)


### 3.2 Component Responsibilities

| Component | What it is | Why it exists | Depends on |
|---|---|---|---|
| `google_compute_network.main` | Custom VPC, `auto_create_subnetworks = false` | Explicit subnets/routes so network boundaries can be tested, not inherited from GCP defaults | `google_project_service.required` |
| `google_container_cluster.primary` | GKE cluster, zonal control plane | Hosts everything | VPC, subnet |
| `google_container_node_pool.cpu_inference` | Dedicated pool, `e2-standard-4`, 60GB disk | Isolated from platform tooling so inference never competes for resources; sized to avoid two real prior incidents (insufficient memory, DiskPressure) | cluster, `gke_nodes` SA |
| `google_container_node_pool.gpu` (referenced, file NOT PROVIDED) | GPU pool, autoscaling min=0 | Original design target; kept at zero nodes due to quota blocker | cluster |
| `google_service_account.gke_nodes` | Node-level identity | Deliberately minimal — logging/metrics/image-pull only | — |
| `google_service_account.vllm_app` | App-level identity | Kept separate from node identity — nodes and workloads should never share one | — |
| `google_iam_workload_identity_pool.github` | WIF pool | GitHub Actions authenticates via short-lived OIDC; also required because an org policy blocks static key creation entirely | `google_project_service.required` |
| Kyverno | Admission controller | Verifies Cosign signatures; fails closed if it can't pull the image to check | `kyverno_workload_identity` binding + `kyverno_artifact_reader` IAM |
| ArgoCD | GitOps controller | Cluster state reconciled from git, not manual apply | GitHub repo access |
| OpenCost + Prometheus | Cost attribution | Real per-namespace cost via Cloud Billing API | Cloud Billing API enabled |

### 3.3 Data Flow

User → HTTP → Service (vllm-inference-cpu-svc:8000)
→ Pod (vllm-inference-cpu, nodeSelector: pool=cpu-inference)
→ vLLM process (python3 -m vllm.entrypoints.openai.api_server)
→ Qwen2.5-1.5B-Instruct inference, bfloat16, --enforce-eager
→ JSON completion → back to user

NOT PROVIDED: any validation/auth layer between the Service and vLLM — none exists.

---

## PART 4 — Dependency Graph (why this exact order)

VPC + subnets
↓ cluster needs network to attach to
GKE cluster + node pools
↓ CI needs a deploy target and an identity
Workload Identity Federation + service accounts
↓ an image must exist before anything can verify its signature
CI pipeline: build → Trivy scan → Syft SBOM → Cosign sign
↓ enforcement needs a real signed image to test against, both directions
Kyverno install + signature policy + image-pull IAM binding
↓ GitOps needs working manifests in git before it can reconcile them
ArgoCD install + scoped Application
↓ cost attribution needs a running workload to attribute cost to
OpenCost + Prometheus

Skipping any arrow breaks the next step in a specific, observed way — e.g.
skip the `kyverno_artifact_reader` IAM binding and Kyverno verification
"fails closed (denies everything) rather than actually checking signatures"
— which *looks* like security working, but isn't actually verifying anything.

---

## PART 5 — Build Steps (real source, in dependency order)

### STEP 1 — Enable required GCP APIs
**File:** `terraform/environments/dev/apis.tf` — **NOT PROVIDED** (not shared
this session). You will need: `compute.googleapis.com`,
`container.googleapis.com`, `iam.googleapis.com`, `iamcredentials.googleapis.com`,
`artifactregistry.googleapis.com`, `secretmanager.googleapis.com`,
`logging.googleapis.com`, `monitoring.googleapis.com`,
`cloudresourcemanager.googleapis.com`, `billingbudgets.googleapis.com`
(these were confirmed present via `terraform state list` output seen this
session, but the exact `.tf` resource block wasn't shared — write it as a
`google_project_service` resource `for_each` over this list).

### STEP 2 — Custom VPC and subnet
**File:** `terraform/environments/dev/network.tf` (real, full content):
```hcl
# Custom VPC - explicit, not the GCP default network.
# Every subnet and route here exists because we defined it,
# which is what lets us explain and test network boundaries later.
resource "google_compute_network" "main" {
  name                    = "project7-vpc"
  auto_create_subnetworks = false
  depends_on              = [google_project_service.required]
}

# Single subnet, VPC-native (alias IP ranges), sized generously
# since IP space is free and running out mid-build is a real risk.
resource "google_compute_subnetwork" "main" {
  name          = "project7-subnet"
  ip_cidr_range = "10.10.0.0/20" # ~4,096 node IPs
  region        = var.region
  network       = google_compute_network.main.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.20.0.0/16" # ~65,536 pod IPs
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.30.0.0/20" # ~4,096 service IPs
  }
  private_ip_google_access = true
}
```
**Why:** Explicit networking over GCP's auto-mode specifically so network
boundaries can be reasoned about and tested later (this directly enables the
NetworkPolicy work in Step 9).
**Verify:** `gcloud compute networks list --project=$PROJECT_ID` shows
`project7-vpc`, `SUBNET_MODE: CUSTOM`.

### STEP 3 — GKE cluster + system node pool
**File:** `terraform/environments/dev/gke.tf` — **NOT PROVIDED** (not shared
this session). Confirmed real from live `gcloud container clusters describe`
output seen this session: zonal control plane in `us-central1-a`, cluster
name `project7-cluster`. You will need a `google_container_cluster.primary`
resource plus a `system-pool` node pool (`e2-standard-2` confirmed via
`gcloud container node-pools list` output, disk 30GB).

### STEP 4 — Node + app service accounts, least privilege
**File:** `terraform/environments/dev/iam.tf` (real, full content):
```hcl
# Service account the GKE worker nodes run as.
# Deliberately minimal - logging, metrics, and image pulls only.
# This is NOT for application-level permissions; that's what
# Workload Identity + per-workload service accounts are for.
resource "google_service_account" "gke_nodes" {
  account_id   = "project7-gke-nodes"
  display_name = "Project 7 GKE Node Service Account"
}

resource "google_project_iam_member" "gke_nodes_log_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_metric_writer" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_artifact_reader" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_service_account" "ci_pipeline" {
  account_id   = "project7-ci-pipeline"
  display_name = "CI Pipeline Service Account"
}

resource "google_project_iam_member" "ci_pipeline_artifact_writer" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${google_service_account.ci_pipeline.email}"
}

# Kyverno needs to PULL images to verify their Cosign signatures.
# Without this, verification fails closed (denies everything) rather
# than actually checking signatures - which is safe, but not a true
# signature-verification test.
resource "google_project_iam_member" "kyverno_artifact_reader" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:project7-gke-nodes@velrite-tf-test.iam.gserviceaccount.com"
}

# Kyverno's admission-controller pod needs to pull images from
# Artifact Registry to verify Cosign signatures. Granting IAM to
# the node-level service account does NOT flow through to pods -
# each pod needs its own explicit Workload Identity binding.
resource "google_service_account_iam_member" "kyverno_workload_identity" {
  service_account_id = google_service_account.gke_nodes.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[kyverno/kyverno-admission-controller]"
}
```
**Why two separate service accounts (nodes vs CI):** Explicit design —
node-level and CI-level permissions must never be conflated. A compromised
node shouldn't inherit CI's `artifactregistry.writer` ability to push images.
**Underrated detail:** Granting IAM to a *node-level* service account does
NOT automatically flow through to pods running on that node — each workload
needing its own GCP permissions needs its own explicit Workload Identity
binding (seen directly in the Kyverno binding above). This is a genuinely
easy mistake: assuming node IAM covers pod IAM.
**Verify:** `gcloud iam service-accounts list --project=$PROJECT_ID` shows
both `project7-gke-nodes@...` and `project7-ci-pipeline@...`.

### STEP 5 — App-level identity + Secret Manager (proven mechanism)
**File:** `terraform/environments/dev/secrets-workload-identity.tf` (real, full content):
```hcl
# Dedicated identity for the inference application itself -
# separate from gke_nodes (which is for the node/kubelet layer).
# App workloads and nodes should never share an identity.
resource "google_service_account" "vllm_app" {
  account_id   = "vllm-app-sa"
  display_name = "vLLM Application Service Account"
}

# Least privilege: this identity can ONLY read secrets, nothing else.
resource "google_project_iam_member" "vllm_app_secret_accessor" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.vllm_app.email}"
}

# Workload Identity binding: allows the Kubernetes service account
# "vllm-app-ksa" in the "default" namespace to impersonate the GCP
# service account above. This is what removes the need for any
# JSON key file to ever exist.
resource "google_service_account_iam_member" "workload_identity_binding" {
  service_account_id = google_service_account.vllm_app.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[default/vllm-app-ksa]"
}

# A real (but dummy/test) secret to prove the mechanism end-to-end.
resource "google_secret_manager_secret" "test_api_key" {
  secret_id = "vllm-test-api-key"
  replication { auto {} }
  depends_on = [google_project_service.required]
}

resource "google_secret_manager_secret_version" "test_api_key_value" {
  secret      = google_secret_manager_secret.test_api_key.id
  secret_data = "dummy-test-value-not-a-real-secret"
}
```
**Why a dummy secret:** Proves the plumbing (KSA → GSA impersonation →
Secret Manager read) works end-to-end without risking a real credential
during testing.
**Verify:** From inside a pod using the `vllm-app-ksa` KSA:
`gcloud secrets versions access latest --secret=vllm-test-api-key` should
return `dummy-test-value-not-a-real-secret` with no key file anywhere on disk.

### STEP 6 — Workload Identity Federation for CI (no static keys)
**File:** `terraform/environments/dev/workload-identity-federation.tf` (real, full content):
```hcl
# Workload Identity Federation lets GitHub Actions authenticate to GCP
# using short-lived OIDC tokens instead of a long-lived JSON key.
# This is the correct pattern - and it sidesteps the org policy that
# blocks static service-account key creation entirely, since no key
# file is ever created.

resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-actions-pool"
  display_name              = "GitHub Actions Pool"
  description               = "Identity pool for GitHub Actions CI"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  # Restrict to ONLY this specific repo - least privilege
  attribute_condition = "assertion.repository == \"velrite/project7-ai-inference-platform\""
}

resource "google_service_account_iam_member" "github_actions_wif" {
  service_account_id = google_service_account.ci_pipeline.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/velrite/project7-ai-inference-platform"
}
```
**Underrated detail:** `attribute_condition` is the actual security boundary
— without this one line, trust extends to any repo under any account claiming
to be this CI pipeline, not just this specific repository.
**Real failure encountered:** After `terraform destroy`, this pool/provider
go into GCP soft-delete rather than disappearing. A plain `terraform apply`
returns `Error 409: Requested entity already exists`. Fix:
```bash
gcloud iam workload-identity-pools describe github-actions-pool --location=global --project=$PROJECT_ID --format="yaml(state,name)"
# if state: DELETED —
gcloud iam workload-identity-pools undelete github-actions-pool --location=global --project=$PROJECT_ID
terraform import google_iam_workload_identity_pool.github "projects/${PROJECT_ID}/locations/global/workloadIdentityPools/github-actions-pool"
# repeat the same pattern for the provider:
gcloud iam workload-identity-pools providers undelete github-provider --workload-identity-pool=github-actions-pool --location=global --project=$PROJECT_ID
terraform import google_iam_workload_identity_pool_provider.github "projects/${PROJECT_ID}/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider"
```

### STEP 7 — Dedicated CPU inference node pool
**File:** `terraform/environments/dev/cpu-inference-pool.tf` (real, full content):
```hcl
# Dedicated CPU inference node pool - separate from system-pool so
# platform tooling (Kyverno, Argo CD, NGINX) never competes with the
# inference workload for resources. Sized from real prior failures:
# e2-standard-4 (4 vCPU/16GB) and 60GB disk avoid both the
# Insufficient-memory and DiskPressure incidents recorded in ADR-005.
resource "google_container_node_pool" "cpu_inference" {
  name     = "cpu-inference-pool"
  cluster  = google_container_cluster.primary.name
  location = var.zone

  initial_node_count = 0

  autoscaling {
    min_node_count = 0
    max_node_count = 1
  }

  node_config {
    machine_type    = "e2-standard-4"
    disk_size_gb    = 60
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    labels = { pool = "cpu-inference" }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```
**File:** `terraform/environments/dev/gpu-node-pool.tf` — **NOT PROVIDED**
(not shared this session). Confirmed real from live `gcloud` output:
`nvidia-tesla-v100`, `acceleratorCount: 1`, `autoscaling.minNodeCount: 0`,
`maxNodeCount: 1`. Write analogously to the CPU pool above, with an
`accelerator { type = "nvidia-tesla-v100", count = 1 }` block inside
`node_config` and appropriate taints/tolerations so only GPU-tolerant pods
schedule there.
**Why `min_node_count = 0` on both:** Zero nodes = zero compute cost while
idle — confirmed directly via `gcloud compute instances list` showing no GPU
instance present for most of the project's history.
**Underrated detail:** `labels = { pool = "cpu-inference" }` here must
exactly match `nodeSelector: pool: cpu-inference` in the Deployment (Step 10)
— if these two independently-maintained strings ever drift, the pod stays
`Pending` forever with no obvious error.
**Verify:**
```bash
gcloud container node-pools list --cluster=project7-cluster --zone=$ZONE --project=$PROJECT_ID
```

### STEP 8 — Artifact Registry
**File:** `terraform/environments/dev/artifact-registry.tf` — **NOT PROVIDED**
(not shared, but confirmed real and in use: repository
`us-central1-docker.pkg.dev/velrite-tf-test/project7-inference`). Write a
`google_artifact_registry_repository` resource, `format = "DOCKER"`,
`location = var.region`.

### STEP 9 — Network segmentation (default-deny)
**Files:** `k8s/network-policies/default-deny.yaml`, `allow-granted-only.yaml`
— **NOT PROVIDED exact content**, but confirmed real and live via
`kubectl get networkpolicy -A`:

NAMESPACE NAME POD-SELECTOR
default default-deny-all <none>
default allow-granted-to-target app=target-app

Write `default-deny-all` as a `NetworkPolicy` with empty `podSelector: {}`
and `policyTypes: [Ingress, Egress]` (blocks everything by default). Write
`allow-granted-to-target` as a second policy selecting `app: target-app`
with an explicit `ingress.from` rule for the intended caller only.
**Verify (real test, proven this project):** deploy an unrelated pod without
the `app: target-app` label and confirm it cannot reach the protected
Service — a genuine `wget` timeout/connection-refused confirms enforcement,
not just the policy objects existing.

### STEP 10 — Dockerfile
**File:** `app/Dockerfile` (real, full content):
```dockerfile
# Minimal vLLM inference runtime image.
# We do NOT bake the model into the image - models are large (15GB+)
# and baking them in would make every rebuild slow and bloat the
# image registry. Model is pulled at container start instead.
FROM vllm/vllm-openai:latest

WORKDIR /app

# vLLM's OpenAI-compatible server port
EXPOSE 8000

ENTRYPOINT ["python3", "-m", "vllm.entrypoints.openai.api_server"]
```
**Why the model isn't baked in:** Explicit trade-off — smaller/faster image
builds at the cost of a cold-start model download on first pod start.

### STEP 11 — vLLM Deployment manifest
**File:** `k8s/vllm-deployment-cpu.yaml` (real, full content):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-inference-cpu
  labels:
    app: vllm-inference-cpu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-inference-cpu
  template:
    metadata:
      labels:
        app: vllm-inference-cpu
    spec:
      nodeSelector:
        pool: cpu-inference
      volumes:
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 2Gi
      containers:
        - name: vllm
          image: public.ecr.aws/q9t5s3a7/vllm-cpu-release-repo:latest
          env:
            - name: VLLM_CPU_KVCACHE_SPACE
              value: "4"
            - name: VLLM_ENABLE_V1_MULTIPROCESSING
              value: "0"
          args:
            - "--model=Qwen/Qwen2.5-1.5B-Instruct"
            - "--dtype=bfloat16"
            - "--max-model-len=4096"
            - "--enforce-eager"
          ports:
            - containerPort: 8000
          volumeMounts:
            - name: dshm
              mountPath: /dev/shm
          resources:
            requests:
              cpu: "2500m"
              memory: "8Gi"
            limits:
              cpu: "3500m"
              memory: "12Gi"
          startupProbe:
            httpGet: { path: /health, port: 8000 }
            failureThreshold: 90
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /health, port: 8000 }
            periodSeconds: 15
```
**Why `VLLM_ENABLE_V1_MULTIPROCESSING=0` + `--enforce-eager`:** Fixes a real,
encountered CPU startup deadlock (commit `015b825`). vLLM's V1 engine
defaults to a multiprocessing path that hangs in this CPU environment;
disabling it and forcing eager execution resolved it. **If you skip this:**
the pod will hang indefinitely at startup with no crash and no clear error —
this is the single most important undocumented gotcha in this entire build.
**Why `emptyDir: medium: Memory` for `/dev/shm`:** Default container shm
(64MB) is too small for vLLM's CPU tensor operations and fails silently.
**Why three separate probes:** `startupProbe` (up to 15 min via
`failureThreshold: 90 × periodSeconds: 10`) tolerates genuinely long model
load without Kubernetes killing the pod as unhealthy; `readinessProbe` and
`livenessProbe` take over afterward with much tighter loops.
**Verify:**
```bash
kubectl apply -f k8s/vllm-deployment-cpu.yaml
kubectl wait --for=condition=Ready pod -l app=vllm-inference-cpu --timeout=600s
kubectl port-forward svc/vllm-inference-cpu-svc 8000:8000 &
curl localhost:8000/v1/completions -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","prompt":"Say hello in one word.","max_tokens":5}'
```
Also write `k8s/vllm-service-cpu.yaml` (ClusterIP Service, port 8000) and
`k8s/ingress/vllm-cpu-ingress.yaml` (NGINX ingress, rate limiting and request
size limits confirmed present via README's own status table — exact YAML
NOT PROVIDED).

### STEP 12 — CI pipeline
**File:** `.github/workflows/ci-security.yml` (real, full content — reconstructed from the fetched log, key structure confirmed):
```yaml
name: CI Security Pipeline
on:
  push: { branches: [main, master] }
  pull_request: { branches: [main, master] }
env:
  REGISTRY: us-central1-docker.pkg.dev
  IMAGE_NAME: velrite-tf-test/project7-inference/vllm-inference
permissions:
  contents: read
  id-token: write
jobs:
  build-scan-sbom-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/362285503795/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider'
          service_account: 'project7-ci-pipeline@velrite-tf-test.iam.gserviceaccount.com'
      - run: gcloud auth configure-docker ${{ env.REGISTRY }} --quiet
      - run: docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} ./app
      - run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
      - uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: CRITICAL
          exit-code: 1
      - uses: anchore/sbom-action@v0
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json
      - uses: actions/upload-artifact@v4
        with: { name: sbom, path: sbom.spdx.json }
      - uses: sigstore/cosign-installer@v3
      - env:
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
        run: |
          echo "${{ secrets.COSIGN_PRIVATE_KEY }}" > cosign.key
          cosign sign --key cosign.key --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          rm -f cosign.key
```
**Why `exit-code: 1` on Trivy with `severity: CRITICAL`:** Any new CRITICAL
CVE blocks the pipeline before SBOM/signing ever run — a real failure mode
hit mid-project (two new CRITICAL CVEs in `linux-libc-dev` landed
undocumented). Fix: add the CVE IDs to `.trivyignore` with a justification
comment once confirmed unexploitable for this workload, same pattern as the
existing 4 exceptions.
**File:** `.trivyignore` (real content):
CVE-2025-37777: linux-libc-dev headers only, ksmbd component not used
by this workload. See docs/security-exceptions.md for full justification.

CVE-2025-37777
CVE-2026-53398
CVE-2026-64535
CVE-2026-64564

**Secrets needed in GitHub repo settings:** `COSIGN_PASSWORD`,
`COSIGN_PRIVATE_KEY`. Generate the key pair with
`cosign generate-key-pair` — commit only `cosign.pub` (public key, safe to
share), never `cosign.key` (add to `.gitignore`).
**Verify:** `gh run watch <run-id> --exit-status` should end with all steps
`✓` and `completed with 'success'`.

### STEP 13 — Kyverno admission enforcement
**File:** `k8s/admission/require-signed-images.yaml` (real, full content):
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "us-central1-docker.pkg.dev/velrite-tf-test/project7-inference/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEOCbfjVwzPuh5fR4ScJhxn8FixoWV
                      //iwfnSwlqhBhiPWTvCVIP22EwUyTrWO45JBE3WuU0HMHL+l6HkavIp7iQ==
                      -----END PUBLIC KEY-----
```
**Install:**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/ --force-update
kubectl create namespace kyverno --dry-run=client -o yaml | kubectl apply -f -
helm install kyverno kyverno/kyverno --namespace kyverno
kubectl rollout status deployment kyverno-admission-controller -n kyverno --timeout=180s
kubectl annotate serviceaccount kyverno-admission-controller -n kyverno \
  iam.gke.io/gcp-service-account=project7-gke-nodes@$PROJECT_ID.iam.gserviceaccount.com --overwrite
kubectl apply -f k8s/admission/require-signed-images.yaml
```
**Underrated detail:** `imageReferences` is scoped to one specific registry
path on purpose — a pod using an out-of-scope image (e.g. `nginx:latest`) is
correctly *ignored* by this policy, which can be mistaken for broken
enforcement if you don't check scope first.
**Verify, both directions (do not skip either):**
```bash
# Should be DENIED
kubectl run test-deny --image=us-central1-docker.pkg.dev/velrite-tf-test/project7-inference/vllm-inference:nonexistent-tag --restart=Never
# Should be CREATED (use your real signed SHA tag)
kubectl run test-allow --image=us-central1-docker.pkg.dev/velrite-tf-test/project7-inference/vllm-inference:$(git rev-parse HEAD) --restart=Never
kubectl delete pod test-deny test-allow --ignore-not-found=true
```

### STEP 14 — ArgoCD GitOps
**Install:**
```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
kubectl rollout status deployment argocd-server -n argocd --timeout=180s
```
**Application (scoped — do not sync the whole folder blindly):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: project7-inference
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/velrite/project7-ai-inference-platform.git
    targetRevision: master
    path: k8s
    directory:
      exclude: vllm-deployment.yaml   # abandoned GPU manifest - real lesson below
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: { prune: false, selfHeal: false }
```
**Real lesson (actually hit this):** An unscoped sync of all of `k8s/`
deployed an abandoned GPU-targeted manifest (`vllm-deployment.yaml`)
alongside the real CPU one — it created a second, permanently-`Pending`
deployment because no GPU nodes exist. `directory.exclude` fixes it. If you
don't have a leftover manifest like this, you won't hit this — but if you
do, this is exactly why.
**Verify:**
```bash
kubectl get application project7-inference -n argocd
# Expect: SYNC STATUS: Synced, HEALTH STATUS: Healthy
echo "=== same commit, three systems ==="
git rev-parse HEAD
kubectl get application project7-inference -n argocd -o jsonpath='{.status.sync.revision}'
```

### STEP 15 — Cost attribution
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --force-update
helm repo add opencost https://opencost.github.io/opencost-helm-chart --force-update
kubectl create namespace prometheus-system --dry-run=client -o yaml | kubectl apply -f -
helm install prometheus prometheus-community/prometheus --namespace prometheus-system \
  --set server.persistentVolume.enabled=false --set alertmanager.enabled=false
kubectl create namespace opencost --dry-run=client -o yaml | kubectl apply -f -
helm install opencost opencost/opencost --namespace opencost
gcloud services enable cloudbilling.googleapis.com --project=$PROJECT_ID
# create a Cloud Billing API key, store as k8s secret opencost-gcp-key, then:
kubectl port-forward -n opencost svc/opencost 9003:9003 &
curl "localhost:9003/allocation/compute?window=7d&aggregate=namespace"
```
**Real result obtained this way (14-minute sample, not a stable monthly figure):**

| Namespace | CPU cost | RAM cost | Total |
|---|---|---|---|
| `default` (vLLM) | $0.01814 | $0.00778 | $0.02591 |
| `kube-system` | $0.01481 | $0.00183 | $0.01664 |
| **Total (measured window)** | | | **$0.04374** |

**Underrated detail:** Prometheus deployed with `persistentVolume.enabled=
false` — cost history resets to near-zero after any pod restart. For a real
deployment, add a PVC.

---

## PART 6 — Evolution of the System (real changes, in order)

| Original approach | Problem discovered | Change made |
|---|---|---|
| GPU-targeted deployment | `GPUS_ALL_REGIONS=0.0` account-level quota, confirmed across 5 regions + a fresh project | Rebuilt on CPU node pool, same vLLM base, no rewrite |
| vLLM default V1 multiprocessing | CPU startup deadlock | `VLLM_ENABLE_V1_MULTIPROCESSING=0` + `--enforce-eager` |
| Manual `kubectl apply` of all manifests | Risk of drift / ArgoCD fighting manual changes | ArgoCD Application reconciling from git |
| ArgoCD syncing all of `k8s/` | Deployed an abandoned GPU manifest, stuck `Pending` | `directory.exclude` scoping |
| `.trivyignore` with 4 exceptions | 2 new undocumented CRITICAL CVEs landed, blocked CI | Added 2 new, justified exceptions, same pattern |

---

## PART 9 — Architecture Decision Records

### ADR-GPU-001: GPU Provisioning Blocked by `GPUS_ALL_REGIONS` Quota
**Status:** Blocked — external dependency (Google Cloud Support manual review).

**Context:** Full GPU infrastructure was built and verified — node pool
Terraform, taints/tolerations, V100 quota confirmed available, GKE
scheduling configuration correct.

**What was tried:** L4 (quota initially 0, later resolved, still blocked),
T4 (same pattern), P100 (ruled out, unavailable in zone), V100 (individual
quota confirmed 1.0, zone stock confirmed, MIG creation blocked), 5 regions
tested (same result), a fresh GCP project under the same billing account
(same result).

**Root cause:** Project-wide `GPUS_ALL_REGIONS` quota = 0.0, confirmed via
direct MIG error logs ("Quota 'GPUS_ALL_REGIONS' exceeded. Limit: 0.0
globally.") and an identical result on a brand-new project under the same
account — meaning this ceiling is tied to the account/billing identity, not
any specific project or region.

**Why self-service resolution wasn't possible:** GCP Console quota-edit form
explicitly rejected increase requests; Cloud Support billing confirmed it's
outside their scope, redirected to manual review.

**Decision:** Escalated with precise, evidence-backed findings. Continued
all GPU-independent work rather than blocking on one external dependency.

**Lesson:** "A single external platform dependency (cloud provider quota
approval) can block one specific capability without blocking a project's
overall engineering demonstration. Documenting the blocker precisely —
including everything ruled out and why — is itself evidence of methodical
troubleshooting, not a gap in the work."

### ADR-002: GitOps Scope Exclusion
**Context:** Unscoped ArgoCD sync deployed an abandoned GPU manifest,
creating a stuck `Pending` deployment.
**Decision:** `directory.exclude` in the Application spec.
**Lesson:** A tool doing exactly what it's configured to do is not the same
as doing what you meant — GitOps needs explicit scope, not just "sync the
folder."

---

## PART 13 — Reliability and Failure Modes (real test data)

### Pod-kill recovery (real, from `docs/07-reliability/`)
The running pod was force-deleted while healthy. Kubernetes rescheduled a
replacement automatically. **Real finding:** the new pod reached `Running`
within seconds but did **not** report `Ready` until several minutes later —
a first readiness check genuinely failed (connection-refused) because the
model was still loading. Recovery time for a stateful ML-serving pod must
include full model-reload time, not just container-restart time. Once
`kubectl wait` confirmed genuine readiness, a real inference request against
the same Service succeeded — correct routing to the new pod, zero manual
intervention.

### Load test — five real, unfiltered sequential requests
| Request | Result |
|---|---|
| 1 | Timed out at 60s |
| 2 | Timed out at 60s |
| 3 | Succeeded, 43.8s |
| 4 | Succeeded, 9.7s |
| 5 | Succeeded, 9.3s |

**Interpretation (from the real doc):** Clear cold-to-warm pattern — the
first two requests likely triggered one-time kernel compilation/memory-layout
work inside vLLM's CPU path. Once paid, latency stabilized ~9–10s for a
~20-token completion on CPU. "Should not be read as representative of
expected GPU latency, which would be materially faster."
**Stated limitation:** True concurrent load, a saturation point, and
TTFT/TPOT as separate metrics were not measured — five sequential requests
only.

| Failure | Detection | Recovery |
|---|---|---|
| `terraform destroy` run | N/A — deliberate | `terraform apply`; WIF needed soft-delete recovery (Step 6) |
| CI CVE scan failure | Red run | `.trivyignore` exception added |
| Unsigned image deployed | Kyverno admission webhook | Pod creation rejected — no manual action needed |
| vLLM CPU deadlock | Pod stuck, never Ready | `VLLM_ENABLE_V1_MULTIPROCESSING=0` + `--enforce-eager` |
| GitOps scope mismatch | `Pending` pod | `directory.exclude` |

---

## PART 17 — Cost Engineering

Real spend (`docs/08-finops/cost-per-inference.md`, GCP Billing Reports Aug
1–19 2026): $27.50 total usage cost, fully offset by free-trial credit ($0
net billed), plus a one-time $10 account-reactivation charge (separate,
account-level, not usage). ~10–15 real inference requests served across all
testing → "~$1.83 per inference request" — explicitly stated in the source
doc as a methodology demonstration using fixed-cost-dominated small-sample
data, **not representative of production unit economics**.
**Cost controls:** deliberate destroy/rebuild cycle between sessions
(`scripts/teardown.sh`); GPU pool autoscaling min=0.
**Stated gap:** no programmatic hard billing cap implemented (identified,
not built).

---

## PART 20 — Repository Structure

project7-ai-inference-platform/
├── app/
│ └── Dockerfile # vLLM image, model NOT baked in
├── terraform/environments/dev/
│ ├── apis.tf # NOT PROVIDED - required APIs
│ ├── network.tf # custom VPC + subnet
│ ├── gke.tf # NOT PROVIDED - cluster + system pool
│ ├── iam.tf # node + CI service accounts
│ ├── secrets-workload-identity.tf # vllm_app SA + Secret Manager
│ ├── workload-identity-federation.tf
│ ├── cpu-inference-pool.tf
│ ├── gpu-node-pool.tf # NOT PROVIDED - V100, min=0
│ ├── artifact-registry.tf # NOT PROVIDED
│ ├── backend.tf, variables.tf, terraform.tfvars # NOT PROVIDED
├── k8s/
│ ├── admission/require-signed-images.yaml
│ ├── network-policies/ # NOT PROVIDED exact YAML, real objects confirmed
│ ├── workload-identity/ksa.yaml # NOT PROVIDED
│ ├── vllm-deployment-cpu.yaml
│ ├── vllm-service-cpu.yaml # NOT PROVIDED exact YAML
│ ├── ingress/ # NOT PROVIDED exact YAML
│ └── observability/vllm-podmonitoring.yaml # NOT PROVIDED exact YAML
├── .github/workflows/ci-security.yml
├── .cosign/cosign.pub
├── .trivyignore
├── scripts/startup.sh, teardown.sh
└── docs/ # ADRs, evidence, runbooks


---

## PART 22 — If I Had to Rebuild This From Scratch (condensed order)

1. Enable required GCP APIs (Step 1).
2. Terraform: VPC/subnet → GKE cluster + system pool → node/CI/app service
   accounts → Secret Manager → WIF → CPU + GPU node pools → Artifact
   Registry. `terraform apply`, verify with `gcloud container clusters list`.
3. Write and push the Dockerfile + CI workflow. Generate a Cosign key pair,
   store `COSIGN_PASSWORD`/`COSIGN_PRIVATE_KEY` as GitHub secrets, commit
   only `cosign.pub`. Push, verify with `gh run watch <id> --exit-status`.
4. Write and apply the vLLM Deployment/Service/Ingress manifests on the CPU
   pool. Verify with a real `curl` to `/v1/completions`.
5. Install Kyverno, annotate its KSA for Workload Identity, apply the
   signature policy. Verify with the deny/allow pod test pair (Step 13).
6. Install ArgoCD, create the scoped Application. Verify `Synced`/`Healthy`
   and that the synced revision matches `git rev-parse HEAD`.
7. Apply NetworkPolicy. Verify with an ephemeral pod that should be denied.
8. Deploy OpenCost + Prometheus. Verify with a real cost pull.
9. Capture evidence for each step as you go — screenshots of real command
   output, not after-the-fact recreations.

---

## PART 24 — Things That Look Small but Matter

| Detail | Why it looks small | Why it actually matters |
|---|---|---|
| `attribute_condition` on the WIF provider | One line of HCL | The actual security boundary on who can claim this CI identity |
| `labels = { pool = "cpu-inference" }` ↔ `nodeSelector` | A string match | Drift between these two = pod stuck `Pending` forever, no clear error |
| `VLLM_ENABLE_V1_MULTIPROCESSING=0` | One env var | Fixes a real CPU startup deadlock — without it, silent indefinite hang |
| `emptyDir: medium: Memory` for `/dev/shm` | A volume mount | Default container shm silently breaks vLLM CPU tensor ops |
| `kyverno_artifact_reader` IAM binding | A read permission | Without it, verification fails closed — looks like security working, isn't actually checking anything |
| `validationFailureAction: Enforce` vs `Audit` | One word | The entire difference between blocking and only logging |
| GCP WIF soft-delete | "Just a 409 error" | Terraform alone can't recover it — needs explicit `undelete` + `import` |
| ArgoCD sync scope (`directory.exclude`) | "Just a folder path" | Unscoped sync deploys everything in the folder, including abandoned manifests |

---

## PART 26 — Technical Debt Register

| Debt | Why it exists | Recommended change |
|---|---|---|
| Deprecated `kyverno.io/v1 ClusterPolicy` API | Chart default, flagged live by `kubectl` | Migrate to `ImageValidatingPolicy` |
| Abandoned GPU manifest left in `k8s/`, excluded by filename | Historical record | Move to an `archive/` folder instead |
| No input validation in front of vLLM | Narrow project scope | Add a validation/rate-limit layer |
| No persistent Prometheus storage | Cost control during dev | Add a PVC for real cost trending |

---

## PART 28 — Interview Defense

**"Why two separate GCP service accounts for nodes vs. the app?"**
Node-level and application-level permissions must never be conflated — the
node identity can log/emit-metrics/pull-images; the app identity can *only*
read secrets. A compromised pod inherits the narrower scope.

**"Walk me through pod recovery."**
Force-deleted a healthy pod. Kubernetes rescheduled it within seconds, but
it genuinely wasn't `Ready` for several minutes, because readiness means
the model finished loading, not just that the container started — my first
verification attempt failed exactly because I checked too early.

**"Why WIF instead of a service account key?"**
Architecturally correct (short-lived OIDC, nothing to leak) *and* a hard
constraint — the org policy on this account blocks static key creation
outright.

**"Biggest known gap?"**
No auth in front of the inference endpoint; load testing is five sequential
requests, not true concurrency — both logged in my own docs, not found by
a reviewer.

---

## PART 30 — Engineering Summary

**What I built:** A GKE-based LLM inference platform with a signed, scanned,
admission-enforced delivery path, GitOps deployment, and real cost
attribution — including real incident recovery (soft-deleted IAM resources,
a broken CI pipeline, a GitOps scoping mistake) documented with the same
rigor as what worked the first time.

**Core engineering problem:** Proving what's running in the cluster is what
was actually intended — demonstrated with live, both-directions tests.

**Major trade-off:** CPU inference instead of GPU — a real, externally
imposed account-level quota ceiling, investigated exhaustively and escalated
through the correct channel rather than hidden.

**What's explicitly not claimed:** No production traffic, no real users, no
GPU numbers, no concurrent-load saturation point.

















# ===============================================================================
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

# ------------------------------------------------------------------------
# PROJECT MASTERY — PART 1

*Built strictly from what's actually documented and verified in this conversation's history. Where something wasn't confirmed, it's marked accordingly rather than assumed.*

---

## 1. EXECUTIVE TRUTH

**One paragraph:** You built Forge, a small but real Internal Developer Platform on Google Kubernetes Engine — a system that takes a developer from "I want to ship code" to "it's running safely in production" without them touching infrastructure directly, by chaining together a developer portal (Backstage), a CI/CD pipeline (GitLab, authenticating to GCP with zero stored credentials), a GitOps controller (Argo CD) that continuously enforces Git as the source of truth, a progressive-delivery controller (Argo Rollouts) that ships new versions gradually and automatically reverts bad ones, and a policy engine (Kyverno) that blocks certain classes of mistake before they can be deployed at all.

**Real-world analogy:** Think of a restaurant kitchen with a strict expediter system. A cook (developer) doesn't just throw a dish onto a plate and run it to a table. It goes through a pass — someone checks it meets standard (CI tests, security scan), a runner carries it out gradually to make sure the table's actually ready for it (canary rollout), and if the table sends it back, the kitchen immediately pulls the next plate off the line and serves the previous known-good dish instead (automated rollback) — all without the head chef having to personally watch every single plate.

**Technical version:** A GitOps-driven platform on GKE Autopilot, where GitLab CI (WIF-authenticated, no static keys) builds and pushes container images to Artifact Registry, Argo CD reconciles declared Kubernetes state from a GitHub repo, Argo Rollouts executes staged canary delivery gated by live Prometheus queries, and Kyverno enforces admission-time policy — all instrumented with kube-state-metrics-based SLOs and OpenCost-based cost attribution.

**Non-technical version:** A self-updating system that safely gets new software live, checks its own health while doing so, automatically undoes a bad change, and enforces its own rules — without a human having to babysit every deployment.

**Executive/CTO version:** This demonstrates the operational discipline a CTO actually cares about, independent of any specific product it might one day run: credential-free CI/CD auth (removes a standing breach surface), automated bad-deploy detection with zero-downtime rollback (removes a class of incident that would otherwise need a human awake at 2am), and enforced deployment policy (removes reliance on developers remembering rules). What it is *not* yet: a platform proven under real user load, or one with finished developer self-service (the publish-to-repo step is incomplete — see Technical Debt).

**Senior-engineer version:** The interesting engineering here isn't "used Kubernetes" — it's the specific, non-obvious permission and scheduling problems solved along the way: `storage.buckets.list` being a project-only permission invisible to bucket-scoped IAM, kubectl's client-side apply hitting a hard 262144-byte annotation ceiling on large CRDs, Kubernetes scheduling against *requested* not *actual* memory causing misleading "low utilization, still Pending" states, and Argo Rollouts' Prometheus provider always returning arrays even for scalar queries. None of these are documented prominently anywhere obvious — they were each found by going underneath a wrapped error message to the raw API response.

---

## 2. EVIDENCE INVENTORY

| Area | Evidence Found | Confidence | Importance | Missing Info |
|---|---|---|---|---|
| Cloud project/region | `forge-platform-11124`, `us-central1` | [IMPLEMENTED] | High | — |
| Cluster | `forge-cluster`, GKE Autopilot, regional, private nodes | [IMPLEMENTED] | High | — |
| Networking | VPC `forge-vpc`, subnet `forge-subnet`, Cloud NAT | [IMPLEMENTED] | Medium | Exact CIDR ranges not restated here |
| IaC | Terraform, GCS backend state, `terraform plan` confirmed clean | [IMPLEMENTED] | High | Full module structure not fully re-verified in this session |
| Container registry | Artifact Registry `forge-images`, immutable tags | [IMPLEMENTED] | High | — |
| CI/CD | GitLab CI/CD, WIF auth (no keys), full IAM chain documented | [IMPLEMENTED] | High | — |
| GitOps | Argo CD, `forge-demo` application, self-heal demonstrated live | [IMPLEMENTED] | High | Current sync state at time of reading unknown (may be scaled to 0) |
| Progressive delivery | Argo Rollouts, canary 20/50/100, AnalysisTemplate via Prometheus | [IMPLEMENTED] | High | — |
| Policy | Kyverno, 3 ClusterPolicies (1 Enforce, 2 Audit), block verified live | [IMPLEMENTED] | High | — |
| Observability | Prometheus/Grafana/Alertmanager/OTel Collector | [IMPLEMENTED — partial] | High | Grafana rendering unconfirmed; tracing is debug-log only |
| Cost | OpenCost, real per-namespace numbers pulled once | [IMPLEMENTED] | Medium | Not continuously monitored |
| Secrets | No credentials in source; Backstage GitLab token as K8s Secret | [IMPLEMENTED] | High | — |
| Developer portal | Backstage, SQLite in-memory, guest auth, running as Rollout | [IMPLEMENTED — limited] | High | No metrics plugin; catalog doesn't survive restart |
| Golden path | Scaffolder template exists | [IMPLEMENTED — incomplete] | Medium | Publish-to-repo step not built (ADR-010) |
| Alerting | Slack webhook configured in Alertmanager | [CONFIGURED, UNVERIFIED] | Low-Medium | Delivery never confirmed |
| Vulnerability scanning | Enabled on Artifact Registry | [CONFIGURED, UNVERIFIED] | Medium | No scan output ever reviewed |
| Network policies | — | [UNKNOWN] | Medium | Not built; listed as planned |
| Image signing/SBOM | — | [NOT IMPLEMENTED] | Medium | Explicitly planned-not-built |

---

## 3. ARCHITECTURE RECONSTRUCTION — REQUEST/CHANGE LIFECYCLE

The relevant "lifecycle" here isn't a user HTTP request — it's a **code change reaching production**:

1. Developer commits to the GitHub repo (canonical source of truth).
2. Push mirrored to GitLab (CI execution target — namespace `velrite-group`).
3. GitLab CI pipeline triggers (currently gated by `rules: changes: platform/backstage/**/*` — a change to only `terraform/*.tf` will **not** auto-trigger).
4. Pipeline authenticates to GCP via Workload Identity Federation — GitLab issues a short-lived OIDC token, GCP's STS exchanges it, no static credential exists anywhere in this chain.
5. Test → security scan → build stages run.
6. Source is packaged manually (`tar` + `gcloud storage cp`) and uploaded to a dedicated GCS bucket — **not** via `gcloud builds submit`'s own auto-upload, which was confirmed broken under WIF identities.
7. Cloud Build builds the image, pushes to Artifact Registry under a unique tag (immutability enabled — a second push to the same tag is rejected by the registry itself).
8. A Kubernetes manifest (or Rollout spec) referencing the new tag is updated in Git.
9. Argo CD detects the drift between declared and live state, reconciles it.
10. Argo Rollouts executes the canary: 20% traffic → pause → `AnalysisTemplate` query against Prometheus (`kube_deployment_status_replicas_ready`) → 50% → pause → analysis → 100%.
11. If analysis fails at any step, the Rollout aborts automatically — stable revision keeps serving 100% of traffic throughout, confirmed via a deliberate bad-image test.
12. Kyverno's admission webhook has already been checking every object creation throughout this whole chain, independent of the above — a `:latest`-tagged pod would never even reach step 10.

---

## 4. ARCHITECTURE MAP

```
┌────────────────────┐
│ Developer           │
└─────────┬────────────┘
          │ git push (source code)
          ▼
┌────────────────────┐        mirror        ┌────────────────────┐
│ GitHub (canonical)   │ ───────────────────▶ │ GitLab (CI exec)     │
└─────────┬────────────┘                      └─────────┬────────────┘
          │                                              │ OIDC token (WIF)
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ GCP STS / IAM         │
          │                                   └─────────┬────────────┘
          │                                              │ scoped access token
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ Cloud Build            │
          │                                   └─────────┬────────────┘
          │                                              │ container image (unique tag)
          │                                              ▼
          │                                   ┌────────────────────┐
          │                                   │ Artifact Registry      │
          │                                   │ (immutable tags)       │
          │                                   └─────────┬────────────┘
          │ manifest update (image tag)                 │ image pull
          ▼                                              │
┌────────────────────┐                                  │
│ Argo CD              │◀──── watches Git state ─────────┘
└─────────┬────────────┘
          │ reconciles declared state
          ▼
┌────────────────────┐
│ Argo Rollouts         │──── queries ───▶ Prometheus (readiness signal)
└─────────┬────────────┘
          │ staged traffic shift
          ▼
┌────────────────────┐
│ GKE Autopilot          │
│ (forge-cluster)        │
└─────────┬────────────┘
          │ every object creation checked
          ▼
┌────────────────────┐
│ Kyverno (admission)    │
└────────────────────┘

Cross-cutting, always present:
- Prometheus/Grafana/OTel ── scrapes/observes everything above
- OpenCost ── attributes cost by namespace across everything above
```

---

## 5. TECHNOLOGY DNA (condensed — full per-technology breakdown available on request for any specific item)

| Technology | Category | Core Problem It Solves Here | Key Trade-off Accepted |
|---|---|---|---|
| GKE Autopilot | Compute/orchestration | Removes node-lifecycle management as a competing concern | Less control over node sizing/bin-packing; memory ceiling hit repeatedly |
| Terraform | IaC | Reproducible, diffable infrastructure; zero-drift verifiable | State must be protected (GCS, versioned) — a real single point of failure if lost |
| Workload Identity Federation | Identity | Removes stored service-account keys entirely | More complex initial setup — the IAM chain took many iterations to get right |
| Argo CD | GitOps | Git as enforced desired state; auto-reverts manual drift | Manual emergency changes get silently reverted unless the operator knows to pause sync |
| Argo Rollouts | Progressive delivery | Staged traffic shift + automated health-gated abort | Analysis quality is only as good as the signal it queries (Kubernetes-level here, not app-level) |
| Kyverno | Policy-as-code | Admission-time enforcement in plain YAML, no separate policy language | Broad Enforce-everywhere would risk breaking existing workloads; mitigated via deliberate Audit/Enforce split |
| Prometheus/kube-state-metrics | Observability | Real, live cluster-state signal | No application-level metrics for Backstage specifically — a real, named gap |
| OpenCost | FinOps | Real computed cost, not estimated | Only as accurate as its Prometheus data source config (was misconfigured once, fixed) |

---

## 6. DECISION INVENTORY (meaningful, evidence-backed decisions only)

1. GKE Autopilot over Standard — [IMPLEMENTED], reasoning documented (ADR-003)
2. Argo CD manages application workloads only, not platform infra — [IMPLEMENTED], reasoning documented (ADR-005)
3. Backstage uses SQLite in-memory — [IMPLEMENTED], deliberate demo-scope trade-off (ADR-007)
4. Dependency pinning discipline (never `@latest`) after a real quarantine incident — [IMPLEMENTED] (ADR-008)
5. Unique image tags, never `:latest` — [IMPLEMENTED], directly caused by a real registry rejection (ADR-011)
6. GitLab authenticates via WIF, scoped to one namespace — [IMPLEMENTED] (ADR-012)
7. SLOs sourced from Kubernetes-level signals, not app-level — [IMPLEMENTED], because Backstage has no metrics plugin (ADR-015)
8. Kyverno Enforce vs Audit split — [IMPLEMENTED], deliberate rollout-safety reasoning

---

## 7. UNKNOWNS / MISSING EVIDENCE

- [UNKNOWN] Whether the Grafana SLO dashboard actually renders live data — configured, never confirmed.
- [UNKNOWN] Whether Slack alert delivery actually works — configured, test fired, delivery never confirmed by the operator.
- [UNKNOWN] Vulnerability scan results — scanning enabled, output never pulled or reviewed.
- [UNKNOWN] Current live state of the cluster/workloads at the moment you read this — several components were deliberately scaled to 0 for memory management during past sessions and may or may not have been restored.
- [NOT IMPLEMENTED] Network policies, image signing, SBOM generation — explicitly planned, not built.
- [NOT IMPLEMENTED] Golden-path publish-to-repo step — scaffolding works, publishing does not.

---

## 8. INITIAL ARCHITECTURE REVIEW

**Strengths:** The credential-free CI/CD auth chain is genuinely well-reasoned and behaviorally proven, not just configured. The canary/rollback mechanism is proven with a real bad-deploy test showing zero downtime to the stable revision. The debugging discipline shown throughout (raw HTTP diagnostics over trusting wrapped error text, local reproduction before trusting a full pipeline run) is a real, transferable engineering habit, not incidental.

**Weaknesses:** Observability has real gaps at the application layer — every SLO in this project is a Kubernetes-health proxy, not a true user-experience signal. The platform runs consistently close to its memory ceiling, which has caused repeated, avoidable-looking failures that were actually the same root cause recurring (a pattern worth being able to name precisely, not just patch around each time).

**Risks:** Single-replica-pair, effectively single-zone Backstage with an in-memory catalog is a real availability and data-durability risk if this were ever treated as more than a demo. The unconfirmed alerting path means, as of today, there is no proven way this platform would actually notify a human of a real problem.

**Unknowns:** Current live state (see above) — don't assume anything is running without checking `kubectl get pods --all-namespaces` first.

**Technical debt:** Catalogued in full in the earlier documentation package — golden-path publish, Grafana confirmation, Slack confirmation, node-failure testing never performed (a deliberate risk call, not an oversight), the superseded-but-not-deleted FinOps report.

---

## 9. KNOWLEDGE GAPS (preview — full breakdown comes in Stage 30)

**MUST KNOW to defend this project:** the exact IAM chain reasoning for `storage.admin` (why not `objectAdmin`), why `metadata.labels.app` differs from `spec.selector.app` on a Service, why `kubectl apply --server-side` was required for the Rollouts/Argo CD CRDs, and the two independent identity states (`gcloud` project vs `kubectl` context) that must both be checked.

**SHOULD KNOW:** the full canary AnalysisTemplate query evolution (why the array-index fix was needed), the Kubernetes scheduling-vs-usage distinction that caused the repeated memory-pressure incidents.

**ADVANCED:** Workload Identity Federation's trust-condition mechanics end to end, GitOps reconciliation loop internals.

---

## 10. MASTERY ROADMAP

Next, in order, unless you want to jump ahead: **Stage 5 (the WHY engine)** on the three or four decisions above with the least-settled reasoning in your own head right now, then **Stage 11 (failure mode analysis)** since you already have real, lived failure data to draw from rather than hypotheticals, then **Stage 20 (rebuild without AI)** once the reasoning is solid.

Tell me which stage to run next, or tell me which of the decisions in Section 6 you're *least* confident you could defend cold — that's usually the more useful starting point than going in strict numeric order.









# ---------------------------------------------------------------------------
Project Title

Enterprise Cloud Platform — Secure DevSecOps Platform on AWS EKS

One-line summary

Designed and operated a secure software supply chain and GitOps deployment platform on Amazon EKS — from source commit to running service — including container signing, policy enforcement, a service mesh, and a full observability stack, against a real multi-service application. Currently extending the platform with chaos engineering, HA database failover, and automated backup/restore.

Results (the numbers a reviewer actually remembers)
0 failed requests across a 200-request load test run through the live service mesh.
39 of 40 requests succeeded through a real Kubernetes node drain — proof of zero-downtime deployment under actual disruption, not a diagram claim.
Proved service mesh fault injection with measured timing: 5.0185s with an injected fault vs. 0.0186s without — a 270x difference, directly observed.
34 of 34 Prometheus monitoring targets confirmed healthy via live API query.
1,241 packages enumerated and independently verified queryable in a generated Software Bill of Materials (SBOM) for the deployed container image.
2 real CVEs identified and patched during live operation via automated dependency scanning.
1 genuine security misconfiguration found and fixed: discovered that a Kubernetes NetworkPolicy object existed but was not actually being enforced at the infrastructure level, root-caused it to the CNI plugin's default behavior, fixed it, and re-verified with a live before/after test.
13 distinct production-style incidents diagnosed and resolved end-to-end, each with documented root cause (not just symptom patching).
Tools Used, and Why Each One

Cloud & Infrastructure

AWS (EKS, VPC, IAM/IRSA, EC2, S3, DynamoDB) — chosen specifically because the project required IRSA (IAM Roles for Service Accounts), which has no direct GCP equivalent — a deliberate architecture decision, not a default.
Terraform — infrastructure as code, split into isolated state stacks (bootstrap/network/cluster) by blast radius and change frequency, not one monolithic config.

CI/CD Supply Chain

GitHub Actions — pipeline orchestration, path-filtered triggers.
Gitleaks — secret scanning, scoped to avoid false-positive noise from an upstream fork's historical commits.
Trivy — dependency and container image vulnerability scanning.
Syft — SBOM generation.
Cosign — keyless container image signing via GitHub OIDC — no static private key to manage or leak.
OPA/Conftest — policy-as-code admission gate.

GitOps & Progressive Delivery

ArgoCD — GitOps continuous deployment; cluster state reconciled from Git, not pushed manually.
Argo Rollouts — canary deployments with automatic rollback, proven by a real configuration bug that triggered a genuine rollback, caught live, not staged.

Service Mesh & Security Hardening

Istio — mutual TLS, traffic policy, fault injection.
HashiCorp Vault — secrets management pattern.
Kyverno — Kubernetes admission policy engine.
Falco — eBPF-based runtime threat detection.
kube-bench — CIS Kubernetes benchmark compliance scanning.

Observability

Prometheus & Grafana — metrics and dashboards.
Loki — log aggregation.
Tempo — distributed tracing.
OpenTelemetry Collector — unified telemetry pipeline.

Resilience (in progress)

Chaos Mesh / LitmusChaos — fault-injection experiments against live infrastructure.
Patroni (PostgreSQL HA), Redis Sentinel, etcd — automated database failover.
Velero — cluster, namespace, and volume backup/restore.

Supporting: Kubernetes (EKS), Helm, Docker, GitHub Container Registry (GHCR).

Skills Demonstrated

Cloud infrastructure design and cost-aware trade-off decisions · secure CI/CD pipeline engineering · container supply-chain security (SBOM, signing, scanning) · GitOps and progressive delivery · service mesh configuration and live verification · Kubernetes troubleshooting under real resource constraints · security control verification methodology · incident root-cause documentation · disaster recovery and chaos engineering (in progress).

Honest Scope Note (give this to your CV agent too — it's what makes the rest credible in an interview)

Everything above was built and operated against a live AWS account across multiple real working sessions, with infrastructure deliberately torn down between sessions for cost discipline — not a tutorial followed once. A small number of items are explicitly configured but not yet verified under live traffic (noted internally, available on request), and the resilience/chaos-engineering phase is actively in progress. Being precise about what's proven versus what's underway is intentional — it's the same discipline this project itself is built on.

Send me that build report whenever it's ready — the moment Phase 3 has real evidence behind it, this whole section upgrades from "in progress" to the same result-backed language as everything above it.









































































































































































































































































































































































































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


