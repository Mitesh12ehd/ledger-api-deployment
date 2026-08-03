# ledger-api-deployment

Hardened deployment and secure delivery pipeline for `ledger-api`, a service handling cardholder-adjacent data. This repo covers workload hardening on Kubernetes, policy enforcement with Gatekeeper, secrets management, and a secure CI/CD pipeline to GHCR.

## Stack

- **Cluster:** Minikube
- **Namespace:** `payments`
- **Policy engine:** OPA/Gatekeeper
- **Secrets:** Sealed Secrets
- **Registry:** GHCR (`ghcr.io/<owner>/ledger-api`)
- **Image signing:** Cosign (keyless)
- **Ingress:** NGINX Ingress Controller

## Repository Structure

```
.
├── .github/workflows/
│   └── build.yml                  # CI pipeline: build → push → sign
├── app/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
└── deploy/
    ├── namespace.yaml
    ├── ledger.yaml                # ledger-api: SA, Deployment, Service, Ingress
    ├── neighbour.yaml             # reporting: neighbour service
    ├── rbac.yaml                  # least-privilege Role/RoleBinding
    ├── secret.yaml / docker-secrets.yaml
    ├── sealed-secret.yaml / sealed-docker-secret.yaml
    └── gatekeeper/
        ├── non-root-containers.yaml
        ├── latest-tag-images.yaml
        └── allow-signed-images.yaml
```

## Task 1 — Workload Hardening

**Status: Complete**

`ledger-api` is deployed alongside a `reporting` neighbour service in the `payments` namespace on Minikube.

**Workload security**
- Non-root execution (`runAsNonRoot`, UID/GID `10001`)
- Read-only root filesystem
- All Linux capabilities dropped
- `allowPrivilegeEscalation: false`
- `seccompProfile: RuntimeDefault`
- Dockerfile builds a dedicated non-root user (`appuser`) and drops privileges before `CMD`

**Reliability**
- Resource requests/limits set on `ledger-api`
- Liveness and readiness probes on `/health`

**Identity & access**
- Dedicated ServiceAccount `ledger-api-sa` (default SA not used)
- `automountServiceAccountToken: false`
- RBAC scoped via `rbac.yaml`

**Secrets**
- `STRIPE_API_KEY` and `DB_PASSWORD` injected via Kubernetes Secret, sourced from Sealed Secrets — no plaintext secret material committed to git
- GHCR pull secret also sealed (`sealed-docker-secret.yaml`)

**Admission control (Gatekeeper)**
- `non-root-containers.yaml` — rejects containers not running as non-root
- `latest-tag-images.yaml` — rejects `:latest` tag images
- `allow-signed-images.yaml` — rejects unsigned images

**Neighbour service**
- `reporting`: a `curl`-based service on its own least-privilege ServiceAccount, used to validate in-namespace connectivity and RBAC boundaries.

## Task 2 — CI/CD & Supply Chain

**Status: In progress**

Implemented so far (`.github/workflows/build.yml`):
- GitHub Actions pipeline on GitHub-hosted runners
- Builds the `ledger-api` image and pushes to GHCR
- Signs the pushed image with Cosign (keyless, via GitHub OIDC)

## Future Improvements

**Task 1 — Workload Hardening**
- RBAC personas for developer / operator / admin, each scoped to least privilege
- Pod Security Standards (restricted) enforced at the namespace level
- Demonstration of the admission policy rejecting the original insecure Deployment

**Task 2 — CI/CD & Supply Chain**
- SAST scanning with Semgrep
- Dependency/CVE scanning with Trivy or Grype
- Secrets scanning with gitleaks
- SLSA-style provenance attestation alongside Cosign signing
- SARIF upload so scan results surface in the repo's Security tab
- GitOps with ArgoCD (or Flux) as the source of truth, with drift detection and self-heal
- Canary or blue-green rollout strategy
- Documented fail policy per gate (hard-block vs warn, handling CVEs with no fix yet)

**Task 3 — Service Mesh & Zero-Trust (Istio)**
- Istio installed with `ledger-api` and `reporting` in the mesh
- mTLS STRICT via PeerAuthentication, with a proof that plaintext requests are refused
- Default-deny AuthorizationPolicy with explicit allows keyed on workload identity (SPIFFE / ServiceAccount)
- Kubernetes NetworkPolicy layered underneath for defense-in-depth
- Istio Ingress Gateway with TLS termination, plus a canary release via VirtualService/DestinationRule
- Mesh boundary mapped to the PCI cardholder-data-environment (CDE) scope

**Task 4 — Recon & Penetration Testing**
- Passive OSINT recon of the external attack surface (crt.sh, subfinder, amass, assetfinder, httpx, whatweb, testssl.sh) with an attack-surface report
- Authorized penetration test against the designated target covering OWASP Top 10 classes (IDOR, injection, SSRF, misconfig, secrets exposure, auth/session flaws)
- Professional findings report: executive summary, methodology, CVSS v3.1 scoring, PoC, impact, and remediation per finding
- Chained attack path across two or more findings
- Retest section showing a finding closed after remediation
- Each finding mapped back to the Task 1–3 controls that would have stopped it

All of the above is scoped and buildable on the current setup (Minikube + GHCR + free-tier tooling) — no infrastructure blockers, just remaining implementation time.
