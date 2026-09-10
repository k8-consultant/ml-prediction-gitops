# DevSecOps controls and terminology

## Recommended layered pipeline

| Control | Finds | Suggested tool | Gate |
|---|---|---|---|
| Secrets | Tokens, passwords, private keys | Gitleaks | Every commit; fail on verified secret |
| SAST | Unsafe source-code patterns | SonarQube or Semgrep | Merge request; quality gate |
| SCA | Vulnerable open-source dependencies/licenses | Snyk Open Source or OWASP Dependency-Check | Merge request/build |
| IaC | Terraform/Kubernetes misconfiguration | Checkov (your direct experience) | Before Terraform plan/apply |
| Container | OS and application packages in image | Aqua/Trivy | After build, before push/promotion |
| DAST | Runtime web/API weaknesses | OWASP ZAP baseline/API scan | Against inactive green environment |
| Cloud runtime | Drift, threats, vulnerable deployed images | Security Hub, GuardDuty, Inspector, AWS Config | Continuous |

Do not call Snyk simply a SAST tool: it is a platform with SCA, code, container and IaC capabilities. Do not call Aqua/Trivy SAST: use it primarily for image, filesystem, dependency, secret and configuration scanning. SonarQube is the strongest direct example from your history for code quality/SAST; Checkov is your direct IaC-policy example.

## Interview wording

“At The Very Group I introduced Checkov into Terraform pipelines so misconfigurations and policy violations failed before infrastructure changes were applied. I have also integrated SonarQube, Snyk and Aqua scanning at appropriate stages. I use layered controls: Gitleaks for secrets, SonarQube or Semgrep for SAST, Snyk for dependency risk, Checkov for IaC, Aqua/Trivy for images, and OWASP ZAP against the deployed inactive environment for DAST. Findings are severity-gated with documented exceptions, rather than treating every scanner warning equally.”

## Sensible policy

- Block new critical/high exploitable findings; allow time-bound, owner-approved exceptions.
- Export SARIF/JUnit/HTML reports into GitLab security and pipeline artifacts.
- Avoid duplicated scanners unless they serve different assurance or migration needs.
- Scan dependencies before building, image after building, DAST after deploying to the inactive colour.
- Sign the promoted image with Cosign and generate an SBOM with Syft in a mature implementation.
