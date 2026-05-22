# DevOps Phase 11 — Security & DevSecOps
## Secrets Management · TLS/SSL · Container Security · IAM · Vulnerability Scanning

---

> **Who this is for:** Beginners who want to understand how security is embedded into every layer of DevOps — from code to production infrastructure.

---

## Table of Contents

1. [What is DevSecOps?](#1-what-is-devsecops)
2. [Secrets Management](#2-secrets-management)
3. [TLS/SSL Deep Dive](#3-tlsssl-deep-dive)
4. [Container Security](#4-container-security)
5. [IAM — Identity & Access Management](#5-iam--identity--access-management)
6. [Vulnerability Scanning](#6-vulnerability-scanning)
7. [Network Security](#7-network-security)
8. [Security in CI/CD Pipelines](#8-security-in-cicd-pipelines)
9. [Compliance & Audit](#9-compliance--audit)
10. [Interview Mastery](#10-interview-mastery)

---

## 1. What is DevSecOps?

### Beginner Explanation

Traditional approach: Developers build → Operations deploy → Security team checks at the end.

**Problem:** Security issues found at the end are expensive to fix, delay releases, and often get ignored under pressure.

DevSecOps means: **Security is everyone's responsibility, embedded at every stage.**

Think of building a house:
- Old way: Build the entire house, then hire a security consultant to check if the locks and windows are secure. Expensive to retrofit.
- DevSecOps way: Architect the house with security from day one — reinforced doors, alarm wiring in the walls, security cameras planned into the blueprint.

### The DevSecOps Lifecycle

```mermaid
flowchart LR
    A[Plan] -->|Threat Modeling| B[Code]
    B -->|SAST, Secrets Scanning| C[Build]
    C -->|SCA, Image Scanning| D[Test]
    D -->|DAST, Pen Testing| E[Deploy]
    E -->|IaC Security, Compliance| F[Operate]
    F -->|Runtime Protection, Monitoring| G[Monitor]
    G -->|Incident Response, Feedback| A
```

### Key Principles

| Principle | Meaning |
|---|---|
| Shift Left | Find security issues early (in code, not production) |
| Automate | Security checks in every CI/CD pipeline run |
| Least Privilege | Give minimum access required for a task |
| Defense in Depth | Multiple layers of security (not just firewalls) |
| Zero Trust | Never trust, always verify — even internal traffic |
| Immutable Infrastructure | Don't patch servers; rebuild them from secure images |

### Shift Left — Finding Bugs Earlier Saves Money

```
Cost to fix a security bug:

Design Phase:    $500
Development:     $5,000
Testing/QA:      $15,000
Production:      $100,000+   ← Data breach, reputation damage, legal

DevSecOps goal: Catch it at $500, not $100,000.
```

---

## 2. Secrets Management

### Beginner Explanation

Secrets are **sensitive data your applications need to function**: database passwords, API keys, encryption keys, certificates, tokens.

The #1 security mistake in DevOps: **Hardcoding secrets in code or config files that end up in Git.**

```
❌ NEVER DO THIS:
DATABASE_URL=postgres://admin:SuperSecret123@db.example.com:5432/prod
AWS_SECRET_KEY=AKIAIOSFODNN7EXAMPLE

These end up in git history FOREVER, even if you delete them.
```

### The Problem — How Secrets Leak

```mermaid
flowchart TD
    A[Developer hardcodes\nDB password in code] --> B[Pushes to GitHub]
    B --> C[Automated bots scan\nGitHub within minutes]
    C --> D[Attacker gains\naccess to database]
    D --> E[Data breach\n$$$ + reputation loss]
    
    F[.env file committed\naccidentally] --> B
    G[Docker image contains\nAPI keys in ENV] --> H[Image pushed to\npublic registry]
    H --> D
```

### Solution 1: Environment Variables (Basic)

```bash
# Store secrets in environment, not code
export DATABASE_URL="postgres://admin:secret@db:5432/prod"
export API_KEY="sk-live-abc123"

# Application reads from environment
# Python:
import os
db_url = os.environ["DATABASE_URL"]

# Node.js:
const dbUrl = process.env.DATABASE_URL;
```

**Limitations:** Still visible in process listings, not encrypted at rest, not auditable.

### Solution 2: HashiCorp Vault (Production-Grade)

Vault is the industry standard for secrets management. It provides:
- Centralized secret storage with encryption
- Dynamic secrets (generated on-demand, auto-expire)
- Fine-grained access control
- Full audit log of who accessed what

```mermaid
flowchart LR
    A[Application] -->|Authenticate| B[Vault]
    B -->|Verify identity| C{Auth Method}
    C -->|Kubernetes SA| D[K8s Auth]
    C -->|AWS IAM Role| E[AWS Auth]
    C -->|Token| F[Token Auth]
    B -->|Return secret| A
    B --> G[(Encrypted Storage\nConsul / Raft)]
    B --> H[Audit Log]
```

**Install and configure Vault:**
```bash
# Install
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install vault

# Start dev server (for learning only!)
vault server -dev

# Set address
export VAULT_ADDR='http://127.0.0.1:8200'

# Write a secret
vault kv put secret/myapp/database \
  username="admin" \
  password="s3cr3tP@ss" \
  host="db.example.com"

# Read a secret
vault kv get secret/myapp/database
vault kv get -field=password secret/myapp/database

# List secrets
vault kv list secret/myapp/
```

**Vault policies (access control):**
```hcl
# policy: app-readonly.hcl
# Only allows reading database secrets — cannot write or delete
path "secret/data/myapp/database" {
  capabilities = ["read"]
}

path "secret/data/myapp/api-keys" {
  capabilities = ["read"]
}

# Deny access to everything else
path "secret/*" {
  capabilities = ["deny"]
}
```

**Apply policy:**
```bash
vault policy write app-readonly app-readonly.hcl
vault token create -policy="app-readonly" -ttl=1h
```

**Dynamic secrets (database credentials):**
```bash
# Configure database secrets engine
vault secrets enable database

vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/prod" \
  allowed_roles="app-role" \
  username="vault-admin" \
  password="vault-admin-pass"

# Create a role that generates temporary credentials
vault write database/roles/app-role \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Application requests credentials (auto-generated, auto-expire!)
vault read database/creds/app-role
# Returns: username=v-app-role-abc123, password=random-generated, lease_duration=1h
```

### Solution 3: Kubernetes Secrets

```yaml
# Create a secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=            # base64 encoded "admin"
  password: czNjcjN0UEBzcw==   # base64 encoded "s3cr3tP@ss"
---
# Use secret in a pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    # Or mount as files:
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: db-credentials
```

**Important:** Kubernetes Secrets are only base64-encoded, NOT encrypted by default. Enable encryption at rest:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

### Solution 4: AWS Secrets Manager / SSM Parameter Store

```bash
# AWS Secrets Manager — store a secret
aws secretsmanager create-secret \
  --name "prod/database/credentials" \
  --secret-string '{"username":"admin","password":"s3cr3t"}'

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id "prod/database/credentials" \
  --query 'SecretString' --output text

# Automatic rotation (Lambda function rotates password periodically)
aws secretsmanager rotate-secret \
  --secret-id "prod/database/credentials" \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456:function:rotate-db

# SSM Parameter Store — cheaper, simpler
aws ssm put-parameter \
  --name "/prod/database/password" \
  --value "s3cr3t" \
  --type "SecureString" \
  --key-id "alias/my-kms-key"

aws ssm get-parameter \
  --name "/prod/database/password" \
  --with-decryption
```

### Secret Scanning — Prevent Leaks Before They Happen

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scanning
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0   # Full history for scanning

    - name: TruffleHog (scan for leaked secrets)
      uses: trufflesecurity/trufflehog@main
      with:
        extra_args: --only-verified

    - name: GitLeaks
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Pre-commit hook (prevent secrets from even being committed):**
```bash
# Install pre-commit
pip install pre-commit

# .pre-commit-config.yaml
repos:
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.18.0
  hooks:
  - id: gitleaks

# Install hooks
pre-commit install
```

**If a secret leaks:**
```bash
# 1. IMMEDIATELY rotate the compromised secret
# 2. Revoke old credentials
# 3. Remove from git history (BFG is faster than filter-branch)
bfg --replace-text passwords.txt repo.git
git reflog expire --expire=now --all && git gc --prune=now --aggressive
# 4. Force push (coordinate with team)
# 5. Audit access logs for unauthorized use
```

---

## 3. TLS/SSL Deep Dive

### Beginner Explanation

TLS (Transport Layer Security) is the technology that puts the "S" in HTTPS. It provides:
1. **Encryption** — Data is scrambled so eavesdroppers can't read it
2. **Authentication** — Proves the server is who it claims to be (via certificate)
3. **Integrity** — Detects if data was tampered with in transit

**SSL vs TLS:** SSL is the old name (SSL 1.0, 2.0, 3.0 — all deprecated). TLS is the modern successor (1.0, 1.1, 1.2, 1.3). People still say "SSL" but mean TLS. Only TLS 1.2 and 1.3 should be used today.

### Certificate Types

| Type | Validation | Cost | Use Case |
|---|---|---|---|
| DV (Domain Validated) | Proves you own the domain | Free (Let's Encrypt) | Most websites |
| OV (Organization Validated) | Proves organization identity | $$ | Business sites |
| EV (Extended Validation) | Full legal verification | $$$ | Banking, e-commerce |
| Wildcard | `*.example.com` | $$ | Multiple subdomains |
| SAN (Multi-domain) | Multiple specific domains | $$ | CDNs, multi-tenant |

### Certificate Chain Explained

```
Root Certificate Authority (CA)
│   (pre-installed in browsers/OS — "trust anchors")
│   e.g., DigiCert, Let's Encrypt ISRG Root X1
│
├── Intermediate CA Certificate
│   (signs end-entity certificates)
│   e.g., Let's Encrypt R3
│
└── End-Entity Certificate (your cert)
    (signed by intermediate)
    e.g., example.com

Browser verification:
1. Server sends: End-entity cert + Intermediate cert
2. Browser has: Root cert (pre-installed)
3. Browser verifies: Root → signed Intermediate → signed Your Cert ✓
```

### mTLS — Mutual TLS (Zero Trust)

Standard TLS: Only the server proves its identity.
mTLS: **Both** client and server prove their identity.

```mermaid
sequenceDiagram
    participant Client as Service A
    participant Server as Service B

    Client->>Server: ClientHello
    Server->>Client: ServerHello + Server Certificate
    Client->>Client: Verify server cert
    Server->>Client: Request client certificate
    Client->>Server: Client Certificate
    Server->>Server: Verify client cert
    Note over Client,Server: Both identities verified — encrypted channel established
    Client->>Server: Encrypted request
    Server->>Client: Encrypted response
```

**mTLS in Kubernetes with Istio (service mesh):**
```yaml
# PeerAuthentication — require mTLS for all services in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # Reject any non-mTLS traffic
```

**Use cases for mTLS:**
- Service-to-service communication in microservices
- Zero Trust networks (don't trust internal traffic)
- API authentication without tokens
- IoT device authentication

### Certificate Management

```bash
# Generate self-signed cert (dev/testing only)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -subj "/CN=localhost"

# Generate CSR (Certificate Signing Request) for CA signing
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr \
  -subj "/C=US/ST=CA/O=MyCompany/CN=example.com"

# View certificate details
openssl x509 -in server.crt -text -noout

# Check certificate expiry
openssl x509 -in server.crt -noout -enddate

# Verify certificate chain
openssl verify -CAfile ca-bundle.crt server.crt

# Test remote server's certificate
openssl s_client -connect example.com:443 -servername example.com

# Check TLS version and cipher
openssl s_client -connect example.com:443 -tls1_3
```

**Cert-Manager in Kubernetes (automated certificate lifecycle):**
```yaml
# Install cert-manager
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# ClusterIssuer — uses Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Certificate resource — cert-manager handles renewal automatically
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: production
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
  renewBefore: 720h  # Renew 30 days before expiry
```

---

## 4. Container Security

### Beginner Explanation

Containers (Docker) are lightweight, but they introduce security challenges:
- Images can contain vulnerabilities in their base packages
- Containers can run as root (dangerous)
- Images from public registries might be malicious
- Container breakout is possible if misconfigured

### Secure Dockerfile Practices

```dockerfile
# ❌ BAD Dockerfile
FROM ubuntu:latest       # "latest" is unpinnable — could change
RUN apt-get update && apt-get install -y curl python3
COPY . /app              # Copies everything including .env, .git
RUN pip install -r requirements.txt
USER root               # Running as root is dangerous
CMD ["python3", "app.py"]

# ✅ GOOD Dockerfile
# Pin exact version for reproducibility
FROM python:3.11-slim-bookworm AS builder

WORKDIR /app

# Install dependencies first (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.11-slim-bookworm

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy only what's needed from builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser ./src /app/src

# No secrets in image
# No unnecessary packages
# No package caches

USER appuser

# Read-only filesystem where possible
# Healthcheck for orchestrators
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

CMD ["python3", "src/app.py"]
```

### Docker Security Scanning

```bash
# Scan image for vulnerabilities with Trivy (most popular free scanner)
trivy image myapp:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan a Dockerfile for misconfigurations
trivy config Dockerfile

# Docker Scout (Docker's built-in scanner)
docker scout cves myapp:latest
docker scout recommendations myapp:latest

# Scan in CI pipeline (fail build on critical vulns)
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

### Container Runtime Security

```yaml
# Kubernetes Pod Security — restrict what containers can do
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true           # Must not run as root
    runAsUser: 1000              # Specific UID
    runAsGroup: 3000             # Specific GID
    fsGroup: 2000               # Files created owned by this group
    seccompProfile:
      type: RuntimeDefault       # Default syscall filter
  containers:
  - name: app
    image: myapp:v1.2.3          # Pin exact version
    securityContext:
      allowPrivilegeEscalation: false  # Cannot sudo/setuid
      readOnlyRootFilesystem: true     # Cannot write to filesystem
      capabilities:
        drop:
          - ALL                  # Drop all Linux capabilities
        add:
          - NET_BIND_SERVICE     # Only add what's needed
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"          # Prevent resource exhaustion
      requests:
        cpu: "100m"
        memory: "128Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp            # Writable tmp (since root fs is read-only)
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

### Pod Security Standards (PSS)

Kubernetes defines three security levels:

```yaml
# Enforce restricted (highest security) on a namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

| Level | What it prevents |
|---|---|
| Privileged | Nothing (unrestricted) |
| Baseline | Blocks known privilege escalations (hostNetwork, hostPID, privileged containers) |
| Restricted | Full lockdown — non-root, no privilege escalation, read-only root FS, minimal capabilities |

### Image Supply Chain Security

```mermaid
flowchart LR
    A[Developer writes\nDockerfile] --> B[Build image in CI]
    B --> C[Scan for vulnerabilities\nTrivy / Snyk]
    C -->|Pass| D[Sign image\nCosign / Notation]
    C -->|Fail| X[Block deployment]
    D --> E[Push to private\nregistry]
    E --> F[Admission controller\nverifies signature]
    F -->|Signed| G[Deploy to K8s]
    F -->|Unsigned| X
```

**Sign and verify images with Cosign:**
```bash
# Generate key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry.io/myapp:v1.2.3

# Verify signature before deployment
cosign verify --key cosign.pub myregistry.io/myapp:v1.2.3

# In Kubernetes — use Kyverno or OPA Gatekeeper to enforce signatures
```

**Kyverno policy to enforce image signing:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "myregistry.io/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKo...
              -----END PUBLIC KEY-----
```

---

## 5. IAM — Identity & Access Management

### Beginner Explanation

IAM answers three questions:
1. **Authentication (AuthN):** WHO are you? (Prove your identity)
2. **Authorization (AuthZ):** WHAT can you do? (What permissions do you have)
3. **Audit:** WHAT did you do? (Tracking for accountability)

Think of a building:
- Authentication = Your ID badge (proves who you are)
- Authorization = Your badge only opens certain doors (your access level)
- Audit = Security cameras (record what you did)

### Principle of Least Privilege

```
❌ WRONG:
"Give the app admin access so it works"
→ If compromised, attacker has full admin access to everything

✅ RIGHT:
"Give the app ONLY the specific permissions it needs"
→ If compromised, blast radius is contained
```

```mermaid
flowchart TD
    A[App needs to\nread from S3 bucket] --> B{Least Privilege}
    B -->|❌ Overprivileged| C[s3:* on *\nFull S3 admin on all buckets]
    B -->|✅ Least Privilege| D[s3:GetObject on\narn:aws:s3:::my-bucket/*]
```

### AWS IAM

```json
// IAM Policy — allow specific actions on specific resources
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    },
    {
      "Sid": "DenyDeleteAnything",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

**IAM Roles for Services (no hardcoded keys!):**
```json
// Trust policy: who can assume this role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# Create role
aws iam create-role --role-name MyAppRole \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions
aws iam attach-role-policy --role-name MyAppRole \
  --policy-arn arn:aws:iam::123456789:policy/MyAppPolicy

# Attach to EC2 instance profile
aws iam create-instance-profile --instance-profile-name MyAppProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name MyAppProfile \
  --role-name MyAppRole
```

### Kubernetes RBAC (Role-Based Access Control)

```yaml
# Role — defines what actions are allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: []           # NO access to secrets

---
# RoleBinding — assigns role to a user/service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: production
roleRef:
  kind: Role
  name: app-deployer
  apiGroup: rbac.authorization.k8s.io

---
# ServiceAccount for CI/CD pipeline
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
  annotations:
    # AWS IRSA (IAM Role for Service Account)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/ci-deployer-role
```

**ClusterRole vs Role:**
```
Role       = permissions within ONE namespace
ClusterRole = permissions cluster-wide (all namespaces)

Use Role for: application workloads
Use ClusterRole for: cluster admins, monitoring tools, ingress controllers
```

### OIDC and SSO for DevOps

```mermaid
flowchart LR
    A[Developer] -->|Login| B[Identity Provider\nOkta / Azure AD / Google]
    B -->|OIDC Token| A
    A -->|Token| C[kubectl]
    C -->|Token verification| D[K8s API Server]
    D -->|Check RBAC| E[Authorize/Deny]
    
    A -->|SSO Login| F[AWS Console]
    A -->|SSO Login| G[Vault UI]
    A -->|SSO Login| H[Grafana]
```

---

## 6. Vulnerability Scanning

### Types of Security Scanning

```mermaid
flowchart TD
    A[Security Scanning Types] --> B[SAST\nStatic Application\nSecurity Testing]
    A --> C[DAST\nDynamic Application\nSecurity Testing]
    A --> D[SCA\nSoftware Composition\nAnalysis]
    A --> E[Container\nImage Scanning]
    A --> F[IaC\nScanning]
    
    B --> B1[Scans source code\nfor vulnerabilities\nwithout running it]
    C --> C1[Tests running application\nfor vulnerabilities\nlike a hacker would]
    D --> D1[Checks third-party\ndependencies for\nknown CVEs]
    E --> E1[Scans Docker images\nfor vulnerable\npackages]
    F --> F1[Scans Terraform/K8s\nfor misconfigurations]
```

### SAST — Static Analysis

```yaml
# GitHub Actions with Semgrep (popular free SAST tool)
name: SAST
on: [pull_request]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
    - uses: actions/checkout@v4
    - run: semgrep scan --config auto --error
      # --config auto uses recommended rules for detected languages
```

**Common SAST findings:**
```python
# SQL Injection — SAST catches this
query = f"SELECT * FROM users WHERE id = {user_input}"  # ❌ VULNERABLE

# Fix:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))  # ✅ Parameterized

# Command Injection
os.system(f"ping {user_input}")  # ❌ VULNERABLE (user can input "; rm -rf /")

# Fix:
subprocess.run(["ping", user_input], check=True)  # ✅ No shell interpretation
```

### SCA — Dependency Scanning

```bash
# Snyk — check for vulnerable dependencies
snyk test                    # Scan current project
snyk monitor                 # Continuous monitoring

# npm audit
npm audit
npm audit fix

# pip-audit (Python)
pip install pip-audit
pip-audit

# OWASP Dependency Check
dependency-check --project "MyApp" --scan ./
```

```yaml
# Dependabot config (.github/dependabot.yml)
version: 2
updates:
- package-ecosystem: "npm"
  directory: "/"
  schedule:
    interval: "weekly"
  open-pull-requests-limit: 10
  
- package-ecosystem: "docker"
  directory: "/"
  schedule:
    interval: "weekly"

- package-ecosystem: "github-actions"
  directory: "/"
  schedule:
    interval: "weekly"
```

### Container Image Scanning

```bash
# Trivy — comprehensive scanner
# Scan image
trivy image nginx:latest

# Scan filesystem
trivy fs --security-checks vuln,config .

# Scan Kubernetes cluster
trivy k8s --report summary cluster

# Output as JSON (for CI parsing)
trivy image --format json --output results.json myapp:latest

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed --severity HIGH,CRITICAL myapp:latest
```

**Sample Trivy output:**
```
myapp:latest (debian 12.1)
Total: 15 (HIGH: 12, CRITICAL: 3)

┌─────────────────────┬──────────────────┬──────────┬─────────────────────┐
│      Library        │  Vulnerability   │ Severity │    Fixed Version    │
├─────────────────────┼──────────────────┼──────────┼─────────────────────┤
│ openssl             │ CVE-2023-5678    │ CRITICAL │ 3.0.12-1            │
│ curl                │ CVE-2023-1234    │ HIGH     │ 7.88.1-10+deb12u4   │
│ libc6               │ CVE-2023-9999    │ HIGH     │ 2.36-9+deb12u3      │
└─────────────────────┴──────────────────┴──────────┴─────────────────────┘
```

### IaC Security Scanning

```bash
# Checkov — scan Terraform, Kubernetes, Dockerfiles
pip install checkov
checkov -d .                         # Scan current directory
checkov -f main.tf                   # Scan specific file
checkov --framework terraform -d .   # Terraform only

# tfsec (Terraform specific)
tfsec .

# kube-score (Kubernetes manifest quality)
kube-score score deployment.yaml
```

**What these tools catch:**
```hcl
# ❌ Checkov finds: S3 bucket without encryption
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  # Missing: server_side_encryption_configuration
  # Missing: versioning
  # Missing: public_access_block
}

# ✅ Fixed:
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## 7. Network Security

### Firewalls and Security Groups

```mermaid
flowchart TB
    subgraph VPC ["AWS VPC (10.0.0.0/16)"]
        subgraph Public ["Public Subnet"]
            A[ALB]
        end
        subgraph Private ["Private Subnet"]
            B[App Servers]
            C[Database]
        end
    end
    
    D[Internet] -->|Only ports 80/443| A
    A -->|Only port 8080| B
    B -->|Only port 5432| C
    C -.->|Cannot reach internet| D
```

**AWS Security Groups (stateful firewall):**
```hcl
# ALB Security Group — only allow HTTP/HTTPS from internet
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# App Security Group — only allow traffic FROM ALB
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only from ALB
  }
}

# Database Security Group — only allow traffic FROM app servers
resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Only from app
  }
  
  # No egress to internet
}
```

### Kubernetes Network Policies

```yaml
# Default deny ALL traffic in namespace (start with zero trust)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}    # Applies to ALL pods
  policyTypes:
  - Ingress
  - Egress

---
# Allow frontend to talk to API only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Allow API to reach database only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:   # Allow DNS lookups
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### WAF — Web Application Firewall

```
WAF sits in front of your application and blocks common attacks:

Request → WAF → (Block/Allow) → Application

What a WAF blocks:
- SQL injection attempts
- Cross-site scripting (XSS)
- Path traversal (../../etc/passwd)
- Known exploit payloads
- Bot traffic
- Rate limit violations
- Geographic blocking
```

**AWS WAF rules (Terraform):**
```hcl
resource "aws_wafv2_web_acl" "main" {
  name  = "production-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Block SQL injection
  rule {
    name     = "SQLInjectionRule"
    priority = 1
    action {
      block {}
    }
    statement {
      sqli_match_statement {
        field_to_match {
          body {}
        }
        text_transformation {
          priority = 1
          type     = "URL_DECODE"
        }
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLInjection"
    }
  }

  # AWS Managed Rules (community rules)
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 2
    override_action {
      none {}
    }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSCommonRules"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWAF"
  }
}
```

---

## 8. Security in CI/CD Pipelines

### The Secure Pipeline

```mermaid
flowchart LR
    A[Code Push] --> B[Secret Scan\nGitleaks]
    B --> C[SAST\nSemgrep]
    C --> D[Build Image]
    D --> E[Image Scan\nTrivy]
    E --> F[SCA\nSnyk/Dependabot]
    F --> G[IaC Scan\nCheckov]
    G --> H[DAST\nOWASP ZAP]
    H --> I{All Passed?}
    I -->|Yes| J[Deploy to Staging]
    I -->|No| K[Block + Alert]
    J --> L[Smoke Tests]
    L --> M[Deploy to Prod]
```

### Complete Secure CI/CD Pipeline

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    - uses: gitleaks/gitleaks-action@v2

  sast:
    runs-on: ubuntu-latest
    needs: secret-scan
    steps:
    - uses: actions/checkout@v4
    - name: Semgrep SAST
      uses: semgrep/semgrep-action@v1
      with:
        config: p/owasp-top-ten

  build-and-scan:
    runs-on: ubuntu-latest
    needs: sast
    steps:
    - uses: actions/checkout@v4
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Trivy image scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'HIGH,CRITICAL'
        exit-code: '1'
    
    - name: Upload scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  dependency-check:
    runs-on: ubuntu-latest
    needs: secret-scan
    steps:
    - uses: actions/checkout@v4
    - name: Snyk dependency scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

  iac-scan:
    runs-on: ubuntu-latest
    needs: secret-scan
    steps:
    - uses: actions/checkout@v4
    - name: Checkov IaC scan
      uses: bridgecrewio/checkov-action@master
      with:
        directory: ./terraform
        framework: terraform
        soft_fail: false

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-scan, dependency-check, iac-scan]
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
    - name: Deploy (only if all security checks pass)
      run: |
        echo "Deploying verified, scanned, signed image..."
```

---

## 9. Compliance & Audit

### Common Compliance Frameworks

| Framework | Focus | Who needs it |
|---|---|---|
| SOC 2 | Security, availability, processing integrity | SaaS companies |
| ISO 27001 | Information security management system | Enterprise |
| HIPAA | Healthcare data protection | Health tech |
| PCI DSS | Credit card data | E-commerce, fintech |
| GDPR | EU personal data protection | Any company with EU users |
| FedRAMP | US government cloud security | GovTech vendors |

### Audit Logging

```yaml
# Kubernetes Audit Policy — log all security-relevant events
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  namespaces: ["production"]
  
- level: Metadata
  resources:
  - group: ""
    resources: ["pods", "services"]
    
- level: Request
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

- level: None
  resources:
  - group: ""
    resources: ["events"]
```

**AWS CloudTrail — track ALL API calls:**
```bash
# Who accessed what in the last 24h?
aws cloudtrail lookup-events \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-16T00:00:00Z" \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket

# Enable CloudTrail for all regions
aws cloudtrail create-trail \
  --name my-audit-trail \
  --s3-bucket-name my-audit-logs \
  --is-multi-region-trail \
  --enable-log-file-validation
```

### Security as Code — OPA (Open Policy Agent)

```rego
# policy.rego — Enforce security requirements
package kubernetes.admission

# Deny containers running as root
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container '%v' must set runAsNonRoot=true", [container.name])
}

# Deny images from untrusted registries
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not startswith(container.image, "myregistry.io/")
  msg := sprintf("Container '%v' uses untrusted image '%v'", [container.name, container.image])
}

# Deny pods without resource limits
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container '%v' must have memory limits set", [container.name])
}
```

---

## 10. Interview Mastery

---

### Beginner Questions

---

**Q: What is the principle of least privilege? Give an example.**

**Perfect Answer:**
> "Least privilege means granting only the minimum permissions needed to perform a specific task — nothing more. It limits the blast radius if credentials are compromised.
>
> Example: An application that reads images from an S3 bucket should have an IAM role with ONLY `s3:GetObject` on that specific bucket — not `s3:*` on all buckets. If an attacker compromises the application, they can only read images from that one bucket — they can't delete data, access other buckets, or escalate to other AWS services.
>
> In practice, I start with zero permissions and add only what's needed. When debugging 'access denied' errors, I check CloudTrail to see exactly which API call was denied and add only that specific permission."

**Interviewer expectation:** Specific example, mention of blast radius, practical experience applying it.

---

**Q: What is the difference between authentication and authorization?**

**Perfect Answer:**
> "Authentication (AuthN) verifies WHO you are — it confirms your identity. This could be a password, MFA token, client certificate, or biometric. It answers 'are you really the person you claim to be?'
>
> Authorization (AuthZ) determines WHAT you can do — it checks your permissions after identity is confirmed. Even though I've proven I'm John (authenticated), authorization decides whether John can access the production database or deploy to production.
>
> Real-world example in Kubernetes: A CI pipeline authenticates via a ServiceAccount token (proves it's the CI system). Then RBAC authorization checks whether that ServiceAccount has permission to update Deployments in the production namespace.
>
> Common mistake: giving too much authorization because authentication was strong. Strong MFA doesn't mean a user should have admin access."

---

**Q: Why should you never store secrets in code or environment variables in production?**

**Perfect Answer:**
> "Secrets in code get committed to Git — and git history is permanent. Even if you delete the file, anyone with repo access can find it in the commit history. Automated bots scan public GitHub and exploit leaked credentials within minutes.
>
> Environment variables are better than code but still problematic: they appear in process listings (`/proc/pid/environ`), crash dumps, CI logs, and Docker inspect output. They're also not auditable — you can't track who accessed which secret when.
>
> The production-grade approach is a secrets manager like HashiCorp Vault or AWS Secrets Manager because it provides: encryption at rest, access control policies, audit logs of every access, automatic rotation, and dynamic short-lived credentials. If a secret leaks, you can trace who accessed it and rotate it automatically."

---

### Intermediate Questions

---

**Q: How would you implement secrets management in a Kubernetes environment?**

**Perfect Answer:**
> "I'd use a layered approach depending on the organization's maturity:
>
> **Layer 1 (minimum):** Kubernetes Secrets with encryption at rest enabled (EncryptionConfiguration). Store secrets as K8s Secret objects, reference them as environment variables or volume mounts. Use RBAC to restrict who can read secrets.
>
> **Layer 2 (recommended):** External Secrets Operator (ESO) pulling from a centralized vault. Secrets live in AWS Secrets Manager or HashiCorp Vault. ESO syncs them into Kubernetes Secrets automatically. This gives us centralized management, rotation, and audit logging outside the cluster.
>
> **Layer 3 (advanced):** Vault Agent sidecar injection. A Vault Agent runs alongside each pod, authenticates via Kubernetes ServiceAccount (IRSA for AWS), fetches secrets directly, and renews leases automatically. Secrets never exist as Kubernetes objects at all — they're injected into the pod's filesystem at runtime.
>
> Additionally: I use Sealed Secrets or SOPS for GitOps workflows where encrypted secrets are committed to git and decrypted only inside the cluster."

---

**Q: Explain how container image scanning fits into a CI/CD pipeline.**

**Perfect Answer:**
> "Container image scanning should be a mandatory gate in the build pipeline — a build cannot proceed to deployment if critical vulnerabilities are found. Here's how I implement it:
>
> **When to scan:**
> 1. After `docker build` — scan the freshly built image
> 2. On a schedule — rescan deployed images daily (new CVEs are published daily)
> 3. Before deployment — admission controller verifies the image was recently scanned
>
> **What it scans:**
> - OS packages (apt, yum) — known CVEs in libraries like OpenSSL, glibc
> - Application dependencies (pip, npm) — vulnerable packages
> - Dockerfile misconfigurations — running as root, exposed secrets
>
> **How I handle findings:**
> - CRITICAL: Block the build immediately. No deploy until fixed.
> - HIGH: Block for production, allow for staging (to test the fix)
> - MEDIUM/LOW: Create tracking tickets, fix in next sprint
>
> **Tools I've used:** Trivy (free, excellent), Snyk Container (commercial, better UI), AWS ECR scanning (built-in for ECR users).
>
> **Reduction strategy:** Use minimal base images (distroless, Alpine, slim), update base images weekly, and use multi-stage builds to exclude build tools from production images."

---

**Q: What is a zero-trust security model and how would you implement it?**

**Perfect Answer:**
> "Zero trust means 'never trust, always verify' — no implicit trust is granted based on network location. Even if you're inside the corporate network or inside the Kubernetes cluster, every request must be authenticated and authorized.
>
> Traditional model: 'Trust the internal network, protect the perimeter' (castle-and-moat)
> Zero trust model: 'Verify every request regardless of origin' (no castle, no moat — just identity)
>
> **Implementation in practice:**
>
> 1. **Identity-based access:** All communication requires identity verification. I use mTLS via a service mesh (Istio/Linkerd) so every pod must present a valid certificate to communicate with any other pod.
>
> 2. **Network micro-segmentation:** Kubernetes NetworkPolicies with default-deny. Each pod explicitly declares what it can talk to. No lateral movement if one pod is compromised.
>
> 3. **Short-lived credentials:** Dynamic secrets from Vault with 1-hour TTLs instead of long-lived API keys. Even if stolen, they expire quickly.
>
> 4. **Continuous verification:** Every request is checked — not just at login. JWTs with short expiry + refresh tokens. Session re-validation.
>
> 5. **Least privilege:** Fine-grained RBAC at every layer (AWS IAM, K8s RBAC, database roles).
>
> The key insight: compromising one component should NOT give access to other components."

---

### Advanced Questions

---

**Q: How would you respond to a security incident where an API key was leaked in a public GitHub repository?**

**Perfect Answer:**
> "I follow a structured incident response process:
>
> **Immediate (first 15 minutes):**
> 1. **Revoke/rotate the compromised key immediately** — this is the single most important step. Don't investigate first; stop the bleeding.
> 2. Check if the key provides access to production systems
> 3. Enable detailed logging on the affected service
>
> **Assessment (next 1-2 hours):**
> 4. Check audit logs: was the key used by an unauthorized party? CloudTrail for AWS, access logs for APIs.
> 5. Determine blast radius: what could an attacker access with this key?
> 6. Check git history: are there other secrets in the repo?
> 7. Run automated secret scanner (trufflehog/gitleaks) on full history
>
> **Remediation (same day):**
> 8. Remove secret from git history using BFG Repo-Cleaner
> 9. Implement pre-commit hooks (gitleaks) to prevent future leaks
> 10. Add secret scanning to CI pipeline (fail on detection)
> 11. Review access: does this application actually need long-lived keys? Migrate to IAM roles with temporary credentials
>
> **Post-incident (within 1 week):**
> 12. Write incident report (blameless postmortem)
> 13. Implement systemic fix: all secrets move to Vault/Secrets Manager
> 14. Training: educate team on secrets hygiene
> 15. Verify: no unauthorized resources were created (cryptominers, etc.)
>
> The critical lesson: **response speed matters more than investigation depth** initially. Rotate first, investigate second."

---

**Q: Design a secure multi-tenant Kubernetes cluster. How do you isolate tenants?**

**Perfect Answer:**
> "Multi-tenant isolation in Kubernetes requires defense in depth across multiple layers:
>
> **1. Namespace isolation:**
> Each tenant gets its own namespace. All resources (pods, services, secrets) are scoped to that namespace.
>
> **2. Network isolation:**
> ```yaml
> # Default deny all inter-namespace traffic
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: deny-cross-namespace
>   namespace: tenant-a
> spec:
>   podSelector: {}
>   ingress:
>   - from:
>     - podSelector: {}  # Only same-namespace pods
> ```
> Tenant A pods cannot communicate with Tenant B pods.
>
> **3. RBAC isolation:**
> Each tenant's ServiceAccounts can ONLY access their own namespace. ClusterRoles are restricted to platform admins only.
>
> **4. Resource quotas:**
> ```yaml
> apiVersion: v1
> kind: ResourceQuota
> metadata:
>   name: tenant-a-quota
>   namespace: tenant-a
> spec:
>   hard:
>     requests.cpu: '10'
>     requests.memory: 20Gi
>     limits.cpu: '20'
>     limits.memory: 40Gi
>     pods: '50'
> ```
> Prevents one tenant from consuming all cluster resources (noisy neighbor).
>
> **5. Pod Security Standards:**
> Enforce 'restricted' PSS per namespace — no privilege escalation, no host access.
>
> **6. Secrets isolation:**
> Each namespace has its own secrets. RBAC ensures cross-namespace secret access is impossible.
>
> **7. Node isolation (strong isolation):**
> For highly sensitive tenants, dedicate nodes using taints/tolerations and node affinity. Tenant A pods run only on Tenant A nodes.
>
> **8. Runtime security:**
> Use Falco for runtime anomaly detection — alert if a container exec's a shell, reads /etc/shadow, or makes unexpected network connections.
>
> **Trade-off:** Stronger isolation = higher cost (dedicated nodes) vs. softer isolation = higher risk but better resource utilization. I discuss this trade-off with stakeholders based on their compliance requirements."

---

**Q: Explain the security implications of running containers as root and how you prevent it.**

**Perfect Answer:**
> "Running a container as root (UID 0) is dangerous because of container breakout risks. While containers have namespaces and cgroups for isolation, kernel vulnerabilities can allow a root process inside the container to escape to the host — and if it's root inside, it's root outside.
>
> **Attack scenario:**
> 1. Attacker exploits an application vulnerability (e.g., RCE in a web app)
> 2. They're now running as root inside the container
> 3. They exploit a kernel vulnerability (e.g., CVE-2022-0185) or mount the host filesystem
> 4. They escape to the host with root privileges
> 5. From the host, they compromise other containers and the cluster
>
> **Prevention layers:**
>
> 1. **Dockerfile:** `USER nonroot` — build images that run as non-root
> 2. **Pod Security Context:** `runAsNonRoot: true` — Kubernetes rejects pods that try to run as root
> 3. **Pod Security Standards:** Enforce 'restricted' level at namespace
> 4. **Admission controllers:** OPA/Kyverno policies that block root containers cluster-wide
> 5. **Read-only root filesystem:** Even if escalation occurs, attacker can't write malware
> 6. **Drop all capabilities:** `capabilities: drop: [ALL]` — removes CAP_SYS_ADMIN, CAP_NET_RAW, etc.
> 7. **Seccomp profiles:** Restrict available system calls to absolute minimum
> 8. **Image scanning:** Flag images that have `USER root` in their Dockerfile
>
> The defense-in-depth approach means even if one layer fails, others still protect you."

---

### Scenario-Based Questions

---

**Q: Your monitoring shows unusual outbound traffic from a production pod at 3 AM. What do you do?**

**Perfect Answer:**
> "This could be data exfiltration from a compromised pod. I treat this as a potential security incident and respond carefully to preserve evidence:
>
> **Step 1: Confirm the anomaly (2 minutes)**
> - Check network flow logs (VPC Flow Logs, Calico/Cilium logs)
> - Identify: source pod, destination IP, data volume, protocol
> - Is the destination a known service (e.g., external API, monitoring)? Or unknown?
>
> **Step 2: Isolate without destroying evidence (5 minutes)**
> - Apply a NetworkPolicy that blocks ALL egress from the suspicious pod
> - Do NOT delete the pod — we need it for forensics
> ```bash
> kubectl label pod suspicious-pod quarantine=true
> # NetworkPolicy selects quarantine=true and blocks egress
> ```
>
> **Step 3: Investigate the pod (30 minutes)**
> ```bash
> kubectl exec suspicious-pod -- ps aux
> kubectl exec suspicious-pod -- netstat -tulpn
> kubectl exec suspicious-pod -- cat /proc/1/cmdline
> kubectl logs suspicious-pod --since=6h
> ```
> - Check: was the image tampered with? Is there a crypto miner? Reverse shell?
> - Check: when was the pod last deployed? Was there a recent image change?
>
> **Step 4: Broader investigation**
> - Check other pods from the same deployment
> - Scan the image for malware
> - Review recent deployments and who triggered them
> - Check IAM/RBAC: was there unauthorized access to the cluster?
>
> **Step 5: Remediate**
> - Rebuild and redeploy from trusted image
> - Rotate any credentials the pod had access to
> - Patch the vulnerability that allowed the compromise
>
> **Step 6: Post-incident**
> - Document timeline and root cause
> - Implement detection (Falco rules for outbound anomalies)
> - Review network policies to prevent lateral movement"

---

**Q: You need to pass a SOC 2 audit. What DevOps controls would you implement?**

**Perfect Answer:**
> "SOC 2 focuses on five trust principles: Security, Availability, Processing Integrity, Confidentiality, and Privacy. Here's how I implement controls with DevOps tools:
>
> **Security:**
> - All infrastructure as code (Terraform) with change history
> - Mandatory code review (branch protection rules)
> - Automated vulnerability scanning in CI/CD
> - MFA for all human access (AWS, GitHub, production)
> - Network segmentation (VPC, security groups, network policies)
>
> **Availability:**
> - Multi-AZ deployments with auto-scaling
> - Health checks and automatic failover
> - Disaster recovery plan tested quarterly
> - SLA monitoring with Prometheus + PagerDuty alerting
>
> **Processing Integrity:**
> - Immutable infrastructure (rebuild, don't patch)
> - Automated testing pipeline
> - Deployment approvals for production
> - Rollback procedures tested regularly
>
> **Confidentiality:**
> - Encryption at rest (KMS for all storage)
> - Encryption in transit (TLS everywhere, mTLS for internal)
> - Secrets management (Vault, never in code)
> - Data classification and access controls
>
> **Audit trail (applies to all):**
> - CloudTrail for AWS API calls
> - Kubernetes audit logging
> - Git history for all changes
> - Centralized log aggregation (ELK/Loki) with 90-day retention
> - Regular access reviews (quarterly)
>
> The key DevOps advantage: infrastructure as code means every change is trackable, reviewable, and reproducible — which auditors love."

---

### FAANG-Style Conceptual Questions

---

**Q: Design a secrets rotation system that achieves zero-downtime rotation for database credentials across 100 microservices.**

**Perfect Answer:**
> "This is a coordination challenge — you need to rotate credentials without any service experiencing authentication failures.
>
> **Architecture:**
>
> ```
> Vault (Source of Truth)
>   │
>   ├── Dynamic credentials with 1h TTL
>   │   (each service gets unique, auto-expiring creds)
>   │
>   └── Rotation signal → All services refresh simultaneously
> ```
>
> **Approach 1: Dynamic secrets (preferred)**
>
> Vault generates short-lived, per-service database credentials. Each service gets a unique username/password valid for 1 hour. Vault's lease renewal handles rotation transparently — the service just calls `vault.renew()` periodically. When a lease expires, Vault creates new credentials and the old ones are revoked.
>
> No coordination needed — each service independently manages its own lease lifecycle.
>
> **Approach 2: Dual-credential rotation (for static creds)**
>
> If dynamic secrets aren't possible:
> 1. Database has two credential slots: `user_v1` and `user_v2`
> 2. All services currently use `user_v1`
> 3. Rotation process:
>    - Create `user_v2` with new password
>    - Update secret in Vault to `user_v2`
>    - Services gradually pick up new secret (via sidecar polling every 30s)
>    - After 5 minutes (all services refreshed), delete `user_v1`
> 4. Next rotation swaps back to `user_v1`
>
> Zero downtime because both credentials are valid simultaneously during the transition window.
>
> **Approach 3: Connection proxy (PgBouncer)**
>
> Services connect to PgBouncer (connection proxy). Only PgBouncer knows the real DB credentials. Rotate credentials in PgBouncer's config and `RELOAD` — downstream services never know their credentials changed because they authenticate with the proxy, not the database directly.
>
> **Monitoring:** Alert on authentication failures (spike = rotation problem), track credential age, and alert if any credential exceeds max-age policy."

---

**Q: How do you balance security with developer velocity? Teams complain security tools slow them down.**

**Perfect Answer:**
> "This is the fundamental DevSecOps tension. My approach: make security invisible and fast, not a gate that frustrates developers.
>
> **1. Speed — make security checks fast:**
> - Secret scanning in pre-commit hooks: < 2 seconds (runs locally, no CI delay)
> - SAST with Semgrep: ~30 seconds (fast enough for PR checks)
> - Image scanning cached: only scan new layers, not entire image every time
> - Total pipeline overhead target: < 3 minutes for security gates
>
> **2. Shift left — catch early when fixes are cheap:**
> - IDE plugins that flag issues while coding (before commit)
> - PR-level feedback (not just blocking, but educating: 'this is SQL injection because...')
>
> **3. Guardrails not gates:**
> - Instead of blocking deployments and requiring security team approval, provide golden paths
> - Pre-approved base images that are already scanned and hardened
> - Terraform modules that encode security best practices (S3 module always enables encryption)
> - CI templates with security baked in (developers don't configure it)
>
> **4. Risk-based approach:**
> - CRITICAL/HIGH: block immediately (non-negotiable)
> - MEDIUM: allow deploy with a ticket created (fix within sprint)
> - LOW: informational only (fix during tech debt sprints)
>
> **5. Self-service:**
> - Developers can add exceptions with justification (not blocked indefinitely)
> - Security team reviews exceptions asynchronously (doesn't block the PR)
>
> **Metric I track:** Developer time-to-deploy (if it increases after adding security tooling, we're doing it wrong). The goal is secure AND fast."

---

## Summary

```
DevSecOps — Key Takeaways:

Secrets Management:    Never in code. Use Vault/Secrets Manager.
                       Dynamic secrets > static secrets.
                       Audit all access. Rotate regularly.

TLS/SSL:              TLS 1.2/1.3 only. Automate with cert-manager.
                       mTLS for service-to-service (zero trust).
                       Monitor certificate expiry.

Container Security:    Non-root, read-only FS, drop capabilities.
                       Scan images in CI. Pin versions.
                       Sign and verify images.

IAM:                  Least privilege. Roles, not long-lived keys.
                       RBAC in Kubernetes. OIDC for humans.
                       Audit everything.

Vulnerability Scanning: SAST, SCA, DAST, IaC scanning in pipeline.
                       Block CRITICAL, track the rest.
                       Scan continuously (new CVEs daily).

Network Security:      Default deny. Micro-segmentation.
                       WAF for public-facing apps.
                       VPC isolation for data layers.

CI/CD Security:        Every stage has a security check.
                       Security gates must be fast (< 3 min).
                       Make secure the easy path.
```

---

[⬇️ Download This File](#)
