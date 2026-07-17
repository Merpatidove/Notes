# QTI RAG Pipeline — Infrastructure Report

**Date:** 2026-07-17 (Updated)
**Cluster:** k0s v1.36.2+k0s (Debian 13 trixie)
**Repo:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG)

---

## 1. What's Running on the Cluster

| Component | Status | Access |
|---|---|---|
| **Argo CD** | 7/7 pods Running | `http://<controller-ip>:30080` (admin / `8H2RJZIZPHhsaRts`) |
| **Qdrant** | 1/1 Running | `qdrant.qdrant.svc.cluster.local:6333`, NFS-backed PVC (10Gi) |
| **api-gateway** | 1/1 Running, Healthy | `api-gateway.qti.svc:8080` — returns `{"status":"ok","version":"0.1.0"}` |
| **NFS CSI driver** | 3/3 controller, 2/2 node pods | k0s path: `/var/lib/k0s/kubelet` |
| **Prometheus/Grafana** | All targets up, 29 dashboards | `http://<node-ip>:30000` (admin / `8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx`) |

### 1.1 CI/CD Pipeline (Working End-to-End)

```
Push to main (api-gateway/**)
  -> GitHub Actions builds Docker image (Rust multi-stage)
  -> Pushes to ghcr.io/merpatidove/qti-api-gateway:<git-sha>
  -> Docker smoke test: /v1/health must return 200
  -> Commits updated image tag to kustomization.yaml [skip ci]
  -> Argo CD auto-syncs to cluster
  -> Pod restarts with new image, health check passes
```

- **Concurrency gate:** enabled — only the latest push builds (old in-progress runs are cancelled).
- **Rollback:** manual via `rollback.yml` — specify a previous image SHA/tag to revert instantly.

**Last successful run:** Image `ghcr.io/merpatidove/qti-api-gateway:e40ba85`, 32MB, deployed in ~8s.

### 1.2 Files Created in QTI-MAGANG Repo

| File | Purpose | Status |
|---|---|---|
| `api-gateway/Dockerfile` | Multi-stage Rust build (rust:1-bookworm → debian:bookworm-slim) | Working |
| `api-gateway/Cargo.toml` | Dependencies: axum 0.8, tokio, serde, reqwest (rustls-tls), prometheus | Working |
| `api-gateway/src/main.rs` | `/v1/health`, `/v1/query`, `/metrics` endpoints with Prometheus counters | Working |
| `api-gateway/k8s/deployment.yaml` | Deployment with liveness/readiness on `/v1/health` | Working |
| `api-gateway/k8s/service.yaml` | ClusterIP on port 8080 | Working |
| `api-gateway/k8s/kustomization.yaml` | Image tag managed by CI (`newTag: <sha>`) | Working |
| `api-gateway/k8s/servicemonitor.yaml` | Prometheus ServiceMonitor, scrapes `/metrics` every 15s | Working |
| `.github/workflows/ci.yml` | Build → push → smoke test → commit-back (with concurrency gate) | Working |
| `.github/workflows/rollback.yml` | Manual rollback to any previous image tag/SHA | Working |
| `rag-service/Cargo.toml` | RAG inference service (axum 0.7, tokio, serde) | Scaffold |
| `rag-service/src/main.rs` | Axum server on port 3000, `POST /api/v1/ticket` | Scaffold |
| `rag-service/src/models.rs` | `TicketRequest`, `InferenceResponse` structs | Scaffold |
| `rag-service/src/routes.rs` | Ticket handler (placeholder, logs ticket ID) | Scaffold |
| `k8s/argocd/application.yaml` | Argo CD Application CRD | Working |

### 1.3 rag-service (New Component)

A separate Rust/Axum service (`rag-service/`) has been scaffolded as the RAG inference engine:

- **Package name:** `inference-engine` (v0.1.0)
- **Endpoint:** `POST /api/v1/ticket` on port 3000
- **Models:** `TicketRequest` (ticket_id, raw_text, project_tags), `InferenceResponse` (status, message)
- **Status:** Placeholder only — accepts JSON, logs ticket ID, returns dummy response. No Qdrant or Mistral integration yet.
- **Not deployed** — no K8s manifests or CI pipeline for this service yet.

> **Note:** The `rag-service/target/` directory (compiled Rust artifacts) is committed to the repo. A `.gitignore` should be added.

---

## 2. SSH Deploy Keys

Stored at `/home/ferdi/.ssh/` for persistence across reboots:

| Key | Repo | Path |
|---|---|---|
| QTI-MAGANG deploy key | `Merpatidove/QTI-MAGANG` (write) | `~/.ssh/deploy_key_qti` |
| Notes repo deploy key | `Merpatidove/Notes` (write) | `~/.ssh/notes_deploy_key` |

SSH config (`~/.ssh/config`):
```
Host github.com-qti
    HostName github.com
    IdentityFile ~/.ssh/deploy_key_qti

Host github.com-notes
    HostName github.com
    IdentityFile ~/.ssh/notes_deploy_key
```

Usage:
```bash
git clone git@github.com-qti:Merpatidove/QTI-MAGANG.git
git clone git@github.com-notes:Merpatidove/Notes.git
```

---

## 3. Notable Observations

### 3.1 k0s-Specific
- **kubelet directory:** `/var/lib/k0s/kubelet/` (NOT `/var/lib/kubelet/`). Any Helm chart with `kubeletDir` must override it.
- **Storage class:** `nfs-csi` is the only one. The Qdrant Helm chart key is `persistence.storageClassName`, NOT `persistence.storageClass`.

### 3.2 Security
- **No firewall** on the controller VM — `iptables`, `ufw`, and `nftables` are all absent. NFS, k0s API, Docker Swarm ports are exposed.
- **No SSH keys** for any user except `ferdi` (who has no `authorized_keys`). All SSH is password-based.
- **9 sudo users**, only 2 actively used (ferdi, hapip).
- **Argo CD is `--insecure`** — no TLS. Change the admin password from the default.
- **Qdrant has no authentication**.
- **This VM is a single point of failure** — k0s controller, NFS server, Docker host all run here. No HA.

### 3.3 Ollama Was Removed
Ollama was running on this controller VM, redundant with the Mac Mini inference server. Removed entirely:
- Binary, service file, user, group, data directory all deleted.
- `ferdi` removed from `ollama` group.

### 3.4 No Firewall on the Controller
The controller has all ports open (NFS, k0s API, Docker Swarm). Consider adding iptables/nftables for staging.

### 3.5 Application Logging
No centralized log aggregation (Loki, Fluentd). Logs live in systemd-journald on each node. To see full system logs:
```bash
sudo usermod -aG adm ferdi   # then re-login
journalctl -u k0scontroller   # k0s control plane logs
kubectl get events --all-namespaces  # cluster events
```

---

## 4. What Needs to Be Done

### 4.1 For the Dev Teams (Unblocks Real Functionality)

- [x] **api-gateway skeleton** — `/v1/health`, `/v1/query`, `/metrics` endpoints implemented. Query is placeholder only.
- [x] **rag-service scaffold** — `POST /api/v1/ticket` accepts JSON, returns dummy response. No Qdrant/Mistral integration.
- [ ] **Write actual Rust source code** — teams need to implement:
  - `routes/query.rs` — POST /v1/query handler, Qdrant query, inference forward
  - `clients/qdrant.rs` — Qdrant HTTP client
  - `clients/inference.rs` — Mac Mini inference client
  - `models.rs` — matching `api_contract.md`
- [ ] **Create `qti_knowledge_base` collection** in Qdrant:
  ```bash
  kubectl port-forward -n qdrant svc/qdrant 6333:6333
  curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
    -H 'Content-Type: application/json' \
    -d '{"vectors": {"size": 1024, "distance": "Cosine"}}'
  ```
- [ ] **Set up the Mac Mini inference server** — the pipeline expects it at `INFERENCE_URL`. No server = `/v1/query` will error.
- [ ] **Add secrets** — `QDRANT_URL`, `INFERENCE_URL`, and any API keys should be Kubernetes Secrets, not hardcoded in the Deployment.
- [ ] **Add `.gitignore`** — `rag-service/target/` build artifacts are committed to the repo. Need to exclude `target/`, `*.pdb`, etc.

### 4.2 Infrastructure Improvements

- [ ] **Smoke test scope** — currently only checks `/v1/health` after build. Once Qdrant/inference code lands, add:
  - Query endpoint returns valid JSON matching the API contract
  - Qdrant connectivity responds
  - Response time under X seconds
- [x] **ServiceMonitor for api-gateway** — done. Prometheus scrapes `/metrics` every 15s via `servicemonitor.yaml`.
- [ ] **ServiceMonitor for Qdrant** — Qdrant exposes metrics at `/metrics` already. A simple `ServiceMonitor` would let you see Qdrant query latency, collection sizes, etc. in Grafana.
- [ ] **AlertManager** — currently disabled. Once the stack is stable, enable it for Slack/email alerts on pod crashes, image pull failures, etc.
- [ ] **Change Argo CD admin password** from the default.
- [ ] **TLS for Argo CD** — install cert-manager or configure SSL passthrough.
- [ ] **Ingress** — if the api-gateway needs external access, install nginx-ingress and create an `Ingress` resource.
- [x] **CI concurrency gate** — done. Only the latest push builds; old in-progress runs are cancelled.

### 4.3 Long-Term

- [ ] **Multi-environment** — replicate for production (separate namespace or cluster, Git branch, ApplicationSet).
- [ ] **Qdrant backup strategy** — NFS snapshots or Qdrant's built-in snapshot API.
- [ ] **Network policies** — restrict pod-to-pod traffic (only api-gateway → Qdrant, api-gateway → inference).
- [ ] **data-pipeline CI/CD** — the Python scraping pipeline needs its own Dockerfile, workflow, and deployment manifests (CronJob maybe).
- [ ] **GPU node for inference** — if the Mac Mini becomes a bottleneck, consider adding a GPU worker node to the cluster.

---

## 5. Quick Reference

```bash
# Check CI pipeline status
# https://github.com/Merpatidove/QTI-MAGANG/actions

# Check Argo CD app
kubectl -n argocd get application qti-api-gateway

# Force Argo CD resync
kubectl -n argocd patch application qti-api-gateway \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' \
  --type=merge

# Test api-gateway
kubectl -n qti run test --rm -i --restart=Never --image=curlimages/curl \
  -- curl -s http://api-gateway.qti.svc:8080/v1/health

# Qdrant health
kubectl -n qdrant exec qdrant-0 -- curl -s http://localhost:6333/healthz

# Grafana
# http://<node-ip>:30000 | admin / 8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx

# Create Qdrant collection
curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
  -H 'Content-Type: application/json' \
  -d '{"vectors": {"size": 1024, "distance": "Cosine"}}'
```

### Deploy Key Recovery (if /tmp is wiped)

```bash
# Keys are stored at:
ls -la ~/.ssh/deploy_key_qti ~/.ssh/notes_deploy_key

# Or regenerate and add to GitHub:
ssh-keygen -t ed25519 -C "vm-notes" -f ~/.ssh/notes_deploy_key -N ""
ssh-keygen -t ed25519 -C "github-actions-qti" -f ~/.ssh/deploy_key_qti -N ""
```

The private keys are also stored in GitHub Secrets (`DEPLOY_KEY` for QTI-MAGANG) — if this VM is ever rebuilt, you can pull them from there.
