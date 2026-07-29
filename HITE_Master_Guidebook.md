# HITE — Master Infrastructure Guidebook

**Project:** Hybrid IT Triage Engine (HITE) — QTI-MAGANG
**Compiled:** 2026-07-30 (state reconciled against live machine 2026-07-30)
**Compiled by:** Johan (merged from 5 individual role reports, unedited content preserved below)

**Roles:**
- **Data Scientist** — Johan
- **Data Engineer** — Farrel
- **DevOps (CI/CD)** — Ferdi
- **DevOps (Prometheus / Grafana / Loki)** — Jep
- **Platform Engineering** — Hilmi

---

## 0. Editor's Note — What Was Changed When Merging

Nothing from the original five source files was deleted. Every table, checklist, code block, and "gotcha" is preserved somewhere below — this note tracks exactly where things moved.

1. **Reorganized by the five actual roles**, in this order: **Data Scientist (Johan) → Data Engineer (Farrel) → DevOps/CI-CD (Ferdi) → DevOps/Prometheus-Grafana-Loki (Jep) → Platform Engineering (Hilmi)**.
2. **Split the original "CI_CD_Automation.md" report in two.** That file was written as one combined DevOps report covering both the GitHub Actions/Argo CD pipeline *and* the Prometheus/Grafana/Loki/Jaeger/AlertManager stack. Since CI/CD and observability are separate roles (Ferdi vs. Jep), I split its content:
   - **Everything about the GitHub Actions pipeline, Argo CD, Docker registries, SSH deploy keys, k0s cluster security notes, ingress-nginx, and the `hite-prod` registry namespace → now under §3 (Ferdi).**
   - **Everything about Prometheus, Grafana, Loki, Promtail, Jaeger, and AlertManager → now under §4 (Jep)**, merged together with the separate Prometheus/Grafana observability roadmap document (which was always Jep's).
   - No row, checklist item, or command was dropped — each landed in whichever new section matches who actually owns that piece of work. Where an item was genuinely shared (e.g. Ingress-NGINX routes to both Argo CD *and* Grafana), it's kept once under the primary owner with a note pointing to the other section, rather than duplicated.
3. **Cleaned up escaping artifacts** in the Data Engineering file — the uploaded `.md` had literal backslashes before punctuation (e.g. `\*\*api-gateway\*\*`, `qti\_knowledge\_base`), a markdown-escaping export artifact, not content. I removed the stray backslashes so it renders as normal bold/inline-code. **No words, facts, numbers, or code were changed** — only the escape characters.
4. **Extracted the Prometheus/Grafana file** — `PrometheusGrafana.txt` was actually a `.docx` file saved with a `.txt` extension. I opened it and extracted the real Markdown content it contained. It's kept in Bahasa Indonesia exactly as written, now merged into Jep's section (§4) since that's the observability roadmap doc.
5. **Re-numbered all internal cross-references** (e.g. "see §3.1") to match the new section layout so they still point to the right place.
6. **§6 Mac Mini Full-Hosting Consolidated View and §7 Cross-Role Dependency Map** — both new, synthesized across all five reports, unchanged in substance from the previous version aside from updated role labels.
7. **2026-07-30 — State reconciled against live machine.** This update corrects every discrepancy found between the guidebook and the actual cluster/VM state on 2026-07-30:
   - **Jaeger removed.** Jaeger has been uninstalled (it was not needed). All references removed from §4 running-components tables, checklists, quick-ref commands, Argo CD lists, and Grafana data-source notes.
   - **Sudo count corrected.** 9 → 8 sudo users (hapip, mulyadi, arief, arip, grace, hilmi, zayfa, ferdi).
   - **`authorized_keys` noted.** `ferdi` now has `~/.ssh/authorized_keys` configured for key-based access.
   - **Qdrant curl corrected.** The stale `"size": 1024` in the create-collection example (§3.5) is now `384` to match the actual collection.
   - **Tailscale access documented.** All cluster hosts are reachable only via Tailscale — added §3.2.1 with IPs and SSH commands.
   - **`/tmp/argocd-tls/` cleaned.** The temporary CA cert directory no longer exists; the TLS cert lives in the `argocd-tls` Kubernetes secret.

---

## Table of Contents

- [§1 — Data Scientist (Owner: Johan)](#1-data-scientist-owner-johan)
- [§2 — Data Engineer (Owner: Farrel)](#2-data-engineer-owner-farrel)
- [§3 — DevOps: CI/CD (Owner: Ferdi)](#3-devops-cicd-owner-ferdi)
- [§4 — DevOps: Prometheus / Grafana / Loki (Owner: Jep)](#4-devops-prometheus--grafana--loki-owner-jep)
- [§5 — Platform Engineering (Owner: Hilmi)](#5-platform-engineering-owner-hilmi)
- [§6 — Mac Mini Full-Hosting: Consolidated View (NEW)](#6-mac-mini-full-hosting-consolidated-view-new)
- [§7 — Cross-Role Dependency & Blocker Map (NEW)](#7-cross-role-dependency--blocker-map-new)

---

## §1. Data Scientist (Owner: Johan)

### LLM Inference & 5W1H Evaluation Pipeline — Infrastructure & State Report

**Date:** 2026-07-29 **Owner:** Johan / Data Scientist **Repo/Path:** https://github.com/Merpatidove/QTI-MAGANG/tree/main/llm-inference

#### 1.1 What's Running / Current State

| Component / File | Status | Access / Details |
| :--- | :--- | :--- |
| `llm-inference/test_run.py` | Stable / Blocked | Ready to ping Rust API `/v1/query`. Blocked by cluster ingress configuration. |
| `llm-inference/grade_result.py` | Active / 100% Stable | Locally executed. Successfully parses `evaluation_results.json` without `JSONDecodeError`. |
| `evaluation_results.json` | Generated | Contains 55 synthetic test cases of placeholder Rust API responses. |
| Rust API (`/v1/query`) | Deployed (Mocked) | K0s cluster target: port 8080. Currently returns static `ticket_metadata` payload. |
| Qdrant DB (`qti_knowledge_base`) | Deployed (Empty) | Internal target: `http://qdrant.qdrant.svc.cluster.local:6333` (Namespace: `qdrant`). 10GB NFS PV on Mac Mini host. |

#### 1.2 Data scientist

The Data Science evaluation pipeline is currently executed manually to evaluate LLM inference outputs against a strict 5W1H (Who, What, When, Where, Why, How) schema.

- **Triggers:** Manual execution via `python test_run.py` to generate inference outputs, followed by `python grade_result.py` for parsing and validation.
- **Steps:**
  1. Ping Rust API backend to retrieve RAG-augmented LLM responses.
  2. Write raw responses to local `evaluation_results.json`.
  3. Parse JSON to validate structural integrity and confirm existence of all required 5W1H keys.
  4. Calculate baseline percentage scores.
- **Last Successful Run:** Ran against 55 test cases locally. Achieved 100% Valid JSON Responses rate. Schema completeness is at 0% explicitly because the live backend is serving a mock schema (`ticket_metadata`, `remediation_payload`) while awaiting Data Engineering vector population.

#### 1.3 Notable Observations & "Gotchas"

##### 1.3.1 k0s Networking on Mac Mini Host

Our K0s cluster does not expose internal services externally by default. Because the Mac Mini is hosting the cluster, local development environments cannot reach the internal DNS (`qdrant.qdrant.svc.cluster.local`) or the Rust API pod without an explicit ingress route. Platform Engineering must establish a NodePort, LoadBalancer, or issue `kubeconfig` RBACs for `kubectl port-forward` before Data Science E2E testing or Data Engineer DB population can proceed.

> **Cross-reference note:** this line says "the Mac Mini is hosting the cluster" — worth reconciling against Platform Engineering §5 (the k0s controller/workers are separate VM IPs `10.20.20.201/202/200`) and DevOps CI/CD §3.3.2 ("this VM is a single point of failure"). See §6 below for the consolidated Mac Mini picture and this exact discrepancy.

##### 1.3.2 Evaluation Schema Strictness

The `grade_result.py` script enforces a hard, exact-match key check for the 5W1H schema. If the backend Rust API returns *any* structure other than `Who`, `What`, `When`, `Where`, `Why`, and `How`, the test will record a 0% success rate for that specific run, regardless of JSON validity.

##### 1.3.3 Qdrant Database State

The `qti_knowledge_base` collection is initialized with a 384-dim Cosine configuration and its status is Green, but currently contains **0 points**. The Rust backend route `clients::qdrant::search_sop` will return no actionable vectors until the Data Engineer runs the document ingestion pipeline.

##### 1.3.4 Version Control Artifacts

All `__pycache__` artifacts and `.venv` environments have been actively purged and added to `.gitignore`. Never commit these files as they bloat the inference repository and trigger CI pipeline caching issues.

#### 1.4 What Needs to Be Done (TODOs)

**Platform Engineering Unblocks**
- [ ] Open K0s network route to expose Rust API (port 8080) to allow Data Science inference testing.
- [ ] Open K0s network route to expose Qdrant (port 6333) to allow Data Engineering SOP ingestion.

**Data Engineering Unblocks**
- [ ] Ingest the 18 SOPs from the RAG manual, generate embeddings, and populate the empty Qdrant vector database.
- [ ] Wire the Rust backend (`/v1/query`) to actively call `clients::qdrant::search_sop` instead of returning the placeholder payload.

**Data Science (Johan)**
- [ ] Run full 5W1H evaluation suite (`test_run.py` -> `grade_result.py`) against live vector data once the API backend is fully wired.
- [ ] Finalize the methodology documentation for the structured 5W1H prototype model analysis based on initial live pipeline metrics.

**DevOps (CI/CD & Prometheus)**
- [ ] Integrate `grade_result.py` into a CI/CD pipeline (e.g., GitHub Actions) for automated schema regression testing on all new commits.
- [ ] Automate Data Engineering SOP ingestion trigger for future file updates to the repository.
- [ ] Expose LLM token throughput, Qdrant latency, and JSON decode error metrics to Prometheus/Grafana once the pipeline goes live.

#### 1.5 Quick Reference (Cheat Sheet)

```bash
# Navigate to the Data Science working directory
cd llm-inference

# Run the LLM inference generator (Requires Platform Engineering network unblock on Port 8080)
python test_run.py

# Grade the newly generated evaluation_results.json file against the strict 5W1H baseline
python grade_result.py

# Safely commit verified LLM evaluation pipeline scripts (excluding .venv/pycache artifacts)
git add agent.py prompts.py test_run.py grade_result.py evaluation_results.json
git commit -m "feat(llm-inference): update baseline evaluations against live API route"
git push origin main
```

---

## §2. Data Engineer (Owner: Farrel)

### `api-gateway` + `data-pipeline` — Infrastructure & State Report

**Date:** 2026-07-29
**Owner:** Farrel — Data Engineering (GitHub push author: `K3tsuko`)
**Repo/Path:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG) → `api-gateway/` (live server) and `data-pipeline/` (run-once loader)

#### 2.1 What's Running / Current State

RAG is split across three owners. This report covers the two Rust crates I own: `api-gateway` = the **retrieve** path (online, always-on), `data-pipeline` = the **index** path (offline, run-once). Generation (Ollama + Qwen) is Data Science on the Mac Mini; the Qdrant **server** is DevOps. Qdrant is the shared store between my two crates.

| Component / File | Status | Access / Details |
|---|---|---|
| `api-gateway` Deployment (ns `qti`) | 1/1 Running, Healthy | `api-gateway.qti.svc:8080`. Image `ghcr.io/merpatidove/qti-api-gateway` — CI/CD report §3.1 records tag `e40ba85`; the modularized code (below) shipped at commit `1f34091` (Actions run #6, green). Confirm the live SHA at the Actions tab. |
| `api-gateway/src/main.rs` | Working, deployed | Orchestrator only (no business logic). Declares `mod models/routes/clients`, wires the router, exposes `/metrics`, binds `0.0.0.0:8080`. |
| `api-gateway/src/models.rs` | Working, deployed | `QueryRequest`, `QueryResponse`, `TicketMetadata`, `RemediationPayload` (serde; matches `api_contract.md`). |
| `api-gateway/src/routes/mod.rs` | Working, deployed | Module table of contents: `pub mod health; pub mod query;` |
| `api-gateway/src/routes/health.rs` | Working, deployed | `GET /v1/health` → `{"status":"ok","version":"0.1.0"}`; counter `health_checks_total`. K8s liveness/readiness target. |
| `api-gateway/src/routes/query.rs` | Working (**placeholder**), deployed | `POST /v1/query`. `Json(req): Json<QueryRequest>` validates the body (bad JSON → auto 422). Returns a **placeholder** `QueryResponse`; counter `queries_total`. |
| `api-gateway/src/clients/mod.rs` | Working, deployed | Module table of contents: `pub mod qdrant;` |
| `api-gateway/src/clients/qdrant.rs` | Written, compiled, **NOT wired** | `search_sop(Vec<f32>) -> Result<...>` via `reqwest` (REST, :6333). Constants `QDRANT_URL = http://qdrant.qdrant.svc.cluster.local:6333`, `COLLECTION_NAME = qti_knowledge_base`. Not yet called from `query.rs`. |
| `api-gateway/src/clients/inference.rs` | **NOT written** | Mac Mini inference client — possibly obsolete (see §2.3.6 / §2.4). |
| `api-gateway/Cargo.toml` | Working | axum 0.8, tokio `[full]`, serde `[derive]`, serde_json, reqwest 0.12 `[rustls-tls, json]`, tracing, tracing-subscriber `[env-filter]`, prometheus 0.13 `[process]`, lazy_static 1.4, anyhow 1.0. |
| `api-gateway/Dockerfile` | Working | Multi-stage `rust:1-bookworm` → `debian:bookworm-slim`; image ~32 MB. |
| `api-gateway/k8s/deployment.yaml` | Working | Liveness + readiness probes on `/v1/health`. |
| `api-gateway/k8s/service.yaml` | Working | ClusterIP on 8080. |
| `api-gateway/k8s/kustomization.yaml` | Working | `newTag: <sha>` managed by CI commit-back. |
| `api-gateway/k8s/servicemonitor.yaml` | Working | Prometheus scrapes `/metrics` every 15s. |
| `data-pipeline/src/main.rs` | Working (**parser only**), **NOT deployed** | Reads `RAG_Manual.md`, splits on `"\n# SOP-"`, prints all 18 SOPs (id + title). Chunk → embed → upsert **not written**. |
| `data-pipeline/Cargo.toml` | Working | Parser uses std only; staged for next step: `fastembed`, `qdrant-client`, `uuid`, `serde`, `serde_json`, `anyhow`, `tokio`. |
| `data-pipeline/RAG_Manual.md` | Present (data) | 18 structured SOPs — the ingestion source. |
| `data-pipeline/golden_datasets.json` | Present (data) | Sample tickets — DS evaluation harness; **not** consumed by the Rust code yet. |
| Collection `qti_knowledge_base` | Created, **green, 0 points** | **384-dim / Cosine** (NOT 1024 — see §2.3.1). Created by DevOps per my spec; empty until I run ingestion. |
| Gateway → Qdrant | Reachable (in-cluster) | REST `http://qdrant.qdrant.svc.cluster.local:6333`. |
| Pipeline → Qdrant | **NOT reachable yet** | Needs an external path (gRPC :6334 by default) — see §2.3.3 / §2.4. |
| `.gitignore` (repo root) | Added | `target/`, `*.pdb`, `*.exe`. |
| `rag-service/` | **DELETED** from repo | DevOps CI/CD report §3.1.2/§3.1.3 (see §3 below) still describe it — stale, see §2.3.5. |

**Expected placeholder response from `/v1/query`** (so testers don't file a bug):

```json
{"ticket_metadata":{"ticket_id":"","classification":"placeholder_classification"},"remediation_payload":{"proposed_fix":"Rust backend is not yet connected to Qdrant.","requires_type_check":false}}
```

#### 2.2 CI/CD & Automation (If applicable)

**`api-gateway` — working end-to-end.**

- **Trigger:** push to `main` touching `api-gateway/**`.
- **Steps:** GitHub Actions builds the Docker image (Rust multi-stage) → pushes `ghcr.io/merpatidove/qti-api-gateway:<sha>` → Docker smoke test (`/v1/health` must return 200) → commits the new image tag to `kustomization.yaml` with `[skip ci]` → Argo CD auto-syncs → pod restarts, health check passes.
- **Concurrency gate:** enabled — only the latest push builds; older in-progress runs are cancelled.
- **Rollback:** manual via `.github/workflows/rollback.yml` (specify a previous image SHA/tag).
- **Last runs:** modularization build = Actions run #6, ~2m47s, green; image ~32 MB; Argo deploy ~8s. DevOps CI/CD report §3.1 records latest tag `e40ba85` — verify the current SHA at https://github.com/Merpatidove/QTI-MAGANG/actions.

**`data-pipeline` — no CI/CD.** Local-only script today. A Dockerfile / workflow / CronJob is a long-term TODO (DevOps CI/CD report §3.4.3). Note: that report §3.4.3 mislabels this as the "Python scraping pipeline" — it is **Rust** (see §2.3.5).

#### 2.3 Notable Observations & "Gotchas"

##### 2.3.1 Vector dimension mismatch (most important — will silently break ingestion)

The embedding model is `all-MiniLM-L6-v2` → **384-dim**. The Platform/Infra report's create-collection `curl` (its §5.4/§5.5, see §5) says `"size": 1024` — that is a **stale placeholder** for a heavier model nobody uses. A 1024-dim collection **rejects every 384-dim upsert** with a dimension mismatch. The collection was created at **384 / Cosine** on purpose (DS-signed decision). **Do not** copy the infra report's `curl` as-is. If query-time embeddings ever move to Ollama, the Ollama embedder must also be 384-dim (Ollama's `all-minilm` matches) or search returns garbage — ingest-time and query-time vectors must share one vector space.

##### 2.3.2 Qdrant port / protocol split (the "connection refused" trap)

The two crates reach Qdrant over **different protocols and ports**:

- `api-gateway` uses a hand-rolled `reqwest` **REST** client → port **6333**.
- `data-pipeline` uses the official `qdrant-client` crate, which speaks **gRPC** → port **6334** by default.

Both ports must be reachable for the relevant caller, or one service connects and the other fails. (Alternative: switch the pipeline client to REST/6334→6333 if only one port can be exposed.)

##### 2.3.3 Network boundary — pipeline cannot reach Qdrant (current blocker)

`qdrant.qdrant.svc.cluster.local` resolves **only inside the cluster**. The gateway runs in-cluster, so it's fine. The `data-pipeline` runs **outside** the cluster (a laptop), so it cannot reach Qdrant directly — this is why ingestion is blocked. The Mac Mini is now the full host (Ollama + Qwen) and is on Tailscale at `100.79.30.90`; the reachable path for the pipeline will be either (a) run the pipeline on / over the Mac Mini, (b) a `kubectl port-forward` / SSH tunnel, or (c) a NodePort/LoadBalancer — pending DevOps confirmation. The gateway's hardcoded `QDRANT_URL` does **not** need to change for any of these (it already works in-cluster).

##### 2.3.4 Bind address must be `0.0.0.0`, never `127.0.0.1`

`main.rs` binds `0.0.0.0:8080`. Binding `127.0.0.1` would accept traffic only from inside the container, so the K8s liveness probe (from the node) would get no answer and the pod would `CrashLoopBackOff`. `0.0.0.0` is correct and intentional. (`localhost` / `127.0.0.1` only ever appears in human-run `port-forward` commands, never in source.)

##### 2.3.5 Stale facts in the Platform/Infra Report (do not trust these rows)

- DevOps CI/CD report §3.1.2/§3.1.3 list `rag-service/` as a scaffold and warn `rag-service/target/` is committed — **the folder is deleted**; those rows describe something that no longer exists.
- DevOps CI/CD report §3.4.1 "Write actual Rust source code" still ticks `models.rs`, `routes/query.rs`, `clients/qdrant.rs` as TODO — **all three are written and deployed**; only `clients/inference.rs` remains.
- DevOps CI/CD report §3.4.1 create-collection `curl` says `1024` — wrong, use `384` (§2.3.1).
- DevOps CI/CD report §3.4.1 "Add `.gitignore`" — **done** (root `.gitignore`).
- DevOps CI/CD report §3.4.3 calls `data-pipeline` the "Python scraping pipeline" — it is **Rust**.

##### 2.3.6 `clients/inference.rs` may be obsolete

The Python agent orchestrates and calls Ollama/Qwen **directly** for generation, calling the gateway only for RAG context. If that design holds, the gateway is **retrieval-only** and never forwards to an inference server → `clients/inference.rs` is not needed (one less file). Confirm with DS before building it.

##### 2.3.7 Git `target/` trap

`.gitignore` ignores **only untracked** files. The `api-gateway/target` and `data-pipeline/target` folders were untracked, so creating the root `.gitignore` excluded them. But `rag-service/target/` had been **committed** earlier (DevOps), so `.gitignore` alone would not have stopped tracking it — deleting the `rag-service/` folder is what removed it. If a `target/` ever shows up staged again: `git rm -r --cached <path>` (the `--cached` flag keeps the files on disk). Stage source surgically (`git add api-gateway/src api-gateway/Cargo.toml ...`) rather than `git add .` until `.gitignore` is confirmed.

##### 2.3.8 Apple Silicon build note (Mac Mini)

The Mac Mini is ARM. The first `cargo build` of `data-pipeline` there recompiles `fastembed` / onnxruntime for `aarch64` — it works, but the first build is slow (minutes). Not a failure.

##### 2.3.9 Qdrant has no auth; secrets are TODO

Qdrant currently has **no authentication** (DevOps CI/CD report §3.3.2). My constants hardcode the URL; per §2.4 these should move to K8s Secrets (`QDRANT_URL`, and `INFERENCE_URL` if inference stays) mounted as env vars — that is a DevOps deliverable; my code just needs to read the env var instead of the constant when it exists.

##### 2.3.10 Rust module system (code-structure gotcha)

Rust does not auto-scan folders. Every new file needs a `mod` declaration in `main.rs` (top-level) **and** a `pub mod` line in the folder's `mod.rs`, or you get `unresolved import` / "file ignored". The `pub` on structs/functions is also mandatory for cross-module use. Relevant whenever anyone adds a file to either crate.

##### 2.3.11 `data-pipeline` reads `RAG_Manual.md` by relative path

`fs::read_to_string("RAG_Manual.md")` resolves against the **current working directory**, so `cargo run` must be executed from inside `data-pipeline/` with the manual present there, or it panics with the "Failed to read RAG_Manual.md" message.

#### 2.4 What Needs to Be Done (TODOs)

**My lane — shipped**

- [x] Modularize `api-gateway` (`models.rs`, `routes/health.rs`, `routes/query.rs`, `clients/qdrant.rs`, rewired `main.rs`).
- [x] `/v1/health` live and passing the CI smoke test.
- [x] `/v1/query` accepts + validates JSON (placeholder response).
- [x] `clients/qdrant.rs::search_sop` written and compiling.
- [x] Root `.gitignore` added; `rag-service/` removed from the repo.
- [x] `data-pipeline` parser isolates all 18 SOPs.
- [x] `qti_knowledge_base` collection created at 384 / Cosine (via DevOps, per my spec).

**My lane — pending (the real remaining work)**

- [ ] **Reachable Qdrant path for `data-pipeline`** — get the external address/port (Mac Mini / Tailscale `100.79.30.90`, port-forward, or NodePort) from DevOps. *Currently the blocker.*
- [ ] **Populate the collection** — finish `data-pipeline`: chunk → embed with `all-MiniLM-L6-v2` (384-dim) → upsert with UUID-v4 point IDs and payload `{text, sop_id, title, category, tier}`. Turns 0 points into the 18 SOPs.
- [ ] **Wire `/v1/query`** to call `clients::qdrant::search_sop` so it returns real SOP text instead of the placeholder.
- [ ] **Confirm `clients/inference.rs`** is needed; if the gateway is retrieval-only (§2.3.6), do not build it.
- [ ] Move hardcoded `QDRANT_URL` to an env var once DevOps mounts the `QDRANT_URL` Secret (§2.3.9).

**Long-term**

- [ ] `data-pipeline` CI/CD — Dockerfile + workflow + CronJob (DevOps CI/CD report §3.4.3).
- [ ] Embedding consistency guard — assert at startup that the configured embedder's dimension equals the collection's vector size, to make §2.3.1 fail loudly instead of per-upsert.

#### 2.5 Quick Reference (Cheat Sheet)

```bash
# --- api-gateway: fast local compile check (no binary produced) ---
cd api-gateway && cargo check

# --- api-gateway: build + run locally (binds 0.0.0.0:8080) ---
cd api-gateway && cargo run

# --- data-pipeline: run the SOP parser. MUST run from inside data-pipeline/ (RAG_Manual.md is a relative path; see §2.3.11) ---
cd data-pipeline && cargo run

# --- /v1/health from inside the cluster (expect: {"status":"ok","version":"0.1.0"}) ---
kubectl -n qti run test-health --rm -i --restart=Never --image=curlimages/curl -- curl -s http://api-gateway.qti.svc:8080/v1/health

# --- /v1/query (POST). Body MUST match QueryRequest exactly, else Axum returns 422. Response is the PLACEHOLDER for now (§2.1) — not a bug. ---
kubectl -n qti run test-query --rm -i --restart=Never --image=curlimages/curl -- curl -s -X POST http://api-gateway.qti.svc:8080/v1/query -H "Content-Type: application/json" -d '{"ticket_id":"TKT-1001","raw_text":"nginx bind failed","project_tags":["docker","nginx"]}'

# --- /metrics (Prometheus text). Hit the two endpoints above first; expect health_checks_total / queries_total to be > 0 ---
kubectl -n qti run test-metrics --rm -i --restart=Never --image=curlimages/curl -- curl -s http://api-gateway.qti.svc:8080/metrics

# --- Qdrant: tunnel the REST port to localhost (run on a kubectl-equipped host; leave the process running) ---
kubectl port-forward -n qdrant svc/qdrant 6333:6333

# --- Qdrant: CREATE the collection at the CORRECT size 384. OVERRIDES the infra report's 1024 (§2.3.1). Returns {"result":true,"status":"ok",...} ---
curl -X PUT http://localhost:6333/collections/qti_knowledge_base -H 'Content-Type: application/json' -d '{"vectors": {"size": 384, "distance": "Cosine"}}'

# --- Qdrant: verify schema + point count (expect vectors.size 384, distance Cosine; points_count 0 until ingestion runs) ---
curl -s http://localhost:6333/collections/qti_knowledge_base | head -c 400

# --- Qdrant: liveness ---
kubectl -n qdrant exec qdrant-0 -- curl -s http://localhost:6333/healthz

# --- Mac Mini (full host: Ollama + Qwen) over Tailscale. Template: substitute the Mac Mini login for <user> (only the Tailscale IP 100.79.30.90 is known here). Confirms whether the Mac Mini can see the cluster (decides the §2.3.3 path). ---
ssh <user>@100.79.30.90 'kubectl get pods -n qdrant'

# --- Git: the target/ trap (§2.3.7). Untrack build artifacts WITHOUT deleting them from disk; safe to run even if already clean ---
git rm -r --cached api-gateway/target data-pipeline/target 2>/dev/null; git status

# --- CI: watch the build ---
# https://github.com/Merpatidove/QTI-MAGANG/actions

# --- Force Argo CD to pick up a freshly committed image tag ---
kubectl -n argocd patch application qti-api-gateway -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge
```

---

## §3. DevOps: CI/CD (Owner: Ferdi)

### QTI RAG Pipeline — CI/CD, Argo CD & Cluster Infrastructure Report

**Date:** 2026-07-17 (Updated 2026-07-30 — ServiceMonitors, Argo CD TLS, Tailscale access documented)
**Cluster:** k0s v1.36.2+k0s (Debian 13 trixie)
**Repo:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG)

> This section covers the CI/CD pipeline, Argo CD, container registries, SSH deploy keys, and general cluster/security notes. The Prometheus/Grafana/Loki/AlertManager observability stack that was originally reported alongside this content now lives in **§4 (Jep's section)** — see the editor's note at the top of this document for why it was split.

#### 3.1 What's Running on the Cluster (CI/CD & Platform components)

| Component | Status | Access |
|---|---|---|
| **Argo CD** | 7/7 pods Running | `https://argocd.hite.local` (admin / `12qwaszx`) |
| **Qdrant** | 1/1 Running | `qdrant.qdrant.svc.cluster.local:6333`, NFS-backed PVC (10Gi) |
| **api-gateway** | 1/1 Running, Healthy | `api-gateway.qti.svc:8080` — returns `{"status":"ok","version":"0.1.0"}` |
| **NFS CSI driver** | 3/3 controller, 2/2 node pods | k0s path: `/var/lib/k0s/kubelet` |
| **Ingress-NGINX** | 1/1 Running (LoadBalancer) | NodePort 31084 (HTTP), 30616 (HTTPS) — routes to Grafana, Prometheus, AlertManager, ArgoCD. *(Grafana/Prometheus/AlertManager themselves are Jep's — see §4.)* |
| **Local Registry** | Running on controller (HTTPS, self-signed cert) | `10.20.20.201:5000` — stores all deployment images |
| **hite-prod Registry** | 1/1 Running (Deployment) | `private-registry-svc.hite-prod:5000` (NodePort 32000) — Kubernetes-native registry:2 |

> **Note:** the original combined report also listed Prometheus/Grafana, Loki, Promtail, and AlertManager in this same table — those rows are preserved in §4.1 (Jep's section) instead of being duplicated here.

##### 3.1.1 CI/CD Pipeline (Working End-to-End)

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

##### 3.1.2 Files Created in QTI-MAGANG Repo

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

> **Note (per §2.3.5 above, from Data Engineering):** the `rag-service/*` rows in this table are now **stale** — the folder has since been deleted from the repo. Kept here unedited as the original report recorded it.

##### 3.1.3 rag-service (New Component)

A separate Rust/Axum service (`rag-service/`) has been scaffolded as the RAG inference engine:

- **Package name:** `inference-engine` (v0.1.0)
- **Endpoint:** `POST /api/v1/ticket` on port 3000
- **Models:** `TicketRequest` (ticket_id, raw_text, project_tags), `InferenceResponse` (status, message)
- **Status:** Placeholder only — accepts JSON, logs ticket ID, returns dummy response. No Qdrant or Mistral integration yet.
- **Not deployed** — no K8s manifests or CI pipeline for this service yet.

> **Note:** The `rag-service/target/` directory (compiled Rust artifacts) is committed to the repo. A `.gitignore` should be added.

#### 3.2 SSH Deploy Keys

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

##### 3.2.1 Tailscale Access

All cluster VMs are reachable exclusively via the Tailscale mesh (no public IPs, no office-network bridge).

| Host | Tailscale IP | Cluster Role |
|---|---|---|
| `debian13` (controller) | `100.94.99.125` | k0s controller, Docker registry, SSH jump host |
| `worker-2` | `100.106.122.68` | k0s worker node (10.20.20.200) |
| `worker-1` | `100.68.225.41` | k0s worker node (10.20.20.202) |

```bash
# SSH to controller
ssh ferdi@100.94.99.125

# Port-forward a service through the controller (e.g. Qdrant REST API)
ssh -L 6333:qdrant.qdrant.svc.cluster.local:6333 ferdi@100.94.99.125

# From the controller, reach workers via Tailscale
ssh ferdi@100.68.225.41   # worker-1
ssh ferdi@100.106.122.68  # worker-2
```

#### 3.3 Notable Observations

##### 3.3.1 k0s-Specific
- **kubelet directory:** `/var/lib/k0s/kubelet/` (NOT `/var/lib/kubelet/`). Any Helm chart with `kubeletDir` must override it.
- **Storage class:** `nfs-csi` is the only one. The Qdrant Helm chart key is `persistence.storageClassName`, NOT `persistence.storageClass`. *(NFS storage class itself is provisioned by Platform Engineering — see §5.1.)*

##### 3.3.2 Security
- **No firewall** on the controller VM — `iptables`, `ufw`, and `nftables` are all absent. NFS, k0s API, Docker Swarm ports are exposed.
- **No SSH keys** for any user except `ferdi` (has `~/.ssh/authorized_keys` configured for key-based access). All other SSH access is key-based via Tailscale.
- **8 sudo users**, only 2 actively used (ferdi, hapip).
- **Argo CD** — admin password changed, TLS enabled via self-signed cert + ingress.
- **Qdrant has no authentication**.
- **This VM is a single point of failure** — k0s controller, NFS server, Docker host all run here. No HA.
- **Telegram bot token in plaintext** — stored in AlertManager Secret and Helm values (AlertManager itself is Jep's — see §4.3). Rotate if the token is reused elsewhere.

##### 3.3.3 Ollama Was Removed
Ollama was running on this controller VM, redundant with the Mac Mini inference server. Removed entirely:
- Binary, service file, user, group, data directory all deleted.
- `ferdi` removed from `ollama` group.

##### 3.3.4 No Firewall on the Controller
The controller has all ports open (NFS, k0s API, Docker Swarm). Consider adding iptables/nftables for staging.

##### 3.3.5 Local Docker Registry

A local HTTPS Docker registry is running on the controller for storing deployment images:
- **Address:** `10.20.20.201:5000` (HTTPS, self-signed cert for IP 10.20.20.201)
- **Cert files:** `/tmp/certs/registry.crt`, `/tmp/certs/registry.key`
- **Docker trust cert:** `/etc/docker/certs.d/10.20.20.201:5000/ca.crt`
- **Containerd trust:** Workers configured via `/etc/k0s/containerd/certs.d/10.20.20.201:5000/hosts.toml` and `/etc/k0s/containerd.d/registry-certs.toml`

**Currently stored images:**

| Image | Tag |
|---|---|
| `grafana/loki` | `2.9.3` |
| `grafana/promtail` | `3.5.1` |
| `busybox` | `1.36` |

**Pushing new images:**
```bash
docker tag <image> 10.20.20.201:5000/<image>:<tag>
docker push 10.20.20.201:5000/<image>:<tag>
```

**Note:** Worker nodes have DNS UDP (port 53) blocked — they cannot resolve external registries (Docker Hub, etc.). All images must come from the local registry or be pre-loaded.

##### 3.3.6 Ingress-NGINX (Deployed 2026-07-16)

nginx ingress controller running as a LoadBalancer in `ingress-nginx` namespace (NodePort 31084/30616).

**Ingress resources:**

| Ingress | Host | Backend | Created |
|---|---|---|---|
| `grafana-ingress` | `grafana.hite.local` | `prometheus-grafana:80` | 2026-07-16 |
| `prometheus-ingress` | `prometheus.hite.local` | `prometheus-kube-prometheus-prometheus:9090` | 2026-07-22 |
| `alertmanager-ingress` | `alertmanager.hite.local` | `prometheus-kube-prometheus-alertmanager:9093` | 2026-07-22 |
| `argocd-server` | `argocd.hite.local` | `argocd-server:443` (HTTPS backend) | 2026-07-27 |

All ingress rules use `ingressClassName: nginx` and `pathType: Prefix`. Hostnames are `.hite.local` (requires /etc/hosts or internal DNS). Argo CD ingress uses TLS termination at the ingress with self-signed cert (`argocd-tls` secret). CA cert was temporarily stored at `/tmp/argocd-tls/ca.crt` during setup but has since been cleaned up — the TLS cert is stored in the `argocd-tls` Kubernetes secret and still functions.

> Three of these four ingress rules route to Jep's observability stack (Grafana, Prometheus, AlertManager) — kept here since the ingress controller itself is a shared CI/CD-owned resource.

##### 3.3.7 hite-prod Namespace + Private Registry (Deployed 2026-07-16)

A `hite-prod` namespace runs a Kubernetes-native Docker registry (official `registry:2` image) as a Deployment with a NodePort Service on `32000`.

| Resource | Details |
|---|---|
| Namespace | `hite-prod` |
| Deployment | `private-registry` — 1 replica, `registry:2` |
| Service | `private-registry-svc` — NodePort 5000:32000 |

This is separate from the host-level Docker registry at `10.20.20.201:5000`. Two registries now exist on the cluster.

##### 3.3.8 Argo CD Repo-Server Connectivity Issue

The Argo CD repo-server (`10.109.94.133:8081`) was intermittently unreachable from the application controller via ClusterIP. This was a kube-router networking issue between nodes. The repo-server pod was always healthy (verified via port-forward). After a repo-server restart, the issue resolved — both Loki and qti-api-gateway Applications are now `Synced` / `Healthy`. *(Loki is Jep's — see §4; api-gateway is Ferdi's — see §3.1.1.)*

#### 3.4 What Needs to Be Done

##### 3.4.1 For the Dev Teams (Unblocks Real Functionality)

- [x] **api-gateway skeleton** — `/v1/health`, `/v1/query`, `/metrics` endpoints implemented. Query is placeholder only.
- [x] **rag-service scaffold** — `POST /api/v1/ticket` accepts JSON, returns dummy response. No Qdrant/Mistral integration.
- [ ] **Write actual Rust source code** — teams need to implement: *(per Data Engineering §2.3.5 above, `models.rs`/`routes/query.rs`/`clients/qdrant.rs` are actually already done — only `clients/inference.rs` remains, and may be unnecessary — see §2.3.6)*
  - `routes/query.rs` — POST /v1/query handler, Qdrant query, inference forward
  - `clients/qdrant.rs` — Qdrant HTTP client
  - `clients/inference.rs` — Mac Mini inference client
  - `models.rs` — matching `api_contract.md`
- [ ] **Create `qti_knowledge_base` collection** in Qdrant: *(per Data Engineering §2.3.1 above, use size 384, NOT 1024 — the `curl` below is stale)*
  ```bash
  kubectl port-forward -n qdrant svc/qdrant 6333:6333
  curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
    -H 'Content-Type: application/json' \
    -d '{"vectors": {"size": 1024, "distance": "Cosine"}}'
  ```
- [ ] **Set up the Mac Mini inference server** — the pipeline expects it at `INFERENCE_URL`. No server = `/v1/query` will error.
- [ ] **Add secrets** — `QDRANT_URL`, `INFERENCE_URL`, and any API keys should be Kubernetes Secrets, not hardcoded in the Deployment.
- [ ] **Add `.gitignore`** — *(done — see Data Engineering §2.3.5)* `rag-service/target/` build artifacts are committed to the repo. Need to exclude `target/`, `*.pdb`, etc.

##### 3.4.2 Infrastructure Improvements (CI/CD & Argo CD ownership)

- [ ] **Smoke test scope** — currently only checks `/v1/health` after build. Once Qdrant/inference code lands, add:
  - Query endpoint returns valid JSON matching the API contract
  - Qdrant connectivity responds
  - Response time under X seconds
- [x] **Change Argo CD admin password** — changed from default. Initial admin secret deleted.
- [x] **TLS for Argo CD** — done. Self-signed CA + server cert for `argocd.hite.local`. TLS terminates at nginx-ingress, HTTPS backend to Argo CD. See §3.3.6.
- [x] **Ingress** — nginx-ingress deployed, Ingress resources for Grafana, Prometheus, AlertManager on `.hite.local` hosts. See §3.3.6.
- [x] **CI concurrency gate** — done. Only the latest push builds; old in-progress runs are cancelled.

> The remaining infra-improvement checklist items from the original combined report (ServiceMonitors, AlertManager, centralized logging, Loki in Argo CD, Grafana Loki data source) are **Jep's** and now live in §4.2.

##### 3.4.3 Long-Term (CI/CD & Platform)

- [ ] **Multi-environment** — replicate for production (separate namespace or cluster, Git branch, ApplicationSet).
- [ ] **Qdrant backup strategy** — NFS snapshots or Qdrant's built-in snapshot API.
- [ ] **Network policies** — restrict pod-to-pod traffic (only api-gateway → Qdrant, api-gateway → inference).
- [ ] **data-pipeline CI/CD** — *(the pipeline is Rust, not Python — see Data Engineering §2.3.5)* the scraping pipeline needs its own Dockerfile, workflow, and deployment manifests (CronJob maybe).
- [ ] **GPU node for inference** — if the Mac Mini becomes a bottleneck, consider adding a GPU worker node to the cluster.

#### 3.5 Quick Reference (CI/CD)

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

# Create Qdrant collection
curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
  -H 'Content-Type: application/json' \
  -d '{"vectors": {"size": 384, "distance": "Cosine"}}'

# Push image to local registry
docker tag <image>:<tag> 10.20.20.201:5000/<image>:<tag>
docker push 10.20.20.201:5000/<image>:<tag>

# Ingress — list all
kubectl get ingress --all-namespaces

# hite-prod registry — test
curl -k https://<node-ip>:32000/v2/_catalog
```

##### Deploy Key Recovery (if /tmp is wiped)

```bash
# Keys are stored at:
ls -la ~/.ssh/deploy_key_qti ~/.ssh/notes_deploy_key

# Or regenerate and add to GitHub:
ssh-keygen -t ed25519 -C "vm-notes" -f ~/.ssh/notes_deploy_key -N ""
ssh-keygen -t ed25519 -C "github-actions-qti" -f ~/.ssh/deploy_key_qti -N ""
```

The private keys are also stored in GitHub Secrets (`DEPLOY_KEY` for QTI-MAGANG) — if this VM is ever rebuilt, you can pull them from there.

---

## §4. DevOps: Prometheus / Grafana / Loki (Owner: Jep)

> This section merges two originally separate documents that both belong to the same role: (a) the observability-stack portion of the combined DevOps report (Prometheus, Grafana, Loki, Promtail, AlertManager), split out from what's now §3 (Ferdi/CI-CD), and (b) the standalone Observability Roadmap document. Nothing was cut — see the editor's note at the top for exactly what moved where.

### 4.1 What's Running — Observability Stack

| Component | Status | Access |
|---|---|---|
| **Prometheus/Grafana** | All targets up, 29 dashboards | `http://<node-ip>:30000` (admin / `8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx`) |
| **Loki** | 1/1 Running (StatefulSet) | `loki.monitoring.svc:3100` — log aggregation backend |
| **Promtail** | 2/2 Running (DaemonSet, both nodes) | Ship logs from all nodes to Loki |
| **AlertManager** | 2/2 Running (StatefulSet) | `prometheus-kube-prometheus-alertmanager.monitoring:9093` — Telegram notifications active |

> Ingress routes for Grafana/Prometheus/AlertManager are managed by the shared nginx ingress controller — see §3.3.6 (Ferdi).

#### 4.1.1 Application Logging (Loki + Promtail)

Centralized log aggregation is running via **Loki + Promtail**:
- **Loki** deployed as a StatefulSet in `monitoring` namespace (NFS-backed PVC)
- **Promtail** deployed as a DaemonSet on both worker nodes — ships all container/system logs to Loki
- **Loki endpoint:** `http://loki.monitoring.svc:3100/loki/api/v1/push`
- **Grafana integration:** Loki already provisioned as a data source via `loki-loki-stack` ConfigMap (label `grafana_datasource: "1"`). Alertmanager data source also pre-configured in `prometheus-kube-prometheus-grafana-datasource`.

To query logs via Grafana UI:
```
1. Open Grafana at http://<node-ip>:30000
2. Go to Explore → select "Loki" data source
3. Query: {namespace="monitoring"} or {job="varlogs"}
```

To query logs via CLI:
```bash
curl -G http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="monitoring"}' \
  --data-urlencode 'limit=10'
```

#### 4.1.2 Monitoring Stack — What Was Done (2026-07-17)

1. Set up local Docker registry at `10.20.20.201:5000` with HTTPS self-signed cert *(registry itself is Ferdi's — see §3.3.5; noted here because it's how the monitoring images below were staged)*
2. **Pushed all deployment images** to the local registry (Loki, Promtail, busybox)
3. **Fixed containerd trust on workers:**
   - Worker nodes couldn't pull from local HTTPS registry (`x509: certificate signed by unknown authority`)
   - k0s `containerd.configOverride` does not propagate to workers in v1.36
   - Deployed a privileged DaemonSet using cached NFS plugin image (`registry.k8s.io/sig-storage/nfsplugin:v4.13.4`) to nsenter into host and:
     - Installed CA cert into system trust store
     - Created `/etc/k0s/containerd/certs.d/10.20.20.201:5000/hosts.toml` with CA reference
     - Created `/etc/k0s/containerd.d/registry-certs.toml` to set `config_path` for CRI plugin
     - Restarted containerd on both workers
4. **Verified Loki + Promtail** working (was already deployed via `loki-stack` Helm chart)
5. **Loki transferred to Argo CD** — old Helm release uninstalled. Application is `Synced` / `Healthy`. *(Argo CD ownership/sync mechanics are Ferdi's — see §3.3.8.)*

#### 4.1.3 AlertManager + Telegram Notifications (Deployed 2026-07-22)

AlertManager is deployed as a StatefulSet in `monitoring` namespace, pulling its image from the local registry (`10.20.20.201:5000/prometheus/alertmanager:v0.33.1`).

**Telegram receivers configured:**

| Receiver | Chat ID | Purpose |
|---|---|---|
| `telegram-notif` | `6983435580` | Default — all alerts |
| `telegram-team-group` | `-1004300626827` | `HITE_Log*` alerts only |

**Routing:** 30s group_wait, 5m group_interval, 3h repeat_interval. `Watchdog` alert suppressed (sent to null receiver).

**Custom PrometheusRule `hite-infra-alerts`** (created 2026-07-22):

| Alert | Severity | Trigger |
|---|---|---|
| `HITE_NodeCPUHigh` | warning | CPU > 90% for 5m |
| `HITE_NodeMemoryHigh` | warning | Memory > 85% for 5m |
| `HITE_NodeDiskHigh` | critical | Disk > 90% for 5m |
| `HITE_PodCrashLooping` | critical | CrashLoopBackOff for 2m |
| `HITE_NodeDown` | critical | node-exporter unreachable 2m |
| `HITE_QdrantDown` | critical | Qdrant unreachable 2m |

Alert annotations are in Indonesian (e.g., "CPU node tinggi di atas 90% selama 5 menit").

> **Security note:** The Telegram bot token is stored in plaintext in the AlertManager Secret and Helm values. Consider rotating if this token is used elsewhere.

### 4.2 What Needs to Be Done — Observability

- [x] **ServiceMonitor for api-gateway** — done. Prometheus scrapes `/metrics` every 15s via `servicemonitor.yaml`.
- [x] **ServiceMonitor for Qdrant** — done. `qdrant-monitor` deployed in `qdrant` namespace, scrapes `/metrics` every 30s.
- [x] **AlertManager** — deployed with Telegram notifications (2 receivers). Custom alerts in `hite-infra-alerts` PrometheusRule. See §4.1.3.
- [x] **Centralized logging (Loki + Promtail)** — done. Logs ship from both nodes to Loki. Loki data source already provisioned in Grafana via ConfigMap.
- [x] **Loki in Argo CD** — Application created (`k8s/loki/application.yaml`). Synced/Healthy. Old Helm release uninstalled.
- [x] **Grafana Loki data source** — provisioned via `loki-loki-stack` ConfigMap. Alertmanager data source also configured.
- [ ] **Expose LLM token throughput, Qdrant latency, and JSON decode error metrics** to Prometheus/Grafana once the pipeline goes live (see Data Scientist §1.4).

### 4.3 Quick Reference (Observability)

```bash
# Grafana
# http://<node-ip>:30000 | admin / 8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx

# Loki log query (via port-forward)
kubectl port-forward -n monitoring svc/loki 3100:3100 &
curl -G http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="monitoring"}' \
  --data-urlencode 'limit=10'

# All pods in monitoring namespace
kubectl get pods -n monitoring -o wide

# AlertManager — check status
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager

# AlertManager — view config
kubectl get secret -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Custom alerts — verify loaded
kubectl get prometheusrule hite-infra-alerts -n monitoring
```

---

### 4.4 Observability Roadmap (originally a separate document)

> Kept in the original Bahasa Indonesia, unedited, as extracted from the source `.docx`.

# ROADMAP & PROGRESS REPORT — HITE OBSERVABILITY STACK

## BAGIAN 1 — Apa yang Sedang Dibangun (Scope Penuh)

### Hybrid IT Triage Engine (HITE)
Sistem **AI Ticket Triage**, *fully on-premise*, tanpa data keluar ke internet.

```text
┌─────────────────────────────────────────────────────────┐
│ Data Pipeline (Scraping → Chunking → Qdrant)              │
├─────────────────────────────────────────────────────────┤
│ API Gateway (Rust/Axum) — terima ticket, forward request   │
├─────────────────────────────────────────────────────────┤
│ Inference Engine (Mac Mini) — LLM, retrieval, keputusan     │
│   Output: confidence_tier (A/B/C), routing_decision         │
├─────────────────────────────────────────────────────────┤
│ Observability Layer (yang sedang kita bangun)                │
│   Metrics + Logs + Alerting + Dashboard                     │
└─────────────────────────────────────────────────────────┘
```

---

## BAGIAN 2 — Progress Detail Per Phase

| Phase | Nama Phase | Status | Catatan / Highlight |
| --- | --- | --- | --- |
| **Phase 1** | Audit Infrastruktur | ✅ | Hostname, timezone, swap, firewall, kernel modules, containerd, CoreDNS, storage class (`nfs-csi`) sehat. |
| **Phase 2** | Diagram Arsitektur | ⏭️ | Dilewati atas kesepakatan, tidak menghalangi progress. |
| **Phase 3** | Instalasi Stack Inti | ✅ | Prometheus, Grafana, Loki + Promtail, Alertmanager running. *Jaeger telah dihapus (removed).* |
| **Phase 5** | Konfigurasi Production | ⚠️ | Datasource & Ingress siap, Contact Point Telegram aktif. *Retention Loki belum diatur*. |
| **Phase 6** | Dashboard | ⚠️ | Node Exporter & K8s Monitoring siap. *Dashboard bisnis HITE blocked (nunggu Farrel)*. |
| **Phase 7** | Alert Rules | ⚠️ | 12 alert infra + 1 log-based alert aktif. *Alert `HITE_ApiGatewayDown` & alert bisnis belum*. |
| **Phase 8** | Monitoring AI Pipeline | ❌ | Belum mulai. Blocked total oleh masalah routing network Mac Mini. |
| **Phase 9** | Testing | 🔴 | Testing infra E2E selesai. *Isu aktif: Log `api-gateway` (namespace `qti`) belum masuk Loki (dugaan RBAC)*. |
| **Phase 10** | Production Checklist | ❌ | Belum dimulai, menunggu Phase 8-9 tuntas. |
| **Phase 11** | Troubleshooting | ⏳ | Berjalan reaktif, akan didokumentasikan di akhir. |

---

## BAGIAN 3 — Checklist Requirement Awal (Detail Observability Target)

### 1. Infrastructure & Node

- [x] CPU, RAM, Disk Monitoring
- [ ] Network Monitoring (Dashboard / Alert)
- [x] Node Health, Availability, Pressure, Restart Status (Lengkap)

### 2. Container & Kubernetes Layer

- [x] Container CPU, Memory, Restart Count, Health (Lengkap)
- [x] Pod Status
- [x] Deployment & Replica Status
- [ ] Namespace-specific View
- [ ] Kubernetes Events (Perlu tool tambahan, helm repo sempat error 404)

### 3. Application Layer

- [x] **Qdrant**: Lengkap (ServiceMonitor + Alerting)
- [⚠️] **Rust API (`api-gateway`)**: Metric infra ✅, log ❌ (sedang debug RBAC), alert Down ⬜
- [❌] **Mac Mini / Inference / LLM**: Belum tersentuh sama sekali

### 4. Business Metrics

- [❌] **Metrics Target**: Tier A/B/C, Confidence, Schema Validation, Escape Hatch, Inference/Retrieval Time, Request Rate, Latency, Error Rate
> *Status: Blocked total, menunggu Farrel melakukan instrumentasi kode.*

---

## BAGIAN 4 — Job Breakdown & Bantuan yang Dibutuhkan

```text
              ┌──────────────────────────────────────────┐
              │              HITE PROJECT                │
              └────────────────────┬─────────────────────┘
                                   │
      ┌────────────────┬───────────┴───────────┬────────────────┐
      │                │                       │                │
┌─────┴───────┐  ┌─────┴───────┐         ┌─────┴───────┐  ┌─────┴───────┐
│ Anda (Jep)  │  │    Ferdi    │         │    Hilmi    │  │    Farrel   │
│ Monitoring  │  │ Host / VM   │         │ Platform/K8s│  │  Backend/AI  │
└─────────────┘  └─────────────┘         └─────────────┘  └─────────────┘
```

### 👤 Jep (Monitoring)

- **Sedang Dikerjakan:** Debug RBAC Promtail untuk namespace `qti`.
- **Keputusan Pending:** Tentukan strategi solusi network Mac Mini (Dual-NIC Hilmi atau opsi alternatif).

### 🔧 Ferdi (VM / Node Host)

- **Action Item:** Konfirmasi ke XOA apakah Mac Mini benar-benar merupakan host fisik hypervisor (`Home` → `Hosts` di XOA).
- *Catatan:* Jika benar, strategi solusi network Mac Mini akan jauh lebih sederhana.

### 🔧 Hilmi (Platform — K0rdent & Networking)

- **Action Item 1:** Jelaskan alokasi RBAC Promtail saat ini (mengapa hanya mengizinkan 4 namespace: `argocd`, `hite-prod`, `kube-system`, `monitoring`).
- **Action Item 2:** Jelaskan isi & fungsi dari namespace `hite-prod`.
- **Action Item 3:** Bantu akses XOA untuk penyelesaian network Mac Mini (jika dibutuhkan).

### 🔧 Farrel (Backend — Rust / Axum / Qdrant)

- **Action Item 1:** Implementasi instrumentasi metric bisnis sesuai spesifikasi:
  - `qti_confidence_tier_total`
  - `qti_routing_decision_total`
  - `qti_qdrant_match_total`
  - `qti_fact_coverage_score`
  - `qti_request_duration_seconds`
- **Action Item 2:** Pastikan log `api-gateway` dialirkan ke `stdout`/`stderr` (bukan disimpan di file terpisah).
- **Action Item 3:** Klarifikasi dokumentasi `api_contract.md` (menyebutkan "Python Inference Engine", namun repository menggunakan `mistral.rs`/Rust).

### 👤 Johanes

- **Status:** Non-teknis. Cukup pastikan tetap berada di grup Telegram **"HITE QTI Logs"** untuk penerimaan alert log.

---

## §5. Platform Engineering (Owner: Hilmi)

### K0rdent (k0s) Infrastructure & Network — Infrastructure & State Report

**Date:** 2026-07-28
**Owner:** Hilmi
**Repo/Path:** On-Premises VM Cluster (worker-1: 10.20.20.202, worker-2: 10.20.20.200)

#### 5.1 What's Running / Current State

| Component / File | Status | Access / Details |
| :--- | :--- | :--- |
| **K0s Worker Nodes** | Active | `worker-1` (10.20.20.202), `worker-2` (10.20.20.200). Controller on 10.20.20.201. |
| **Private Container Registry** | Active | External: `10.20.20.202:32000`. Internal: `private-registry-svc:5000` (namespace `hite-prod`). |
| **NFS StorageClass (`nfs-csi`)** | Bound | NFS Server: `10.20.20.143`, Path: `/upload/intern`. Binds PVC `qdrant-storage-qdrant-0` (namespace `qdrant`) and `pvc-uji-coba`. |
| **Firewall (SOP-06 Air-Gapped)** | Active | Managed by `iptables-persistent`. Blocks 100% of outbound internet traffic (including `api.anthropic.com` and `8.8.8.8`). Traffic is allowed to subnets `10.20.20.0/24`, `10.244.0.0/16`, `10.96.0.0/12`, and `192.168.0.0/16`. |
| **Mac Mini Interconnect** | Active (Tunnel) | Reverse SSH Tunnel forwards Mac Mini ports to `worker-1`. Port `19100` (Node Exporter metrics) and `11434` (Ollama/LLM). |
| **Prometheus Telemetry** | Active | Authentication via `prometheus-scraper` ServiceAccount token over port `10250`. |

#### 5.2 CI/CD & Automation (If applicable)

- **Reverse Tunnel Daemon**: Automates the connection from the Mac Mini to the VM using macOS `launchctl` (`com.hite.tunnel.plist` with `644` permissions). This daemon will continuously attempt to reconnect (`KeepAlive`) and reroute port `9100` to `19100`, and `11434` to `11434` on the `worker-1` side if the SSH connection drops.
- **Firewall Persistence**: Uses the `netfilter-persistent` and `iptables-persistent` packages to ensure air-gapped rules are automatically reloaded if `worker-1` reboots.

#### 5.3 Notable Observations & "Gotchas"

##### 5.3.1 Service Mesh & DNS Mismatch (CRITICAL)
- The blueprint specifies the Qdrant DNS target as `qdrant-service.qdrant.svc.cluster.local`, but those calls return `NXDOMAIN`.
- The valid and currently active Service is `qdrant.qdrant.svc.cluster.local` (resolves to IP `10.108.156.131`).
- **Gotcha**: The Data Engineering team (Farrel) needs to update the target URL variable in the `api-gateway` and `data-pipeline` repositories to `qdrant` to avoid internal connection failures.

##### 5.3.2 Networking & Mac Mini Location
- The Mac Mini is currently located at an employee's home with a local IP of `192.168.1.18`, not at the office (`192.168.20.163`).
- **Gotcha**: Creating an official Network Bridge on the office router is currently impossible due to the physical network segment difference. The Reverse SSH Tunnel script is the only path connecting the LLM to the cluster.
- **Gotcha**: The Windows `.ssh/config` configuration has been updated to use the mDNS `Qtis-Mac-mini.local` as the `ProxyJump` target to anticipate dynamic IP changes for the Mac Mini.

##### 5.3.3 Security & Offline Image Loading
- The `worker-1` machine rejects all `OUTPUT` traffic to the public internet.
- **Gotcha**: You cannot run `docker pull` on `worker-1`. All new Docker images (such as `registry:2`) must be pulled from a local PC with internet access, exported via `docker save` to a `.tar` file, sent to `worker-1` using `scp`, and imported using `k0s ctr images import`.

#### 5.4 What Needs to Be Done (TODOs)

- [x] Install k0s Worker Nodes (`worker-1`, `worker-2`) and clean up the old K3s installation.
- [x] Create the Private Container Registry (`10.20.20.202:32000`).
- [x] Provision NFS-based Persistent Storage (`nfs-csi`).
- [x] Execute SOP-06 Firewall Lockdown (Air-Gapped Absolute).
- [ ] **Dev Team Unblocks**: Farrel needs to revise the Rust API code to use the `qdrant.qdrant.svc.cluster.local` DNS URL target instead of `qdrant-service`.
- [ ] **Infra Improvements**: Replace the Reverse SSH Tunnel with an official Network Bridge from the `10.20.20.0/24` block to the Mac Mini (Ports `11434` and `9100`), once the Mac Mini is returned to the physical office infrastructure.
- [ ] **Infra Improvements**: Request the office network admin to apply a DHCP Reservation (Static IP) for the Mac Mini's MAC Address (`192.168.20.163`) when the device returns.

#### 5.5 Quick Reference (Cheat Sheet)

**Check Private Registry status from worker-1**
```bash
curl -X GET http://10.20.20.202:32000/v2/_catalog
```

**Push an image to the Private Registry locally**
```bash
sudo docker push 10.20.20.202:32000/<image-name>:<tag>
```

**Check Mac Mini telemetry via Reverse Tunnel from within worker-1**
```bash
curl -s -I http://10.20.20.202:19100/metrics
```

**Test Qdrant DNS resolution from within the Kubernetes cluster**
```bash
kubectl run -it --rm test-dns --image=busybox:1.28 -n hite-prod -- nslookup qdrant.qdrant.svc.cluster.local
```

**Force restart the Reverse SSH Tunnel service (Run in Mac Mini terminal)**
```bash
launchctl unload ~/Library/LaunchAgents/com.hite.tunnel.plist && launchctl load ~/Library/LaunchAgents/com.hite.tunnel.plist
```

**Pull Kubelet metrics using a Bearer Token (Run on worker-2 / Jep)**
```bash
curl -k -H "Authorization: Bearer $TOKEN" https://10.20.20.202:10250/metrics
```

---

## §6. Mac Mini Full-Hosting: Consolidated View (NEW)

> **This section is synthesis, not a verbatim report.** You said HITE is moving to fully host on the Mac Mini — here's every Mac-Mini-related fact scattered across the five reports, pulled into one place, plus the open questions they leave unanswered. Nothing here is a new decision; it's the source material you'll need to make one.

### 6.1 What each report says about the Mac Mini today

| Source | What it says |
|---|---|
| Platform Eng §5.1 / §5.3.2 | Mac Mini is currently **at an employee's home** (`192.168.1.18`), not the office (`192.168.20.163`). Connected to `worker-1` only via a **Reverse SSH Tunnel** (ports `19100`→Node Exporter, `11434`→Ollama). Office network bridge is currently impossible due to the network segment mismatch. SSH config uses mDNS `Qtis-Mac-mini.local` as a ProxyJump target to survive IP changes. |
| Platform Eng §5.4 (TODO) | Long-term plan is the *opposite* of full-hosting: **replace** the tunnel with an office network bridge and get the Mac Mini a DHCP reservation *once it returns to the office* — i.e. the Mac Mini was planned as a **satellite inference box**, not the primary host. |
| Data Engineering §2.3.3 | The Mac Mini is described as "**now the full host** (Ollama + Qwen)" and is reachable over **Tailscale at `100.79.30.90`** — a different, VPN-based path than the SSH tunnel Platform Eng describes. |
| Data Engineering §2.3.8 | Mac Mini is **Apple Silicon (ARM/aarch64)** — first `cargo build` of `data-pipeline` there recompiles `fastembed`/onnxruntime and is slow (minutes) but works. |
| Data Scientist §1.3.1 | States directly: "**Because the Mac Mini is hosting the cluster**..." — implying the k0s cluster itself (not just inference) may run on/through the Mac Mini. |
| DevOps CI/CD §3.3.3 | Ollama was **removed from the k0s controller VM** specifically because it was "redundant with the Mac Mini inference server" — confirms the Mac Mini's role as the LLM host, separate from the controller VM. |
| DevOps Prometheus/Grafana/Loki roadmap, Phase 8 (§4.4) | "**Blocked total** oleh masalah routing network Mac Mini" — the AI pipeline monitoring phase cannot start until Mac Mini networking is resolved. |
| DevOps Prometheus/Grafana/Loki roadmap, Ferdi's action item (§4.4) | Ferdi is asked to confirm with XOA (the hypervisor manager) whether the **Mac Mini is literally the physical hypervisor host** (`Home → Hosts` in XOA) — if true, the whole networking problem gets much simpler. **This question appears unresolved in all five reports.** |
| DevOps Prometheus/Grafana/Loki roadmap, Jep's action item (§4.4) | "Keputusan Pending: Tentukan strategi solusi network Mac Mini (**Dual-NIC Hilmi** atau opsi alternatif)" — a dual-NIC approach from Hilmi is on the table but not decided. |

### 6.2 Open contradictions to resolve before committing to full-hosting

1. **Is the k0s cluster running on the Mac Mini, or does the Mac Mini just host inference (Ollama/Qwen) alongside a separate cluster on VM IPs 10.20.20.201/202/200?** Platform Eng and DevOps CI/CD reports both describe the cluster living on those VM IPs with **no HA** and a **single point of failure** — this reads as physically separate hardware from the Mac Mini. But the Data Scientist report and the "XOA hypervisor" question both suggest the Mac Mini might *be* that physical box underneath. This is the single biggest open question and should be confirmed with Ferdi/Hilmi before any full-hosting plan is finalized.
2. **Two different network paths to the Mac Mini exist simultaneously and neither team seems aware of the other's:** Platform Eng relies on a **Reverse SSH Tunnel** (`launchctl`-managed, ports 19100/11434) while Data Engineering relies on **Tailscale** (`100.79.30.90`). If you fully host on the Mac Mini, you'll want to standardize on one — running both increases the surface area for the exact kind of connectivity issues already blocking Phase 8 of the observability roadmap.
3. **Location is still in flux.** As of Platform Eng's report the Mac Mini was at an employee's home, not the office — full-hosting plans should nail down the physical location first, since it changes which network path (tunnel vs. Tailscale vs. office LAN) is even viable.
4. **ARM build implications for full-hosting:** if `data-pipeline`, `api-gateway`, and any future services also move onto the Mac Mini, expect the same first-build ARM recompilation tax described in Data Engineering §2.3.8 for any new Rust crate, and check that all Docker images used cluster-wide have `linux/arm64` variants — the current setup assumes `x86_64`-style workers.

### 6.3 Suggested next step

Given the above, before writing a full-hosting migration plan it's worth getting three yes/no answers on record: (1) is the Mac Mini the XOA hypervisor host — yes/no; (2) tunnel or Tailscale as the one standard path — pick one; (3) final physical location — office or remote. Everything else (DNS, firewall rules, ARM builds, Qdrant reachability for `data-pipeline`) hangs off those three answers.

---

## §7. Cross-Role Dependency & Blocker Map (NEW)

> Synthesized from the TODO/gotcha sections of all five reports — shows who's currently blocked on whom.

| Blocked | Blocked on | Because |
|---|---|---|
| Data Science E2E testing (§1.4) | Platform Engineering (Hilmi) | Rust API (port 8080) and Qdrant (port 6333) need a K0s network route (NodePort/LoadBalancer/kubeconfig RBAC) — see §1.3.1. |
| Data Engineering `data-pipeline` ingestion (§2.4) | DevOps CI/CD (Ferdi) confirmation | `data-pipeline` runs outside the cluster and can't reach `qdrant.qdrant.svc.cluster.local` — needs a decided external path (Mac Mini, port-forward, or NodePort) — see §2.3.3. |
| `/v1/query` returning real data (§2.4, §1.4) | Data Engineering (Farrel) | `clients::qdrant::search_sop` is written but not wired into the route handler yet, and the collection has 0 points until ingestion runs. |
| DevOps Prometheus/Grafana/Loki Phase 8 (AI pipeline monitoring) | Mac Mini networking resolution (Hilmi + Ferdi) | "Blocked total" per §4.4, Phase 8 — same root cause as the two rows above. |
| DevOps Prometheus/Grafana/Loki business-metric dashboards (Phase 6/7) | Farrel (Data Engineering) | Business metrics (`qti_confidence_tier_total`, etc.) require code instrumentation not yet written — see §4.4, Farrel's action items. |
| Promtail logs for `api-gateway` (namespace `qti`) reaching Loki | RBAC debugging (Jep), explanation from Hilmi | Promtail currently only has RBAC for 4 namespaces (`argocd`, `hite-prod`, `kube-system`, `monitoring`) — `qti` is not among them; per §4.4 this needs Hilmi to explain the current allocation. |
| Any full-hosting decision (§6) | Ferdi + Hilmi | Needs the XOA hypervisor question answered and a single network path (tunnel vs. Tailscale) chosen. |
| `clients/inference.rs` (api-gateway) | Design confirmation from Data Science (Johan) | May be entirely unnecessary if the Python agent calls Ollama/Qwen directly — see §2.3.6. |

---

*End of Master Guidebook. All five source reports are preserved in full above; only escaping artifacts were cleaned, the DevOps report was split by role (CI/CD vs. observability), and organizational headers/cross-references were added.*
