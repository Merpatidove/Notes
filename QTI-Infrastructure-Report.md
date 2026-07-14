# QTI RAG Pipeline — Infrastructure Report

**Date:** 2026-07-14
**Cluster:** k0s v1.36.2+k0s (Debian 13 trixie)
**Repo:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG)

---

## 1. What Was Done Today

### 1.1 Fixed CSI NFS Driver (k0s Compatibility)

The NFS CSI node pods (`csi-nfs-node`) were stuck in `ContainerCreating` on both worker nodes.

**Root cause:** k0s stores kubelet data at `/var/lib/k0s/kubelet/`, not the standard `/var/lib/kubelet/`. The Helm chart defaulted to the standard path.

**Fix:** Upgraded the Helm release with the correct path:
```bash
helm upgrade csi-driver-nfs csi-driver-nfs/csi-driver-nfs -n kube-system \
  --set kubeletDir=/var/lib/k0s/kubelet \
  --set controller.replicas=1
```

**Takeaway:** Any Helm chart that mounts hostPath volumes from kubelet directories will fail on k0s. Always verify `kubeletDir` when deploying CSI drivers or similar system-level charts.

### 1.2 Installed Argo CD

- **Namespace:** `argocd`
- **Access:** NodePort `30080` (HTTP) / `30081` (HTTPS)
- **Insecure mode:** Enabled (`--insecure` flag) since no TLS cert is configured yet
- **Initial password:** `8H2RJZIZPHhsaRts` (from `argocd-initial-admin-secret`)
- **Auto-sync:** Enabled globally via `syncPolicy.automated`

### 1.3 Deployed Qdrant

- **Namespace:** `qdrant`
- **Storage:** NFS-backed PVC via `nfs-csi` storage class (10Gi)
- **Internal access only:** ClusterIP at `qdrant.qdrant.svc.cluster.local:6333`

**Gotcha:** The Helm chart key is `persistence.storageClassName`, NOT `persistence.storageClass`. The wrong key silently renders an empty `storageClassName`, leaving the PVC unbound. If you reinstall Qdrant, always use:
```bash
helm install qdrant qdrant/qdrant -n qdrant \
  --set persistence.storageClassName=nfs-csi \
  --set persistence.size=10Gi
```

### 1.4 Created CI/CD Pipeline

**Files added to `Merpatidove/QTI-MAGANG`:**

| File | Purpose |
|---|---|
| `api-gateway/Dockerfile` | Multi-stage Rust build (rust:1.82 builder -> debian:bookworm-slim runtime) |
| `api-gateway/k8s/deployment.yaml` | Deployment with liveness/readiness probes on `/v1/health` |
| `api-gateway/k8s/service.yaml` | ClusterIP service on port 8080 |
| `api-gateway/k8s/kustomization.yaml` | Image tag management (updated by CI) |
| `.github/workflows/ci.yml` | Build, push to ghcr.io, commit updated tag back to repo |
| `k8s/argocd/application.yaml` | Argo CD Application CRD |

**Pipeline flow:**
```
Push to main (api-gateway/**) 
  -> GitHub Actions builds Docker image 
  -> Pushes to ghcr.io/merpatidove/qti-api-gateway:<sha>
  -> Commits updated image tag to kustomization.yaml [skip ci]
  -> Argo CD auto-syncs to cluster
```

### 1.5 Generated SSH Deploy Key

- **Type:** ed25519
- **Public key:** Added as deploy key on GitHub (write access)
- **Private key:** Stored as `DEPLOY_KEY` repository secret
- **Used by:** GitHub Actions to push manifest updates back to the repo

---

## 2. Current Cluster State

### 2.1 Nodes

| Node | IP | CPU | RAM | Role |
|---|---|---|---|---|
| worker-1 | 10.20.20.202 | 4 cores | ~4GB | k0s worker |
| worker-2 | 10.20.20.200 | 4 cores | ~4GB | k0s worker |

### 2.2 Namespaces

| Namespace | Workload |
|---|---|
| `kube-system` | NFS CSI driver, CoreDNS, k0s system pods |
| `monitoring` | Prometheus + Grafana (kube-prometheus-stack) |
| `argocd` | Argo CD (7 pods) |
| `qdrant` | Qdrant vector database (StatefulSet) |
| `qti` | api-gateway (Deployment + Service, currently ImagePullBackOff) |

### 2.3 Resource Usage (at time of setup)

| Node | CPU | RAM |
|---|---|---|
| worker-1 | 2% | 19% |
| worker-2 | 10% | 45% |

**Plenty of headroom** for additional workloads.

---

## 3. Architecture Overview

```
                    ┌─────────────────────────────────────────────┐
                    │              GitHub (Merpatidove/QTI-MAGANG) │
                    │                                             │
                    │  api-gateway/    data-pipeline/   inference/ │
                    └──────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   GitHub Actions     │
                    │   (CI Pipeline)      │
                    │   Build -> ghcr.io   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │     Argo CD          │
                    │   (Auto-sync)        │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
┌─────────▼────────┐ ┌────────▼───────┐  ┌─────────▼────────┐
│   api-gateway     │ │    Qdrant      │  │  (Future: more)  │
│   Axum/Rust       │ │  Vector DB     │  │                  │
│   Port 8080       │ │  Port 6333     │  │                  │
└───────────────────┘ └────────────────┘  └──────────────────┘
          │
          │  HTTP
          │
┌─────────▼────────┐
│  Mac Mini         │
│  Inference Server │
│  Mistral-7B       │
│  mistral.rs       │
└──────────────────┘
```

### 3.1 Data Flow

1. **Client** sends `POST /v1/query` to **api-gateway**
2. **api-gateway** embeds the query and queries **Qdrant** for relevant chunks
3. **api-gateway** forwards query + chunks to **Mac Mini inference server**
4. **Mac Mini** runs Mistral-7B, returns structured JSON with answer + citations
5. **api-gateway** aggregates and returns the response to the client

### 3.2 Team Responsibilities

| Team | Component | Runs On |
|---|---|---|
| DevOps | api-gateway (Axum) | k0s cluster |
| Data Engineering | data-pipeline (Python) | TBD (likely CronJob or standalone) |
| Inference | mistral.rs server | Mac Mini (external) |

---

## 4. What Needs to Be Done Next

### 4.1 Immediate (Unblocks Development)

- [ ] **Add actual Rust source code** to `api-gateway/` — the pipeline is ready, but there's no `Cargo.toml` or `src/main.rs` yet. The Docker build will fail until these exist.
- [ ] **Create the `qti_knowledge_base` collection** in Qdrant:
  ```bash
  kubectl port-forward -n qdrant svc/qdrant 6333:6333
  curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
    -H 'Content-Type: application/json' \
    -d '{
      "vectors": {
        "size": 1024,
        "distance": "Cosine"
      }
    }'
  ```
- [ ] **Set up Argo CD authentication** — change the admin password, consider integrating with GitHub OAuth or a SSO provider.

### 4.2 Short-Term (Before Production)

- [ ] **TLS for Argo CD** — currently using `--insecure` mode. Either:
  - Install an Ingress controller (nginx/traefik) with cert-manager, or
  - Use a self-signed cert and configure SSL passthrough
- [ ] **Secrets management** — the api-gateway needs env vars like `QDRANT_URL`, `INFERENCE_URL`, and potentially API keys. Consider:
  - Sealed Secrets (gitops-friendly)
  - External Secrets Operator (with a cloud vault)
  - Or at minimum, a Kubernetes Secret that's not committed to git
- [ ] **Resource limits review** — current limits are conservative (100m-500m CPU, 128Mi-512Mi RAM). Monitor actual usage and adjust.
- [ ] **HorizontalPodAutoscaler** — once the api-gateway is serving traffic, add HPA for scaling.

### 4.3 Medium-Term

- [ ] **Data pipeline deployment** — the Python scraping/chunking/embedding pipeline could run as:
  - A Kubernetes CronJob (for periodic ingestion)
  - A standalone Deployment (for continuous processing)
  - Needs its own Dockerfile + k8s manifests
- [ ] **Monitoring for QTI workloads** — extend the existing Prometheus/Grafana stack:
  - ServiceMonitor for api-gateway metrics
  - Grafana dashboards for request latency, Qdrant query performance
- [ ] **Ingress** — when you need external access to the api-gateway:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: api-gateway
    namespace: qti
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - host: api.qti.local
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: api-gateway
              port:
                number: 8080
  ```

### 4.4 Long-Term / Production

- [ ] **Multi-environment** — if staging works well, replicate the pattern for production with:
  - Separate Argo CD Application (or ApplicationSet)
  - Separate k8s namespace or cluster
  - Git branch or directory-based environment separation
- [ ] **Backup strategy** for Qdrant (NFS snapshots or Qdrant's built-in snapshot feature)
- [ ] **Network policies** — restrict which pods can talk to Qdrant and the api-gateway

---

## 5. Best Practices & Notes

### 5.1 k0s-Specific

- **kubelet directory:** `/var/lib/k0s/kubelet/` (NOT `/var/lib/kubelet/`)
- **Storage classes:** `nfs-csi` is the only one available. Any Helm chart that specifies a different `storageClassName` will leave PVCs unbound.
- **Control plane:** Runs on a separate node (not one of the two workers). k0s manages its own containerd.

### 5.2 GitOps / Argo CD

- **Commit messages matter** — Argo CD uses them to display sync history. Use conventional commits (`feat:`, `chore:`, `fix:`).
- **`[skip ci]` in manifest update commits** — prevents infinite CI loops when the pipeline commits back to the repo.
- **Auto-sync + self-heal** — Argo CD will revert manual `kubectl edit` changes. This is by design.
- **Argo CD Application** lives in `k8s/argocd/application.yaml` — a good pattern for GitOps-of-GitOps. You can also register it via the UI or CLI.

### 5.3 CI/CD Pipeline

- **Image tags use git SHA** (`ghcr.io/...:<sha>`) — deterministic, traceable, no `latest` ambiguity.
- **Kustomize** is used for image tag management — `sed` updates the `newTag` field in `kustomization.yaml`.
- **GitHub Actions cache** is enabled (`cache-from/to: type=gha`) — Rust builds are slow, this significantly speeds up rebuilds.
- **`GITHUB_TOKEN`** is used for ghcr.io login (automatic, no extra secret needed). The **deploy key** is only for pushing manifest commits.

### 5.4 Security Observations

- The deploy key has write access to the repo — this is necessary for the pipeline but means a leaked key can push code. Keep the private key secure in GitHub Secrets.
- Argo CD admin password is static — change it after initial setup.
- Qdrant has no authentication enabled by default. For staging it's fine, but for production enable API key auth.
- The api-gateway connects to Qdrant over plain HTTP (cluster-internal). This is acceptable within the cluster but consider mTLS for production.

### 5.5 Peculiar / Interesting Observations

1. **k0s vs standard Kubernetes paths** — this is the #1 gotcha. Every Helm chart that touches kubelet directories needs `kubeletDir` overridden. This includes CSI drivers, device plugins, and monitoring agents.

2. **Mac Mini as an inference server** — unusual but practical. The Mistral-7B model benefits from Apple Silicon's unified memory architecture. The trade-off is that it's a single point of failure outside the cluster. Consider:
   - Health check from api-gateway with circuit breaker pattern
   - Fallback behavior when inference is unreachable
   - Whether a GPU-equipped worker node would be more reliable

3. **The repo is multi-team from day one** — `inference/`, `api-gateway/`, `data-pipeline/` with clear ownership. This is good but means the CI pipeline needs to be extended per-component. The current pipeline only handles `api-gateway/`. When `data-pipeline/` gets a Dockerfile, it'll need its own workflow or a matrix build.

4. **NFS as the only storage class** — works well for shared data but has latency implications for Qdrant's random I/O. Monitor query performance. If it becomes a bottleneck, consider local SSD storage for Qdrant on a specific node.

5. **Only 6 commits in the repo** — the project is very early stage. The infrastructure is set up ahead of the code, which is the right order. When the dev team starts pushing, the pipeline will just work.

---

## 6. Quick Reference

### Access

| Service | URL | Credentials |
|---|---|---|
| Argo CD UI | `http://<controller-ip>:30080` | admin / `8H2RJZIZPHhsaRts` |
| Grafana | `http://<any-node>:30000` | (check via `kubectl -n monitoring`) |
| Qdrant Dashboard | Port-forward: `kubectl port-forward -n qdrant svc/qdrant 6333:6333` | No auth |

### Useful Commands

```bash
# Check Argo CD app status
kubectl -n argocd get application qti-api-gateway

# Check all QTI workloads
kubectl -n qti get all

# View Argo CD logs
kubectl -n argocd logs deployment/argocd-server

# Force Argo CD sync
kubectl -n argocd patch application qti-api-gateway -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge

# Check Qdrant health
kubectl -n qdrant exec qdrant-0 -- curl -s http://localhost:6333/healthz

# Redeploy api-gateway
kubectl -n qti rollout restart deployment/api-gateway
```

### File Locations on This VM

| File | Path |
|---|---|
| Deploy key (private) | `/tmp/deploy_key` |
| Deploy key (public) | `/tmp/deploy_key.pub` |
| Cloned repo | `/tmp/QTI-MAGANG` |
