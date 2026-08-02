# HITE — Master Infrastructure Guidebook

**Project:** Hybrid IT Triage Engine (HITE) — QTI-MAGANG
**Compiled:** 2026-07-30 (state reconciled against live machine 2026-07-31; DS E2E baseline reconciled 2026-08-02)
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
   - **Wireguard added.** New §3.2.2 documents the VM↔Mac Mini Wireguard tunnel (`10.10.10.0/24`) for LLM traffic. Tunnel verified operational 2026-07-30: handshake established, 26-30ms latency, Ollama models reachable from VM at `10.10.10.2:11434`.
8. **2026-07-31 — Deployment reconciled; HITE RAG path now live end-to-end.** First real-data deployment of the retrieval path, replacing placeholders throughout:
   - **`/v1/query` wired to Qdrant and deployed.** Commit `10898a1` wired `clients::qdrant::search_sop` into the route handler and populated the collection. CI run #7 for that commit **built the image and passed smoke but never deployed**: the commit-back push failed because `main` had moved — Argo CD faithfully stayed on the old `1f34091`. Root cause: the workflow's `git push` after the auto-tag bump needed a `git pull --rebase` first. Fixed in `ci.yml` (see §3.1.1).
   - **Embedding model baked into the image.** Query-time embedding failed in the air-gapped cluster (`Failed to retrieve model.onnx`) because fastembed can't download at runtime. The Dockerfile now runs `--download-only` at build time and ships the model under `EMBED_CACHE_DIR=/opt/fastembed` (env added to the Deployment). Deployed image is now `ghcr.io/merpatidove/qti-api-gateway:a0b4bec`.
   - **api-gateway exposed on NodePort 30082** (was ClusterIP). Reachable at `http://100.106.122.68:30082` (worker-2 Tailscale IP) — unblocks Data Science E2E testing.
   - **CI/CD hardened.** `ci.yml` gained `workflow_dispatch`, a `cargo check` step, an extended smoke test (`POST /v1/query` + `grep remediation_payload`), and the `git pull --rebase` commit-back fix. Runs #8/#9 green; tag auto-bump working again.
   - **Qdrant populated.** `qti_knowledge_base` now holds **83 points** (18 SOPs, chunked) at 384-dim/Cosine — Data Engineering ingestion is done.
   - **Promtail RBAC for `qti` confirmed.** `api-gateway` logs (namespace `qti`) reach Loki — row 6 of the §7 map is resolved.
   - **Still open:** business metrics (`qti_*`) not yet instrumented; DS 5W1H schema mismatch with the API contract; `clients/inference.rs` design question; XOA hypervisor question (full-hosting §6).
9. **2026-07-31 — Follow-up reconciliation vs. live machine + repo; Jep's PR #1 merged.** Cross-checked every §3/§4 status against the deployed cluster and the manifests in `QTI-MAGANG`:
   - **Jep's updates (PR #1) incorporated:** `hite-infra-alerts` now counts **13 rules** (added `HITE_ApiGatewayDown`), observability Phase 9 log E2E marked done — `api-gateway` logs (ns `qti`) confirmed reaching Loki; §4.5 api-gateway row updated (metric/log/alert ✅).
   - **§4.1.3 alert table extended** from 6 → 13 rows to match the actual PrometheusRule (source of truth: `k8s/prometheus/hite-infra-alerts.yaml`).
   - **`QDRANT_URL` env-read is already implemented** (`clients/qdrant.rs` reads the env var, falling back to in-cluster DNS) — §2.4 TODO moved to done; only the Secret mounting (§3.4.1) remains open.
   - **Image size note corrected** — the "~32 MB" figure predates baking `all-MiniLM-L6-v2` into the image (`a0b4bec` is larger).
   - **`/metrics` correction** — `http_requests_total` is registered but never incremented, so it does not appear in the exported metrics; only `health_checks_total` and `queries_total` are live.
   - **Live check 2026-07-31:** `GET http://100.106.122.68:30082/v1/health` → `{"status":"ok","version":"0.1.0"}`.
10. **2026-08-02 — Data Science evaluation pipeline runs end-to-end on real data; first measured baseline.** The DS lane moved from "harness that pings endpoints" to "a pipeline that produces a graded 5W1H triage," reconciled against the live cluster + Mac Mini Ollama on 2026-08-02:
    - Gateway reachable from a Tailscale laptop with NO port-forward: `GET http://100.106.122.68:30082/v1/health` → `{"status":"ok","version":"0.1.0"}`; `POST /v1/query` returns `classification: "retrieved"` with real SOP text (e.g. "nginx bind failed" → SOP-DOC-002; TKT-1001 → SOP-DB-001). Retrieval is content-driven.
    - Gateway response carries NO 5W1H keys — it is `{ticket_metadata, remediation_payload}`. The 5W1H schema mismatch with the API contract (flagged open since 2026-07-31) is now PROVEN on live data, not guessed: the 5W1H object lives in the DS agent's output, not the gateway response.
    - Ollama is NOT reachable from a Tailscale laptop directly (`100.79.30.90:11434` and worker-1 `100.68.225.41:11434` both refused — Ollama binds to the WireGuard interface `10.10.10.2` only, §3.2.2). The working door is an SSH local-forward through the controller, after an ED25519 key was generated (`ssh-keygen -t ed25519 -C "johan-hite"`) and added to `ferdi`'s `authorized_keys`:
      `ssh -L 11434:10.10.10.2:11434 ferdi@100.94.99.125` → `curl http://localhost:11434/api/tags` returns the model list (all-minilm 384-dim, Qwen2.5-Coder-7B-Instruct Q4_K_M, Ornith-1.0-9B Q4_K_M, llama3 Q4_0).
    - The real generation brain is `agent.py` (a FastAPI ReAct orchestrator on :8000, `/process-ticket`), NOT the gateway-direct path. The gateway-direct runs of 2026-07-31 were the retrieval diagnostic; the agent is the retrieval+generation join. Five fixes landed in `llm-inference/`: (A) `grade_result.py` reads `5w1h_output` (was the never-written `output` key — the root cause of the eternal 0%); (B) `test_run.py` points at the agent `/process-ticket` and extracts the flat 6-key dict; (C) `agent.py` Ollama URL → `127.0.0.1:11434` (was the dead `192.168.100.35`), timeout 45→300; (D) `tools.search_sop` → live gateway NodePort 30082 `/v1/query` (was the deleted `rag-service:8000`); (E) `agent.py` synthesis prompt now demands the exact six keys. `prompts.py` unchanged (already correct).
    - First real-data baseline (2026-08-02, 55 tickets): Valid JSON 55/55 (100%); Complete 5W1H Schema 55/55 (100%); Grounded (how ≠ "Pending SOP search") 24/55 (43.6%). `grade_result.py` now reports schema AND grounding as two metrics from one file. See §1.6.
    - Architecture locked (joint DS+DE decision 2026-08-02): do NOT build `clients/inference.rs`; gateway stays retrieval-only; generation lives in the DS agent. §7 row 8 → Resolved; §2.4 TODO → done; §2.3.6 / §2.3.12 / §3.4.1 annotated; the dead `INFERENCE_URL` Deployment env is confirmed for removal.
    - Still open (DS): raise grounding above 43.6% (synthesis-gate fix — *proposed, not yet applied*); methodology doc; business-metric/tier spec doc; agent-side observability hooks; `grade_result.py` into CI. The error→eval feedback loop (Loki → golden_datasets curator) is **PROPOSED only, not built**.
    - **Repo-state note (crosscheck 2026-08-02):** the five `llm-inference/` fixes (A–E) and the 55-ticket 100%/43.6% baseline were produced on Johan's laptop and are **NOT yet committed** to `Merpatidove/QTI-MAGANG`. The committed repo still has Fixes A–E unapplied (`grade_result.py` reads `output`; `test_run.py` → gateway `/v1/query`; `agent.py` → `192.168.100.35:11434`, timeout 45; `tools.search_sop` → deleted `rag-service:8000`) and its `evaluation_results.json` holds placeholder gateway responses (grades 0% schema, not 100%). §1.1 / §1.2 / §1.5 / §1.6 reflect the laptop state; re-crosscheck after the laptop code is committed.

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
| --- | --- | --- |
| llm-inference/agent.py | Active / generation brain | FastAPI ReAct orchestrator on :8000 (`POST /process-ticket`). Calls Ollama (Qwen2.5-Coder-7B-Instruct Q4_K_M) over the SSH tunnel (§1.3.7) for the 5W1H + tool choice; calls the gateway `/v1/query` via `tools.search_sop` for RAG context; synthesis phase grounds `why`/`how` in the retrieved SOP. Ollama URL env-driven (`OLLAMA_URL`, default `http://127.0.0.1:11434`). |
| llm-inference/test_run.py | Active / unblocked | POSTs each ticket to the **agent** `/process-ticket` (`AGENT_URL`, default `http://127.0.0.1:8000`) and extracts the flat 6-key 5W1H dict. No longer points at the gateway directly — that path was the 2026-07-31 retrieval diagnostic only. |
| llm-inference/grade_result.py | Active / two metrics | Reads `5w1h_output` (Fix A — was the never-written `output` key, the root cause of the eternal 0%). Reports **two** metrics: Complete 5W1H Schema (six keys present) and Grounded (`how` ≠ the `Pending SOP search` placeholder, constant `PLACEHOLDER_HOW`). Single source of truth for both definitions (handed to Ferdi for CI). |
| llm-inference/prompts.py | Stable | `REACT_SYSTEM_PROMPT` defines the six lowercase keys + tool-selection directives; proven to elicit the shape. |
| llm-inference/tools.py | Active / fixed | `search_sop` hits the live gateway NodePort 30082 `/v1/query` (was the deleted `rag-service:8000`). `execute_safe_cli` sandbox not deployed — non-critical; the agent catches the error and falls back to the analysis-phase 5W1H (still six keys). |
| evaluation_results.json | Generated (real) | 55 **real agent** 5W1H triages (2026-08-02), not placeholders: 100% schema-complete, 43.6% grounded. |
| Rust API ( /v1/query ) | Deployed, live / retrieval-only | NodePort 30082. Returns retrieved SOP text in `remediation_payload.proposed_fix` (no 5W1H). Reached directly over Tailscale for the retrieval diagnostic; reached by the agent's `search_sop` for RAG context. |
| Qdrant DB ( qti_knowledge_base ) | Deployed, 83 points | 384-dim / Cosine, 18 SOPs chunked. Internal `http://qdrant.qdrant.svc.cluster.local:6333`; the agent reaches it only via the gateway. |

> **Repo note (2026-08-02 crosscheck):** rows above describe the laptop state — the five `llm-inference/` fixes and the real 55-ticket baseline are NOT yet committed to `QTI-MAGANG` (see §0 item 10).

#### 1.2 Data scientist

The Data Science evaluation pipeline is currently executed manually to evaluate LLM inference outputs against a strict 5W1H (Who, What, When, Where, Why, How) schema.

- **Triggers:** Manual execution via `python test_run.py` to generate inference outputs, followed by `python grade_result.py` for parsing and validation.
- **Steps (2026-08-02 pipeline):**
  1. `test_run.py` POSTs each ticket to the agent `/process-ticket`.
  2. The agent runs a ReAct loop: Ollama produces a 5W1H analysis + tool choice; if `search_sop` is chosen, the agent retrieves RAG context from the gateway `/v1/query`; a synthesis call grounds `why`/`how` in it.
  3. `test_run.py` stores the flat six-key 5W1H dict under `5w1h_output` in `evaluation_results.json`.
  4. `grade_result.py` reports schema completeness AND grounding.
- **Last Successful Run (2026-08-02):** full agent pipeline over 55 tickets — Valid JSON 55/55 (100%); Complete 5W1H Schema 55/55 (100%); **Grounded 24/55 (43.6%)**. This is the first run where the schema score is measured on real generated data (it was 0% on placeholders and on the wrong key before Fix A). The 43.6% is the first measurement of RAG grounding in the project. (The 2026-07-31 gateway-direct run that returned `ticket_metadata`/`remediation_payload` was the retrieval diagnostic, not the grading target.)

> **Repo note (2026-08-02 crosscheck):** steps + run above describe the laptop state, NOT yet committed to `QTI-MAGANG` (see §0 item 10).

#### 1.3 Notable Observations & "Gotchas"

##### 1.3.1 k0s Networking on Mac Mini Host

Our K0s cluster does not expose internal services externally by default. Because the Mac Mini is hosting the cluster, local development environments cannot reach the internal DNS (`qdrant.qdrant.svc.cluster.local`) or the Rust API pod without an explicit ingress route. Platform Engineering must establish a NodePort, LoadBalancer, or issue `kubeconfig` RBACs for `kubectl port-forward` before Data Science E2E testing or Data Engineer DB population can proceed.

> **Cross-reference note:** this line says "the Mac Mini is hosting the cluster" — worth reconciling against Platform Engineering §5 (the k0s controller/workers are separate VM IPs `10.20.20.201/202/200`) and DevOps CI/CD §3.3.2 ("this VM is a single point of failure"). See §6 below for the consolidated Mac Mini picture and this exact discrepancy.

**Update 2026-08-02 (DS unblocked):** the Rust API route no longer needs a port-forward for DS — api-gateway is on NodePort 30082, reachable directly over Tailscale at `http://100.106.122.68:30082`. The remaining network door DS needs is Ollama, which is reached via the SSH local-forward in §1.3.7 (not a Platform Eng NodePort). The "Platform Engineering must establish a route" sentence above is therefore resolved for DS E2E; it remains accurate as historical context and for any component that still needs cluster-internal DNS.

##### 1.3.2 Evaluation Schema Strictness

The `grade_result.py` script enforces a hard, exact-match key check for the 5W1H schema. If the backend Rust API returns *any* structure other than `Who`, `What`, `When`, `Where`, `Why`, and `How`, the test will record a 0% success rate for that specific run, regardless of JSON validity.

##### 1.3.3 Qdrant Database State

The `qti_knowledge_base` collection is initialized with a 384-dim Cosine configuration and its status is Green. **As of 2026-07-31 it holds 83 points** (18 SOPs, chunked) — the Data Engineer has run the document ingestion pipeline, so `clients::qdrant::search_sop` now returns actionable vectors. *(Written when the collection was empty at 0 points.)*

##### 1.3.4 Version Control Artifacts

All `__pycache__` artifacts and `.venv` environments have been actively purged and added to `.gitignore`. Never commit these files as they bloat the inference repository and trigger CI pipeline caching issues.

##### 1.3.5 Architecture locked — gateway is retrieval-only (decided 2026-08-02)

Joint DS+DE decision: the Rust api-gateway does retrieval only (returns the retrieved SOP text in `remediation_payload.proposed_fix`); it never calls Ollama and never generates the 5W1H. Generation is the DS Python agent's job (`agent.py`), which calls Ollama over the SSH tunnel (§1.3.7) and uses the gateway only for RAG context. Consequences already reflected elsewhere: `clients/inference.rs` is intentionally never built (§2.3.6 / §7 row 8); the 5W1H object lives in the agent's output, not the gateway response (so `grade_result.py` reads `5w1h_output` from the agent, not `/v1/query`); the `INFERENCE_URL` Deployment env is dead (§2.3.12). Recorded so the design is not re-opened by a future reader.

##### 1.3.6 Schema completeness vs grounding — two metrics, one grader

`grade_result.py` now reports two independent numbers, on purpose, because they measure different things and conflating them hid the real state for weeks:
- **Complete 5W1H Schema** = are all six keys (`Who/What/When/Where/Why/How`, matched case-insensitively via `.capitalize()`) present and non-empty? Measures *shape*.
- **Grounded** = is `how` backed by a real retrieved SOP, i.e. NOT the prompt's placeholder example `"Pending SOP search"` (constant `PLACEHOLDER_HOW`)? Measures *whether the RAG half of RAG reached the answer*. A schema-complete triage can still be ungrounded (the model filled the form from the ticket alone).
Both definitions live in this one file so they cannot drift (the same drift that produced the old never-written `output` key). The Tier mapping derived from them: **Tier A** = schema-complete AND grounded; **Tier B** = complete but ungrounded; **Tier C** = schema-incomplete / escape-hatch. On 2026-08-02: A = 24, B = 31, C = 0. Hand this file (not a copy of the logic) to Ferdi for CI (§1.4).
Caveat: the grounding check is naive — an escape-hatch string like `"no matching SOP"` would be mis-counted as grounded because it lacks the placeholder substring. Revisit the `PLACEHOLDER_HOW` list from the real distribution before treating 43.6% as exact.

> **Repo note (2026-08-02 crosscheck):** this two-metric grader is laptop-side, NOT yet committed to `QTI-MAGANG` (see §0 item 10).

##### 1.3.7 Ollama reachability from a Tailscale laptop — the SSH-tunnel door

Ollama on the Mac Mini binds to the WireGuard interface only (`OLLAMA_HOST=10.10.10.2:11434`, §3.2.2); on the Mac Mini's Tailscale interface nothing listens on 11434. A Windows laptop on Tailscale therefore CANNOT reach Ollama directly — `curl http://100.79.30.90:11434` and worker-1 `http://100.68.225.41:11434` both refuse. The working door is a local-forward through the controller (which is on the WireGuard subnet at 10.10.10.1):

```bash
ssh -L 11434:10.10.10.2:11434 ferdi@100.94.99.125     # leave this session OPEN; it IS the tunnel
curl http://localhost:11434/api/tags                    # expect the model list
```

Auth is key-only (§3.3.2): the laptop needs an ED25519 key in `ferdi`'s `~/.ssh/authorized_keys` (`ssh-keygen -t ed25519 -C "johan-hite"`; hand the `.pub` line to Ferdi). Without it, SSH closes at auth (`Connection closed by 100.94.99.125 port 22`, no password prompt). `agent.py` reads `OLLAMA_URL` (default `http://127.0.0.1:11434`), so with the tunnel up the default works. First Ollama call cold-loads the 4.7 GB model (20–60 s) — that is why `call_ollama` timeout is 300, not 45. This tunnel is a foreground dependency: closing the SSH session kills generation mid-run. For a reproducible/monitored pipeline the agent should eventually run on a WireGuard-side host (controller/Mac Mini) so the path stops depending on a laptop.

#### 1.4 What Needs to Be Done (TODOs)

**Platform Engineering Unblocks**
- [x] Open K0s network route to expose Rust API (port 8080) to allow Data Science inference testing. *(done 2026-07-31 — api-gateway now NodePort 30082, reachable at `http://100.106.122.68:30082`)*
- [x] Open K0s network route to expose Qdrant (port 6333) to allow Data Engineering SOP ingestion. *(done — ingestion ran via SSH port-forward tunnel; Qdrant intentionally not exposed permanently, see §3.2.3)*

**Data Engineering Unblocks**
- [x] Ingest the 18 SOPs from the RAG manual, generate embeddings, and populate the empty Qdrant vector database. *(done 2026-07-31 — `qti_knowledge_base` = 83 points, 384-dim/Cosine)*
- [x] Wire the Rust backend (`/v1/query`) to actively call `clients::qdrant::search_sop` instead of returning the placeholder payload. *(done 2026-07-31 — commit `10898a1`, deployed as `a0b4bec`)*
- [x] Confirm `clients/inference.rs` is needed — confirmed NOT needed (joint DS+DE decision 2026-08-02); gateway is retrieval-only (§1.3.5 / §2.3.6), file not built.

**Data Science (Johan)**
- [x] Run full 5W1H evaluation suite against live data — done 2026-08-02 via the agent pipeline: 55/55 valid, 55/55 schema, 24/55 (43.6%) grounded (§1.6). *(laptop-side, uncommitted — see §0 item 10)*
- [ ] Raise grounding above 43.6% — the retrieval→synthesis join lands on 24/55 only; prime suspect is the empty-retrieval falsy gate in `agent.py` (`if tool_output and ...` treats a 200-OK empty retrieval as "nothing" and skips synthesis). Apply the observe-first `test_run.py` fields + the `is not None` gate fix, re-run, compare to the 43.6% control. (Proposed fix — NOT yet applied.)
- [ ] Spec the business metrics from data — Tier A/B/C, escape-hatch rate, confidence: the 2026-08-02 distribution (A=24/B=31/C=0) anchors the definitions; write the spec so Farrel can instrument `qti_confidence_tier_total` etc. (§4.4).
- [ ] Add agent-side observability hooks — JSON-decode / Ollama-timeout / empty-retrieval counters + per-ticket latency (the `inference_time_sec` field is a primitive start); emit so Jep's Phase 8 has DS-side signal.
- [ ] Finalize the methodology documentation (§1.4 original) — write it OFF the 2026-08-02 baseline + a model comparison (Qwen2.5-Coder-7B vs Ornith-1.0-9B via `OLLAMA_MODEL`), citing the schema-vs-grounding finding.
- [ ] (PROPOSED, not built) Error→eval feedback loop — a curator that pulls error-shaped tickets from Loki into `golden_datasets.json` so the baseline is graded on production shapes; raw errors must NOT be auto-embedded into Qdrant (human-gated SOP authoring only). See §4.2 note.

**DevOps (CI/CD & Prometheus)**
- [ ] Integrate `grade_result.py` into a CI/CD pipeline (e.g., GitHub Actions) for automated schema regression testing on all new commits.
- [ ] Automate Data Engineering SOP ingestion trigger for future file updates to the repository.
- [ ] Expose LLM token throughput, Qdrant latency, and JSON decode error metrics to Prometheus/Grafana once the pipeline goes live.

#### 1.5 Quick Reference (Cheat Sheet)

```bash
# --- Data Science working directory ---
cd llm-inference

# --- THREE TERMINALS (the agent is a service, not a batch script) ---
# Terminal A — Ollama door (KEEP OPEN; closing it kills generation). See §1.3.7.
ssh -L 11434:10.10.10.2:11434 ferdi@100.94.99.125
# Terminal B — the agent (FastAPI on :8000)
pip install uvicorn fastapi requests     # once
uvicorn agent:app --host 127.0.0.1 --port 8000
# Terminal C — the harness + grader (env vars are PER-TERMINAL; defaults already correct)
#   AGENT_URL  -> agent      (default http://127.0.0.1:8000)
#   OLLAMA_URL -> Ollama     (default http://127.0.0.1:11434, the tunnel)
#   QTI_API_URL-> gateway    (default http://100.106.122.68:30082, used by tools.search_sop)
python test_run.py        # ~10-20 min; first ticket slow (cold model load)
python grade_result.py    # prints BOTH lines:
#   Complete 5W1H Schema:      55/55 (100.0%)
#   Grounded (how != pending): 24/55 (43.6%)

# --- one-shot cross-tab (diagnose WHY ungrounded; uses only guaranteed fields) ---
python -c "import json,collections as c; d=json.load(open('evaluation_results.json',encoding='utf-8')); g=lambda i: bool(((i.get('5w1h_output') or {}).get('how') or '').strip()) and 'pending sop search' not in (((i.get('5w1h_output') or {}).get('how') or '').lower()); print('grounded mean %.1fs ungrounded mean %.1fs'%(sum(i['inference_time_sec'] for i in d if g(i))/max(1,sum(1 for i in d if g(i))), sum(i['inference_time_sec'] for i in d if not g(i))/max(1,sum(1 for i in d if not g(i))))"

# --- commit (exclude .venv / __pycache__ per §1.3.4) ---
git add llm-inference/agent.py llm-inference/prompts.py llm-inference/tools.py llm-inference/test_run.py llm-inference/grade_result.py llm-inference/evaluation_results.json
git commit -m "feat(llm-inference): real-data 5W1H baseline (100% schema / 43.6% grounded)"
git push origin main
```

---

#### 1.6 Evaluation Results — 2026-08-02 (first real-data baseline)

| Metric | Value | Meaning |
| --- | --- | --- |
| Total test cases | 55 | golden_datasets.json synthetic tickets |
| Valid JSON | 55/55 (100%) | agent returned parseable 5W1H on every ticket |
| Complete 5W1H Schema | 55/55 (100%) | all six keys present + non-empty, reproducibly |
| Grounded (how ≠ pending) | 24/55 (43.6%) | retrieval→synthesis join landed on 24 tickets |
| Tier A (complete + grounded) | 24 | |
| Tier B (complete, ungrounded) | 31 | |
| Tier C (incomplete) | 0 | schema is 100%, so none |

Latency is bimodal: grounded tickets (~18–34 s, two Ollama calls: analysis + synthesis) vs ungrounded (~5–11 s, one call, synthesis skipped). The ungrounded count ≈ the fast cluster — evidence the synthesis phase is being skipped on the 31, consistent with the empty-retrieval falsy-gate hypothesis (§1.4).

What this proves: the retrieval→generation join works end-to-end on a real fraction; the shape problem that printed 0% for weeks is closed; and — because the grader now measures grounding — the project can finally SEE that "schema-complete" ≠ "RAG-grounded." That distinction is the binding constraint going forward, not a bug.
What it does NOT prove: that grounding is high (it is 43.6%), nor reproducibility of the grounding rate across runs, nor quality of the grounded `how` beyond "not the placeholder." Those are the next measurements.

> **Repo note (2026-08-02 crosscheck):** this baseline was produced on Johan's laptop and is NOT yet committed to `QTI-MAGANG` — the committed `evaluation_results.json` still holds placeholder gateway responses (grades 0% schema). See §0 item 10.

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
| `api-gateway` Deployment (ns `qti`) | 1/1 Running, Healthy | NodePort **30082** (was `api-gateway.qti.svc:8080`). Image `ghcr.io/merpatidove/qti-api-gateway:a0b4bec` (deployed 2026-07-31; live SHA verified at the Actions tab — CI/CD report §3.1). |
| `api-gateway/src/main.rs` | Working, deployed | Orchestrator only (no business logic). Declares `mod models/routes/clients`, wires the router, exposes `/metrics`, binds `0.0.0.0:8080`. |
| `api-gateway/src/models.rs` | Working, deployed | `QueryRequest`, `QueryResponse`, `TicketMetadata`, `RemediationPayload` (serde; matches `api_contract.md`). |
| `api-gateway/src/routes/mod.rs` | Working, deployed | Module table of contents: `pub mod health; pub mod query;` |
| `api-gateway/src/routes/health.rs` | Working, deployed | `GET /v1/health` → `{"status":"ok","version":"0.1.0"}`; counter `health_checks_total`. K8s liveness/readiness target. |
| `api-gateway/src/routes/query.rs` | Working, deployed | `POST /v1/query`. `Json(req): Json<QueryRequest>` validates the body (bad JSON → auto 422). Returns a real `QueryResponse` from `search_sop` since commit `10898a1` (2026-07-31; was placeholder); counter `queries_total`. |
| `api-gateway/src/clients/mod.rs` | Working, deployed | Module table of contents: `pub mod qdrant;` |
| `api-gateway/src/clients/qdrant.rs` | Written, compiled, **WIRED + deployed** | `search_sop(Vec<f32>) -> Result<...>` via `reqwest` (REST, :6333). Constants `QDRANT_URL = http://qdrant.qdrant.svc.cluster.local:6333`, `COLLECTION_NAME = qti_knowledge_base`. Called from `query.rs` since commit `10898a1` (2026-07-31). |
| `api-gateway/src/clients/inference.rs` | **NOT written** | Mac Mini inference client — possibly obsolete (see §2.3.6 / §2.4). |
| `api-gateway/Cargo.toml` | Working | axum 0.8, tokio `[full]`, serde `[derive]`, serde_json, reqwest 0.12 `[rustls-tls, json]`, tracing, tracing-subscriber `[env-filter]`, prometheus 0.13 `[process]`, lazy_static 1.4, anyhow 1.0, **fastembed 4**. |
| `api-gateway/Dockerfile` | Working | Multi-stage `rust:1-bookworm` → `debian:bookworm-slim`; was ~32 MB before `--download-only` baked the embedding model in (image `a0b4bec` is larger). |
| `api-gateway/k8s/deployment.yaml` | Working | Liveness + readiness probes on `/v1/health`; `EMBED_CACHE_DIR=/opt/fastembed` env (2026-07-31, model baked into image). |
| `api-gateway/k8s/service.yaml` | Working | NodePort 30082 (was ClusterIP on 8080; changed 2026-07-31). |
| `api-gateway/k8s/kustomization.yaml` | Working | `newTag: <sha>` managed by CI commit-back. |
| `api-gateway/k8s/servicemonitor.yaml` | Working | Prometheus scrapes `/metrics` every 15s. |
| `data-pipeline/src/main.rs` | Working (**parser only**), **NOT deployed** | Reads `RAG_Manual.md`, splits on `"\n# SOP-"`, prints all 18 SOPs (id + title). Chunk → embed → upsert **not written in this crate** — the collection was nonetheless populated to 83 points on 2026-07-31 (ingestion path per §2.4); parser crate remains local-only. |
| `data-pipeline/Cargo.toml` | Working | Parser uses std only; staged for next step: `fastembed`, `qdrant-client`, `uuid`, `serde`, `serde_json`, `anyhow`, `tokio`. |
| `data-pipeline/RAG_Manual.md` | Present (data) | 18 structured SOPs — the ingestion source. |
| `data-pipeline/golden_datasets.json` | Present (data) | Sample tickets — DS evaluation harness; **not** consumed by the Rust code yet. |
| Collection `qti_knowledge_base` | Created, **green, 83 points** | **384-dim / Cosine** (NOT 1024 — see §2.3.1). 18 SOPs ingested + chunked on 2026-07-31 (commit `10898a1`). |
| Gateway → Qdrant | Reachable (in-cluster) | REST `http://qdrant.qdrant.svc.cluster.local:6333`. |
| Pipeline → Qdrant | **Reachable (via SSH port-forward / tunnel)** | Ingestion ran over the tunnel path on 2026-07-31; Qdrant stays internal by design (no permanent NodePort exposure). |
| `.gitignore` (repo root) | Added | `target/`, `*.pdb`, `*.exe`. |
| `rag-service/` | **DELETED** from repo | DevOps CI/CD report §3.1.2/§3.1.3 (see §3 below) still describe it — stale, see §2.3.5. |

**Expected placeholder response from `/v1/query`** (so testers don't file a bug — *historical as of 2026-07-31, when the route was wired to Qdrant and now returns real retrieved SOP text*):

```json
{"ticket_metadata":{"ticket_id":"","classification":"placeholder_classification"},"remediation_payload":{"proposed_fix":"Rust backend is not yet connected to Qdrant.","requires_type_check":false}}
```

#### 2.2 CI/CD & Automation (If applicable)

**`api-gateway` — working end-to-end.**

- **Trigger:** push to `main` touching `api-gateway/**`.
- **Steps:** GitHub Actions builds the Docker image (Rust multi-stage) → pushes `ghcr.io/merpatidove/qti-api-gateway:<sha>` → Docker smoke test (`/v1/health` must return 200) → commits the new image tag to `kustomization.yaml` with `[skip ci]` → Argo CD auto-syncs → pod restarts, health check passes.
- **Concurrency gate:** enabled — only the latest push builds; older in-progress runs are cancelled.
- **Rollback:** manual via `.github/workflows/rollback.yml` (specify a previous image SHA/tag).
- **Last runs:** modularization build = Actions run #6, ~2m47s, green; image ~32 MB; Argo deploy ~8s. 2026-07-31: run #7 (tag `10898a1`) built + passed smoke but the commit-back push failed on a moved `main` — fixed with `git pull --rebase`; runs #8/#9 green, latest tag `a0b4bec` (embedding model baked into image).

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

**Decision 2026-08-02 (Johan + Farrel):** confirmed — do NOT build `clients/inference.rs`. The Python agent orchestrates generation and calls Ollama directly (§1.3.7); the gateway only returns RAG context. This locks the architecture: the air-gapped gateway needs no Ollama route and no generation-time embedding. Consequence: the `INFERENCE_URL` env in `k8s/deployment.yaml` (§2.3.12) is permanently dead and should be removed.

##### 2.3.7 Git `target/` trap

`.gitignore` ignores **only untracked** files. The `api-gateway/target` and `data-pipeline/target` folders were untracked, so creating the root `.gitignore` excluded them. But `rag-service/target/` had been **committed** earlier (DevOps), so `.gitignore` alone would not have stopped tracking it — deleting the `rag-service/` folder is what removed it. If a `target/` ever shows up staged again: `git rm -r --cached <path>` (the `--cached` flag keeps the files on disk). Stage source surgically (`git add api-gateway/src api-gateway/Cargo.toml ...`) rather than `git add .` until `.gitignore` is confirmed.

##### 2.3.8 Apple Silicon build note (Mac Mini)

The Mac Mini is ARM. The first `cargo build` of `data-pipeline` there recompiles `fastembed` / onnxruntime for `aarch64` — it works, but the first build is slow (minutes). Not a failure.

##### 2.3.9 Qdrant has no auth; secrets are TODO

Qdrant currently has **no authentication** (DevOps CI/CD report §3.3.2). My constants hardcode the URL; per §2.4 these should move to K8s Secrets (`QDRANT_URL`, and `INFERENCE_URL` if inference stays) mounted as env vars — that is a DevOps deliverable; my code just needs to read the env var instead of the constant when it exists. *(Update 2026-07-31: the env-var read is done in `clients/qdrant.rs` — see §2.4; the Secret mounting is the remaining piece.)*

##### 2.3.10 Rust module system (code-structure gotcha)

Rust does not auto-scan folders. Every new file needs a `mod` declaration in `main.rs` (top-level) **and** a `pub mod` line in the folder's `mod.rs`, or you get `unresolved import` / "file ignored". The `pub` on structs/functions is also mandatory for cross-module use. Relevant whenever anyone adds a file to either crate.

##### 2.3.11 `data-pipeline` reads `RAG_Manual.md` by relative path

`fs::read_to_string("RAG_Manual.md")` resolves against the **current working directory**, so `cargo run` must be executed from inside `data-pipeline/` with the manual present there, or it panics with the "Failed to read RAG_Manual.md" message.

##### 2.3.12 `INFERENCE_URL` in the Deployment is dead config

`k8s/deployment.yaml` sets `INFERENCE_URL=http://inference-mac-mini:8080`, but no such Service/DNS exists in the cluster and the gateway is retrieval-only (§2.3.6) — it never makes an outbound inference call, so the env var is unused. Harmless, but it should be removed (or replaced by a real secret) when the Deployment is next touched (§3.4.1 "Add secrets").

**2026-08-02:** with the retrieval-only decision final (§1.3.5 / §2.3.6), this env is confirmed dead with no future — remove it from `k8s/deployment.yaml` (1-line edit on next Deployment touch; §3.4.1).

#### 2.4 What Needs to Be Done (TODOs)

**My lane — shipped**

- [x] Modularize `api-gateway` (`models.rs`, `routes/health.rs`, `routes/query.rs`, `clients/qdrant.rs`, rewired `main.rs`).
- [x] `/v1/health` live and passing the CI smoke test.
- [x] `/v1/query` accepts + validates JSON (placeholder → **real data 2026-07-31**, returns retrieved SOP text).
- [x] `clients/qdrant.rs::search_sop` written and compiling.
- [x] Root `.gitignore` added; `rag-service/` removed from the repo.
- [x] `data-pipeline` parser isolates all 18 SOPs.
- [x] `qti_knowledge_base` collection created at 384 / Cosine (via DevOps, per my spec).

**My lane — done 2026-07-31; remaining work below**

- [x] **Reachable Qdrant path for `data-pipeline`** — resolved via SSH port-forward tunnel (Qdrant stays internal by design); ingestion completed. *Previously the blocker.*
- [x] **Populate the collection** — done 2026-07-31: chunk → embed `all-MiniLM-L6-v2` (384-dim) → upsert UUID-v4 point IDs with payload `{text, sop_id, title, category, tier}`. `qti_knowledge_base` = **83 points**.
- [x] **Wire `/v1/query`** to call `clients::qdrant::search_sop` — done (commit `10898a1`); verified live returning real SOP text.
- [x] Confirm `clients/inference.rs` is needed — confirmed NOT needed (joint DS+DE decision 2026-08-02); gateway is retrieval-only (§1.3.5), file not built.
- [x] Move hardcoded `QDRANT_URL` to an env var — **done** (`clients/qdrant.rs` reads `QDRANT_URL`, falls back to in-cluster DNS). Remaining: DevOps mounts it as a K8s Secret (§3.4.1).

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

**Date:** 2026-07-17 (Updated 2026-07-30 — ServiceMonitors, Argo CD TLS, Tailscale access documented; Updated 2026-07-31 — `/v1/query` wired, NodePort 30082, model baked into image)
**Cluster:** k0s v1.36.2+k0s (Debian 13 trixie)
**Repo:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG)

> This section covers the CI/CD pipeline, Argo CD, container registries, SSH deploy keys, and general cluster/security notes. The Prometheus/Grafana/Loki/AlertManager observability stack that was originally reported alongside this content now lives in **§4 (Jep's section)** — see the editor's note at the top of this document for why it was split.

#### 3.1 What's Running on the Cluster (CI/CD & Platform components)

| Component | Status | Access |
|---|---|---|
| **Argo CD** | 7/7 pods Running | `https://argocd.hite.local` (admin / `12qwaszx`) |
| **Qdrant** | 1/1 Running | `qdrant.qdrant.svc.cluster.local:6333`, NFS-backed PVC (10Gi) |
| **api-gateway** | 1/1 Running, Healthy | NodePort **30082** — `http://100.106.122.68:30082/v1/health` returns `{"status":"ok"}`; `POST /v1/query` returns real SOP data. (Was ClusterIP `api-gateway.qti.svc:8080`.) |
| **NFS CSI driver** | 3/3 controller, 2/2 node pods | k0s path: `/var/lib/k0s/kubelet` |
| **Ingress-NGINX** | 1/1 Running (LoadBalancer) | NodePort 31084 (HTTP), 30616 (HTTPS) — routes to Grafana, Prometheus, AlertManager, ArgoCD. *(Grafana/Prometheus/AlertManager themselves are Jep's — see §4.)* |
| **Local Registry** | Running on controller (HTTPS, self-signed cert) | `10.20.20.201:5000` — stores all deployment images |
| **hite-prod Registry** | 1/1 Running (Deployment) | `private-registry-svc.hite-prod:5000` (NodePort 32000) — Kubernetes-native registry:2 |

> **Note:** the original combined report also listed Prometheus/Grafana, Loki, Promtail, and AlertManager in this same table — those rows are preserved in §4.1 (Jep's section) instead of being duplicated here.

##### 3.1.1 CI/CD Pipeline (Working End-to-End)

```
Push to main (api-gateway/**)
  -> GitHub Actions builds Docker image (Rust multi-stage + fastembed model baked via --download-only)
  -> cargo check (added 2026-07-31)
  -> Pushes to ghcr.io/merpatidove/qti-api-gateway:<git-sha>
  -> Docker smoke test: /v1/health must return 200; POST /v1/query must return remediation_payload (extended 2026-07-31)
  -> git pull --rebase then commit updated image tag to kustomization.yaml [skip ci] (fix for run #7, see below)
  -> Argo CD auto-syncs to cluster
  -> Pod restarts with new image, health check passes
```

- **Concurrency gate:** enabled — only the latest push builds (old in-progress runs are cancelled).
- **Rollback:** manual via `rollback.yml` — specify a previous image SHA/tag to revert instantly.
- **Known failure (2026-07-31):** CI run #7 built the image and passed smoke for commit `10898a1` but **never deployed** — the commit-back push failed because `main` had moved mid-run, so Argo CD stayed on old tag `1f34091`. Fixed with `git pull --rebase` before the tag commit-back. Runs #8/#9 green.
- **Manual trigger:** `workflow_dispatch` added 2026-07-31 (was push-only).

**Last successful run:** Image `ghcr.io/merpatidove/qti-api-gateway:a0b4bec` (bakes `all-MiniLM-L6-v2` under `/opt/fastembed`), deployed 2026-07-31; health check passing at NodePort 30082.

##### 3.1.2 Files Created in QTI-MAGANG Repo

| File | Purpose | Status |
|---|---|---|
| `api-gateway/Dockerfile` | Multi-stage Rust build (rust:1-bookworm → debian:bookworm-slim); runs `--download-only` to bake `all-MiniLM-L6-v2` into `/opt/fastembed` (2026-07-31, air-gapped cluster can't fetch at runtime) | Working |
| `api-gateway/Cargo.toml` | Dependencies: axum 0.8, tokio, serde, reqwest (rustls-tls), prometheus | Working |
| `api-gateway/src/main.rs` | `/v1/health`, `/v1/query`, `/metrics` endpoints with Prometheus counters | Working |
| `api-gateway/k8s/deployment.yaml` | Deployment with liveness/readiness on `/v1/health` | Working |
| `api-gateway/k8s/service.yaml` | **NodePort 30082** (was ClusterIP 8080; changed 2026-07-31) | Working |
| `api-gateway/k8s/kustomization.yaml` | Image tag managed by CI (`newTag: <sha>`) | Working |
| `api-gateway/k8s/servicemonitor.yaml` | Prometheus ServiceMonitor, scrapes `/metrics` every 15s | Working |
| `.github/workflows/ci.yml` | Build → `cargo check` → push → smoke (`/v1/health` + `POST /v1/query`) → commit-back via `git pull --rebase` (concurrency gate, `workflow_dispatch`) | Working |
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

##### 3.2.2 Wireguard — VM ↔ Mac Mini LLM Link

Wireguard provides a dedicated encrypted tunnel between the controller VM and the Mac Mini for LLM traffic, avoiding the Tailscale DERP relay overhead.

**Subnet:** `10.10.10.0/24`

| Side | Wireguard IP | Wireguard Public Key |
|---|---|---|
| Controller VM (debian13) | `10.10.10.1` | `IMDZ7747SpFcdPJaNNydUUJ3DPi1sq4Z/jFLDE03lXk=` |
| Mac Mini (qtis-mac-mini) | `10.10.10.2` | `W/ZjEHMjrkDq+rv3QJxZieL7MZlz6guijDN0i+RSmwA=` |

**VM config** (`/etc/wireguard/wg0.conf`):
```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = GNAX6apvqInSr/p15179ib+hUVydn+m7h5mP5SL/Ylc=

[Peer]
PublicKey = W/ZjEHMjrkDq+rv3QJxZieL7MZlz6guijDN0i+RSmwA=
AllowedIPs = 10.10.10.2/32
```

**Mac Mini config** (`/opt/homebrew/etc/wireguard/wg0.conf` on macOS — Homebrew prefix):
```ini
[Interface]
Address = 10.10.10.2/24
PrivateKey = <MacMini-PrivateKey>   # fresh key, NOT reusing mac-mini-ops tunnel key

[Peer]
PublicKey = IMDZ7747SpFcdPJaNNydUUJ3DPi1sq4Z/jFLDE03lXk=
Endpoint = 100.94.99.125:51820
AllowedIPs = 10.10.10.1/32
PersistentKeepalive = 25
```

**Mac Mini (macOS) — live state (verified 2026-07-30):**

| Item | Value |
|---|---|
| Interface | `utun6` (wg0) |
| Config path | `/opt/homebrew/etc/wireguard/wg0.conf` |
| Private key | `/opt/homebrew/etc/wireguard/mm-vm-private.key` |
| Public key | `W/ZjEHMjrkDq+rv3QJxZieL7MZlz6guijDN0i+RSmwA=` |
| Ollama binding | `10.10.10.2:11434` (`OLLAMA_HOST=10.10.10.2:11434`) |
| Auto-start | LaunchDaemon `/Library/LaunchDaemons/com.wireguard.wg0.plist` |
| Latency (VM↔Mac Mini) | 26-30ms, 0% loss |
| Mac Mini hostname | `Qtis-Mac-mini.local` (mDNS), Tailscale IP `100.79.30.90` |

**Models available on Mac Mini via WireGuard (Ollama):**

| Model | Size |
|---|---|
| `all-minilm:latest` | 46MB (embedding, 384-dim) |
| `Ornith-1.0-9B (Q4_K_M)` | 5.6GB |
| `Qwen2.5-Coder-7B (Q4_K_M)` | 4.7GB |
| `llama3:latest (Q4_0)` | 4.7GB |

**Getting LLM access from the VM:**
```bash
# After both sides have wg-quick up wg0, Ollama is reachable at:
curl http://10.10.10.2:11434/api/tags

# Verify handshake
sudo wg show wg0 | grep handshake
```

**Important notes:**
- **Tailscale must be up first** on both sides — the Mac Mini's `Endpoint` is a Tailscale IP (`100.94.99.125`), so it won't resolve if Tailscale is down.
- **Fresh key required** — the Mac Mini's Wireguard key must be generated with `wg genkey`; do not reuse the key from any existing macOS Wireguard tunnel (`mac-mini-ops` or similar).
- **Ollama binding** — use `OLLAMA_HOST=10.10.10.2:11434` (binds directly to the WireGuard interface; `0.0.0.0` on macOS binds to IPv6 `[::]` which may not accept IPv4 connections from the tunnel).
- **macOS auto-start** — LaunchDaemon installed at `/Library/LaunchDaemons/com.wireguard.wg0.plist` with `KeepAlive`. Load with: `sudo launchctl load /Library/LaunchDaemons/com.wireguard.wg0.plist`.
- **`wg-quick` on macOS** searches configs in order: `/etc/wireguard/`, `/usr/local/etc/wireguard/`, `/opt/homebrew/etc/wireguard/`. The config lives under the Homebrew prefix since that directory is user-writable.
- **Pre-existing tunnel** — the Mac Mini also has a separate WireGuard tunnel **"mac-mini-ops"** (utun4, IP `10.7.0.63`, endpoint `117.54.250.111:51820`) managed by the macOS WireGuard app — connects to the ops infrastructure controller at `10.20.20.201`. Recovery script at `~/wg-recover.sh`. Do not confuse or reuse keys between tunnels.

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

- [x] **api-gateway skeleton** — `/v1/health`, `/v1/query`, `/metrics` endpoints implemented. Query returned placeholder initially; **returns real retrieved SOP text since 2026-07-31** (commit `10898a1`, deployed `a0b4bec`).
- [x] **rag-service scaffold** — `POST /api/v1/ticket` accepts JSON, returns dummy response. No Qdrant/Mistral integration.
- [x] **Write actual Rust source code** — *(per Data Engineering §2.3.5 above)*: `models.rs`, `routes/query.rs`, `clients/qdrant.rs` are **written, wired, and deployed** (2026-07-31). Only `clients/inference.rs` remains, and may be unnecessary — see §2.3.6.
  - `routes/query.rs` — POST /v1/query handler, Qdrant query, inference forward
  - `clients/qdrant.rs` — Qdrant HTTP client
  - `clients/inference.rs` — Mac Mini inference client
  - `models.rs` — matching `api_contract.md`
- [x] **Create `qti_knowledge_base` collection** in Qdrant — **done** (size 384, Cosine; now holds **83 points** after 2026-07-31 ingestion). The curl below is historical (note the size is already corrected to 384):
  ```bash
  kubectl port-forward -n qdrant svc/qdrant 6333:6333
  curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
    -H 'Content-Type: application/json' \
    -d '{"vectors": {"size": 384, "distance": "Cosine"}}'
  ```
- [x] **Set up the Mac Mini inference server** — Ollama live via WireGuard tunnel (§3.2.2). The gateway is retrieval-only (§2.3.6) so it does not consume `INFERENCE_URL`; the Python agent calls Ollama directly.
- [ ] **Add secrets** — `QDRANT_URL` and any API keys should be Kubernetes Secrets, not hardcoded. (`INFERENCE_URL` dropped 2026-08-02 — retrieval-only decision §1.3.5; the env is dead and should be removed, not secretized.)
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

**Custom PrometheusRule `hite-infra-alerts`** (created 2026-07-22; **13 rules** as of 2026-07-31 — source of truth: `k8s/prometheus/hite-infra-alerts.yaml`):

| Alert | Severity | Trigger |
|---|---|---|
| `HITE_NodeCPUHigh` | warning | CPU > 90% for 5m |
| `HITE_NodeMemoryHigh` | warning | Memory > 85% for 5m |
| `HITE_NodeDiskHigh` | critical | Disk > 90% for 5m |
| `HITE_PodCrashLooping` | critical | CrashLoopBackOff for 2m |
| `HITE_NodeDown` | critical | node-exporter unreachable 2m |
| `HITE_QdrantDown` | critical | Qdrant unreachable 2m |
| `HITE_NodeMemoryPressure` | critical | MemoryPressure node condition for 2m |
| `HITE_NodeDiskPressure` | critical | DiskPressure node condition for 2m |
| `HITE_NodeRecentlyRestarted` | warning | node booted < 5m ago (immediate) |
| `HITE_ContainerMemoryNearLimit` | warning | container memory working set > 90% of limit for 5m |
| `HITE_ContainerCPUThrottled` | warning | > 50% of CFS periods throttled for 5m |
| `HITE_DeploymentReplicasMismatch` | warning | spec replicas ≠ status replicas for 5m |
| `HITE_ApiGatewayDown` | critical | api-gateway (`job="api-gateway"`) unreachable 2m |

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
- [ ] (DS-side, proposed) Error→eval feedback loop: a curator pulls error-shaped tickets from Loki into `golden_datasets.json` so the 5W1H baseline is graded on production shapes. The error LOG is already in Loki (§4.1.1); the Telegram message is only the alert derived from it — never parse Telegram back into a store. Raw errors must NOT be auto-embedded into Qdrant (human-gated SOP authoring via §2.4 only). Status: PROPOSED, not built.
- [ ] (DS-side) Agent error counters (JSON-decode / Ollama-timeout / empty-retrieval) are DS-owned and ride with the agent observability hooks (§1.4); the gateway-side `qti_*` business counters remain Farrel's (§4.4). Note for Phase 8: the Mac Mini *network* blocker is gone (WireGuard §3.2.2 + DS SSH tunnel §1.3.7); the only thing left blocking AI-pipeline monitoring is the metrics themselves (Farrel's `qti_*` + DS hooks).

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
| **Phase 5** | Konfigurasi Production | ✅ | Datasource & Ingress siap, Contact Point Telegram aktif. *Retention Loki sudah diatur menjadi 720 jam = 30 hari*. |
| **Phase 6** | Dashboard | ⚠️ | Node Exporter & K8s Monitoring siap. *Dashboard bisnis HITE blocked (nunggu Farrel)*. |
| **Phase 7** | Alert Rules | ⚠️ | 13 alert infra + 1 log-based alert aktif. *Alert bisnis belum (menunggu instrumentasi Farrel).*. |
| **Phase 8** | Monitoring AI Pipeline | ❌ | Belum mulai. Blocked total oleh masalah routing network Mac Mini. |
| **Phase 9** | Testing | ✅ | Testing infra E2E selesai. *Testing log E2E selesai. Log api-gateway (namespace qti) berhasil masuk ke Loki.*. |
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
- [✅] **Rust API (`api-gateway`)**: Metric infra ✅, log ✅, alert Down ✅
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
2. **Two different network paths to the Mac Mini exist simultaneously and neither team seems aware of the other's:** Platform Eng relies on a **Reverse SSH Tunnel** (`launchctl`-managed, ports 19100/11434) while Data Engineering relies on **Tailscale** (`100.79.30.90`). **Update 2026-07-30:** a third path has been added — **WireGuard over Tailscale** (`10.10.10.0/24`, §3.2.2) — which provides a dedicated encrypted tunnel between the VM and Mac Mini for LLM traffic (5x latency improvement over Tailscale DERP relay). This is now the recommended path for inference traffic. The Reverse SSH Tunnel remains in place for Node Exporter metrics and legacy access.
3. **Location is still in flux.** As of Platform Eng's report the Mac Mini was at an employee's home, not the office — full-hosting plans should nail down the physical location first, since it changes which network path (tunnel vs. Tailscale vs. office LAN) is even viable.
4. **ARM build implications for full-hosting:** if `data-pipeline`, `api-gateway`, and any future services also move onto the Mac Mini, expect the same first-build ARM recompilation tax described in Data Engineering §2.3.8 for any new Rust crate, and check that all Docker images used cluster-wide have `linux/arm64` variants — the current setup assumes `x86_64`-style workers.

### 6.3 Suggested next step

Given the above, before writing a full-hosting migration plan it's worth getting three yes/no answers on record: (1) is the Mac Mini the XOA hypervisor host — yes/no; (2) tunnel or Tailscale as the one standard path — **answered 2026-07-30: WireGuard over Tailscale (§3.2.2) is now operational and recommended for inference traffic**; (3) final physical location — office or remote. Everything else (DNS, firewall rules, ARM builds, Qdrant reachability for `data-pipeline`) hangs off those three answers.

---

## §7. Cross-Role Dependency & Blocker Map (NEW)

> Synthesized from the TODO/gotcha sections of all five reports — shows who's currently blocked on whom. Status column reconciled against live machine 2026-07-31.

| Blocked | Blocked on | Because | Status (2026-07-31) |
|---|---|---|---|
| Data Science E2E testing (§1.4) | Platform Engineering (Hilmi) | Rust API (port 8080) and Qdrant (port 6333) need a K0s network route (NodePort/LoadBalancer/kubeconfig RBAC) — see §1.3.1. | ✅ **Resolved** — api-gateway on NodePort 30082 (`http://100.106.122.68:30082`). Qdrant kept internal; DS E2E runs against the gateway. |
| Data Engineering `data-pipeline` ingestion (§2.4) | DevOps CI/CD (Ferdi) confirmation | `data-pipeline` runs outside the cluster and can't reach `qdrant.qdrant.svc.cluster.local` — needs a decided external path (Mac Mini, port-forward, or NodePort) — see §2.3.3. | ✅ **Resolved** — ingestion ran via SSH port-forward tunnel; `qti_knowledge_base` = **83 points**. |
| `/v1/query` returning real data (§2.4, §1.4) | Data Engineering (Farrel) | `clients::qdrant::search_sop` is written but not wired into the route handler yet, and the collection has 0 points until ingestion runs. | ✅ **Resolved** — wired (commit `10898a1`), deployed (`a0b4bec`), verified returning SOP text (e.g. SOP-GIT-003, SOP-DOC-001). |
| DevOps Prometheus/Grafana/Loki Phase 8 (AI pipeline monitoring) | Mac Mini networking resolution (Hilmi + Ferdi) | "Blocked total" per §4.4, Phase 8 — same root cause as the two rows above. | ⏳ **Partial** — Mac Mini reachable via WireGuard (§3.2.2), so the infra blocker is gone; pipeline-monitoring work (Loki logs + metrics) still in progress. |
| DevOps Prometheus/Grafana/Loki business-metric dashboards (Phase 6/7) | Farrel (Data Engineering) | Business metrics (`qti_confidence_tier_total`, etc.) require code instrumentation not yet written — see §4.4, Farrel's action items. | ⏳ Partial — DS now provides the Tier distribution from data (A=24/B=31/C=0, §1.6), anchoring the spec; Farrel's `qti_*` instrumentation still not written; `/metrics` still exposes only `health_checks_total` / `queries_total`. |
| Promtail logs for `api-gateway` (namespace `qti`) reaching Loki | RBAC debugging (Jep), explanation from Hilmi | Promtail currently only has RBAC for 4 namespaces (`argocd`, `hite-prod`, `kube-system`, `monitoring`) — `qti` is not among them; per §4.4 this needs Hilmi to explain the current allocation. | ✅ **Resolved** — `qti` namespace logs confirmed in Loki. |
| Any full-hosting decision (§6) | Ferdi + Hilmi | Needs the XOA hypervisor question answered and a single network path (tunnel vs. Tailscale) chosen. | ⏳ **Open** — XOA question still unanswered; WireGuard tunnel adopted as the LLM path. |
| `clients/inference.rs` (api-gateway) | Design confirmation from Data Science (Johan) | Possibly unnecessary if the Python agent calls Ollama/Qwen directly — see §2.3.6. | ✅ **Resolved 2026-08-02** — joint DS+DE decision: do NOT build it; gateway retrieval-only, generation in the DS agent (§1.3.5). Dead `INFERENCE_URL` env to remove (§2.3.12 / §3.4.1). |
| DS grounding rate (43.6%) / retrieval→synthesis join | Data Science (Johan) | Synthesis lands on 24/55; the empty-retrieval falsy gate in `agent.py` is the prime suspect (§1.4 / §1.6). | ⏳ Open (DS) — schema locked at 100%; raising grounding is the next DS experiment (observe-first fields + `is not None` gate fix, then re-run vs the 43.6% control). |

---

*End of Master Guidebook. All five source reports are preserved in full above; only escaping artifacts were cleaned, the DevOps report was split by role (CI/CD vs. observability), and organizational headers/cross-references were added.*
