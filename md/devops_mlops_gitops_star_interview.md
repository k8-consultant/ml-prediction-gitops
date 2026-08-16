# DevOps/MLOps GitOps & Progressive Delivery --- STAR Interview Notes

## Interview Question

**Can you describe a CI/CD or GitOps solution you implemented in a
previous environment?**

## Situation

In one of the environments I supported as a DevOps/MLOps Engineer, we
had containerised Python-based services, including APIs used to expose
machine-learning predictions.

The existing release process had too much dependency on engineers
manually deploying or modifying workloads in Kubernetes. That created
operational concerns around deployment consistency, configuration drift,
release traceability, and the level of Kubernetes access required by the
CI system.

We wanted to move towards a GitOps operating model where application
teams could push code normally, while deployment state remained
controlled, auditable, and reproducible from Git.

## Task

My responsibility was to help design and implement the CI/CD and
Kubernetes deployment workflow.

I was responsible for the pipeline from source-code validation through
container creation and security scanning, and for integrating that
process with our GitOps deployment model.

The main objective was to establish a clear separation between **CI and
CD**:

-   **GitHub Actions** would validate and package the application.
-   **Argo CD** would reconcile the desired deployment state into
    Kubernetes.
-   **Argo Rollouts** would provide progressive delivery for workloads
    requiring safer releases.

## Action

### 1. CI Pipeline with GitHub Actions

I developed and maintained GitHub Actions workflows triggered by
application changes.

The pipeline:

1.  Checked out the source code.
2.  Provisioned the required Python environment.
3.  Installed application dependencies.
4.  Executed automated tests using `pytest`.
5.  Built the application into a Docker image.
6.  Scanned the container image with Trivy.
7.  Published approved images to the container registry.

I configured dependencies between pipeline stages so downstream jobs
could not proceed when tests failed.

### 2. Immutable Image Tagging and Traceability

Rather than relying on mutable `latest` tags, I tagged container images
using the Git commit SHA.

This gave us a traceable release chain:

``` text
Git commit
    ↓
Docker image
    ↓
GitOps manifest
    ↓
Kubernetes workload
```

If we identified a running image in Kubernetes, we could trace it back
to the source-code revision that produced it.

### 3. Container Security Scanning

I integrated Trivy into the CI workflow to scan container images for
known vulnerabilities before promotion.

This introduced a security gate between image creation and
publication/deployment and allowed security findings to become part of
the normal CI/CD workflow.

### 4. GitOps with Argo CD

On the deployment side, I maintained a separate GitOps repository
containing the Kubernetes desired state.

We separated application source code from deployment configuration:

``` text
Application Repository
        │
        ├── Python/FastAPI
        ├── Tests
        ├── Dockerfile
        └── GitHub Actions

GitOps Repository
        │
        ├── Kubernetes manifests
        ├── Environment configuration
        └── Argo Rollout definitions
```

Argo CD continuously monitored the GitOps repository and reconciled the
desired state against the Kubernetes cluster.

The deployment flow became:

``` text
Application change
        ↓
GitHub Actions
        ↓
Container Registry
        ↓
GitOps change
        ↓
Argo CD
        ↓
Kubernetes
```

GitHub Actions therefore did not need to perform `kubectl apply`
directly against production clusters.

### 5. Configuration Drift and Self-Healing

I configured Argo CD automated synchronization, pruning, and
self-healing.

Git became the source of truth.

For example, if Git specified five replicas but someone manually scaled
the deployment to eight replicas, Argo CD detected that the live cluster
no longer matched the desired state and reconciled it back to five.

``` text
Git desired state:       replicas = 5
                              │
                              │ compare
                              ▼
Kubernetes live state:   replicas = 8
                              │
                         DRIFT DETECTED
                              │
                              ▼
                         Argo CD reconcile
                              │
                              ▼
Kubernetes live state:   replicas = 5
```

This prevented undocumented manual changes from becoming permanent
cluster configuration.

### 6. Progressive Delivery with Argo Rollouts

For services requiring greater release control, I worked with Argo
Rollouts rather than relying solely on the standard Kubernetes
Deployment rolling-update strategy.

I defined Rollout resources with canary strategies that progressively
introduced new application versions.

For example:

``` text
Stable v1     Canary v2
   80%           20%
                  ↓
   60%           40%
                  ↓
   40%           60%
                  ↓
    0%          100%
```

This allowed us to maintain a stable ReplicaSet alongside the new canary
ReplicaSet while a release progressed.

I used the Argo Rollouts CLI and Argo CD UI to inspect rollout status,
ReplicaSets, pods, image revisions, synchronization state, and
application health.

I also worked with promotion and abort mechanisms so unsuccessful
releases could be stopped instead of blindly continuing through
deployment.

### 7. Automated Image Promotion

Another responsibility was removing the manual hand-off between CI and
GitOps.

Initially, after CI produced a new container image, an engineer had to
update the image SHA in the GitOps repository.

We automated that process.

After testing, scanning, and publishing the image, the pipeline updated
the relevant GitOps manifest with the new immutable image SHA and
committed the change back to the GitOps repository.

Argo CD detected the new Git commit and initiated reconciliation
automatically.

The complete flow became:

``` text
Developer commit
        ↓
GitHub Actions
        ↓
Automated testing
        ↓
Docker build
        ↓
Trivy security scan
        ↓
Container registry
        ↓
GitOps repository update
        ↓
Argo CD reconciliation
        ↓
Argo Rollouts
        ↓
Kubernetes
```

The key architectural point was that CI still did not deploy directly
into Kubernetes. CI changed the desired state in Git, and Argo
controlled cluster-side deployment.

## Result

The result was a more controlled and traceable deployment process.

We:

-   Reduced manual Kubernetes deployment activity.
-   Established Git as the source of truth for deployment configuration.
-   Improved traceability between source commits, container images,
    GitOps changes, and running workloads.
-   Reduced the Kubernetes privileges required by CI runners.
-   Introduced configuration-drift detection and self-healing.
-   Established progressive delivery through Argo Rollouts.
-   Improved troubleshooting by making it easier to identify exactly
    what version was running and where it originated.

------------------------------------------------------------------------

## 90-Second Interview Version

> In one environment I supported, I was responsible for implementing and
> maintaining part of our containerised application delivery platform
> around Kubernetes.
>
> One of the problems we wanted to address was the amount of manual
> deployment activity and the fact that CI systems deploying directly
> with kubectl can create both security and configuration-drift
> problems.
>
> I implemented a clear separation between CI and CD. On the CI side, I
> used GitHub Actions to run pytest, build Docker images, perform
> vulnerability scanning with Trivy, and publish successful images to
> our container registry. Images were tagged using the Git commit SHA
> rather than relying on `latest`, which gave us end-to-end release
> traceability.
>
> On the CD side, we maintained a separate GitOps repository containing
> the Kubernetes desired state. Argo CD monitored that repository and
> reconciled changes into the cluster, so GitHub Actions didn't need to
> deploy directly to Kubernetes.
>
> I also configured automated sync, pruning, and self-healing, meaning
> that if somebody manually changed something like the replica count in
> the cluster, Argo detected the configuration drift and reconciled it
> back to what was defined in Git.
>
> For safer application releases, I worked with Argo Rollouts to
> implement canary deployment strategies so a new version could be
> progressively introduced rather than replacing the stable version
> immediately.
>
> We then automated the hand-off between CI and GitOps: once a new image
> passed testing and security checks, the pipeline updated the GitOps
> repository with the new image SHA. Argo CD detected that change and
> Argo Rollouts controlled the progressive deployment.
>
> The result was a much more reproducible and auditable deployment
> process, with less manual Kubernetes intervention, better release
> traceability, and a clear separation between build responsibilities
> and production deployment control.

------------------------------------------------------------------------

## Technical Responsibilities Summary

**CI/CD** - Designed and maintained GitHub Actions workflows. -
Implemented automated Python testing with pytest. - Built and tagged
Docker images using immutable Git SHAs. - Published images to a
container registry. - Integrated Trivy vulnerability scanning into CI. -
Configured pipeline dependencies and quality gates.

**GitOps** - Maintained separate application and GitOps repositories. -
Managed Kubernetes desired state through Git. - Configured Argo CD
applications and automated synchronization. - Implemented pruning and
self-healing. - Investigated synchronization and configuration-drift
issues.

**Kubernetes** - Managed Deployments, Services, ReplicaSets, Pods,
probes, namespaces, and container configuration. - Troubleshot
application and deployment state using `kubectl`. - Validated
application health and deployment behaviour.

**Progressive Delivery** - Implemented Argo Rollouts resources. -
Configured canary deployment strategies and traffic weights. - Monitored
stable and canary ReplicaSets. - Used rollout promotion and abort
mechanisms. - Used Argo CD and Argo Rollouts tooling to troubleshoot
releases.

**Security and Governance** - Reduced the need for CI runners to hold
direct cluster-deployment credentials. - Used immutable image versions
for release traceability. - Integrated container vulnerability
scanning. - Kept Git as the auditable source of truth for deployment
state.

------------------------------------------------------------------------

## Architecture Summary

``` text
Developer
    │
    │ git push
    ▼
Application Repository
    │
    ▼
GitHub Actions
    │
    ├── pytest
    ├── Docker build
    ├── Trivy scan
    └── Push image
            │
            ▼
     Container Registry
            │
            ▼
     GitOps Repository
            │
            ▼
         Argo CD
            │
            ▼
      Argo Rollouts
            │
       ┌────┴────┐
       ▼         ▼
    Stable     Canary
            │
            ▼
        Kubernetes
```

## Interview Follow-Up: Why Not Deploy Directly from GitHub Actions?

The main reasons were:

-   **Git as the source of truth** --- the desired deployment state
    remained version controlled.
-   **Reduced cluster credentials in CI** --- CI did not require broad
    Kubernetes deployment access.
-   **Auditability** --- deployment changes could be traced through Git
    history.
-   **Drift detection** --- Argo CD could detect differences between Git
    and the live cluster.
-   **Self-healing** --- unauthorised or accidental manual changes could
    be automatically reconciled.
-   **Separation of concerns** --- CI built and validated artifacts; the
    GitOps controller handled deployment reconciliation.

## Interview Follow-Up: DevOps/MLOps vs Developer Responsibility

Application developers primarily owned application functionality and
application-level tests.

My DevOps/MLOps/platform responsibilities covered:

-   CI workflow design and maintenance.
-   Containerisation standards.
-   Container image lifecycle and security scanning.
-   GitOps deployment manifests.
-   Argo CD configuration and reconciliation.
-   Kubernetes deployment behaviour and troubleshooting.
-   Progressive-delivery configuration with Argo Rollouts.
-   Release traceability and operational troubleshooting.
