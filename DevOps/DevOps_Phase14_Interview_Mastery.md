# DevOps Phase 14 — DevOps Interview Mastery
## Top Interview Questions · Troubleshooting · Production Incidents · System Design

---

> **Who this is for:** Engineers preparing for DevOps/SRE/Platform Engineering interviews at any level — from startups to FAANG. This is your final preparation guide.

---

## Table of Contents

1. [How DevOps Interviews Work](#1-how-devops-interviews-work)
2. [Core Concepts — Rapid-Fire Questions](#2-core-concepts--rapid-fire-questions)
3. [Linux & Troubleshooting](#3-linux--troubleshooting)
4. [Docker & Kubernetes Deep Questions](#4-docker--kubernetes-deep-questions)
5. [CI/CD & Automation](#5-cicd--automation)
6. [Cloud & Infrastructure](#6-cloud--infrastructure)
7. [Monitoring, Logging & Observability](#7-monitoring-logging--observability)
8. [Production Incident Scenarios](#8-production-incident-scenarios)
9. [System Design for DevOps](#9-system-design-for-devops)
10. [Behavioral & Culture-Fit Questions](#10-behavioral--culture-fit-questions)
11. [Interview Cheat Sheet](#11-interview-cheat-sheet)

---

## 1. How DevOps Interviews Work

### Interview Stages

```
Stage 1: Recruiter Screen (30 min)
  → Experience overview, salary expectations, team fit
  
Stage 2: Technical Phone Screen (45-60 min)
  → Linux troubleshooting, CI/CD concepts, basic coding
  
Stage 3: System Design (60 min)
  → Design a deployment pipeline, scalable infrastructure, monitoring system
  
Stage 4: Hands-On / Live Coding (60 min)
  → Write Terraform, Dockerfile, K8s manifests, bash scripts
  → Debug a broken system live
  
Stage 5: Behavioral / Team Fit (45 min)
  → Incident response stories, collaboration, conflict resolution
```

### What Interviewers Actually Evaluate

```
Technical Skills (40%):
  - Can you build and operate production systems?
  - Do you understand the WHY behind tools, not just the WHAT?
  - Can you troubleshoot under pressure?

Problem-Solving (30%):
  - How do you approach unknown problems?
  - Can you break complex issues into steps?
  - Do you consider trade-offs?

Communication (20%):
  - Can you explain technical concepts clearly?
  - Do you ask clarifying questions?
  - Can you write clear runbooks/docs?

Culture Fit (10%):
  - Blameless postmortem mindset?
  - Collaborative or solo?
  - Comfortable with on-call?
```

### How to Answer Interview Questions (Framework)

```
The STAR-T Method for DevOps:

S — Situation: What was the context? (production, scale, constraints)
T — Task: What was your responsibility?
A — Action: What did YOU specifically do? (be precise, use "I" not "we")
R — Result: What was the outcome? (metrics: uptime improved X%, deploy time reduced Y%)
T — Takeaway: What did you learn? What would you do differently?

Example:
S: "Our e-commerce platform was experiencing 5-minute deployments with 30 seconds of downtime each time."
T: "As the DevOps lead, I was responsible for achieving zero-downtime deployments."
A: "I implemented rolling updates with readiness probes, added a preStop hook with 15-second drain time, and configured PDB to maintain 80% availability during updates."
R: "Deployments became zero-downtime. We went from 2 deploys/week (fear of downtime) to 5 deploys/day."
T: "I learned that zero-downtime isn't just about deployment strategy — it requires the application to handle graceful shutdown."
```

---

## 2. Core Concepts — Rapid-Fire Questions

These are "warmup" questions that test fundamental understanding. Answer each in 2-3 sentences.

---

**Q: What is DevOps?**
> "DevOps is a culture and set of practices that unifies software development and operations. The goal is to shorten the development lifecycle and deliver high-quality software continuously. It's built on automation, CI/CD, monitoring, and shared responsibility between dev and ops teams."

---

**Q: What is Infrastructure as Code (IaC)?**
> "IaC means managing infrastructure (servers, networks, databases) through machine-readable definition files rather than manual configuration. Tools like Terraform and Ansible let you version-control infrastructure, review changes via PR, and reproduce environments reliably. If a server dies, you recreate it from code in minutes rather than spending hours configuring manually."

---

**Q: What is immutable infrastructure?**
> "Instead of updating (mutating) existing servers with patches and config changes, you build a completely new server image with the updates baked in, deploy it, and destroy the old one. This eliminates configuration drift — every server is identical to its definition. If something breaks, you don't debug it; you destroy and rebuild from the known-good image."

---

**Q: Explain the difference between containers and virtual machines.**
> "VMs virtualize hardware — each VM has its own full OS kernel, taking gigabytes of RAM and minutes to boot. Containers virtualize the operating system — they share the host kernel, use only megabytes, and start in milliseconds. Containers are lighter and faster but provide weaker isolation. VMs are heavier but provide complete isolation (different OS kernels, stronger security boundaries)."

---

**Q: What is a service mesh?**
> "A service mesh is an infrastructure layer that handles service-to-service communication — things like load balancing, encryption (mTLS), retries, and observability — without requiring application code changes. A sidecar proxy (Envoy) is injected beside each pod, intercepting all traffic. Istio and Linkerd are popular implementations. It's essential at scale when you have 50+ microservices communicating."

---

**Q: Explain GitOps.**
> "GitOps uses Git as the single source of truth for declarative infrastructure and application state. A controller (ArgoCD/Flux) continuously watches a Git repo and reconciles the actual cluster state to match. Changes happen exclusively through pull requests — not kubectl. Benefits: full audit trail, self-healing (drift correction), and easy rollback (git revert)."

---

**Q: What are SLI, SLO, and SLA?**
> "SLI (Service Level Indicator) is the metric itself — like '99.5% of requests returned 200 OK.' SLO (Service Level Objective) is the internal target — 'we aim for 99.9% success rate.' SLA (Service Level Agreement) is the external contract with customers — 'we guarantee 99.5% or refund.' SLO is always stricter than SLA to provide a safety buffer."

---

**Q: What is the 12-factor app methodology?**
> "It's a set of principles for building cloud-native applications. Key factors: store config in environment variables (not code), treat backing services as attached resources, make services stateless and horizontally scalable, execute apps as one or more stateless processes, export services via port binding, and treat logs as event streams. Following these principles makes apps easy to deploy, scale, and operate in containers and cloud platforms."

---

**Q: Explain eventual consistency.**
> "In a distributed system, eventual consistency means that after a write, not all nodes immediately have the latest data — but given enough time without new writes, all nodes will converge to the same value. It's a trade-off for availability: the system stays responsive even during network partitions, accepting that reads might briefly return stale data. Good for: social media likes, DNS. Bad for: bank balances."

---

**Q: What is the difference between stateful and stateless applications?**
> "A stateless app stores no data between requests — every request is independent, and any instance can handle any request. This makes horizontal scaling trivial. A stateful app maintains data between requests (sessions, connections, local files). Stateful apps are harder to scale because you need sticky sessions or shared state. Best practice: make apps stateless and externalize state to Redis/database."

---

## 3. Linux & Troubleshooting

---

**Q: A server is slow. How do you diagnose it?**

**Perfect Answer:**
> "I follow the USE method (Utilization, Saturation, Errors) systematically:
>
> **CPU:**
> ```bash
> top -bn1 | head -20       # Overall CPU usage + top processes
> mpstat -P ALL 1 3         # Per-CPU breakdown
> # Look for: high %user (app), high %system (kernel), high %iowait (disk)
> ```
>
> **Memory:**
> ```bash
> free -h                   # Total/used/available
> vmstat 1 5               # Swap in/out activity
> # Look for: high swap usage (means RAM is exhausted)
> ```
>
> **Disk I/O:**
> ```bash
> iostat -xz 1 5           # Disk utilization and latency
> iotop                    # Which process is doing I/O
> # Look for: %util near 100%, high await (ms per I/O op)
> ```
>
> **Network:**
> ```bash
> ss -s                    # Socket summary
> ss -tuln                 # Listening ports
> iftop                    # Bandwidth per connection
> # Look for: many TIME_WAIT connections, packet drops
> ```
>
> **Process-level:**
> ```bash
> ps aux --sort=-%mem | head  # Top memory consumers
> ps aux --sort=-%cpu | head  # Top CPU consumers
> strace -p <pid> -c          # System call summary (what's it waiting on?)
> ```
>
> My first 60 seconds: `uptime` (load average), `dmesg | tail` (kernel errors), `free -h`, `top`, `iostat`. This covers 80% of issues."

---

**Q: Explain file permissions in Linux. What does chmod 755 mean?**

**Perfect Answer:**
> "Linux file permissions have three sets: owner, group, others. Each set has read (4), write (2), execute (1).
>
> `chmod 755` means:
> - Owner: 7 = rwx (read + write + execute)
> - Group: 5 = r-x (read + execute, no write)
> - Others: 5 = r-x (read + execute, no write)
>
> Common patterns:
> ```
> 644 — files (owner rw, everyone r): config files, HTML
> 755 — directories and scripts (owner rwx, everyone rx)
> 600 — sensitive files (owner rw only): SSH keys, .env
> 700 — private directories (owner only): .ssh/
> ```
>
> Special permissions:
> - SUID (4xxx): Execute as file owner (e.g., `/usr/bin/passwd` runs as root)
> - SGID (2xxx): Execute as group / new files inherit directory's group
> - Sticky bit (1xxx): Only owner can delete files (e.g., `/tmp`)"

---

**Q: How do you find what's consuming disk space?**

**Perfect Answer:**
> ```bash
> # Quick overview
> df -h                        # Filesystem usage (which partition is full?)
> 
> # Find largest directories
> du -sh /* 2>/dev/null | sort -rh | head -10
> du -sh /var/* | sort -rh | head
>
> # Find large files
> find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
>
> # Common culprits:
> # /var/log/     — unrotated logs (fix: logrotate)
> # /tmp/         — temp files never cleaned
> # /var/lib/docker/ — Docker images/volumes (docker system prune)
> # Old kernels in /boot
>
> # Docker specifically:
> docker system df             # Show Docker disk usage
> docker system prune -a       # Remove unused images/containers/volumes
>
> # Find deleted files still held open (invisible disk usage):
> lsof | grep deleted
> # Fix: restart the process holding the file handle
> ```"

---

**Q: A process is running but not responding to requests. How do you debug it?**

**Perfect Answer:**
> "The process is likely stuck — either deadlocked, waiting on I/O, or out of threads/connections.
>
> ```bash
> # Step 1: Is it actually running?
> ps aux | grep <process-name>
> # Check state: S (sleeping), R (running), D (uninterruptible — stuck on I/O), Z (zombie)
>
> # Step 2: What is it doing RIGHT NOW?
> strace -p <pid> -e trace=network,write  # System calls (is it stuck waiting?)
> # If stuck on futex() → deadlock
> # If stuck on read()/recv() → waiting for network response
> # If stuck on write()/writev() → output buffer full
>
> # Step 3: Thread dump (for Java/Go/Python)
> kill -3 <java-pid>     # Java: prints thread dump to stdout
> curl localhost:8080/debug/pprof/goroutine?debug=2  # Go: goroutine dump
>
> # Step 4: Network connections
> ss -tnp | grep <pid>   # What connections does it have?
> # Many connections in ESTABLISHED but idle → connection pool to dead backend
> # Many in CLOSE_WAIT → app not closing connections properly
>
> # Step 5: Resource limits
> cat /proc/<pid>/limits       # Check file descriptor limit
> ls /proc/<pid>/fd | wc -l   # Current open file descriptors
> # If current ≈ limit → can't open new connections
>
> # Step 6: If all else fails
> gdb -p <pid>           # Attach debugger (last resort)
> # Or: generate core dump and analyze offline
> ```"

---

**Q: Explain the Linux boot process.**

**Perfect Answer:**
> "The boot sequence:
>
> 1. **BIOS/UEFI** — Hardware initialization, finds bootable device
> 2. **Bootloader (GRUB2)** — Loads kernel + initramfs into memory
> 3. **Kernel** — Initializes hardware, mounts root filesystem, starts init
> 4. **Init system (systemd)** — PID 1, starts all services in correct order
> 5. **Services** — Networking, SSH, Docker, your applications
>
> When troubleshooting boot issues:
> - Kernel panic → corrupt kernel or missing driver (boot from rescue image)
> - Filesystem errors → fsck from single-user mode
> - Service won't start → `journalctl -u <service>` for logs
> - Network not coming up → check systemd-networkd/NetworkManager
>
> In DevOps context: we rarely debug boot issues because we use immutable infrastructure — if a machine won't boot, we replace it. But understanding this helps debug systemd service ordering issues."

---

## 4. Docker & Kubernetes Deep Questions

---

**Q: Explain Docker networking modes and when to use each.**

**Perfect Answer:**
> "Docker has several network drivers:
>
> **Bridge (default):**
> - Creates a virtual bridge network on the host
> - Containers get private IPs (172.17.x.x)
> - Containers on same bridge can communicate
> - Requires port publishing (-p) to be reachable from outside
> - Use for: single-host development, Docker Compose
>
> **Host:**
> - Container shares host's network namespace directly
> - No network isolation — container uses host's IP and ports
> - Best performance (no NAT overhead)
> - Use for: performance-sensitive apps, network monitoring tools
>
> **None:**
> - No networking at all
> - Container is completely isolated
> - Use for: batch jobs that don't need network, security-sensitive workloads
>
> **Overlay:**
> - Multi-host networking (Docker Swarm / overlay across nodes)
> - VXLAN encapsulation
> - Use for: Docker Swarm clusters
>
> **macvlan:**
> - Container gets its own MAC address on the physical network
> - Appears as a physical device to the network
> - Use for: legacy apps that need to be on the LAN directly
>
> In production Kubernetes, we don't use Docker networking — the CNI plugin (Calico, Cilium, Flannel) handles pod networking. Each pod gets a real routable IP."

---

**Q: A pod is in CrashLoopBackOff. Walk me through your debugging process.**

**Perfect Answer:**
> "CrashLoopBackOff means the container starts, crashes, K8s restarts it, it crashes again — repeating with exponential backoff delays.
>
> ```bash
> # Step 1: Get the error details
> kubectl describe pod <pod-name>
> # Look at: Events section (pull errors, OOM, probe failures)
> # Look at: Last State → Reason (OOMKilled, Error, Completed)
> # Look at: Exit code (137=OOMKilled, 1=app error, 143=SIGTERM)
>
> # Step 2: Check logs
> kubectl logs <pod-name>              # Current attempt (may be empty if crash is instant)
> kubectl logs <pod-name> --previous   # Previous crashed container's logs (THE KEY COMMAND)
>
> # Step 3: Based on what you find:
>
> # Exit code 137 (OOMKilled):
> # → Container exceeded memory limit
> # → Fix: increase resources.limits.memory OR fix memory leak
> kubectl top pod <pod-name>
>
> # Exit code 1 (application error):
> # → App crashed during startup
> # → Check logs for: missing env vars, can't connect to DB, config error
> # → Common: database not ready yet, missing secret, wrong image tag
>
> # Exit code 0 (Completed):
> # → Container exited successfully but K8s expected it to keep running
> # → Fix: ensure process runs in foreground (not daemon mode)
>
> # Image pull errors:
> # → Wrong image tag, auth issues with private registry
> kubectl get events --field-selector reason=Failed
>
> # Step 4: Interactive debugging (if logs aren't enough)
> # Override the entrypoint to get a shell:
> kubectl run debug --image=<same-image> --rm -it -- /bin/sh
> # Or if the pod spec allows:
> kubectl debug <pod-name> -it --image=busybox
> ```"

---

**Q: Explain how Kubernetes scheduling works. How does the scheduler decide where to place a pod?**

**Perfect Answer:**
> "The kube-scheduler places pods onto nodes through a two-phase process:
>
> **Phase 1: Filtering (which nodes CAN run this pod?)**
> - Does the node have enough CPU and memory (requests)?
> - Does the node satisfy nodeSelector/nodeAffinity?
> - Does the node have the required taints/tolerations?
> - Are there enough ports available?
> - Does it satisfy topology spread constraints?
> - Result: list of feasible nodes
>
> **Phase 2: Scoring (which feasible node is BEST?)**
> - LeastRequestedPriority: prefer nodes with more available resources
> - BalancedResourceAllocation: prefer nodes with balanced CPU/memory usage
> - NodeAffinity weight: prefer nodes matching preferred (soft) affinity
> - PodAntiAffinity: prefer nodes without conflicting pods
> - ImageLocality: prefer nodes that already have the container image cached
> - Result: highest-scoring node wins
>
> **Key scheduling features I use in production:**
>
> ```yaml
> # Topology spread (distribute across zones)
> topologySpreadConstraints:
> - maxSkew: 1
>   topologyKey: topology.kubernetes.io/zone
>   whenUnsatisfiable: DoNotSchedule
>
> # Taints/tolerations (dedicated GPU nodes)
> tolerations:
> - key: \"gpu\"
>   operator: \"Equal\"
>   value: \"true\"
>   effect: \"NoSchedule\"
>
> # Pod anti-affinity (don't colocate replicas)
> affinity:
>   podAntiAffinity:
>     preferredDuringSchedulingIgnoredDuringExecution:
>     - weight: 100
>       podAffinityTerm:
>         topologyKey: kubernetes.io/hostname
> ```
>
> **Important:** If a pod is Pending with 'no nodes available to schedule,' check: resource requests vs available capacity, taints without matching tolerations, or PVC binding issues."

---

**Q: What happens when you run `kubectl apply -f deployment.yaml`?**

**Perfect Answer:**
> "The full lifecycle:
>
> 1. **kubectl** validates YAML syntax, sends HTTP POST/PUT to API server
> 2. **API Server** authenticates the request (who is this?), authorizes via RBAC (can they do this?), runs admission controllers (webhooks that mutate/validate the resource)
> 3. **API Server** persists the Deployment object to **etcd**
> 4. **Deployment Controller** (in controller-manager) detects the new/changed Deployment, creates/updates a **ReplicaSet**
> 5. **ReplicaSet Controller** detects the ReplicaSet needs N pods, creates **Pod** objects
> 6. **Scheduler** detects unscheduled Pods, assigns each to a node (writes node assignment to etcd)
> 7. **Kubelet** on the assigned node detects a pod scheduled to it, instructs the **container runtime** (containerd) to pull the image and start containers
> 8. **Container runtime** pulls image, creates container, starts process
> 9. Kubelet reports pod status back to API server
> 10. **kube-proxy** updates iptables/IPVS rules so the Service can route traffic to the new pod
> 11. **Readiness probe** passes → pod is added to Service endpoints → receives traffic
>
> For a rolling update: a new ReplicaSet is created with the new image, pods are gradually added to the new RS and removed from the old RS, respecting maxSurge and maxUnavailable."

---

**Q: Explain Kubernetes resource requests vs limits.**

**Perfect Answer:**
> "**Requests** = what the pod is guaranteed. The scheduler uses this to place the pod — it ensures the node has this much available. Think of it as a reservation.
>
> **Limits** = the maximum the pod can use. If a pod exceeds its memory limit, it's OOM-killed. If it exceeds CPU limit, it's throttled (slowed, not killed).
>
> ```yaml
> resources:
>   requests:
>     cpu: 200m      # Guaranteed 0.2 CPU cores
>     memory: 256Mi  # Guaranteed 256MB RAM
>   limits:
>     cpu: 1000m     # Can burst up to 1 full core
>     memory: 512Mi  # Killed if exceeds 512MB
> ```
>
> **Best practices:**
> - Always set both requests and limits
> - Set requests based on normal usage (profile your app!)
> - Set limits at 2-3x requests for burst headroom
> - Memory limit close to request (OOM is better than node instability)
> - CPU limit can be generous (throttling is graceful)
>
> **QoS Classes (determined by requests/limits):**
> - **Guaranteed:** requests == limits (first to be scheduled, last to be evicted)
> - **Burstable:** requests < limits (middle priority)
> - **BestEffort:** no requests/limits (first evicted under memory pressure)
>
> **Common mistake:** Setting requests too high wastes cluster resources (pods reserved but not using). Too low → pods evicted under pressure. Profile with `kubectl top pods` over several days to find the right values."

---

## 5. CI/CD & Automation

---

**Q: Design a CI/CD pipeline for a monorepo with 5 services.**

**Perfect Answer:**
> "Monorepo CI challenge: you don't want to build ALL 5 services when only one changed.
>
> **Strategy: Path-based triggering + selective builds**
>
> ```yaml
> # .github/workflows/ci.yml
> on:
>   push:
>     paths:
>       - 'services/auth/**'
>       - 'services/api/**'
>       - 'services/worker/**'
>       - 'shared/**'        # Shared libraries trigger all
>
> jobs:
>   detect-changes:
>     runs-on: ubuntu-latest
>     outputs:
>       auth: ${{ steps.changes.outputs.auth }}
>       api: ${{ steps.changes.outputs.api }}
>       worker: ${{ steps.changes.outputs.worker }}
>     steps:
>     - uses: dorny/paths-filter@v2
>       id: changes
>       with:
>         filters: |
>           auth:
>             - 'services/auth/**'
>             - 'shared/**'
>           api:
>             - 'services/api/**'
>             - 'shared/**'
>           worker:
>             - 'services/worker/**'
>             - 'shared/**'
>
>   build-auth:
>     needs: detect-changes
>     if: needs.detect-changes.outputs.auth == 'true'
>     # ... build only auth service
>
>   build-api:
>     needs: detect-changes
>     if: needs.detect-changes.outputs.api == 'true'
>     # ... build only api service
> ```
>
> **Key design decisions:**
> - Shared code changes trigger all dependent services
> - Each service has its own Dockerfile and test suite
> - Docker layer caching shared across services (common base image)
> - Deploy only changed services (don't restart the world)
> - Integration tests run only if dependent services changed
>
> **Alternative at scale (Bazel/Nx):** For 50+ services, use a build system that understands the dependency graph and only builds/tests affected targets. GitHub Actions path filters don't scale to complex dependency trees."

---

**Q: How do you handle secrets in CI/CD pipelines?**

**Perfect Answer:**
> "Secrets in CI/CD must never appear in logs, artifacts, or code. My approach:
>
> **Storage:** Secrets stored in the CI platform's secret store (GitHub Encrypted Secrets, GitLab CI Variables with 'masked' flag) or an external vault (HashiCorp Vault, AWS Secrets Manager).
>
> **Access:** Pipelines authenticate to secret stores using OIDC federation — no long-lived credentials. GitHub Actions can assume an AWS IAM role directly:
> ```yaml
> permissions:
>   id-token: write
> steps:
> - uses: aws-actions/configure-aws-credentials@v4
>   with:
>     role-to-assume: arn:aws:iam::123456:role/ci-role
>     aws-region: us-east-1
> ```
>
> **Protection:**
> - Secrets are masked in logs (CI platform auto-redacts)
> - Never echo/print secret values
> - Use environment scoping (production secrets only available in production deployment job)
> - Require approval for jobs that access production secrets
> - Rotate secrets regularly (automated via Vault)
>
> **Common mistakes I've seen:**
> - Printing environment variables for debugging (leaks secrets)
> - Using long-lived access keys instead of OIDC
> - Same secrets for all environments (staging compromise = production compromise)
> - Not rotating after team member departure"

---

**Q: How would you implement rollback in a deployment pipeline?**

**Perfect Answer:**
> "Rollback strategy depends on the deployment method:
>
> **1. Kubernetes native rollback:**
> ```bash
> # Instant — reverts to previous ReplicaSet
> kubectl rollout undo deployment/myapp
> kubectl rollout undo deployment/myapp --to-revision=3  # Specific version
> ```
> K8s keeps revision history (revisionHistoryLimit). Fast because old ReplicaSet still exists.
>
> **2. GitOps rollback (ArgoCD):**
> ```bash
> # Revert the commit in config repo
> git revert HEAD
> git push
> # ArgoCD syncs automatically — cluster matches previous state
> ```
> Or: `argocd app rollback myapp <revision-number>`
>
> **3. Blue-green rollback:**
> ```bash
> # Switch traffic back to previous environment
> kubectl patch svc myapp -p '{\"spec\":{\"selector\":{\"slot\":\"blue\"}}}'
> ```
> Instant — previous environment is still running.
>
> **4. Canary rollback (Argo Rollouts):**
> ```bash
> kubectl argo rollouts abort myapp  # Stops canary, routes all traffic to stable
> ```
>
> **Automated rollback triggers:**
> - Error rate > 5% for 2 minutes
> - P99 latency > 2x baseline
> - Readiness probe failures
> - OOM kills in new pods
>
> **Critical rule:** Rollback should be possible WITHOUT the CI/CD pipeline being available. If your pipeline is down, you must still be able to rollback via `kubectl` or git revert."

---

## 6. Cloud & Infrastructure

---

**Q: Explain VPC architecture. How would you design a VPC for a production application?**

**Perfect Answer:**
> "A VPC is an isolated virtual network in the cloud. My production VPC design:
>
> ```
> VPC: 10.0.0.0/16 (65,536 IPs)
> 
> Public Subnets (internet-facing):
>   10.0.1.0/24 (AZ-a) — Load balancers, NAT gateways, bastion
>   10.0.2.0/24 (AZ-b)
>   10.0.3.0/24 (AZ-c)
>
> Private Subnets (application):
>   10.0.10.0/24 (AZ-a) — App servers, EKS nodes
>   10.0.11.0/24 (AZ-b)
>   10.0.12.0/24 (AZ-c)
>
> Data Subnets (isolated):
>   10.0.20.0/24 (AZ-a) — Databases, caches
>   10.0.21.0/24 (AZ-b)
>   10.0.22.0/24 (AZ-c)
>   (No route to internet — even outbound)
> ```
>
> **Key components:**
> - **Internet Gateway:** Allows public subnets to reach internet
> - **NAT Gateway (per AZ):** Allows private subnets outbound internet (for pulling images, updates)
> - **Route tables:** Public routes to IGW, private routes to NAT
> - **Security groups:** Stateful firewall per resource (ALB → App → DB chain)
> - **NACLs:** Stateless subnet-level firewall (defense in depth)
>
> **Design principles:**
> - Minimum 3 AZs for HA
> - Private subnets for compute (no direct internet access)
> - Isolated data subnets (no internet even outbound)
> - NAT Gateway per AZ (if one AZ's NAT dies, others still work)
> - /16 VPC with /24 subnets gives plenty of room for growth"

---

**Q: What is Terraform state and why does it matter?**

**Perfect Answer:**
> "Terraform state is a JSON file that maps your Terraform configuration to real-world resources. It contains the resource IDs, attributes, and metadata that Terraform needs to know what already exists.
>
> **Why it matters:**
> - Without state, Terraform doesn't know what it created vs. what already existed
> - `terraform plan` compares desired state (code) to current state (state file) to determine what to change
> - If state is lost, Terraform thinks nothing exists and will try to recreate everything (causing conflicts with existing resources)
>
> **Production state management:**
> ```hcl
> # backend.tf — store state remotely (NEVER in git!)
> terraform {
>   backend \"s3\" {
>     bucket         = \"my-terraform-state\"
>     key            = \"production/terraform.tfstate\"
>     region         = \"us-east-1\"
>     encrypt        = true
>     dynamodb_table = \"terraform-locks\"  # Prevents concurrent modifications
>   }
> }
> ```
>
> **Common state issues:**
> - **State lock conflict:** Someone else is running terraform. Wait or force-unlock (carefully).
> - **State drift:** Someone changed infrastructure manually. Run `terraform plan` to detect, `terraform apply` to reconcile.
> - **State corruption:** Restore from S3 versioning backup.
> - **Importing existing resources:** `terraform import aws_instance.my_server i-1234567890`
>
> **Critical rules:**
> - Never store state in git (contains secrets in plain text)
> - Always use remote backend with locking
> - Enable versioning on the S3 bucket (rollback capability)
> - Use workspaces or separate state files per environment"

---

**Q: How do you manage multiple environments (dev/staging/prod) with Terraform?**

**Perfect Answer:**
> "There are three common approaches, with trade-offs:
>
> **Approach 1: Workspaces**
> ```bash
> terraform workspace new staging
> terraform workspace new production
> terraform workspace select production
> terraform apply -var-file=production.tfvars
> ```
> Pros: Simple, same code, separate state.
> Cons: All environments share the same backend config; easy to accidentally apply to wrong workspace.
>
> **Approach 2: Directory structure (my preference)**
> ```
> terraform/
> ├── modules/
> │   ├── vpc/
> │   ├── eks/
> │   └── rds/
> ├── environments/
> │   ├── dev/
> │   │   ├── main.tf      # Calls modules with dev-sized params
> │   │   ├── backend.tf   # Separate state file
> │   │   └── terraform.tfvars
> │   ├── staging/
> │   └── production/
> ```
> Pros: Clear separation, independent state, can promote changes env-by-env.
> Cons: Some code duplication (mitigated by modules).
>
> **Approach 3: Terragrunt (DRY wrapper)**
> ```
> terragrunt/
> ├── terragrunt.hcl        # Common config
> ├── dev/
> │   └── terragrunt.hcl   # Inherits + overrides
> ├── staging/
> └── production/
> ```
> Pros: Minimal duplication, consistent patterns.
> Cons: Extra tool/complexity.
>
> **I recommend Approach 2** for most teams — it's explicit, each environment can evolve independently, and there's zero risk of applying production changes to dev accidentally. Modules ensure consistency."

---

## 7. Monitoring, Logging & Observability

---

**Q: Design a monitoring and alerting strategy for a microservices application.**

**Perfect Answer:**
> "I follow the four golden signals (from Google SRE) plus USE method for infrastructure:
>
> **Application-level (Golden Signals):**
>
> | Signal | What to measure | Alert threshold |
> |---|---|---|
> | Latency | P50, P90, P99 response time | P99 > 2x baseline for 5 min |
> | Traffic | Requests per second | Sudden drop > 50% (anomaly) |
> | Errors | 5xx rate / error ratio | > 1% for 2 min (page), > 5% (critical) |
> | Saturation | CPU, memory, connection pool usage | > 80% for 5 min |
>
> **Infrastructure-level (USE):**
> - CPU utilization per node (> 85% = warning)
> - Memory utilization (> 90% = critical)
> - Disk usage (> 80% = warning, > 90% = critical)
> - Network errors/drops
>
> **Stack:**
> ```
> Metrics: Prometheus (collection) + Grafana (visualization)
> Logs: Loki or ELK (aggregation) + structured JSON logs
> Traces: OpenTelemetry (instrumentation) + Jaeger (visualization)
> Alerting: Alertmanager → PagerDuty (critical) / Slack (warning)
> ```
>
> **Alert design principles:**
> - Alert on symptoms (error rate), not causes (CPU high) — high CPU without errors isn't a problem
> - Every alert must be actionable — if you can't do anything about it, it's not an alert
> - Use severity levels: critical (pages someone at 3 AM), warning (Slack, fix during business hours)
> - Include runbook link in every alert annotation
> - Group related alerts (don't send 100 alerts for one incident)
>
> **Dashboard hierarchy:**
> 1. Executive: Are customers happy? (error rate, apdex score)
> 2. Service: Is each service healthy? (golden signals per service)
> 3. Infrastructure: Are nodes/pods healthy? (USE metrics)
> 4. Debug: Deep-dive during incidents (per-pod, per-endpoint)"

---

**Q: What is the difference between monitoring and observability?**

**Perfect Answer:**
> "Monitoring tells you WHEN something is broken (predefined metrics, known failure modes). Observability lets you understand WHY it's broken (explore unknown failure modes).
>
> **Monitoring = known-unknowns:**
> 'Alert me when error rate exceeds 1%.' You knew to look for this in advance.
>
> **Observability = unknown-unknowns:**
> 'Why are only users from Germany experiencing latency on the /checkout endpoint during business hours?' You didn't anticipate this specific question, but your system has enough data to answer it.
>
> **How to achieve observability:**
> - **High-cardinality metrics:** Don't just measure 'error rate' — measure it by endpoint, by status code, by user region, by customer tier
> - **Structured logs:** JSON with consistent fields (request_id, user_id, trace_id) so you can filter/aggregate on any dimension
> - **Distributed traces:** Follow a single request across all services — see exactly where time is spent
> - **Correlation:** The same trace_id appears in metrics, logs, AND traces — jump between them seamlessly
>
> Modern systems are too complex for predefined dashboards. You need the ability to ask arbitrary questions of your system data in real-time. Tools like Honeycomb and Grafana with Loki/Tempo enable this exploratory approach."

---

## 8. Production Incident Scenarios

---

**Q: It's 3 AM. PagerDuty alerts you: 'API error rate above 10%.' Walk through your response.**

**Perfect Answer:**
> "I follow a structured incident response:
>
> **0-2 minutes: Acknowledge and assess**
> - Acknowledge alert in PagerDuty (stops escalation)
> - Open laptop, check dashboard: is it getting worse? Scope?
> - Quick assessment: is this partial degradation or total outage?
>
> **2-5 minutes: Triage**
> - When did it start? (correlate with recent deploys, config changes)
> - Check: was there a deployment in the last hour? → prime suspect
>   ```bash
>   kubectl rollout history deployment/api-service
>   ```
> - Check: which endpoints are failing? All of them or specific ones?
> - Check: are dependent services healthy? (database, cache, third-party APIs)
>
> **5-15 minutes: Mitigate**
> - If recent deploy → rollback immediately, investigate later
>   ```bash
>   kubectl rollout undo deployment/api-service
>   ```
> - If no recent deploy → is it resource exhaustion?
>   ```bash
>   kubectl top pods -l app=api-service
>   kubectl describe pod <failing-pod>  # OOMKilled? CrashLoop?
>   ```
> - If dependency is down → circuit breaker should activate; verify graceful degradation
> - Scale up if traffic spike: `kubectl scale deployment/api-service --replicas=20`
>
> **15-30 minutes: Communicate**
> - Update status page if customer-facing
> - Post in incident Slack channel: what's happening, what you've done, ETA
> - If not resolved in 30 min → escalate (wake up second person)
>
> **After resolution:**
> - Monitor for 15 minutes to confirm stability
> - Post brief resolution note
> - Schedule blameless postmortem within 48 hours
>
> **Key principle:** Mitigate first, investigate later. At 3 AM with 10% errors, your job is to stop the bleeding — the root cause analysis can wait until morning."

---

**Q: Your database is experiencing connection exhaustion — applications can't get connections. What do you do?**

**Perfect Answer:**
> "Connection exhaustion means the database has hit `max_connections` — all slots are used, new requests are rejected.
>
> **Immediate (stop the bleeding):**
> ```sql
> -- Check: how many connections, by who?
> SELECT usename, application_name, state, count(*)
> FROM pg_stat_activity
> GROUP BY usename, application_name, state
> ORDER BY count DESC;
> 
> -- Kill idle connections that have been open too long
> SELECT pg_terminate_backend(pid)
> FROM pg_stat_activity
> WHERE state = 'idle'
>   AND state_change < now() - interval '10 minutes';
> ```
>
> **Short-term (survive the incident):**
> - Increase `max_connections` temporarily (requires restart or use `ALTER SYSTEM` + reload)
> - Identify the source: which application is holding connections?
> - Is one service leaking connections (opening but never closing)?
>
> **Root cause analysis:**
>
> 1. **No connection pooler:** Each app instance opens its own connections. 10 pods × 20 connections = 200 connections. Add PgBouncer.
>
> 2. **Connection leak:** App opens connections in error path without closing. Code fix needed.
>    ```python
>    # BAD: connection never closed on exception
>    conn = db.connect()
>    result = conn.execute(query)  # If this throws...
>    conn.close()                  # ...this never runs
>
>    # GOOD: context manager guarantees cleanup
>    with db.connect() as conn:
>        result = conn.execute(query)
>    ```
>
> 3. **Long-running queries:** Blocking slots. Find and kill them:
>    ```sql
>    SELECT pid, duration, query
>    FROM pg_stat_activity
>    WHERE state = 'active'
>    ORDER BY duration DESC;
>    ```
>
> 4. **Sudden traffic spike:** More pods auto-scaled, each opened connections.
>    Fix: connection pool should have a per-pod limit that respects DB max.
>
> **Prevention:**
> - PgBouncer: multiplexes 1000 app connections into 50 DB connections
> - Connection pool settings: max 20 per pod, timeout 5s (fail fast vs. queue forever)
> - Monitor: alert when connections > 80% of max
> - Set `idle_in_transaction_session_timeout` to kill abandoned transactions"

---

**Q: Users report intermittent 503 errors. It happens randomly and clears within 30 seconds. What could cause this?**

**Perfect Answer:**
> "Intermittent 503s that self-resolve in 30 seconds — classic symptoms of pod lifecycle issues during deployments or scaling events.
>
> **Most likely causes:**
>
> **1. Rolling deployment without proper graceful shutdown:**
> - Old pod receives SIGTERM → immediately stops → in-flight requests get 503
> - Fix: `preStop` hook (sleep 10-15s) to allow load balancer to drain
> - Fix: App must handle SIGTERM gracefully (finish in-flight, reject new)
>
> **2. Readiness probe delay:**
> - New pod starts → registered in Service before it's truly ready → requests fail
> - Pod doesn't respond to readiness probe → removed from endpoints → 503 for a few seconds
> - Fix: Configure `initialDelaySeconds` correctly, readiness probe checks actual dependencies
>
> **3. HPA scaling down:**
> - HPA removes pods → endpoints updated → brief window where clients still route to terminated pod
> - Fix: Slower scale-down (stabilizationWindowSeconds: 300)
>
> **4. Node NotReady (brief network blip):**
> - Node loses connectivity for 10-20s → pods marked Unknown → endpoints removed → 503
> - Node recovers → pods back to Ready → traffic resumes
> - Fix: Tolerate short disruptions with `tolerationSeconds` for not-ready taint
>
> **5. Connection draining mismatch:**
> - Load balancer deregistration delay < pod shutdown time
> - Or: pod shutdown time > terminationGracePeriodSeconds (pod force-killed)
>
> **How to diagnose:**
> ```bash
> # Correlate 503 timing with pod events
> kubectl get events --sort-by=.lastTimestamp | grep -E 'Killing|Unhealthy|Scheduled'
>
> # Check if it aligns with deployments
> kubectl rollout history deployment/api-service
>
> # Check HPA scaling events
> kubectl describe hpa api-service
> ```
>
> **The fix that covers most cases:**
> ```yaml
> spec:
>   terminationGracePeriodSeconds: 60
>   containers:
>   - lifecycle:
>       preStop:
>         exec:
>           command: ['sh', '-c', 'sleep 15']
>     readinessProbe:
>       initialDelaySeconds: 10
>       periodSeconds: 5
>       failureThreshold: 3
> ```"

---

## 9. System Design for DevOps

---

**Q: Design a deployment pipeline that supports 50 microservices across 3 environments.**

**Perfect Answer:**
> "At this scale, self-service with guardrails is critical. Engineers shouldn't wait for a DevOps team to deploy.
>
> **Architecture:**
>
> ```
> Application Repos (50 repos)     Config Repo (1 monorepo)
> ├── service-auth/                ├── apps/
> ├── service-payments/            │   ├── auth/
> ├── service-notifications/       │   │   ├── base/
> │   ├── src/                     │   │   └── overlays/
> │   ├── Dockerfile               │   │       ├── dev/
> │   └── .github/workflows/       │   │       ├── staging/
> └── ...                          │   │       └── production/
>                                  │   ├── payments/
>     CI builds image              │   └── ...
>     Updates config repo ─────────┘
>                                  ArgoCD watches config repo
>                                  Syncs each environment
> ```
>
> **CI Pipeline (per service, reusable workflow):**
> ```yaml
> # .github/workflows/ci.yml (template)
> jobs:
>   test:      # Unit + integration tests
>   build:     # Docker build + push (tagged with git SHA)
>   scan:      # Trivy + Semgrep
>   promote:   # Update config repo with new image tag
>     # dev: auto-promoted on every push to main
>     # staging: auto-promoted after dev is healthy for 30 min
>     # production: canary with automated analysis
> ```
>
> **Environment promotion flow:**
> ```
> main branch merge
>   → CI builds + pushes image :abc123
>   → Auto-updates dev overlay (image: :abc123)
>   → ArgoCD syncs dev (auto)
>   → Dev healthy for 30 min? → Auto-updates staging overlay
>   → ArgoCD syncs staging
>   → Staging healthy for 2 hours? → PR created for production overlay
>   → Engineer approves PR (or auto-merge if canary analysis passes)
>   → ArgoCD syncs production (canary rollout)
> ```
>
> **Progressive delivery (production):**
> - All 50 services use Argo Rollouts
> - Default canary strategy: 5% → 25% → 50% → 100%
> - Analysis templates check: error rate, latency, pod restarts
> - Automatic rollback on failure
>
> **Platform features:**
> - Shared CI templates (one to maintain, 50 use it)
> - Standardized Kubernetes manifests via Kustomize
> - External Secrets Operator (secrets from Vault, not in git)
> - Namespace per service with resource quotas
> - Service mesh for mTLS and observability
>
> **Operational tooling:**
> - Backstage portal: view all services, deploy status, dependencies
> - Slack notifications: deploy started/succeeded/failed per team channel
> - Dashboards: DORA metrics (deploy frequency, lead time, failure rate, MTTR)"

---

**Q: Design a monitoring and incident response system for a platform serving 10 million users.**

**Perfect Answer:**
> "At 10M users, you need proactive detection (catch issues before users notice) and fast response.
>
> **Monitoring architecture:**
>
> ```
> Data Sources:
> ├── Application metrics (Prometheus, custom metrics)
> ├── Infrastructure metrics (node-exporter, cAdvisor)
> ├── Logs (structured JSON → Loki/ELK)
> ├── Traces (OpenTelemetry → Jaeger/Tempo)
> ├── Synthetic monitoring (Pingdom, Checkly — tests from external)
> └── Real User Monitoring (RUM — actual user experience)
>
> Processing:
> ├── Prometheus (metrics storage, 30-day retention)
> ├── Thanos/Cortex (long-term storage, cross-region)
> └── Alertmanager (deduplication, routing, silencing)
>
> Alerting:
> ├── Critical → PagerDuty → oncall engineer's phone (24/7)
> ├── Warning → Slack #alerts channel (business hours)
> └── Info → Dashboard only (for investigation)
> ```
>
> **Alert design:**
>
> ```yaml
> # Symptom-based alerts (what users experience)
> - alert: HighErrorRate
>   expr: error_rate > 0.01 for 2m
>   severity: critical
>   runbook: https://wiki/runbooks/high-error-rate
>
> - alert: HighLatency
>   expr: p99_latency > 2s for 5m
>   severity: warning
>
> # Proactive alerts (catch before users notice)
> - alert: DiskFilling
>   expr: predict_linear(disk_free[4h], 24*3600) < 0
>   severity: warning
>   # Disk will be full in 24 hours at current rate
>
> - alert: CertExpiringSoon
>   expr: cert_expiry_seconds < 7*86400
>   severity: warning
>   # Certificate expires in < 7 days
> ```
>
> **Incident response system:**
>
> ```
> Detection (< 1 min):
>   Alert fires → PagerDuty escalation policy
>
> Response (< 5 min):
>   Oncall acknowledges → Checks dashboard
>   IF critical: opens incident channel, starts status page
>
> Mitigation (< 30 min target):
>   Rollback / scale / failover / restart
>   Communicate on status page every 15 min
>
> Resolution:
>   Root cause fixed, monitoring confirms recovery
>   Status page updated: resolved
>
> Post-incident (within 48 hours):
>   Blameless postmortem document
>   Action items with owners and due dates
>   Follow-up to prevent recurrence
> ```
>
> **Oncall structure:**
> - Primary oncall (responds first) + secondary (escalation)
> - Weekly rotation (prevent burnout)
> - Oncall handoff meeting: open issues, known risks
> - SLO-based: if error budget is being consumed, oncall focuses on reliability
>
> **Key metrics to display prominently:**
> - Current availability vs SLO (are we within budget?)
> - Active incidents count
> - Mean time to detect (MTTD)
> - Mean time to resolve (MTTR)"

---

## 10. Behavioral & Culture-Fit Questions

---

**Q: Tell me about a time you caused a production outage. What happened and what did you learn?**

**Perfect Answer (example):**
> "Early in my career, I ran a database migration on production that added a NOT NULL column without a default value to a table with 50 million rows. The ALTER TABLE locked the table for 8 minutes. All queries queued up, connection pool exhausted, and the entire application went down.
>
> **What I did immediately:**
> - Killed the migration (`SELECT pg_cancel_backend(pid)`)
> - Confirmed the table was unlocked and traffic recovered
> - Communicated the issue to the team within 5 minutes
>
> **Root cause:** I didn't test the migration against production-sized data. In staging (with 1000 rows), it completed instantly.
>
> **What I changed:**
> 1. All migrations must be tested against a production-sized dataset before approval
> 2. Added a pre-deploy checklist that includes 'check for table locks'
> 3. Adopted the expand/contract pattern: add column WITH DEFAULT (instant in Postgres 11+), then add NOT NULL constraint separately
> 4. Set `statement_timeout` to 5 seconds for DDL in production — fail fast rather than lock
>
> **What I learned:** Never trust staging to represent production. Size matters for database operations. And most importantly — every outage is a learning opportunity, not a failure."

---

**Q: How do you handle disagreements with developers about deployment practices?**

**Perfect Answer:**
> "I approach it collaboratively, not adversarially. Developers and DevOps have the same goal — deliver reliable software to users — we just have different perspectives on risk.
>
> **My approach:**
> 1. **Listen first.** Understand their constraint. Often they're pushing back because our process is genuinely too slow or friction-heavy.
>
> 2. **Find common ground.** We both want: fast deployments, low risk, and happy users.
>
> 3. **Use data.** If a developer wants to skip testing in CI because 'it's slow,' I don't say 'no' — I show that 15% of our bugs were caught by that specific test stage. Then I work with them to make it faster rather than removing it.
>
> 4. **Offer alternatives.** Instead of blocking, provide a path that meets both our needs. 'You need to deploy urgently? Let's do a canary at 5% with tight monitoring — faster to production but with a safety net.'
>
> **Real example:** A team wanted to deploy directly from their laptops to production because CI was 'too slow' (20 minutes). Instead of blocking them, I reduced CI to 8 minutes by adding better caching, parallelizing tests, and using faster runners. The developer friction was legitimate — the solution was to fix the pipeline, not enforce a policy they'd work around."

---

**Q: How do you prioritize work when everything seems urgent?**

**Perfect Answer:**
> "I use a simple framework:
>
> 1. **Is production broken RIGHT NOW?** → Drop everything, mitigate immediately
> 2. **Is production about to break?** (disk filling, cert expiring, approaching capacity) → Fix today
> 3. **Is this blocking other teams?** (broken CI, blocked deploys) → Fix within hours
> 4. **Is this important but not urgent?** (tech debt, automation improvements) → Scheduled sprint work
>
> For competing priorities at the same level, I ask:
> - What's the blast radius? (affects 1 team vs. entire org)
> - What's the opportunity cost of NOT doing it?
> - Can someone else handle it while I focus on the higher-impact item?
>
> I'm also explicit about trade-offs with stakeholders. 'I can set up the new monitoring system this week OR fix the flaky CI pipeline. The CI pipeline affects 20 engineers daily. Which is more valuable?' Making the trade-off visible helps them prioritize for me."

---

**Q: Describe your experience with on-call. How do you manage burnout?**

**Perfect Answer:**
> "I've been on-call for production systems at scale. My approach to making on-call sustainable:
>
> **During on-call:**
> - I keep runbooks updated so I'm not debugging blind at 3 AM
> - I prioritize sleep — if paged, I mitigate quickly and go back to sleep (root cause can wait)
> - I track every page: was it actionable? If not, fix the alert to prevent noise
>
> **Preventing burnout (systemic):**
> - **Reduce page volume:** Every false alert and noisy alert is toil. I spend 20% of each sprint reducing alert noise.
> - **Automate recovery:** If I'm doing the same manual fix repeatedly, I automate it (self-healing)
> - **Fair rotation:** Weekly rotation, never back-to-back, compensatory time off after heavy weeks
> - **Error budget policy:** If error budget is consumed, feature work stops and we focus on reliability. This prevents chronic 'everything on fire' culture.
>
> **Personal:** I distinguish between 'being on-call' (available) and 'being paged' (active incident). A good on-call week has 0-2 pages. If I'm getting 10+ pages per week, that's a systemic problem to fix — not a badge of honor."

---

## 11. Interview Cheat Sheet

### Quick Reference — Key Numbers to Know

```
Latency numbers every DevOps engineer should know:
  L1 cache reference:           0.5 ns
  RAM reference:                100 ns
  SSD random read:              150,000 ns (150 μs)
  HDD seek:                     10,000,000 ns (10 ms)
  Round trip same datacenter:   500,000 ns (0.5 ms)
  Round trip US coast-to-coast: 40,000,000 ns (40 ms)

Availability math:
  99.9%  = 43 min downtime/month
  99.95% = 22 min/month
  99.99% = 4.4 min/month

Container startup: 1-5 seconds
VM startup: 30-90 seconds
K8s rolling update (10 pods): 2-5 minutes
DNS propagation: seconds to 48 hours (TTL dependent)
TLS handshake: 1-3 round trips (50-150ms)

Scale numbers:
  Single PostgreSQL: ~10,000 connections, ~50K simple queries/sec
  Single Redis: ~100K ops/sec
  Single Nginx: ~50K concurrent connections
  Kafka single broker: ~1M messages/sec (small messages)
```

### Commands to Know Cold

```bash
# Kubernetes
kubectl get pods -A                          # All pods, all namespaces
kubectl describe pod <name>                  # Full pod details + events
kubectl logs <pod> --previous                # Previous container logs
kubectl exec -it <pod> -- /bin/sh            # Shell into pod
kubectl rollout undo deployment/<name>       # Rollback
kubectl top pods --sort-by=cpu               # Resource usage
kubectl get events --sort-by=.lastTimestamp  # Recent events

# Docker
docker ps -a                                 # All containers
docker logs -f <container>                   # Follow logs
docker exec -it <container> bash             # Shell into container
docker system prune -a                       # Clean everything
docker inspect <container>                   # Full details

# Linux troubleshooting
top / htop                                   # Process overview
free -h                                      # Memory
df -h                                        # Disk
ss -tlnp                                     # Listening ports
journalctl -u <service> -f                   # Service logs
dmesg | tail                                 # Kernel messages
lsof -i :8080                                # What's using port 8080

# Git
git log --oneline -20                        # Recent commits
git diff HEAD~1                              # Last change
git bisect start                             # Find breaking commit

# Networking
curl -vI https://example.com                 # HTTP debug
dig +trace example.com                       # DNS resolution
tcpdump -i eth0 port 80                      # Packet capture
mtr example.com                              # Network path + packet loss
```

### Architecture Patterns Quick Reference

```
Pattern              | When to Use                    | Trade-off
─────────────────────┼────────────────────────────────┼──────────────────
Microservices        | Large teams, independent deploy | Complexity
Monolith             | Small team, rapid prototyping   | Scaling limits
Event-driven         | Async, decoupled services       | Eventual consistency
CQRS                 | Read-heavy with complex queries | Dual data models
Saga                 | Distributed transactions        | Compensation logic
Circuit breaker      | Unreliable dependencies         | Latency detection
Bulkhead             | Isolate failure domains         | Resource overhead
Sidecar              | Cross-cutting concerns (mesh)   | Memory per pod
Ambassador           | Connection pooling, auth        | Extra hop
Strangler fig        | Gradual legacy migration        | Long transition
```

### Interview Do's and Don'ts

```
DO:
✓ Ask clarifying questions before designing
✓ Think out loud — show your reasoning process
✓ Mention trade-offs for every decision
✓ Reference real experience ("In my previous role...")
✓ Discuss what could go wrong and how you'd handle it
✓ Be honest about what you don't know ("I haven't used X, but...")

DON'T:
✗ Give one-word answers
✗ Claim to know everything
✗ Skip over failure scenarios
✗ Blame others for past incidents
✗ Jump to a solution without understanding the problem
✗ Use buzzwords without being able to explain them
✗ Say "we" when they ask what YOU specifically did
```

### Final Tips

```
1. DevOps interviews test DEPTH, not breadth.
   Knowing how DNS works internally > knowing 100 tool names.

2. Always connect to PRODUCTION experience.
   "In production, I've seen this cause..." carries 10x more weight than
   "Theoretically, you could..."

3. Ask "at what scale?" for system design.
   The answer for 100 users vs 10 million users is completely different.

4. Every trade-off has a "it depends."
   The best answer acknowledges the constraints that would change your choice.

5. Practice failure stories.
   "Tell me about a time something went wrong" is in EVERY interview.
   Have 3-4 polished stories ready (with the STAR-T framework).

6. Debugging > knowledge.
   Showing HOW you'd approach an unknown problem is more impressive
   than having memorized the answer.
```

---

## Summary

```
DevOps Interview Mastery — The Complete Path:

Foundation:
  Understand WHY tools exist, not just HOW to use them.
  Every technology solves a specific problem — articulate that problem.

Technical depth:
  Linux internals, container architecture, K8s control plane.
  Be able to draw architectures on a whiteboard.
  Know the failure modes of every component you mention.

Production experience:
  Incidents you've handled, systems you've built, problems you've solved.
  Specific numbers: reduced deployment time from X to Y, improved uptime to Z%.

System design:
  Start with requirements. Ask clarifying questions.
  Design incrementally. Show trade-offs at every decision.
  Consider: scale, failure modes, cost, team size, timeline.

Communication:
  Explain complex topics simply.
  Be honest about gaps.
  Show curiosity and willingness to learn.

The goal: Demonstrate that you can be trusted to run
production systems that real users depend on.
```

---

[⬇️ Download This File](#)
