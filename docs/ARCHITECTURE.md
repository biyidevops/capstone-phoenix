# TaskApp Capstone — Architecture

## Overview

TaskApp runs as a containerized React + Flask + Postgres application on a
self-managed k3s cluster, provisioned with Terraform and configured with
Ansible. The cluster runs on 3 AWS EC2 nodes (1 control-plane + 2 workers)
in `eu-north-1`, spread across two subnets/availability zones for resilience.

The app is live at `https://taskapp.techproconsults.com` with a valid
Let's Encrypt certificate, GitHub Container Registry images, login,
task creation, and the Kanban board all confirmed working end to end.

## Infrastructure

| Layer        | Tool      | Notes                                                        |
|--------------|-----------|---------------------------------------------------------------|
| Compute      | Terraform | 3 × t3.medium EC2, modular (vpc, security_groups, ec2, core) |
| Config       | Ansible   | roles: common (hardening), k3s-server, k3s-agent            |
| Remote state | S3 + DynamoDB (`use_lockfile`) | `techproconsults-capstone-tfstate-eun1`        |
| Networking   | k3s + Flannel (vxlan backend) | see incident write-up below                    |
| TLS          | cert-manager + Let's Encrypt (prod)          | HTTP-01 challenge via Traefik   |
| Public IP    | AWS Elastic IP | pinned to control-plane instance so it survives stop/start   |

### Node topology
Control-plane public IP is a pinned Elastic IP: `13.61.205.222`
(originally a dynamic public IP, which changed across several stop/start
cycles during setup — see note in RUNBOOK.md on why this matters for
kubeconfig / k3s TLS SAN / DNS).

### Request flow
## Security group / firewall

- SSH (22): restricted to admin IP range, not `0.0.0.0/0`
- HTTP/HTTPS (80/443): open to internet (app + ACME challenge)
- k3s API (6443): VPC-internal only (`10.0.0.0/16`), never exposed publicly
- VXLAN overlay (8472/udp), kubelet (10250/tcp): VPC-internal only
- Source/Destination Check: **disabled** on all 3 instances (required for
  any gateway-style pod routing between nodes — see incident below)

## Networking incident: multi-AZ + Flannel backend incompatibility

**Symptom:** Cross-node pod-to-pod traffic (and therefore Service/DNS
resolution) failed intermittently between the control-plane/worker-1 pair
and worker-2, with kernel errors like `network is unreachable` when Flannel
tried to install routes, and later `address already in use` when the
`flannel.1` VXLAN interface got stuck in a bad kernel state after repeated
CNI backend changes.

**Root cause:** The cluster's 3 nodes are split across two subnets for
availability (control-plane + worker-1 in `10.0.1.0/24`, worker-2 in
`10.0.2.0/24`). Flannel's `host-gw` backend routes pod traffic by adding a
direct kernel route via each peer node's private IP as the gateway. Linux
refuses to install a route whose gateway isn't in the same L2-reachable
subnet as the outgoing interface — so any route from worker-2 to the other
nodes' pod CIDRs was rejected outright. This is a fundamental limitation of
`host-gw`, not a bug: **it requires all nodes on one flat L2 network.**

Separately, AWS's **Source/Destination Check** (enabled by default on every
EC2 instance) blocks any packet where the instance is not the actual source
or destination — which further breaks gateway-style routing between nodes
even on a single subnet, and had to be disabled on all 3 instances.

**Fix:**
1. Disabled Source/Destination Check on all 3 EC2 instances.
2. Reinstalled k3s server with `--flannel-backend=vxlan` instead of the
   default. VXLAN encapsulates pod traffic inside UDP packets addressed
   directly node-to-node, so it works across subnets/AZs without relying on
   L2-adjacent gateway routes.
3. After the backend switch, the `flannel.1` interface entered a stuck
   kernel state (`address already in use` on every restart — confirmed via
   `restart counter is at 73` in systemd). Interface deletion and process
   restarts did not clear this; a full **instance reboot** was required to
   reset the kernel network namespace cleanly on both the control-plane and
   both workers.

**Verification:** confirmed pod-to-pod ping, Service DNS resolution via
CoreDNS, and direct TCP connectivity to the Postgres pod, all initiated from
worker-2 (the node in the separate subnet) reaching pods on the other two
nodes.

**Lesson for the "single-server assumption" writeup:** a single-server
deployment has no concept of pod-to-pod routing at all — this entire class
of failure is specific to going multi-node, and is worse when nodes aren't
on a single L2 subnet (a natural outcome of spreading nodes across AZs for
HA, which itself fixes a different single-server assumption: node/AZ
failure taking down the whole app).

## Core Kubernetes resources

| Resource                   | Single-server assumption it fixes                                                                                   |
|-----------------------------|------------------------------------------------------------------------------------------------------------------|
| `taskapp` Namespace         | Logical isolation; a single server has no equivalent multi-tenancy boundary                                       |
| ConfigMap / Secret split    | A single `.env` file mixes secrets and config; Kubernetes requires (and enforces) separating them                 |
| Postgres StatefulSet + PVC  | A single server's local Postgres data directory has no concept of rescheduling; StatefulSet + PVC keeps storage bound to the pod identity even if the underlying node changes |
| Migration Job               | A single Flask process running `alembic upgrade head` on startup is safe alone, but 2+ replicas starting simultaneously would race on migrations |
| Backend/Frontend Deployments (2 replicas, topologySpreadConstraints) | A single server has no replica placement problem; multi-replica requires explicit spread across nodes so a single node failure doesn't take down all instances |
| Ingress + cert-manager       | A single server terminates its own TLS or has none; Kubernetes needs an Ingress controller + automated certificate issuance to serve HTTPS across multiple backend pods |

## App wiring issues discovered during deployment

**Environment variable naming mismatch:** the ConfigMap/Secret were
initially built with assumed names (`DB_HOST`, `DB_USER`, etc.). The actual
app reads `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`,
`DATABASE_USER`, `DATABASE_PASSWORD` (confirmed by inspecting
`/app/migrations/env.py` inside the running image directly, rather than
guessing). Fixed by renaming the ConfigMap/Secret keys to match.

**Backend Service name is hardcoded in the frontend image:** the frontend's
nginx config (baked into the image at build time) proxies to an upstream
literally named `backend` — not `backend-service`. Confirmed via
`kubectl logs` showing `host not found in upstream "backend"`. Fixed by
naming the backend Service `backend` instead of `backend-service`. This is
a tight coupling between the pre-built frontend image and Kubernetes Service
naming that isn't visible without inspecting the image's nginx config.

**Verified image tags:** the pinned tags provided in secondary
documentation (`c2b906d`) did not exist in the GHCR registry for the
backend image. The actual working tags, confirmed via direct
`docker pull`, are:
- `ghcr.io/ts-a-devops/taskapp-backend:5d6b8fc`
- `ghcr.io/ts-a-devops/taskapp-frontend:26da2b0`

**No seeded demo accounts:** the login page displays demo credentials
(admin/admin123, user/user123) but the `users` table was empty after
migration — no seed script or Flask CLI command exists in the image.
Manually inserted both accounts with password hashes generated via the
app's own `werkzeug.security.generate_password_hash` (pbkdf2:sha256) to
ensure compatibility with the app's login verification logic.

**Tasks table schema:** confirmed already correct after migration
(includes `priority`, `status`, `updated_at`) — no manual ALTER TABLE
needed.

**Verified functionality:** login (both accounts), task creation, and the
Kanban board all confirmed working through the browser over HTTPS.

## TLS / Ingress

- cert-manager v1.20.3 installed via Helm
- `ClusterIssuer` `letsencrypt-prod` using HTTP-01 challenge via Traefik
- `Ingress` routes `/api` → `backend` Service (port 5000), `/` → 
  `frontend-service` (port 80)
- Certificate `taskapp-tls` issued and verified: `READY=True`,
  `curl -vI` confirms valid chain, correct SAN, HTTP/2 200 response

## Advanced Requirements

| Requirement | Status | Evidence |
|-------------|--------|----------|
| HPA (backend, CPU 50%, min 2 max 6) | ✅ Proven — scaled 2→6 under load | HPA_Active.png |
| PodDisruptionBudget (minAvailable: 1, both tiers) | ✅ Proven — node drain respected PDB | kubectl_drain_terminal.png |
| Observability (Grafana + metrics-server) | ✅ Running in monitoring namespace | Grafana_dashboard.png |
| NetworkPolicy (default-deny + explicit allows) | ✅ Applied — Flannel does not enforce; would be enforced with Cilium/Calico CNI swap | NetworkPolicy rules applied |
| GitOps (Argo CD) | ✅ Installed, syncing manifests/main, auto-sync confirmed | argo_cd_dashboard.png |
| securityContext hardening | ✅ runAsNonRoot, readOnlyRootFilesystem, drop ALL caps, RuntimeDefault seccomp | Applied to all workloads |

### NetworkPolicy — CNI enforcement note

NetworkPolicy manifests are present in `manifests/11-networkpolicy.yaml` with correct rules:
default-deny ingress in taskapp namespace, explicit allows for ingress→frontend,
frontend→backend (port 5000), backend→postgres (port 5432), monitoring→backend.
Flannel (vxlan) does not enforce these at the kernel level. To enforce:

    helm install cilium cilium/cilium --namespace kube-system \
      --set operator.replicas=1 --set kubeProxyReplacement=false

After Cilium replaces Flannel, the existing NetworkPolicy objects enforce immediately
with no manifest changes needed.
