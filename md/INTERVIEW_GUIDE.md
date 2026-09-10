# EPAM first-round interview guide

## 60-second introduction

“I am a DevOps and MLOps engineer with production experience across AWS and Azure. At Kainos I supported the DWP Common Risk Engine on AWS using services including Lambda, SQS, API Gateway, KMS and SageMaker, with GitLab CI and least-privilege IAM in an SC-cleared environment. At The Very Group I worked on AWS delivery platforms and implemented Checkov controls for Terraform, alongside pipeline security scanning. At Virtusa, on the Lloyds Banking Group mortgage programme, I worked with Azure, Kubernetes, Azure DevOps, Ansible and blue-green delivery. More recently at Kernlume I have built an EKS-based GitOps/MLOps platform using Terraform, GitHub Actions, Argo CD, Argo Rollouts, MLflow and Prometheus/Grafana. This EPAM role appeals to me because it brings together AWS, infrastructure as code, GitLab and enterprise CI/CD modernisation.”

## Why EPAM and this role?

“EPAM works on complex client transformations rather than a single internal product. I enjoy taking an existing delivery process, understanding its risks and constraints, and incrementally improving automation, security and reliability. The Jenkins-to-GitLab dimension matches my pipeline migration experience, while the AWS and Terraform focus is closely aligned with my delivery history.”

## How I would design the AWS platform

Use a multi-AZ VPC, public subnets only for ingress/NAT, private subnets for EKS nodes and data, ECR for immutable images, EKS managed node groups, ALB ingress, Route 53/ACM/WAF at the edge, RDS Multi-AZ for relational state, Secrets Manager with external-secrets, CloudWatch plus Prometheus/Grafana, and AWS Backup. Terraform is modular, remote state is encrypted/versioned/locked, and GitLab assumes an AWS role with OIDC instead of storing access keys.

## Best GitLab CI blue-green answer

1. Build once, tag the image with the immutable commit SHA, scan it, sign it and push it to ECR.
2. Read which colour the production Service currently selects.
3. Deploy the same image to the inactive colour and wait for readiness.
4. Route a preview endpoint only to the inactive colour; run smoke, integration and ZAP DAST checks.
5. Require a protected production approval.
6. Atomically patch the production Service selector from blue to green (or change ALB target-group weights if gradual exposure is required).
7. Watch error rate, latency and saturation; rollback is the reverse selector switch.
8. Keep the old colour for a defined observation window, then scale it down.

GitLab should orchestrate the process, but Kubernetes/AWS should own traffic routing. For many teams or advanced progressive delivery, use GitLab for CI and Argo Rollouts for the in-cluster state machine; for a straightforward blue-green deployment, two Deployments plus stable/preview Services is transparent and reliable.

## Jenkins to GitLab CI migration

Inventory jobs, shared libraries, credentials, agents, triggers, artifacts and deployment dependencies. Classify pipelines by complexity and business criticality. Build reusable GitLab `include` templates, replace credentials with OIDC/Vault integrations, migrate a low-risk pilot, compare outputs, then dual-run high-risk jobs before cutover. Preserve audit evidence, rollback routes and ownership. Measure lead time, failure rate, recovery time and runner utilisation.

## Likely technical questions

### How do you structure Terraform?

Separate reusable modules from environment composition. Pin providers/modules, use remote state and locking, format/validate/lint/Checkov in merge requests, generate a saved plan, require review, and apply that exact plan only from a protected branch. Avoid workspaces when environments require different access boundaries; separate state is clearer.

### How do you secure AWS access from GitLab?

Configure GitLab as an IAM OIDC identity provider. A job receives an ID token with the expected audience and assumes a narrowly scoped role through STS. Restrict the role trust policy by project, branch/tag and protected environment. This removes static access keys and gives short-lived, auditable sessions.

### How do you handle Terraform drift?

Run scheduled read-only plans, alert on non-empty drift, identify whether the change was emergency/manual or an unmanaged dependency, and reconcile through code. Do not automatically overwrite unknown production drift.

### What happens when deployment health checks fail?

The active Service is unchanged, so customers remain on the old colour. Capture pod events/logs, deployment status and test reports; fix forward or remove the failed inactive release. If failure happens after promotion, switch the Service selector back and investigate.

### How do you avoid database failure during blue-green?

Use backward-compatible expand-and-contract migrations: add compatible schema first, deploy code able to operate with both versions, migrate/backfill, switch traffic, then remove old schema in a later release. Never bundle a destructive schema change with the traffic switch.

## STAR examples to prepare

### Checkov at The Very Group

- Situation: Terraform changes needed consistent security/compliance checks.
- Task: Move misconfiguration detection earlier than deployment review.
- Action: Integrate Checkov in the pipeline, establish severity/policy gates, publish findings, and use reviewed suppressions with reasons where necessary.
- Result: Use your real metric if known. If not, say it reduced late-stage findings and made controls repeatable; do not invent a percentage.

### DWP Common Risk Engine

Explain your AWS services, GitLab pipeline responsibilities, least-privilege/KMS work, collaboration with security, and production outcome. Emphasise the constraints of an SC-cleared government environment without revealing protected information.

### Lloyds blue-green delivery

Explain how parallel application versions, health verification and controlled traffic switching reduced release risk. State clearly that the environment was Azure; then bridge the same deployment principle to EKS/AWS.

## Questions to ask EPAM

1. What is the present Jenkins estate—job count, shared-library complexity and migration deadline?
2. Is the target GitLab platform SaaS, Dedicated or self-managed, and how are runners isolated?
3. Is the AWS landing zone already established, including identity, networking and guardrails?
4. What workloads are moving—EC2, containers/EKS, serverless, Windows, or a mixture?
5. How will success be measured in the first 90 days?

## Accuracy boundaries

Say “implemented” only for Checkov and the scanners/pipeline work you personally performed. Describe the supplied EKS/GitLab design as “the implementation I prepared for this interview” unless you deploy it. Keep Azure blue-green experience distinct from the proposed AWS implementation.
