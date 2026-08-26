# Credit Risk MLOps SRE Platform

![Credit Risk MLOps SRE architecture](docs/credit-risk-sre-architecture.png)

## Project overview

This project demonstrates how Site Reliability Engineering practices can be applied to a production-style machine-learning inference service. A credit-risk model is trained and registered in MLflow, loaded dynamically by a FastAPI service, deployed on Kubernetes, and observed through Prometheus and Grafana. Prometheus alert rules detect service, model-loading and Kubernetes reliability failures, while Alertmanager routes actionable notifications to Slack.

The objective is not simply to expose a model through an API. The objective is to operate that model as a reliable service with measurable targets, failure detection, dashboards and response procedures.

## Business and engineering problem

A credit-risk model may perform well during offline evaluation but still fail as a production service. Typical production risks include:

- The approved model cannot be downloaded from MLflow.
- The inference API returns errors even though its pods are running.
- Predictions become too slow under load.
- Containers exceed their memory limits and are terminated.
- Pods repeatedly restart or deployment replicas become unavailable.
- A technical deployment remains healthy while model quality deteriorates over time.

This project addresses the first five risks directly through application and platform observability. Data drift, concept drift and delayed ground-truth evaluation are identified as the next layer of model monitoring.

## Architecture

The system contains two related paths:

1. **Model lifecycle:** training pipeline → evaluation → MLflow Tracking and Model Registry → champion alias → FastAPI model loading → predictions.
2. **Reliability lifecycle:** FastAPI and Kubernetes metrics → ServiceMonitor → Prometheus → Grafana and PrometheusRule → Alertmanager → Slack.

### Main components

| Component | Responsibility |
|---|---|
| Training pipeline | Trains and evaluates candidate credit-risk models. |
| MLflow Tracking | Records parameters, metrics and model artifacts for each experiment. |
| MLflow Model Registry | Versions approved models and exposes the `champion` alias. |
| FastAPI | Loads `credit-risk-model@champion` and exposes health, model, prediction, reload and metrics endpoints. |
| Kubernetes Deployment | Runs two API replicas with probes, resource controls and a restricted security context. |
| Kubernetes Service | Provides stable access to the API pods. |
| ServiceMonitor | Instructs Prometheus Operator to scrape the API's `/metrics` endpoint every 15 seconds. |
| Prometheus | Stores time-series metrics and evaluates reliability rules. |
| Grafana | Displays model availability, success rate, latency, traffic and prediction distribution. |
| Alertmanager | Groups and routes firing alerts. |
| Slack | Receives operational notifications for the SRE team. |

## End-to-end model lifecycle

### 1. Train and evaluate

The training workflow reads prepared data, trains a candidate model and calculates offline metrics such as accuracy, precision, recall and ROC-AUC. Training parameters, evaluation metrics and artifacts are logged to MLflow so every run is reproducible and comparable.

An offline evaluation gate should determine whether a candidate is eligible for registration or promotion. A candidate should not become the production model solely because its training job completed successfully.

### 2. Register and promote

The accepted model is registered under:

```text
credit-risk-model
```

The production-approved version is assigned the alias:

```text
champion
```

The alias decouples the serving application from a hard-coded model version. FastAPI requests `models:/credit-risk-model@champion`; MLflow resolves that reference to the model version currently approved for production.

### 3. Load the model into FastAPI

Kubernetes provides the following configuration to the API:

```yaml
MLFLOW_TRACKING_URI: "http://mlflow.mlflow.svc.cluster.local:5000"
MODEL_NAME: "credit-risk-model"
MODEL_ALIAS: "champion"
```

At startup, the model loader constructs the MLflow model URI and downloads the champion artifact. The loaded model remains in the API process memory and is reused for predictions. FastAPI does not need to contact MLflow for every request.

The `/reload-model` endpoint allows the service to resolve the champion alias again after a promotion. In a stronger production implementation, that endpoint should be authenticated and the reload should be rolled out safely or coordinated across replicas.

### 4. Serve predictions

Clients send credit features to:

```text
POST /predict
```

FastAPI validates the request, passes the features to the loaded model and returns the result. At the same time, application instrumentation records request count, result status, latency and prediction distribution.

## API endpoints

| Endpoint | Purpose |
|---|---|
| `GET /health` | Kubernetes readiness and service-health check. |
| `GET /model` | Reports information about the currently loaded model. |
| `POST /predict` | Runs credit-risk inference. |
| `POST /reload-model` | Reloads the model referenced by the MLflow champion alias. |
| `GET /metrics` | Exposes Prometheus-format application and model-serving metrics. |

## Kubernetes design

The API runs in the `credit-risk` namespace as a two-replica Deployment. The manifest includes:

- Readiness and liveness probes.
- CPU and memory requests and limits.
- Non-root execution.
- No privilege escalation.
- Dropped Linux capabilities.
- `RuntimeDefault` seccomp profile.
- A ClusterIP Service with a named `http` port.

The named Service port is important because the ServiceMonitor refers to `port: http`. The Service selects pods with `app: credit-risk-api`, and the ServiceMonitor selects the Service using the same label.

## Observability flow

FastAPI exposes the following custom metrics:

| Metric | Type and meaning |
|---|---|
| `credit_risk_requests_total` | Counter of requests labelled by endpoint and status. |
| `credit_risk_request_latency_seconds` | Histogram used to calculate latency percentiles. |
| `credit_risk_predictions_total` | Counter showing prediction distribution. |
| `credit_risk_model_loaded` | Gauge: `1` when the model is available and `0` when loading failed. |
| `credit_risk_model_version_info` | Information metric identifying the loaded model version. |

Prometheus Operator watches the `ServiceMonitor` resource. The ServiceMonitor selects the `credit-risk-api` Service in the `credit-risk` namespace and instructs Prometheus to scrape `/metrics` every 15 seconds. Prometheus stores the samples and makes them available to PromQL, Grafana and alert rules.

## SLI, SLO and alerting model

An **SLI** is a measured reliability signal. An **SLO** is the target for that signal. An **alert** indicates that the target is being breached or that a failure condition requires action.

| Area | SLI | Example SLO | Demo alert condition |
|---|---|---|---|
| Prediction availability | Successful `/predict` requests ÷ all `/predict` requests | At least 99% successful requests | Error rate above 5% for 5 minutes |
| Prediction latency | P95 `/predict` latency | P95 below 500 ms | P95 above 1 second for 5 minutes |
| Model availability | `credit_risk_model_loaded` | Champion model loaded on all serving replicas | Value equals `0` for 2 minutes |
| Pod stability | Container restarts and termination reason | No crash loops or OOM terminations | More than 3 restarts in 10 minutes, or OOMKilled |
| Deployment availability | Unavailable replicas | All required replicas available | One or more unavailable replicas for 2 minutes |

The demo alert thresholds are deliberately less strict than some proposed SLOs. In a production system, warning and critical alerts would normally be designed around error-budget consumption and user impact rather than using every SLO boundary as a paging threshold.

## Alert scenarios and response

### 1. Champion model not loaded

**Alert:** `CreditRiskModelNotLoaded`

**Scenario:** MLflow is unavailable, the model alias is missing, artifact storage cannot be accessed, or the artifact is incompatible with the serving image.

**Signal:**

```promql
credit_risk_model_loaded == 0
```

**First response:**

```bash
kubectl logs -n credit-risk deployment/credit-risk-api
kubectl get pods -n mlflow
kubectl get svc -n mlflow
```

Confirm that `credit-risk-model@champion` exists, that DNS resolves `mlflow.mlflow.svc.cluster.local`, and that the serving environment contains compatible dependencies.

### 2. High prediction error rate

**Alert:** `CreditRiskHighErrorRate`

**Scenario:** Invalid request handling, model inference exceptions, schema incompatibility, dependency failures or a defective application release cause more than 5% of prediction requests to fail for five minutes.

**Signal:**

```promql
sum(rate(credit_risk_requests_total{endpoint="/predict",status="error"}[5m]))
/
clamp_min(
  sum(rate(credit_risk_requests_total{endpoint="/predict"}[5m])),
  0.000001
)
```

**First response:** correlate the increase with application logs, recent releases, input-validation errors, model changes and downstream dependency health. If the alert began immediately after a deployment, pause or roll back the release.

### 3. High P95 prediction latency

**Alert:** `CreditRiskHighLatency`

**Scenario:** At least 95% of requests are no slower than the calculated P95 value, but that value remains above one second for five minutes. Likely causes include CPU throttling, large request payloads, slow feature processing, an expensive model or overloaded pods.

**Signal:**

```promql
histogram_quantile(
  0.95,
  sum(rate(credit_risk_request_latency_seconds_bucket{endpoint="/predict"}[5m])) by (le)
)
```

**First response:** compare latency with request rate, CPU, memory, throttling and replica availability. Determine whether the issue is load-related, model-related or release-related before scaling.

### 4. OOMKilled container

**Alert:** `CreditRiskPodOOMKilled`

**Scenario:** A container consumes more than its configured memory limit and Kubernetes terminates it.

**First response:**

```bash
kubectl get pods -n credit-risk
kubectl describe pod -n credit-risk POD_NAME
kubectl top pod -n credit-risk
```

Check the termination reason, model size, memory growth and whether the configured `768Mi` limit is justified. Raising the limit without understanding consumption may only postpone the next failure.

### 5. Repeated container restarts

**Alert:** `CreditRiskPodRestartingFrequently`

**Scenario:** A container restarts more than three times within ten minutes because of application crashes, failed probes, missing configuration or resource exhaustion.

**First response:** examine current and previous container logs:

```bash
kubectl logs -n credit-risk POD_NAME
kubectl logs -n credit-risk POD_NAME --previous
kubectl describe pod -n credit-risk POD_NAME
```

### 6. Unavailable deployment replicas

**Alert:** `CreditRiskDeploymentUnavailable`

**Scenario:** Kubernetes cannot maintain the two desired API replicas because of image-pull failures, failed readiness checks, resource shortages or application startup failures.

**First response:**

```bash
kubectl get deployment,pods -n credit-risk
kubectl describe deployment credit-risk-api -n credit-risk
kubectl get events -n credit-risk --sort-by='.lastTimestamp'
```

## Slack notification flow

Prometheus evaluates the `PrometheusRule` resources continuously. When a condition becomes true, the alert initially enters the `pending` state. If it remains true for the configured `for` duration, it becomes `firing` and is sent to Alertmanager.

Alertmanager groups alerts and matches the label:

```yaml
service: credit-risk-api
```

The matching route sends the notification through a Slack webhook stored in the `alertmanager-slack` Kubernetes Secret. The secret must never be committed to Git. The configured destination is `#mlops-alerts`.

```text
Metric breach
  → PrometheusRule pending
  → PrometheusRule firing
  → Alertmanager grouping and routing
  → Slack webhook
  → #mlops-alerts
```

## Grafana dashboard

The supplied dashboard displays:

- Champion model loaded state.
- Prediction success rate.
- P95 prediction latency.
- Prediction request rate grouped by status.
- Prediction distribution grouped by prediction value.

Grafana queries Prometheus; it does not scrape FastAPI directly. In this design, alert evaluation is also performed by Prometheus rather than Grafana.

## Operational validation

### Verify the API and metrics

```bash
kubectl port-forward -n credit-risk svc/credit-risk-api 8000:80
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/model
curl http://127.0.0.1:8000/metrics
```

### Verify Prometheus discovery

```bash
kubectl get servicemonitor -n credit-risk
kubectl get prometheus -n monitoring
```

Useful PromQL:

```promql
up{namespace="credit-risk"}
```

```promql
credit_risk_model_loaded
```

```promql
sum(rate(credit_risk_requests_total{endpoint="/predict"}[5m])) by (status)
```

### Verify monitoring resources

```bash
kubectl get prometheusrule -n monitoring
kubectl get alertmanagerconfig -n monitoring
kubectl get configmap credit-risk-grafana-dashboard -n monitoring
```

## Troubleshooting approach

I troubleshoot the system layer by layer instead of treating it as one connection:

1. **Workload:** Are the FastAPI and MLflow pods running and ready?
2. **Service:** Do the Services have endpoints?
3. **Application:** Do `/health`, `/model` and `/metrics` return expected results?
4. **Discovery:** Does the ServiceMonitor select the correct labelled Service and named port?
5. **Scraping:** Is the target visible and `UP` in Prometheus?
6. **Query:** Do the raw metrics and PromQL expressions return time series?
7. **Rules:** Are PrometheusRule objects loaded, pending or firing?
8. **Routing:** Does Alertmanager select the alert labels and locate the Slack Secret?
9. **Notification:** Does Slack receive the firing and resolved notifications?

This layered method separates installation failures, Kubernetes discovery problems, empty metrics, incorrect PromQL and notification-routing errors.

## Model monitoring versus service monitoring

This project primarily covers **online serving reliability**:

- Is the model loaded?
- Is the API available?
- Are requests succeeding?
- Is prediction latency acceptable?
- Are pods and replicas stable?

It does not yet prove that live predictions are correct. Production accuracy often cannot be calculated immediately because the real outcome arrives later. A complete model-quality feedback loop would store prediction inputs, output, model version and correlation identifiers; ingest delayed outcomes; calculate performance by model version and cohort; and publish drift and quality metrics for monitoring.

Future model-observability extensions include:

- Input feature drift and missing-value monitoring.
- Prediction distribution drift.
- Data-schema validation.
- Delayed ground-truth accuracy, precision and recall.
- Fairness metrics across approved cohorts.
- Automated retraining triggers with human approval gates.

## Reliability improvements for production

- Define formal availability and latency SLOs with an error budget.
- Use multi-window, multi-burn-rate alerts to reduce noisy paging.
- Protect `/reload-model` with authentication and authorization.
- Store Slack credentials in a managed secret store.
- Add NetworkPolicies between FastAPI, MLflow and monitoring components.
- Persist Prometheus data and define retention requirements.
- Add PodDisruptionBudgets and topology spreading.
- Use an HPA only after selecting a signal correlated with real load.
- Add distributed tracing to separate API, feature-processing and model-inference latency.
- Promote models and application releases through controlled GitOps and progressive-delivery gates.

## Interview explanation

> I built an SRE layer around a Kubernetes-hosted credit-risk inference service. The training workflow records experiments and model artifacts in MLflow, and an approved version receives the champion alias. FastAPI resolves that alias at runtime, loads the model into memory and exposes prediction and operational endpoints. I instrumented the service with Prometheus counters, histograms and model-availability gauges. Prometheus Operator discovers the API through a ServiceMonitor, while Grafana visualises success rate, P95 latency, traffic, prediction distribution and model state. Prometheus rules detect high errors, high latency, model-loading failures, OOM kills, restart loops and unavailable replicas. Alertmanager groups and routes actionable events to Slack. I also defined a layered troubleshooting process from workload and service discovery through PromQL, rule evaluation and notification delivery. The design separates service reliability from model quality and provides a clear extension path for drift and delayed ground-truth monitoring.

## Example STAR answer

**Situation:** A machine-learning model could be trained and served, but the platform did not provide a reliable way to detect model-loading failures, slow predictions, API errors or Kubernetes instability.

**Task:** Design an observable inference platform with measurable reliability indicators, dashboards and actionable alerting while keeping model lifecycle management separate from application deployment.

**Action:** I registered approved model versions in MLflow and used a champion alias so FastAPI could load the approved artifact dynamically. I deployed the API with two Kubernetes replicas, health probes, resource controls and a restricted security context. I instrumented prediction traffic with Prometheus metrics, configured cross-namespace ServiceMonitor discovery, built PromQL rules for errors and P95 latency, added Kubernetes workload alerts, provisioned a Grafana dashboard and routed service-labelled alerts through Alertmanager to Slack. I validated every connection independently, from `/metrics` through Prometheus targets and rule state to Slack delivery.

**Result:** The platform gained end-to-end visibility across model availability, API reliability and Kubernetes health. Failures became measurable and diagnosable, the team could see service behaviour in Grafana, and operational issues could be routed to Slack with enough context to begin incident response.

## Repository structure

```text
credit-risk-sre-demo/
├── app/
│   ├── main.py
│   └── model_loader.py
├── docs/
│   └── credit-risk-sre-architecture.png
├── k8s/
│   ├── app/
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── servicemonitor.yaml
│   │   └── kustomization.yaml
│   └── monitoring/
│       ├── values-kube-prometheus-stack.yaml
│       ├── credit-risk-alerts.yaml
│       ├── alertmanager-config.yaml
│       ├── grafana-dashboard.yaml
│       └── test-alert.yaml
├── scripts/
│   ├── install-monitoring.sh
│   ├── deploy-monitoring-config.sh
│   ├── create-slack-secret.sh
│   ├── deploy-app.sh
│   └── generate-traffic.sh
├── Dockerfile
├── requirements.txt
└── README.md
```

## Key takeaway

Reliable MLOps requires more than a successfully trained model. The model, inference code, Kubernetes workload, metrics pipeline, alert rules and response process form one production system. This project makes those connections explicit and demonstrates how SRE principles can be applied to machine-learning serving workloads.
