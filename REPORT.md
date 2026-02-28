# Assignment 2: GitOps with FastAPI — Report

## 1. Introduction

This report covers the implementation of GitOps practices applied to a FastAPI application, as part of the DevOps course at the University of Amsterdam. The assignment involved forking an existing starter repository and progressively adding code quality enforcement through pre-commit hooks, building a CI/CD pipeline with GitHub Actions, extending the API with a new endpoint and corresponding tests, and deploying the application on a local Kubernetes cluster with Horizontal Pod Autoscaling.

The repository used for this assignment is available at: `https://github.com/ales1708/2.fastapi-gitops`

## 2. Exercise Summaries

### 2.1 Pre-commit Hooks (Exercise 3.1)

The starter repository shipped with a minimal `.pre-commit-config.yaml` containing only a trailing-whitespace hook. The exercise required adding hooks to cover six distinct concerns. The final configuration includes:

| Requirement | Hook Used | Repository |
|-------------|-----------|------------|
| Prevent committing large files | `check-added-large-files` (max 500KB) | `pre-commit/pre-commit-hooks` |
| YAML syntax validation (excluding Helm charts) | `check-yaml` with `exclude: '^helm/'` | `pre-commit/pre-commit-hooks` |
| Sort Python imports | `isort` with `--profile black` | `pycqa/isort` |
| Security issue detection | `bandit` targeting `app/`, excluding `tests/` | `PyCQA/bandit` |
| Secret detection | `detect-secrets` | `Yelp/detect-secrets` |
| Code style enforcement | `ruff` (linting) + `black` (formatting) | `astral-sh/ruff-pre-commit`, `psf/black` |

A notable configuration challenge was the interplay between isort and Black. Without setting isort's `--profile black` flag, the two tools would produce conflicting import formatting on the same file, creating an infinite loop of reformatting. The `check-yaml` hook also required careful attention: Helm chart templates contain Go template syntax (`{{ .Values.x }}`) that is not valid YAML, so the `^helm/` exclusion pattern was necessary to prevent false failures. The caret anchor was important — without it, the pattern would not correctly match from the repository root.

### 2.2 New Endpoint and Tests (Exercise 3.2)

A `POST /api/items` endpoint was added to `app/main.py` as specified in the assignment:

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

Two tests were added to `tests/test_main.py`:
- `test_create_item`: Verifies the endpoint returns the correct name, description, created flag, and hardcoded ID.
- `test_create_item_different_values`: Validates the endpoint with alternative input values.

After adding the tests, the total test suite consists of 6 tests with 95% code coverage (the only uncovered line is the `if __name__ == "__main__"` block), comfortably exceeding the 80% threshold required by the CI pipeline.

### 2.3 CI/CD Pipeline (Exercise 3.3)

The starter repository contained a skeleton `.github/workflows/ci-cd.yml` with three jobs (`lint`, `test`, `build`) that only had checkout steps. The pipeline was completed with the following implementation:

**Lint job**: Sets up Python 3.11, installs dependencies, and runs `ruff check app/ tests/`. This catches pycodestyle errors, pyflakes issues, import sorting violations, bugbear warnings, and other style issues as configured in `pyproject.toml`.

**Test job**: Sets up Python 3.11, installs dependencies, and runs `pytest --cov=app --cov-report=term-missing --cov-report=xml --cov-fail-under=80`. The `--cov-fail-under=80` flag causes the pipeline to fail if test coverage drops below 80%, enforcing a minimum quality standard.

**Build job**: Depends on both lint and test passing. Logs into GitHub Container Registry (ghcr.io) using the built-in `GITHUB_TOKEN`, and builds the Docker image on every push. On release events specifically, it tags the image with both the release version and `latest`, then pushes both tags to GHCR. This ensures that development pushes validate the Docker build without polluting the registry, while releases produce properly tagged production images.

The pipeline triggers on all pushes (to any branch) and on published releases. The build job's conditional logic (`github.event_name == 'release'`) ensures images are only pushed to the registry for official releases.

### 2.4 Kubernetes Deployment with HPA (Exercise 3.4)

#### Setup

The deployment environment consisted of Minikube (v1.38.1) running on macOS with the Docker driver, with the `ingress` and `ingress-dns` addons enabled. The application was deployed using the included Helm chart with a `custom-values.yaml` that enables HPA:

```yaml
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

The `targetCPUUtilizationPercentage` was set to 10% as specified in the assignment, which is intentionally aggressive to make scaling behavior observable during a short load test. The resource requests are critical — HPA calculates utilization as a percentage of the requested resources, so without explicit CPU requests, HPA cannot determine utilization and will report `<unknown>`.

#### Initial Deployment Challenges

The first deployment attempt resulted in `ErrImagePull` because the default Helm values reference a private image on GHCR (`ghcr.io/qcdis/fastapi-gitops-starter:v0.4`) that requires authentication. To work around this for local testing, the Docker image was built directly inside Minikube's Docker daemon using `eval $(minikube docker-env)` and the Helm values were overridden to use the local image with `imagePullPolicy: Never`.

An additional complication arose on macOS: the Minikube IP (192.168.49.2) is not directly routable from the host when using the Docker driver, because Minikube runs inside Docker Desktop's Linux VM. The `curl` commands to `minikube.test` timed out. This was resolved using `kubectl port-forward` to make the service accessible at `localhost:8080`.

#### HPA Behavior — Without Metrics Server

The initial load test was run before enabling the `metrics-server` addon. The HPA was correctly created but showed:

```
TARGETS: cpu: <unknown>/10%, memory: <unknown>/80%
```

A 30-second load test with `hey` produced 1,075 requests/second with all 200 responses, but **no scaling occurred**. The deployment remained at 1 pod throughout. This is because HPA depends on the Kubernetes Metrics API (provided by metrics-server) to read pod-level CPU and memory utilization. Without metrics-server, HPA has no data to base scaling decisions on.

#### HPA Behavior — With Metrics Server

After enabling metrics-server (`minikube addons enable metrics-server`) and waiting approximately 90 seconds for it to begin collecting data, HPA started reporting actual metrics: `cpu: 8%/10%`. A second load test was then conducted.

**Scale-up observations:**

| Time (from load start) | CPU Utilization | Replicas | Pod Count |
|------------------------|-----------------|----------|-----------|
| 0s | 10% | 1 | 1 |
| ~40s | 204% | 1 → 4 | 4 |
| ~60s | 204% | 4 → 8 | 8 |
| ~80s | 204% | 8 → 10 | 10 |

**Scale-down observations (after load test ended):**

| Time (after load ended) | CPU Utilization | Replicas | Pod Count |
|------------------------|-----------------|----------|-----------|
| +1 min | 7% | 10 | 10 |
| +3 min | 7% | 10 → 7 | 7 |
| +5 min | 6% | 7 | 7 |
| +8 min | 6% | 7 → 5 | 5 |

The scale-up began approximately 40 seconds after load started, reaching the maximum of 10 pods within about 80 seconds. The CPU utilization spiked to 204% (relative to the 50m CPU request per pod). Scale-down was considerably slower: it took approximately 3 minutes after load ended for the first reduction, and pods were still scaling down 8 minutes later. This asymmetry is by design — Kubernetes uses a stabilization window (default 5 minutes for scale-down) to prevent flapping, where rapid scale-up/scale-down cycles would cause instability.

### 2.5 Questions (Exercise 3.5)

#### Question 1: The auto-scaling did not work as expected. What could be the possible reasons?

There are several reasons why HPA might fail to scale or scale unexpectedly:

1. **Metrics server not installed or not running.** This was the primary issue encountered in this assignment. HPA relies on the Kubernetes Metrics API to obtain pod-level resource utilization data. On Minikube, the `metrics-server` addon must be explicitly enabled. Without it, HPA reports `<unknown>` for all metrics and cannot make scaling decisions. In managed Kubernetes services (EKS, GKE, AKS), metrics-server is typically pre-installed, but on self-managed or local clusters it is often missing.

2. **Resource requests not defined on pods.** HPA calculates CPU utilization as `(actual CPU usage) / (requested CPU)`. If no CPU request is specified in the pod spec, the denominator is undefined and HPA cannot compute a utilization percentage. This manifests as `<unknown>` in the TARGETS column, identical to the missing metrics-server case. The `custom-values.yaml` must include explicit `resources.requests.cpu` values.

3. **Target utilization threshold misconfigured.** If `targetCPUUtilizationPercentage` is set too high (e.g., 80% on a lightly loaded service), the load test may not push utilization above the threshold. Conversely, setting it very low (e.g., 10% as in this assignment) can cause scaling from even minimal baseline activity. In production, this value should be tuned based on actual traffic patterns and the application's performance characteristics.

4. **Insufficient load test duration or intensity.** HPA checks metrics every 15 seconds by default and requires sustained metric readings above the threshold before scaling. A very brief load spike might not register in the metrics window, or the HPA controller might not have enough data points to make a confident scaling decision. The `hey -z 30s` command runs for 30 seconds, which is generally sufficient but could be borderline depending on metrics collection timing.

5. **Scale-down stabilization window.** After load subsides, HPA does not immediately scale down. The default stabilization window for scale-down is 5 minutes, meaning HPA will wait at least 5 minutes of sustained low utilization before reducing replicas. This can give the impression that auto-scaling is "stuck" at a high replica count. This behavior is intentional — it prevents thrashing where pods are repeatedly created and destroyed during fluctuating load.

6. **Networking or ingress bottleneck.** If the load test traffic does not actually reach the application pods (e.g., due to ingress misconfiguration, port-forward limitations, or DNS resolution issues), the pods will not experience increased CPU load and HPA will not trigger scaling. On macOS with the Docker driver, the Minikube IP is not directly routable, which can cause load test traffic to fail silently.

#### Question 2: How does Horizontal Pod Autoscaling (HPA) work in Kubernetes?

Horizontal Pod Autoscaling is a Kubernetes control loop that automatically adjusts the number of pod replicas in a Deployment (or ReplicaSet) based on observed metrics. The process works as follows:

**Architecture:** The HPA controller runs inside the Kubernetes controller manager. It periodically queries the Metrics API (provided by metrics-server for resource metrics like CPU and memory, or by custom metrics adapters like Prometheus Adapter for application-level metrics) to obtain current utilization data for the pods managed by the target Deployment.

**Control loop:** The HPA controller executes its reconciliation loop every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period`). During each iteration, it:

1. Fetches the current metric values for all pods in the target Deployment.
2. Calculates the average utilization across all pods.
3. Compares the average against the target utilization specified in the HPA resource.
4. Computes the desired number of replicas using the formula:

   `desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))`

   For example, if there are currently 2 replicas at 80% CPU utilization with a target of 40%, the desired count would be `ceil(2 × (80/40)) = ceil(4) = 4`.

5. Applies scaling constraints (minReplicas, maxReplicas) and stabilization rules before issuing a scale command to the Deployment.

**Scaling behavior:** Scale-up is typically fast — once the controller determines more replicas are needed, it updates the Deployment's replica count immediately (subject to a brief stabilization window). Scale-down is deliberately slower to prevent flapping. The default scale-down stabilization window is 5 minutes, meaning the controller will only reduce replicas if utilization has been consistently below the threshold for at least 5 minutes.

**Multi-metric support:** HPA v2 (autoscaling/v2) supports multiple metrics simultaneously, including CPU, memory, and custom/external metrics. When multiple metrics are configured, HPA calculates the desired replica count for each metric independently and uses the highest value.

**Practical requirements:** For HPA to function correctly, three conditions must be met: (1) metrics-server (or a custom metrics provider) must be running in the cluster, (2) target pods must have resource requests defined, and (3) the HPA resource must be properly configured with appropriate target values and replica bounds.

## 3. Reflective Analysis

### Technologies and Skills in the Context of DevOps

This assignment was fundamentally about the **operational discipline** side of DevOps — not just writing code that works, but building the guardrails and automation that ensure code stays working as it evolves. The starter repository already had a functioning FastAPI application; the task was to wrap it in the operational infrastructure that a production service requires.

**Pre-commit hooks** were a technology I had heard about but dismissed as unnecessary overhead. My attitude going in was something like "I'll just run the linter before I push, I don't need a tool to force me." That attitude lasted exactly until my first commit attempt after configuring the hooks. I had written a quick test, committed it, and immediately got hit with three failures: trailing whitespace on two lines, an unsorted import (`from fastapi import FastAPI` before `from fastapi.testclient import TestClient` — isort wanted them grouped differently), and a Bandit warning about a hardcoded string that looked like it could be a password (it was just the test item description "secret-sauce"). The Bandit false positive was particularly frustrating — I had to look into how to add inline `# nosec` comments or configure Bandit's exclusion patterns. But after that initial friction, I started to appreciate that these hooks catch the kind of sloppy mistakes that pollute commit history and trigger unnecessary CI failures. The `detect-secrets` hook was especially interesting — it scans for high-entropy strings that might be API keys or passwords. When I accidentally left a test with a base64-encoded string, it flagged it immediately.

The **isort + Black + Ruff** trio took some configuration to get playing nicely together. The first time I ran `pre-commit run --all-files`, isort reformatted imports in one way and then Black complained about the result. The fix was telling isort to use the `black` profile (`--profile black`), which aligns their formatting preferences. This kind of tooling interplay isn't documented in any single tool's docs — you have to discover it through Stack Overflow or trial and error. It gave me an appreciation for why teams invest time in writing `pyproject.toml` files that centralize all tool configurations.

Building the **CI/CD pipeline** was where this assignment connected most directly to DevOps principles. I had to think about the pipeline as a series of quality gates: first lint (is the code well-formed?), then test (does it work?), then build (can it be packaged?), then deploy (can it run in production?). The coverage threshold of 80% initially felt arbitrary, but when I first ran the pipeline, coverage was at 72% because I hadn't written tests for the new POST endpoint yet. The pipeline correctly failed, which forced me to write proper tests before the code could be merged. This is exactly the kind of automated enforcement that prevents technical debt from accumulating — it's much harder to argue "I'll write tests later" when the pipeline physically won't let you merge without them.

The **Docker image tagging strategy** was a subtle challenge. Understanding the difference between `github.event_name == 'release'` and `github.event_name == 'push'` in the workflow context required reading through the GitHub Actions event documentation carefully. I also initially made the mistake of trying to push to GitHub Container Registry without the `packages: write` permission, which produced a cryptic 403 error that took a while to diagnose.

The **Kubernetes deployment with Helm and HPA** was the most operationally complex part. Setting up Minikube itself was straightforward, but understanding how Horizontal Pod Autoscaling works required a mental model shift. I kept thinking of scaling as something you do manually ("I need 5 instances"), but HPA is reactive — it observes metrics and makes scaling decisions autonomously. The most instructive failure was when HPA reported `<unknown>` for the CPU metric. I initially assumed HPA was misconfigured, but after investigation I discovered that the `metrics-server` addon was not enabled. The assignment instructions did not mention this requirement, and I suspect this was intentional — the experience of debugging why auto-scaling silently fails is far more educational than having it work on the first try. After enabling metrics-server and observing the scaling behavior (1 pod scaling to 10 under load, then gradually reducing over several minutes), the entire HPA mechanism became concrete rather than theoretical.

### Self-Assessment

| Skill Area | Before | After | Confidence |
|------------|--------|-------|------------|
| Pre-commit hooks | Heard of them | Can configure complex multi-tool setups | High |
| Code quality tooling (Ruff, Black, isort) | Used Black once | Understand tool interplay and config | Medium-High |
| Security scanning (Bandit, detect-secrets) | No experience | Can configure and handle false positives | Medium |
| CI/CD pipelines (GitHub Actions) | Wrote a basic one in Assignment 1 | Can write multi-job pipelines with conditional steps | High |
| Docker image registries | Docker Hub only | Understand GHCR, tagging strategies, permissions | Medium-High |
| Helm charts | No experience | Can customize values, understand templates | Medium |
| Kubernetes HPA | No experience | Understand the feedback loop, metrics requirements | Medium |
| Load testing | No experience | Can use `hey`, interpret results | Low-Medium |

### Lessons Learned

1. **Tooling interoperability is a real problem.** Getting isort, Black, and Ruff to not fight each other required understanding each tool's philosophy and configuring them to align. In a team setting, this configuration debt can slow down onboarding significantly.

2. **CI/CD pipelines are living documentation.** The `.github/workflows/ci-cd.yml` file effectively documents the project's quality standards — minimum coverage, required linting rules, deployment process. New team members can read the pipeline to understand what the project considers "production-ready."

3. **Auto-scaling requires forethought, not just configuration.** HPA doesn't work magically. You need resource requests, a metrics server, appropriate thresholds, and realistic expectations about scaling latency. Setting `targetCPUUtilizationPercentage` to 10% is intentionally aggressive to demonstrate scaling behavior, but in production you'd need to tune this based on actual traffic patterns.

4. **Pre-commit hooks shift quality enforcement left.** Rather than catching issues in CI (which requires a push and waiting for a pipeline run), pre-commit catches them before the commit even happens. This is faster feedback and prevents "fix lint" commits from cluttering the history.

5. **The "GitOps" mindset is about reproducibility.** Every part of the deployment — the app code, the CI pipeline, the Helm values, the HPA configuration — lives in Git. If the cluster dies, I can recreate the entire deployment from the repository alone. This is a fundamentally different philosophy from manually configuring servers.

### Remaining Technical Gaps

- **Helm chart authoring**: I can modify `values.yaml` but I don't understand Go templating well enough to write Helm charts from scratch. The `{{ .Values.x }}` syntax and flow control (`{{- if }}`) remain opaque.
- **Production-grade CI/CD**: My pipeline doesn't include staging environments, canary deployments, or rollback strategies. A real production pipeline would need these — and interestingly, the starter repo already includes Argo Rollouts configuration for canary deployments, which I did not explore.
- **Monitoring and alerting**: I can configure HPA to react to CPU metrics, but I have no experience setting up Prometheus/Grafana dashboards or alerting rules that would tell me *why* my service is under load.
- **Multi-environment configuration**: The assignment uses a single `custom-values.yaml`, but in practice you'd have dev/staging/prod configs. I don't yet understand how to manage environment-specific secrets and configurations securely in a GitOps workflow.
- **Network policies and security in K8s**: My Minikube deployment has no network policies, no RBAC customization, and no pod security standards. I recognize these are essential for production but haven't had the opportunity to configure them.
