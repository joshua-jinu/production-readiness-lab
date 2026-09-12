# Production Readiness Evidence Pack

This document organizes the production-readiness evidence for the `production-readiness-lab` service into four pillars: Deployment evidence, Monitoring visibility, Recovery readiness, and Engineering decisions. It also contains an honest readiness review and a short cloud mapping note.

---

## 1. Deployment evidence

- CI configuration: this repository has a GitHub Actions workflow at [.github/workflows/ci.yml](.github/workflows/ci.yml) that runs lint, tests and builds a Docker image.

- Compose configuration and a healthy stack:

  - The service is defined in `docker-compose.yml` and exposes the app on port 8080. The app container has a healthcheck that probes `http://localhost:8080/health`.
  - Relevant files:
    - [docker-compose.yml](docker-compose.yml)
    - [app/app.js](app/app.js)
    - [prometheus/prometheus.yml](prometheus/prometheus.yml)

  - Config verification (inspected from files in the repo):

    - `app` listens on port `8080` when `PORT=8080` (see [app/app.js](app/app.js)).
    - `docker-compose.yml` maps `8080:8080` and sets `APP_VERSION` and `PORT=8080` for the `app` service.
    - Prometheus scrapes the app at `app:8080` with `metrics_path: /metrics` (see [prometheus/prometheus.yml](prometheus/prometheus.yml)).

  - Local run note (evidence collection instructions):

    I attempted to start the stack from this environment but the Docker daemon was unavailable here. To collect the deployment evidence locally, run:

    ```sh
    cd production-readiness-lab
    docker compose up -d
    docker compose ps
    ```

    Example healthy `docker compose ps` output you should capture and commit as a screenshot or paste here:

    ```text
    Name                 Command               State           Ports
    -----------------------------------------------------------------
    prl-app              node app.js           Up (healthy)    0.0.0.0:8080->8080/tcp
    prl-prometheus       /bin/prometheus ...   Up              0.0.0.0:9090->9090/tcp
    prl-grafana          /run.sh               Up              0.0.0.0:3000->3000/tcp
    ```

  - CI pass evidence: after opening a PR, capture the green GitHub Actions run that shows `Lint`, `Test`, and `Build Docker image` succeeded. The workflow file is at [.github/workflows/ci.yml](.github/workflows/ci.yml).

---

## 2. Monitoring visibility

- Prometheus is provisioned and configured to scrape the app. Key config: [prometheus/prometheus.yml](prometheus/prometheus.yml) shows job `app` targets `['app:8080']` and `metrics_path: /metrics` with a 5s scrape interval.

- Grafana dashboard: A pre-provisioned dashboard is included at [grafana/dashboards/service-overview.json](grafana/dashboards/service-overview.json). It contains panels for Request Rate, Error Rate, Latency (p95) and Health (uses `up{job="app"}`). Panels refresh at `5s`.

- Local validation instructions (collect this during the demo):

  1. Start the stack locally (see Deployment section).
 2. Generate traffic to populate metrics: `./scripts/generate-traffic.sh 500 http://localhost:8080`.
 3. Verify Prometheus sees the target as UP:

    ```sh
    curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job=="app")'
    ```

    Look for `"health":"UP"` and `"discoveredLabels"` showing `job="app"` and `instance":"app:8080"` in the JSON — include this snippet in the evidence.

  4. Open Grafana at `http://localhost:3000` (admin/admin) and open the `Service Overview` dashboard (UID `service-overview`) — record a short screen capture showing Request Rate, Error Rate, Latency, and Health change when you generate traffic or cause failure.

---

## 3. Recovery readiness

- Rollback strategy (manual container/image rollback for this lab):

  - The Compose setup runs the `app` as a single container. To demonstrate rollback during the demo:

    1. Build and tag a known-good image: `docker build -t prl-app:good .`
    2. Deploy it (via Compose or `docker run`) and confirm health.
    3. Build a bad image (e.g., change `APP_VERSION` or inject a trivial bug) and deploy it as `prl-app:bad`.
    4. Observe the healthcheck fail and Grafana `Health` panel go to `DOWN` and/or increased error rate.
    5. Roll back by re-tagging and deploying the `prl-app:good` image and show the health returning to `UP` and metrics normalizing.

  - Commands to simulate a bad release and rollback (run locally during demo):

    ```sh
    # tag the current working build as the known-good image
    docker build -t prl-app:good .

    # Simulate a bad release: modify the app to fail health (example: set PORT to an incorrect value), rebuild
    docker build -t prl-app:bad .

    # Update Compose to use the bad image (or docker stop/start the container with the bad image)
    # Then, to rollback:
    docker tag prl-app:good production-readiness-lab_app:latest
    docker compose up -d
    ```

  - Evidence to collect and commit: the `docker ps`/`docker compose ps` and `docker logs` snippets during failure and after rollback, plus a short screen capture of Grafana showing Health dropping then returning after rollback.

---

## 4. Engineering decisions

- Health check: `docker-compose.yml` defines a `healthcheck` for the `app` service that calls `/health`. This makes automated orchestrators and load balancers able to detect unhealthy containers earlier (improves recoverability).

- Instrumentation: the app exposes `/metrics` and registers the standard `prom-client` metrics plus request counters, histograms and error counters. Grafana panels are prebuilt to show the golden signals (rate, errors, latency, health).

- Scrape frequency: Prometheus is configured with `scrape_interval: 5s`. This gives low-latency detection at the cost of slightly more scrape load; for a single-instance lab this is appropriate for demo purposes.

- Restart policy: `restart: unless-stopped` is set for containers in `docker-compose.yml`, which is useful for recovery in a simple environment but should be coupled with deployment automation in production.

---

## Readiness Review

- **Strengths**:
  - Instrumentation is present and wired into Prometheus and Grafana dashboards for the golden signals.
  - A healthcheck is defined and a short scrape interval ensures quick detection of failures.
  - CI workflow exists to run lint, tests and produce a Docker image.

- **Unresolved risks (honest list)**:
  - CI: I could not observe a passing GitHub Actions run from this environment. The workflow file exists but a committed run (green check) must be included in the PR evidence.
  - Single-instance deployment: the `docker-compose` setup runs one `app` container with no replication — this is a single point of failure for production traffic.
  - No automated alerts: there are dashboards but no alerting rules are defined; there is no pager or notification automation shown here.
  - No resource limits: containers have no CPU/memory limits in `docker-compose.yml`, so noisy neighbors or memory leaks could destabilize the host.
  - No tested automated rollback: rollback steps are manual in this lab; there is no CI/CD pipeline automating a safe rollback.

- **Verdict**: Ready with caveats.

  Rationale: the system demonstrates the essential observability and health checks required for a production release (metrics, dashboard, health probe). However, because CI run evidence, automated alerts, replication, resource limits, and an automated rollback mechanism are not proven here, the release should proceed only with the listed mitigations in place or as a limited canary rollout.

---

## Demo checklist (what I recorded or will record for the PR/video)

1. Start stack: `docker compose up -d` and show `docker compose ps` with healthy states.
2. Generate traffic: `./scripts/generate-traffic.sh 500 http://localhost:8080` and show Grafana panels populate.
3. Show Prometheus `api/v1/targets` JSON showing the `app` target `UP`.
4. Deploy a bad release (small change), show health fail and error rate spike.
5. Roll back to the good image and show health return to `UP` and metrics normalize.
6. Show GitHub Actions CI green run (lint, tests, docker build) in the PR checks.

Include short screen recordings or GIF snippets of steps 2–5 and paste `curl` outputs and `docker compose ps` output in the PR description.

---

## Cloud mapping note

Local flow -> cloud recommended mapping (documentation only):

- Build image locally -> push image to Artifact Registry (or Docker Hub / Container Registry).
- Deploy image to a managed service (Cloud Run, ECS, or Kubernetes): in GCP, you'd push to Artifact Registry then deploy to Cloud Run or GKE.
- Observability mapping:
  - Prometheus metrics -> Cloud Monitoring (export or use Managed Prometheus). Grafana dashboards can remain as dashboards pointing at Managed Prometheus or Cloud Monitoring metrics.
  - Health checks -> Cloud Load Balancer / Cloud Run readiness probes.
- Rollback:
  - Use image tags and Traffic-splitting or revision rollback in Cloud Run/K8s for safe rollback; automate via CI/CD (Cloud Build / GitHub Actions) to revert to the last good image.

---

If you want, I can now:

- attempt to run the stack locally here again (Docker desktop must be running), collect the `docker compose ps` output and Prometheus `/api/v1/targets` JSON and commit them into this README as evidence, or
- help you produce the short demo recording script and sample PR description that includes all required artifacts.
