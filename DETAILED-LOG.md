# Assignment 2: GitOps with FastAPI — Detailed Implementation Log

This document records every step taken during the implementation of Assignment 2, including commands executed, outputs observed, errors encountered, and how they were resolved.

---

## 1. Project Setup

### 1.1 Forking and Cloning

The starter repository was forked from `https://github.com/DevOps-and-Cloud-based-Software/fastapi-gitops.git` to `https://github.com/ales1708/2.fastapi-gitops.git`.

The forked repository was cloned to the local machine at:
```
/Users/alessiosilverio/Desktop/uni/MSc AI/DevOps/assignments/2.fastapi-gitops/
```

### 1.2 Initial Repository Structure

The starter repository contained:
```
2.fastapi-gitops/
├── app/
│   ├── __init__.py
│   └── main.py                    # FastAPI app with 4 endpoints
├── tests/
│   ├── __init__.py
│   └── test_main.py               # 4 existing tests
├── docker/
│   └── Dockerfile                 # Python 3.11-slim based
├── helm/fastapi-gitops-starter/
│   ├── Chart.yaml
│   ├── values.yaml                # Default values (HPA disabled)
│   ├── example-values.yaml        # Example with HPA enabled
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── service.yaml
│   │   └── ... (other templates)
│   └── ... (other example values)
├── .github/workflows/
│   ├── ci-cd.yml                  # SKELETON — only checkout steps
│   ├── tests_md-urls.yml
│   └── markdown2pdf.yml
├── .pre-commit-config.yaml        # MINIMAL — only trailing-whitespace
├── pyproject.toml                 # Ruff, Black, pytest config
├── requirements.txt               # FastAPI, uvicorn, pytest, ruff, black
└── README.md
```

### 1.3 Existing Application Endpoints

The starter `app/main.py` provided:
- `GET /` — Returns `{"message": "Welcome to FastAPI GitOps Starter!"}`
- `GET /health` — Returns `{"status": "healthy", "service": "fastapi-gitops-starter"}`
- `GET /api/items` — Returns list of 3 hardcoded items
- `GET /api/items/{item_id}` — Returns dynamically generated item by ID

All endpoints are served under the root path `/GitOps-Starter` (configurable via `ROOT_PATH` env var).

### 1.4 Existing Test Suite

`tests/test_main.py` contained 4 tests:
- `test_root()` — Validates welcome message
- `test_health_check()` — Validates health endpoint response
- `test_list_items()` — Checks 3 items are returned
- `test_get_item()` — Tests dynamic item retrieval with ID 5

### 1.5 Existing CI/CD Pipeline (Skeleton)

The `.github/workflows/ci-cd.yml` had three jobs defined but incomplete:
- `lint` — Only had checkout step
- `test` — Only had checkout step
- `build` — Only had checkout step, with correct permissions for GHCR

---

## 2. Exercise 3.1: Pre-commit Hooks

### 2.1 Original Configuration

`.pre-commit-config.yaml` contained only:
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
```

### 2.2 Requirements Analysis

The assignment specified six types of hooks to add:
1. Prevent committing large files
2. Check YAML files for syntax errors (excluding Helm charts)
3. Sort imports in Python files
4. Check for security issues
5. Make sure we do not commit secrets
6. Check code style

### 2.3 Tool Selection and Configuration Decisions

**Large files**: Used `check-added-large-files` from `pre-commit-hooks` with a 500KB limit. This is the standard pre-commit hook for this purpose.

**YAML validation**: Used `check-yaml` from `pre-commit-hooks`. The critical decision was the exclusion pattern. Helm chart templates contain Go template syntax like `{{ .Values.replicaCount }}` which is not valid YAML. The exclusion pattern `'^helm/'` was chosen, with the `^` anchor ensuring it only matches from the repository root. Without the anchor, the pattern might miss files in subdirectories.

**Import sorting**: Used `isort` (v5.13.2) with the `--profile black` argument. This was a deliberate choice — isort and Black have different default opinions about import formatting. The `black` profile tells isort to format imports in a way that Black will not subsequently reformat, preventing the two tools from fighting each other.

**Security scanning**: Used `bandit` (v1.8.3) configured to scan `app/` and exclude `tests/`. Bandit is a standard Python security linter that checks for common security issues like hardcoded passwords, SQL injection patterns, and unsafe function calls. Tests are excluded because they often contain deliberate security anti-patterns (test credentials, mock data).

**Secret detection**: Used `detect-secrets` (v1.5.0) from Yelp. This tool scans for high-entropy strings, known secret patterns (AWS keys, private keys, tokens), and other indicators of accidentally committed secrets.

**Code style**: Used both `ruff` (v0.8.4, matching the version in `requirements.txt`) for linting with auto-fix, and `black` (v24.10.0, matching `requirements.txt`) for formatting. Ruff handles linting rules (pycodestyle, pyflakes, isort, bugbear, comprehensions, pyupgrade as configured in `pyproject.toml`), while Black handles consistent formatting.

### 2.4 Final Configuration

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: check-yaml
        exclude: '^helm/'

  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort
        args: ['--profile', 'black']

  - repo: https://github.com/PyCQA/bandit
    rev: 1.8.3
    hooks:
      - id: bandit
        args: ['-r', 'app/']
        exclude: tests/

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff
        args: ['--fix']

  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black
```

---

## 3. Exercise 3.2: New Endpoint

### 3.1 Endpoint Implementation

Added to `app/main.py` before the `if __name__` block:

```python
@app.post("/api/items")
async def create_item(name: str, description: str):
    """Create a new item."""
    return {
        "id": 999,
        "name": name,
        "description": description,
        "created": True,
    }
```

This matches the specification in the assignment PDF exactly. The endpoint accepts `name` and `description` as query parameters and returns a JSON response with a hardcoded ID of 999.

### 3.2 Test Implementation

Added two tests to `tests/test_main.py`:

```python
def test_create_item():
    """Test the create item endpoint."""
    response = client.post(
        "/api/items",
        params={"name": "Test Item", "description": "A test item"},
    )
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Test Item"
    assert data["description"] == "A test item"
    assert data["created"] is True
    assert data["id"] == 999


def test_create_item_different_values():
    """Test creating an item with different values."""
    response = client.post(
        "/api/items",
        params={"name": "Another Item", "description": "Another description"},
    )
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Another Item"
    assert data["description"] == "Another description"
```

### 3.3 Test Results

```
tests/test_main.py ......                                                [100%]

Name              Stmts   Miss  Cover   Missing
-----------------------------------------------
app/__init__.py       0      0   100%
app/main.py          22      1    95%   64
-----------------------------------------------
TOTAL                22      1    95%

Required test coverage of 80% reached. Total coverage: 95.45%
============================== 6 passed in 1.59s
```

All 6 tests passed. The only uncovered line (line 64) is `uvicorn.run(app, host="0.0.0.0", port=8000)` inside the `if __name__ == "__main__"` guard, which is standard and not testable via the test client.

### 3.4 Linting Verification

```
$ ruff check app/ tests/
All checks passed!
```

---

## 4. Exercise 3.3: CI/CD Pipeline

### 4.1 Pipeline Structure

The completed `.github/workflows/ci-cd.yml` implements three jobs:

**Trigger configuration:**
- Runs on every push to any branch (except tags)
- Runs on release events (type: published)

**Job 1 — lint:**
- Checks out code
- Sets up Python 3.11
- Installs dependencies from `requirements.txt`
- Runs `ruff check app/ tests/`

**Job 2 — test:**
- Checks out code
- Sets up Python 3.11
- Installs dependencies from `requirements.txt`
- Runs `pytest --cov=app --cov-report=term-missing --cov-report=xml --cov-fail-under=80`
- Pipeline fails if coverage drops below 80%

**Job 3 — build:**
- Depends on lint and test jobs passing
- Runs on push and release events
- Logs into GitHub Container Registry using `GITHUB_TOKEN`
- Builds Docker image tagged with commit SHA
- On release only: tags image with release version and `latest`, pushes both to GHCR

### 4.2 Design Decisions

- **Lint and test run in parallel** (no dependency between them), reducing pipeline duration
- **Build depends on both** (using `needs: [lint, test]`), ensuring only quality code gets built
- **Docker build on every push** validates the Dockerfile works, but images are only pushed to the registry on release
- **GHCR over Docker Hub** was chosen because it integrates natively with GitHub — the `GITHUB_TOKEN` provides authentication automatically without configuring external secrets

---

## 5. Exercise 3.4: Kubernetes Deployment

### 5.1 Tool Installation

Prerequisites installed via Homebrew on macOS:
```bash
brew install minikube helm hey
```

Versions installed:
- Minikube: v1.38.1
- Helm: v4.1.1
- hey: latest
- Docker: v28.0.4 (pre-existing)
- kubectl: pre-existing

### 5.2 Minikube Cluster Setup

```bash
minikube start --addons=ingress,ingress-dns
```

Minikube started with the Docker driver on macOS (ARM64). Addons enabled:
- `ingress` — NGINX Ingress Controller
- `ingress-dns` — DNS resolution for ingress hostnames

**Note:** `metrics-server` was NOT enabled at this point (intentional — see Section 5.6).

### 5.3 DNS Configuration

```bash
$ minikube ip
192.168.49.2

$ echo "192.168.49.2 minikube.test" | sudo tee -a /etc/hosts
```

Added `minikube.test` hostname resolution to `/etc/hosts`.

### 5.4 Custom Values File

Created `custom-values.yaml`:
```yaml
replicaCount: 1

ingress:
  enabled: true
  hosts:
    - host: minikube.test
      paths:
        - path: /GitOps-Starter
          pathType: Prefix

app:
  rootPath: "/GitOps-Starter"

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 10

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

### 5.5 Deployment — First Attempt (Failed)

```bash
$ helm install my-release ./helm/fastapi-gitops-starter -f custom-values.yaml
```

**Result:** Deployment created but pod entered `ErrImagePull` state.

```
$ kubectl get pods
NAME                                                READY   STATUS         RESTARTS   AGE
my-release-fastapi-gitops-starter-9957d9cc7-jrjfv   0/1     ErrImagePull   0          6s
```

**Root cause:** The default `values.yaml` references `ghcr.io/qcdis/fastapi-gitops-starter:v0.4`, a private image on GitHub Container Registry. The Helm chart also expects an `image-pull-secret` with GHCR credentials that we don't have.

**Error from kubectl describe:**
```
Failed to pull image "ghcr.io/qcdis/fastapi-gitops-starter:v0.4": unauthorized
```

### 5.5.1 Fix: Building Image Locally in Minikube

```bash
# Point Docker CLI to Minikube's Docker daemon
eval $(minikube docker-env)

# Build the image inside Minikube
docker build -t fastapi-gitops-starter:local -f docker/Dockerfile .
```

Image built successfully (sha256:ea8c2664a128...).

### 5.5.2 Fix: Helm Upgrade Attempt (Failed)

```bash
$ helm upgrade my-release ./helm/fastapi-gitops-starter -f custom-values.yaml \
  --set image.repository=fastapi-gitops-starter \
  --set image.tag=local \
  --set image.pullPolicy=Never \
  --set registry.createImagePullSecret=false \
  --set "imagePullSecrets={}"
```

**Error:**
```
UPGRADE FAILED: failed to create typed patch object: .spec.template.spec.imagePullSecrets:
element 0: associative list with keys may not have non-map elements
```

**Root cause:** Kubernetes treats `imagePullSecrets` as an associative list keyed by `name`. Setting it to an empty list `{}` via `--set` creates an invalid patch because the existing deployment has a non-empty list. Helm cannot reconcile the type mismatch.

### 5.5.3 Fix: Clean Reinstall

```bash
$ helm uninstall my-release

$ helm install my-release ./helm/fastapi-gitops-starter -f custom-values.yaml \
  --set image.repository=fastapi-gitops-starter \
  --set image.tag=local \
  --set image.pullPolicy=Never \
  --set registry.createImagePullSecret=false \
  --set imagePullSecrets=null
```

**Result:** Successful deployment. Pod running:
```
NAME                                                READY   STATUS    RESTARTS   AGE
my-release-fastapi-gitops-starter-64f58d775-lsplx   1/1     Running   0          15m
```

### 5.6 Networking — macOS Docker Driver Issue

```bash
$ curl http://minikube.test/GitOps-Starter/api/items
# Exit code 28 — connection timed out
```

**Root cause:** Minikube with the Docker driver on macOS runs inside Docker Desktop's Linux VM. The Minikube IP (192.168.49.2) is internal to Docker's virtual network and is not routable from the macOS host. The assignment instructions (`minikube ip` → add to `/etc/hosts`) assume a driver where the IP is directly accessible (e.g., VirtualBox, hyperkit on macOS, or bare-metal on Linux).

**Workaround:** Used `kubectl port-forward` to expose the service:
```bash
$ kubectl port-forward svc/my-release-fastapi-gitops-starter 8080:80

$ curl http://localhost:8080/GitOps-Starter/api/items
{"items":[{"id":1,"name":"Item 1","description":"First item"},{"id":2,"name":"Item 2","description":"Second item"},{"id":3,"name":"Item 3","description":"Third item"}]}
```

Application confirmed accessible and functional.

### 5.7 HPA Status — Before Metrics Server

```
$ kubectl get hpa
NAME                                REFERENCE                                      TARGETS                                     MINPODS   MAXPODS   REPLICAS
my-release-fastapi-gitops-starter   Deployment/my-release-fastapi-gitops-starter   cpu: <unknown>/10%, memory: <unknown>/80%   1         10        1
```

Both CPU and memory targets show `<unknown>` because `metrics-server` is not running.

### 5.8 Load Test #1 — Without Metrics Server

```bash
$ hey -z 30s http://localhost:8080/GitOps-Starter/api/items
```

**Results:**
```
Summary:
  Total:        30.0175 secs
  Slowest:      0.4176 secs
  Fastest:      0.0018 secs
  Average:      0.0465 secs
  Requests/sec: 1074.6067
  Total data:   5419176 bytes

Status code distribution:
  [200] 32257 responses
```

**HPA after load test:**
```
TARGETS: cpu: <unknown>/10%, memory: <unknown>/80%
REPLICAS: 1
```

**No scaling occurred.** HPA could not read metrics, so it could not determine that scaling was needed.

### 5.9 Enabling Metrics Server

```bash
$ minikube addons enable metrics-server
```

Waited approximately 90 seconds for metrics-server to start collecting pod-level resource data.

```
$ kubectl get hpa
TARGETS: cpu: 8%/10%, memory: 52%/80%
REPLICAS: 1
```

Metrics now reporting correctly.

### 5.10 Load Test #2 — With Metrics Server

```bash
$ hey -z 30s http://localhost:8080/GitOps-Starter/api/items
```

**Load test results:**
```
Summary:
  Total:        30.0132 secs
  Slowest:      0.3134 secs
  Fastest:      0.0013 secs
  Average:      0.0465 secs
  Requests/sec: 1076.0925
  Total data:   5425896 bytes

Status code distribution:
  [200] 32297 responses
```

### 5.11 HPA Scaling Observations — Scale Up

Monitored every 10 seconds during the load test:

| Timestamp | CPU | Replicas | Pod Count |
|-----------|-----|----------|-----------|
| 13:13:42 | 10% | 1 | 1 |
| 13:13:52 | 10% | 1 | 1 |
| 13:14:03 | 10% | 1 | 1 |
| 13:14:13 | 10% | 1 | 1 |
| 13:14:23 | 204% | 1 → 4 | 4 |
| 13:14:33 | 204% | 4 | 8 |
| 13:14:43 | 204% | 4 → 8 | 8 |
| 13:14:53 | 204% | 8 → 10 | 10 |
| 13:15:04 | 8% | 10 | 10 |

**Scale-up summary:**
- First scaling event: ~40 seconds after load began
- Reached maximum (10 pods): ~80 seconds after load began
- Peak CPU utilization reported: 204% (relative to 50m request per pod)

### 5.12 HPA Scaling Observations — Scale Down

Monitored every 60 seconds after load test completed:

| Time After Load | CPU | Replicas | Pod Count |
|----------------|-----|----------|-----------|
| +1 min | 7% | 10 | 10 |
| +2 min | 6% | 10 | 10 |
| +3 min | 7% | 10 → 7 | 7 |
| +4 min | 7% | 7 | 7 |
| +5 min | 6% | 7 | 7 |
| +6 min | 7% | 7 | 7 |
| +7 min | 6% | 7 | 7 |
| +8 min | 6% | 7 → 5 | 5 |

**Scale-down summary:**
- First reduction: ~3 minutes after load ended (10 → 7)
- Still reducing at 8 minutes (7 → 5)
- Scale-down is deliberately slow due to the 5-minute stabilization window
- The idle CPU of 6-7% is close to the 10% target, which slows further scale-down

### 5.13 Cleanup

```bash
$ helm uninstall my-release
$ minikube stop
```

---

## 6. Git Workflow

### 6.1 Branch and Commit

```bash
$ git checkout -b feature/gitops-exercises

$ git add .pre-commit-config.yaml app/main.py tests/test_main.py .github/workflows/ci-cd.yml custom-values.yaml

$ git commit -m "Add GitOps exercises: pre-commit hooks, create item endpoint, CI/CD pipeline, and HPA config

- Exercise 3.1: Configure pre-commit hooks for large files, YAML validation (excluding Helm),
  isort, bandit security scanning, detect-secrets, ruff linting, and black formatting
- Exercise 3.2: Add POST /api/items endpoint with tests (95% coverage)
- Exercise 3.3: Implement CI/CD pipeline with ruff linting, pytest coverage gate (80%),
  and Docker build/push to GHCR on release
- Exercise 3.4: Add custom-values.yaml with HPA enabled at 10% CPU target"
```

### 6.2 Push

```bash
$ git push -u origin feature/gitops-exercises
```

**Note:** First push attempt succeeded but the branch was not visible on GitHub. Investigation with `git ls-remote origin` confirmed only `main` existed on the remote. The branch was pushed again successfully on the second attempt.

### 6.3 Pull Request

PR created on GitHub from `feature/gitops-exercises` → `main` within the fork (`ales1708/2.fastapi-gitops`).

**Important note:** When creating a PR from a forked repository, GitHub defaults the base repository to the upstream (original) repo. This must be manually changed to point to the fork's own `main` branch to keep the PR within the fork.

---

## 7. Files Changed Summary

| File | Type | Lines Added | Description |
|------|------|------------|-------------|
| `.pre-commit-config.yaml` | Modified | +40 | 7 hooks across 6 repositories |
| `app/main.py` | Modified | +11 | POST /api/items endpoint |
| `tests/test_main.py` | Modified | +26 | 2 new test functions |
| `.github/workflows/ci-cd.yml` | Modified | +42 | Complete lint, test, build pipeline |
| `custom-values.yaml` | New | +19 | HPA config with 10% CPU target |
| **Total** | | **+138** | |
