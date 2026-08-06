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
    - **Repo-state note (crosscheck 2026-08-02, post-commit):** the five `llm-inference/` fixes (A–E) and the 55-ticket 100%/43.6% baseline are now committed to `Merpatidove/QTI-MAGANG` `main` as **`ef915cf`** (2026-08-02, author ardhityo1 — "feat(llm-inference): wire agent + first real-data 5W1H baseline (100% schema / 43.6% grounded)"). Verified against the local checkout the same day: `grade_result.py` reads `5w1h_output` + reports `grounded` (Fix A); `test_run.py` → agent `/process-ticket` via `AGENT_URL` (Fix B); `agent.py` → `127.0.0.1:11434` via `OLLAMA_URL`, timeout 300 (Fix C); `tools.search_sop` → gateway NodePort 30082 `/v1/query` (Fix D); synthesis prompt demands the exact six keys (Fix E); `.gitignore` fixed (no BOM, `venv`/`__pycache__`/ ignored); `evaluation_results.json` holds the 55 real agent triages (6-key `5w1h_output` on all 55, 24/55 grounded). §1.1 / §1.2 / §1.5 / §1.6 now match the committed state.
11. **2026-08-03 — Data Engineering sprint deliverables complete; gateway-side business metrics live.** Farrel closed the remaining Data Engineer items from §4.4 and the 2026-08-02 open list:
    - **Dead `INFERENCE_URL` env removed** from `api-gateway/k8s/deployment.yaml` (§2.3.12) — permanently dead under the retrieval-only decision (§1.3.5); removed, not secretized.
    - **`api-gateway-secrets` mounted** via `envFrom: secretRef` in `deployment.yaml` (Secret created by Hilmi in ns `qti`; resolves §3.4.1 "add secrets" / §2.3.9 Secret-mounting). Note: `QDRANT_URL` is still hardcoded in the `env:` block as a safety net — `env:` takes precedence over `envFrom`, so the Secret's value is currently shadowed; full migration to the Secret is an optional cleanup.
    - **Three gateway-side business metrics instrumented + verified live** in `api-gateway/src/routes/query.rs`, exported on `/metrics` and scraped by the existing ServiceMonitor:
      - `qti_qdrant_match_total{found="true|false"}` (counter) — did the Qdrant search return a usable SOP match.
      - `qti_request_duration_seconds` (histogram) — end-to-end `/v1/query` latency.
      - `qti_ticket_classification_total{classification="..."}` (counter) — responses by classification (retrieved / embed_error).

      Verified 2026-08-03: `POST /v1/query` → classification `"retrieved"` + real SOP; `GET /metrics` shows all three series. Deployed via the standard CI path (image SHA per the Actions tab).
    - **Re-scoping of the remaining business metrics:** `qti_confidence_tier_total`, `qti_routing_decision_total`, `qti_fact_coverage_score` are **not** gateway metrics — under retrieval-only (§1.3.5) the gateway never produces a Tier, routing decision, or fact-coverage score. Those measure the DS agent's 5W1H output and move to Johan's agent-side observability hooks (§1.4). Jep's Phase 6/7 business dashboards can now proceed against the three live retrieval metrics; Tier-based dashboards wait on the DS agent hooks + Tier spec (§1.4).
    - **Still open (DE):** `http_requests_total` registered but never incremented (§0 item 9); `data-pipeline` CI/CD; embedding consistency guard; optional `QDRANT_URL` full-Secret migration; optional `api_contract.md` wording update.
12. **2026-08-03 (DS grounding experiment completed; reconciled):** Grounding raised from the 24–28/55 control band (43.6–50.9%) to a treatment band of **52–53/55 (94.5–96.4%, ≈95%)**, verified genuine (all grounded tickets carry non-empty SOP context; 0 empty-preview). The archived experiment record (`3a03a55`, `treatment2_errorgate_0803.json`) grades 53/55; the latest committed `evaluation_results.json` (`edb7dff`, HEAD `0cde7e2`) grades 52/55 — the 1-ticket delta is which tickets fell through on each stochastic run (archived: TKT-1054/1055, `action_taken = none`; HEAD: TKT-1016/1041/1047), not a metric change. Root cause fixed: the synthesis gate's `"Error" not in str(tool_output)` substring veto false-positived on SOP text containing "Error"; fix = structured `_is_err` check. Intermediate `is not None` hypothesis falsified first (24/55, in-band).
13. **2026-08-03 (DS agent observability delivered; Jep fully unblocked).** `agent.py` now implements `prometheus_client` to expose AI-pipeline metrics on `/metrics` (bound `0.0.0.0`), scrapeable over the Tailscale mesh at Johan's IP `100.126.65.74:8000/metrics`. Five `qti_*` metrics: `qti_llm_request_duration_seconds{phase}`, `qti_llm_tokens_total{type}`, `qti_agent_parse_errors_total`, `qti_agent_ollama_timeouts_total`, `qti_agent_empty_retrieval_total`. Business-metrics spec formalized (Tier A = complete + grounded; Tier B = complete + ungrounded; Tier C = incomplete/escape) and handed to Jep. §1.4 DS TODOs closed; §4.1.4 documents the scrape target; §4.4 Phases 6/7 unblocked, Phase 8 done; §7 rows 4–5 resolved.
14. **2026-08-04 — Phase 8 (Observability & LLM Integration) 100% completed.** Reconciled live cluster, Mac Mini, and `DEBIAN13` state following end-to-end connectivity verification:
    - **Ollama binding updated.** Mac Mini Ollama reconfigured to bind `0.0.0.0:11434` (`*:11434`), enabling cross-mesh access. Verified via Tailscale from `DEBIAN13` (`http://100.79.30.90:11434/api/tags`).
    - **`qti-agent` deployment live.** Pod `qti-agent` deployed and running (`1/1 Running` in ns `qti`), HTTP gateway responding on port `8080`. *(Superseded 2026-08-06 — no `qti-agent` deployment exists in the cluster; manifests moved to `k8s/archive/qti-agent/`; see §0 item 19.)*
    - **Agent Johan metrics registered.** Scrape target `100.126.65.74:8000` added to `k8s/prometheus/prometheus-additional.yaml` and committed to repo.
    - **Phase 8 officially closed.** Status raised from ~40% to **100% Done**.
15. **2026-08-05 — Stage 2 real-data validation begins (manual curation) + reproducibility experiments.**
   - **Real-data eval (manual, ahead of Loki access):**
     - Built the error→eval pipeline by hand: `collected_error_logs.json` (14 raw records from real debugging — Ollama/WireGuard outage, Loki/Grafana outage, dev noise) → `real_tickets.json` (3 flat tickets: REAL-001 Ollama, REAL-002 Loki, REAL-003 nginx).
     - Added a `DATASET` env override + shape-tolerant loader to `test_run.py` so one harness grades either the synthetic golden set or the real set.
     - **Coverage-gap finding:** all 3 returned `grounded=True`, but REAL-001/002 grounded in the **wrong SOPs** (SOP-KIT-002 liveness-probe, SOP-DB-001 MySQL) because no Ollama/Loki SOP exists; only REAL-003 (nginx) matched its real SOP (SOP-DOC-002). Retrieval always returns a nearest neighbor, so the **naive grounding check (`how != placeholder`) can't distinguish right-SOP from wrong-SOP grounding**. Honest read: 1/3 correctly grounded, 2/3 grounded-in-wrong-SOP.
     - **Next Stage 2 step = SOP knowledge-base expansion:** author `SOP-INF-003` (Ollama unreachable over WireGuard) + `SOP-INF-004` (Loki datasource unreachable) → append to `RAG_Manual.md` → Farrel re-ingests (384-dim) → re-eval. Caveat: once ingested, REAL-001/002 become "seen"; keep collecting fresh tickets for generalization.
   - **Reproducibility experiments (added `OLLAMA_TEMPERATURE` / `OLLAMA_SEED` env knobs to `agent.py`):**
     - default temp, random seed → band **52–54/55** (~11 "fragile" tickets take turns failing per draw — that's the fluctuation).
     - temp **0.1** + seed 42 → **45/55**, 9 parse errors (JSON degeneration — an ablation finding: low temp degrades synthesis reliability).
     - temp **0.8** + seed 42 → **49/55**, then **48/55** with a *different* ungrounded set (only TKT-1055 overlapping) → **a fixed seed is NOT fully reproducible on Metal** (llama.cpp GPU non-determinism).
     - Final clean run: all 55 chose `search_sop`, 0 empty retrievals, but **7 synthesis parse failures** → 48/55. Dominant remaining loss = synthesis JSON decode failures → fix = **synthesis retry-on-parse-failure**.
     - CI guidance: gate on a **threshold** (e.g. `grounded ≥ 45/55`), not an exact number.
   - **Repo hygiene:** archives `evaluation_results_real_0804.json` + `evaluation_results_seed42_0805.json`; committed `agent.py`, `test_run.py`, `real_tickets.json`, `collected_error_logs.json` + archives. HEAD `evaluation_results.json` kept as the documented 52/55 synthetic Stage-1 baseline.
16. **2026-08-05 — Additional Scrape Targets & Business Alert Rules Live (Jep).**
    **Integrasi scrape target eksternal dan aturan alert bisnis AI pipeline berhasil diselesaikan dan terverifikasi secara E2E:**
      - Additional Scrape Configs Live: Secret `prometheus-additional-configs` berhasil dibuat dan ditautkan ke Custom Resource Prometheus (`additionalScrapeConfigs`). Target `mac-mini-   external` (100.79.30.90:9100) dan `qti-agent-johan` (100.126.65.74:8000) aktif ter-scrape. Ingest metrik `qti_llm_tokens_total` (prompt: 34k, completion: 14k) terverifikasi real-time di Grafana Explore.
      - PrometheusRule `qti-business-alerts` Deployed: Aturan alert bisnis AI pipeline (`QTI_OllamaTimeoutSpike` & `QTI_AgentParseErrorHigh`) berhasil dibuat dan di-label dengan `release: prometheus`. Rule berhasil direconcile oleh Prometheus Operator dan berstatus Normal (Hijau) di Grafana Alerting UI.
      - Git Repository Synchronized: File `k8s/prometheus/prometheus-additional.yaml` dan `k8s/prometheus/qti-business-alerts.yaml` telah di-commit dan di-push ke repositori `Merpatidove/QTI-MAGANG` branch `main`.
17. **2026-08-05 — Data Engineering sprint fully closed; `data-pipeline` upgraded to v1.1; Stage 2 KB expanded to 110 points.** Farrel completed all remaining closeout tasks, resolved critical network routing bugs, and executed the Stage 2 SOP expansion:
    - **`api-gateway` configuration finalized:** Hardcoded `QDRANT_URL` removed from `deployment.yaml`; the mounted `api-gateway-secrets` Secret is now the single source of truth. Dead `http_requests_total` counter wired via an Axum `count_requests` middleware in `main.rs`.
    - **`api_contract.md` updated to v2.0:** Explicitly documents the retrieval-only contract. `cognitive_triage`, `grounding_citations`, and `routing_decision` removed from the gateway schema and delegated to the DS agent.
    - **`data-pipeline` upgraded for RAG Manual v1.1:** Replaced the fragile parser with a robust line-by-line state machine. Now extracts explicit `SOP_ID`, `Category`, `Confidence_Tier`, and `Tags` (stored as arrays for Qdrant filtering). Added auto-reset logic (deletes/recreates collection to prevent duplicates) and batched upserts (batch size 32) to prevent SSH tunnel drops.
    - **Stage 2 KB Expansion:** Ingested 24 SOPs (including `SOP-INF-003..008`). `qti_knowledge_base` now holds exactly **110 points**.
    - **Network routing workaround documented:** Discovered a `kube-router` ClusterIP routing bug on the controller. Implemented a definitive 3-terminal workaround (`kubectl port-forward` -> SSH tunnel -> local `cargo run`).
18. **2026-08-06 — Methodology finalized; §8 promoted from draft to final.** §8 now carries the full two-stage methodology (system under test, metrics, Stage 1 + Stage 2 results, findings, limitations). Stage 2 real set expanded to 8 tickets (REAL-001..008); SOP expansion SOP-INF-003..008 → RAG_Manual v1.1 (24 SOPs) ingested by Farrel → qti_knowledge_base = 110 points. Real-set grounding 6/8 (75%); post-ingest correct-SOP re-eval pending.
19. **2026-08-06 — State reconciled against live machine + cluster cleanup.** Every §1–§4 "current state" claim was re-checked against the deployed cluster on 2026-08-06. Corrections below; all cluster-side changes are listed in the same bullet:
   - **`qti-agent` deployment is NOT running.** §0 item 14 / §4.4 Phase 6+8 / §7 claimed a `qti-agent` Deployment "1/1 Running in ns qti". No such deployment/pod exists in the cluster; the manifests were moved to `k8s/archive/qti-agent/` (commit `5ad3042`, same commit that added the additional scrape targets). The live agent is Johan's laptop process (`uvicorn agent:app` on `100.126.65.74:8000`, §4.1.4), not a cluster pod. §4.4 / §7 rows corrected.
   - **`hite-ai-pipeline-alerts` PrometheusRule added (was undocumented).** 5 rules live (`HITE_AIOllamaTimeoutHigh`, `HITE_AILLMLatencyHigh`, `HITE_AIParseJSONErrorsHigh`, `HITE_AIRetrievalEmptyHigh`, `HITE_AIAgentDown`); source `k8s/prometheus/hite-ai-pipeline-alerts.yaml`. **Gotcha:** `HITE_AIAgentDown` fires on `up{job="qti-agent"} == 0`, but no live scrape job is named `qti-agent` (actual jobs: `qti-agent-johan` from the additional-scrape config) — the rule can never fire as written. Follow-up TODO: fix the job matcher.
   - **`hite-infra-alerts` = 14 rules, not 13.** Prose updated (the §4.1.3 table already listed all 14 incl. `HITE_MacMiniDown`).
   - **Qdrant points 83 → 110** in §1.1 / §1.3.3 / §3.4.1 (was already 110 in §2.1). Verified live: 110 points, 384-dim/Cosine, green.
   - **api-gateway deployed image `a0b4bec` → `485024c`** (matches `api-gateway/k8s/kustomization.yaml`). The `a0b4bec` references remain only as historical notes.
   - **`johan-qti-agent` ScrapeConfig CRD added then removed.** It duplicated the `qti-agent-johan` additional-scrape job for the same endpoint (`100.126.65.74:8000`). Follow-up TODO (see §4.1.4): keep only one scrape mechanism. The ScrapeConfig was deleted from the cluster on 2026-08-06; the repo file `k8s/prometheus/johan-agent-scrape.yaml` remains and should be removed or archived.
   - **`hite-qti-agent-monitor` ServiceMonitor deleted (dead).** It selected `app=qti-agent`, but no Service carries that label (qti-agent archived) — matched nothing. Deleted from cluster 2026-08-06; repo file `k8s/prometheus/qti-agent-monitor.yaml` remains and should be removed or archived.
   - **Mac Mini scrape target DOWN as of 2026-08-06.** `mac-mini-external` → `100.79.30.90:9100/metrics` health = down; `qti-agent-johan` → `100.126.65.74:8000` is up. §0 item 16's "both targets live" was true on 2026-08-05; the Mac Mini endpoint is currently unreachable — follow-up TODO.
   - **Orphan `jaeger` ServiceMonitor deleted.** Jaeger was uninstalled (§0 item 7) but a leftover `jaeger` ServiceMonitor (9d) remained in `monitoring`; deleted 2026-08-06.
   - **Leftover test pods deleted.** `trigger-log-loop` (Running) and `curl-test3` (Completed) in `monitoring` removed 2026-08-06.
   - **Grafana dashboards 29 → 28** labeled dashboard ConfigMaps (incl. `grafana-dashboard-hite-llm-retrieval`).
   - **Ollama binding reconciled to `0.0.0.0:11434`** (§0 item 14, 2026-08-04), superseding the `10.10.10.2:11434` text in §1.3.7 / §3.2.2 — noted in both places.
   - **NFS CSI controller 3/3 → 5/5** ready containers (the deployment's container count grew).
20. **2026-08-06 — Platform Engineering §5 refreshed to 2026-08-05 machine state (Hilmi).** Reverse SSH Tunnel (`autossh` / `com.hite.tunnel.plist` via `launchctl`) formally deprecated and removed; resilience is now Tailscale mesh + WireGuard. Workers reach Mac Mini Ollama via the **mac-mini-ops** WireGuard tunnel `10.7.0.63` (firewall bypass `sudo iptables -I OUTPUT 7 -d 10.7.0.0/24 -j ACCEPT`; guidebook's `10.10.10.2` is the **controller-side** wg0 tunnel — verified live on the controller 2026-08-06, peer handshake active; §3.2.2 annotated to clarify the two tunnels' roles). Air-gap firewall (update-firewall-v2.sh) now also allows Tailscale CGNAT `100.64.0.0/10` + WireGuard `10.7.0.0/24`; `iptables-persistent` installed on worker-2. XOA hypervisor confirmation still open (§5.4).
21. **2026-08-06 — Qdrant outage: image-pull fix + `qti_knowledge_base` found EMPTY (0 points).** After a restart to clear a hung Raft state, the scheduler placed `qdrant-0` on **worker-2**, which does not cache the image; the pod stuck in `Init:ImagePullBackOff`. Workers cannot reach Docker Hub (air-gap: outbound + DNS 53 blocked), the in-cluster registry `10.20.20.202:32000` carries no qdrant image, and the controller host registry `10.20.20.201:5000` is DOWN (connection refused — separate open finding; §3.1.2 expects it up).
   - **Fix applied (no guidebook § for this yet):** exported `docker.io/qdrant/qdrant:v1.18.2` on worker-1 (`docker save` → `/tmp/qdrant-v1.18.2.tar`, 66.7 MiB), transferred via the controller, imported on worker-2 (`docker load`, digest `sha256:75eab8c4ba42096724fdcfde8b4de0b5713d529dde32f285a1f86fdcb2c9e50c`). Both nodes now cache the image, so future scheduler placements cannot re-block qdrant.
   - `qdrant-0` recreated → **1/1 Running on worker-2**; `healthz` check passed; collection `qti_knowledge_base` green (384-dim/Cosine, 4 segments). Controller cannot route to pod IPs (`10.244.3.185` refused) — use `kubectl port-forward` for API access.
   - **EMPTY-KB finding:** `qti_knowledge_base` reports **points_count = 0** (was 110 per §0 items 17/19). On the NFS backing store (`pvc-cd0fd51e…`, PV server `10.20.20.201`, share `/mnt/k8s-nfs`) the collection dir was recreated fresh at **2026-08-06 10:56–10:57 WIB** with new empty segment dirs; `.deleted` is empty and no snapshot exists anywhere — the 110 points are **not recoverable from disk**. Timing matches the data-pipeline v1.1 auto-reset (§0 item 17) and pre-dates the restart, i.e. likely intentional pre-ingestion cleanup, not outage data loss. Impact: DS retrieval returns no matches until re-ingestion. **Action:** Farrel re-ingests the RAG_Manual v1.1 72-point set (qdrant now Running; previously blocked).
   - **NFS server reconciliation:** the qdrant PV points at NFS server `10.20.20.201` (controller), not the `10.20.20.143 /upload/intern` referenced in older §5 text — reconcile §5 NFS notes.
22. **2026-08-06 — CI/CD: local registry fixed + pull-through proxy, ghcr.io mirror, data-pipeline CI; §3 refreshed.** The §3 refresh surfaced the local registry was actually crash-looping, not Running. All infra fixes applied 2026-08-06:
   - **Local registry `10.20.20.201:5000` fixed (was DOWN ~7 days).** Root cause: the `registry:2` container bind-mounted `/tmp/certs:/certs` and the self-signed cert in `/tmp` had been wiped → `fatal: open /certs/registry.crt: no such file or directory`, crash-loop. Fix: regenerated a self-signed cert (SAN `IP:10.20.20.201`, 10y) at the **persistent** path `/etc/docker/registry/` (never `/tmp` again), recreated the container with `-p 5000:5000`, the existing data volume (legacy images preserved: `busybox`, `curlimages/curl`, `grafana/loki`, `grafana/promtail`, `jaegertracing/jaeger`, `prometheus/alertmanager`), and **`REGISTRY_PROXY_REMOTEURL=https://ghcr.io`** (pull-through proxy mode). Redistributed the new CA to `/etc/docker/certs.d/10.20.20.201:5000/ca.crt` (controller) and `/etc/k0s/containerd/certs.d/10.20.20.201:5000/ca.crt` (both workers). Verified: catalog served; workers reach it; container can reach ghcr.io. **Caveat:** registry:2 proxy mode ignores pushes — `10.20.20.201:5000` is now a pull-through cache only; the in-cluster `hite-prod` registry (`10.20.20.202:32000`) remains the cluster push target.
   - **ghcr.io pull gap fixed (next CI deploy would otherwise have failed).** Workers' containerd had no mirror for ghcr.io and cannot reach it (air-gap), so the next api-gateway image would `ImagePullBackOff` exactly like qdrant. Fix: added `/etc/k0s/containerd/certs.d/ghcr.io/hosts.toml` on both workers → mirror `server = "https://10.20.20.201:5000"` (+CA). Workers now pull `ghcr.io/merpatidove/...` through the local proxy, which fetches/caches from ghcr.io. **Verified end-to-end 2026-08-06:** a pod on worker-1 with `imagePullPolicy: Always` pulled `ghcr.io/merpatidove/qti-api-gateway:e40ba85` successfully (33 MB, 4.5 s) via the mirror. Argo CD manifests/CI are unchanged — CI still publishes to ghcr.io.
   - **data-pipeline CI added.** New `.github/workflows/data-pipeline.yml` (trigger: push to `data-pipeline/**` + `workflow_dispatch`): `cargo check` → build+push `ghcr.io/merpatidove/qti-data-pipeline:<sha>` (GitHub Actions only; no cluster deploy). New `data-pipeline/Dockerfile` (multi-stage, bakes all-MiniLM-L6-v2 via `--download-only`, mirrors api-gateway pattern). To support that, `data-pipeline/src/main.rs` gained a `--download-only` guard + `EMBED_CACHE_DIR` support (identical to api-gateway's pattern). Ingest still runs manually (`cd data-pipeline && cargo run`, §2.3.10 workaround); a cluster CronJob is a documented follow-up.
   - **§3 refreshed** to match: local-registry row corrected, pipeline flow documents the mirror step, §3.4.1/§3.4.2 checkboxes closed (secrets, `.gitignore`, smoke-test scope), rag-service rows struck, §3.4.3 `data-pipeline CI/CD` closed, quick-ref gotcha added (`curlimages/curl` test pods fail on air-gapped workers — use images from the local registry/mirror).
   - **Still open:** `curlimages/curl`-style `kubectl run` tests need a pre-loaded/mirrored image (documented in §3.5); legacy images on the proxy registry should eventually be re-pushed to `hite-prod` `32000`.
23. **2026-08-06 — KB re-ingestion complete (103 points); `data-pipeline` parser finalized; Qdrant Raft/NFS write-latency flagged to DevOps.** Following the §0 item 21 outage, Farrel completed the re-ingestion and finalized the v1.1 parser:
    - **Fence-tracking parser fix:** the interim "terminate-on-H1" parser wrongly truncated each SOP at the first `# ` comment line *inside* its fenced ` ```bash ` Remediation block (built only 72 truncated points). The final parser tracks markdown code fences so `# ` comments inside bash/python blocks are never mistaken for headers, and terminates SOPs on the `---` horizontal rule (also cleanly excluding the Python Appendix). Result: full Remediation text in every chunk.
    - **Smart-reset logic:** the auto-reset (delete+create) hit Qdrant's Raft consensus timeout (`Waiting for consensus operation commit failed. Timeout set at: 10 seconds`) on the NFS-backed PVC. The pipeline now checks `points_count` first and only resets when the collection has data; when empty it upserts directly, bypassing the heavy delete+create op (§2.3.14).
    - **Final state:** `qti_knowledge_base` = **103 points** (24 SOPs, 384-dim/Cosine), every payload carrying `sop_id`, `category`, `tier`, and a `tags` array. Verified live 2026-08-06. **Supersedes the "110 points" (2026-08-05) and the item-21 "72-point set" figures — 103 is the correct post-fix count** (tighter chunking, no Appendix bleed).
    - **Empty-KB root cause (for the record):** the v1.1 auto-reset's delete+recreate *succeeded* (fresh empty collection at ~10:56), but the refill upsert failed at that exact moment (pod rescheduled to worker-2, stuck in `ImagePullBackOff` / Raft timing out) — so the collection sat at 0 points. Not a data-model problem; bad infra timing.
    - **Infra follow-up (handed to Ferdi):** Qdrant Raft consensus remains slow on heavy writes post-restart (likely NFS WAL fsync latency on the `10.20.20.201` share). DE workaround in place (smart reset); DevOps to check `kubectl logs qdrant-0 -n qdrant` for Raft warnings / NFS latency and consider local (non-NFS) WAL storage or a tuned consensus timeout.
    - **DS unblocked:** Tyo notified the KB is ready for the REAL-003 probe + 8-ticket Stage 2 re-eval (§8.4 post-ingest re-eval pending).

---

## Table of Contents

- [§1 — Data Scientist (Owner: Johan)](#1-data-scientist-owner-johan)
- [§2 — Data Engineer (Owner: Farrel)](#2-data-engineer-owner-farrel)
- [§3 — DevOps: CI/CD (Owner: Ferdi)](#3-devops-cicd-owner-ferdi)
- [§4 — DevOps: Prometheus / Grafana / Loki (Owner: Jep)](#4-devops-prometheus--grafana--loki-owner-jep)
- [§5 — Platform Engineering (Owner: Hilmi)](#5-platform-engineering-owner-hilmi)
- [§6 — Mac Mini Full-Hosting: Consolidated View (NEW)](#6-mac-mini-full-hosting-consolidated-view-new)
- [§7 — Cross-Role Dependency & Blocker Map (NEW)](#7-cross-role-dependency--blocker-map-new)
- [§8 — Data Science Methodology — 5W1H Triage Evaluation (FINAL)](#8-data-science-methodology--5w1h-triage-evaluation-final)

---

## §1. Data Scientist (Owner: Johan)

### LLM Inference & 5W1H Evaluation Pipeline — Infrastructure & State Report

**Date:** 2026-07-29 **Owner:** Johan / Data Scientist **Repo/Path:** https://github.com/Merpatidove/QTI-MAGANG/tree/main/llm-inference

#### 1.1 What's Running / Current State

| Component / File | Status | Access / Details |
| --- | --- | --- |
| llm-inference/agent.py | Active / Generation & Metrics | FastAPI ReAct orchestrator on :8000 (`POST /process-ticket`). Calls Ollama (Qwen2.5-Coder-7B-Instruct Q4_K_M) over the SSH tunnel → WireGuard (§1.3.7 / §3.2.2) for the 5W1H + tool choice; calls the gateway `/v1/query` via `tools.search_sop` for RAG context; synthesis phase grounds `why`/`how` in the retrieved SOP. **Synthesis gate now uses `_is_err` (structured error check) so retrievals whose SOP text contains "Error" still reach synthesis — raised grounding from the 43.6–50.9% control band to a **94.5–96.4% treatment band** (2026-08-03).** **Update 2026-08-03: now implements `prometheus_client` to expose AI-pipeline metrics on `/metrics`; bound to `0.0.0.0` to allow Prometheus scraping over the Tailscale mesh.** **Update 2026-08-05: now also reads `OLLAMA_TEMPERATURE` / `OLLAMA_SEED` env knobs.** Ollama URL env-driven (`OLLAMA_URL`, default `http://127.0.0.1:11434`). |
| llm-inference/test_run.py | Active / unblocked | POSTs each ticket to the **agent** `/process-ticket` (`AGENT_URL`, default `http://127.0.0.1:8000`) and extracts the flat 6-key 5W1H dict. **Writes observe-first fields (`action_taken`, `result_preview`, `grounded`) into each result row for per-ticket mechanism audit (2026-08-03).** **Update 2026-08-05: gains a `DATASET` env override + shape-tolerant loader so one harness grades either the synthetic golden set or the real-ticket set.** No longer points at the gateway directly — that path was the 2026-07-31 retrieval diagnostic only. |
| llm-inference/grade_result.py | Active / two metrics | Reads `5w1h_output` (Fix A — was the never-written `output` key, the root cause of the eternal 0%). Reports **two** metrics: Complete 5W1H Schema (six keys present) and Grounded (`how` ≠ the `Pending SOP search` placeholder, constant `PLACEHOLDER_HOW`). Single source of truth for both definitions (handed to Ferdi for CI). |
| llm-inference/prompts.py | Stable | `REACT_SYSTEM_PROMPT` defines the six lowercase keys + tool-selection directives; proven to elicit the shape. |
| llm-inference/tools.py | Active / fixed | `search_sop` hits the live gateway NodePort 30082 `/v1/query` (was the deleted `rag-service:8000`). `execute_safe_cli` sandbox not deployed — non-critical; the agent catches the error and falls back to the analysis-phase 5W1H (still six keys). |
| `evaluation_results.json` | Generated (real) | Latest committed run (`edb7dff`, HEAD `0cde7e2`): 55/55 valid, 55/55 schema, **52/55 grounded (94.5%)**. Treatment band across runs: 52–53/55 (94.5–96.4%); archived `3a03a55` sample = 53/55. (2026-08-02 baseline: 43.6%.) |
| evaluation_results_{control2,treatment1_isnotnone,treatment2_errorgate}_0803.json | Archived | Experiment samples (2026-08-03): control band (24–28/55) + falsified `is not None` hypothesis + winning Error-gate treatment. |
| Rust API ( /v1/query ) | Deployed, live / retrieval-only | NodePort 30082. Returns retrieved SOP text in `remediation_payload.proposed_fix` (no 5W1H). Reached directly over Tailscale for the retrieval diagnostic; reached by the agent's `search_sop` for RAG context. |
| Qdrant DB ( qti_knowledge_base ) | Deployed, **103 points** | 384-dim / Cosine, **24 SOPs chunked (re-ingested 2026-08-06)**. Internal `http://qdrant.qdrant.svc.cluster.local:6333`; the agent reaches it only via the gateway. |
| llm-inference/collected_error_logs.json | **New** | Staging record of 14 raw error logs (real debugging) — the provenance for the real tickets |
| data-pipeline/real_tickets.json | **New** | Stage 2 real-ticket eval set (REAL-001/002/003); flat array, same schema as golden |
| llm-inference/evaluation_results_real_0804.json | **Archived** | Stage 2 real-ticket run (3/3 grounded, but 2 in wrong SOP) |
| llm-inference/evaluation_results_seed42_0805.json | **Archived** | seed-42 / temp-0.8 reproducibility baseline (49/55) |

> **Repo note (2026-08-02, post-commit):** rows above match the committed state — the five `llm-inference/` fixes and the real 55-ticket baseline are committed to `QTI-MAGANG` `main` as `ef915cf` and verified against the local checkout (see §0 item 10).

> **Repo note (2026-08-03, post-commit):** the grounding experiment (observe-first fields + `_is_err` synthesis gate) and the 96.4% run are committed to `QTI-MAGANG` `main` as `3a03a55` (see §0 item 12).

#### 1.2 Data scientist

The Data Science evaluation pipeline is currently executed manually to evaluate LLM inference outputs against a strict 5W1H (Who, What, When, Where, Why, How) schema.

- **Triggers:** Manual execution via `python test_run.py` to generate inference outputs, followed by `python grade_result.py` for parsing and validation.
- **Steps (2026-08-02 pipeline):**
  1. `test_run.py` POSTs each ticket to the agent `/process-ticket`.
  2. The agent runs a ReAct loop: Ollama produces a 5W1H analysis + tool choice; if `search_sop` is chosen, the agent retrieves RAG context from the gateway `/v1/query`; a synthesis call grounds `why`/`how` in it.
  3. `test_run.py` stores the flat six-key 5W1H dict under `5w1h_output` in `evaluation_results.json`.
  4. `grade_result.py` reports schema completeness AND grounding.
- **Last Successful Run:** Grounded **52–53/55 (94.5–96.4%)** treatment band, up from 43.6% (2026-08-02). HEAD's committed artifact grades 52/55; the archived `3a03a55` run grades 53/55; the delta is run-to-run stochasticity, not a metric change.
- **Baseline Run (2026-08-02):** first run where the schema score is measured on real generated data (was 0% on placeholders and on the wrong key before Fix A); 24/55 (43.6%) grounded was the first measurement of RAG grounding in the project. (The 2026-07-31 gateway-direct run that returned `ticket_metadata`/`remediation_payload` was the retrieval diagnostic, not the grading target.)

> **Repo note (2026-08-02, post-commit):** steps + run above match the committed state — committed to `QTI-MAGANG` `main` as `ef915cf`, verified against the local checkout (see §0 item 10).

> **Repo note (2026-08-03, post-commit):** the Error-gate treatment run above is committed to `QTI-MAGANG` `main` as `3a03a55` (see §0 item 12).

#### 1.3 Notable Observations & "Gotchas"

##### 1.3.1 k0s Networking on Mac Mini Host

Our K0s cluster does not expose internal services externally by default. Because the Mac Mini is hosting the cluster, local development environments cannot reach the internal DNS (`qdrant.qdrant.svc.cluster.local`) or the Rust API pod without an explicit ingress route. Platform Engineering must establish a NodePort, LoadBalancer, or issue `kubeconfig` RBACs for `kubectl port-forward` before Data Science E2E testing or Data Engineer DB population can proceed.

> **Cross-reference note:** this line says "the Mac Mini is hosting the cluster" — worth reconciling against Platform Engineering §5 (the k0s controller/workers are separate VM IPs `10.20.20.201/202/200`) and DevOps CI/CD §3.3.2 ("this VM is a single point of failure"). See §6 below for the consolidated Mac Mini picture and this exact discrepancy.

**Update 2026-08-02 (DS unblocked):** the Rust API route no longer needs a port-forward for DS — api-gateway is on NodePort 30082, reachable directly over Tailscale at `http://100.106.122.68:30082`. The remaining network door DS needs is Ollama, which is reached via the SSH local-forward in §1.3.7 (not a Platform Eng NodePort). The "Platform Engineering must establish a route" sentence above is therefore resolved for DS E2E; it remains accurate as historical context and for any component that still needs cluster-internal DNS.

##### 1.3.2 Evaluation Schema Strictness

The `grade_result.py` script enforces a hard, exact-match key check for the 5W1H schema. If the backend Rust API returns *any* structure other than `Who`, `What`, `When`, `Where`, `Why`, and `How`, the test will record a 0% success rate for that specific run, regardless of JSON validity.

##### 1.3.3 Qdrant Database State

The `qti_knowledge_base` collection is initialized with a 384-dim Cosine configuration and its status is Green. **As of 2026-08-06 it holds 103 points** (24 SOPs, chunked by the fence-tracking parser) — re-ingested after the 2026-08-06 outage (§0 items 21/23). *(First written when empty; interim 83 points / 18 SOPs on 2026-07-31 and 110 points on 2026-08-05.)*

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

> **Repo note (2026-08-02, post-commit):** this two-metric grader is committed to `QTI-MAGANG` `main` as `ef915cf` and verified against the local checkout (`grade_result.py` reads `5w1h_output`, reports schema + `grounded`; see §0 item 10).

##### 1.3.7 Ollama reachability from a Tailscale laptop — the SSH-tunnel door

Ollama on the Mac Mini binds to `0.0.0.0:11434` (`OLLAMA_HOST=0.0.0.0:11434`, updated 2026-08-04, §0 item 14) — reachable over the WireGuard interface at `10.10.10.2:11434` and over Tailscale from the controller. *(Before 2026-08-04 it bound only to the WireGuard interface `10.10.10.2:11434`, §3.2.2; a Windows laptop on Tailscale therefore CANNOT reach Ollama directly — `curl http://100.79.30.90:11434` and worker-1 `http://100.68.225.41:11434` both refused. The working door is still a local-forward through the controller, which is on the WireGuard subnet at 10.10.10.1):* *(2026-08-06, §0 item 20: the workers themselves reach Ollama over the mac-mini-ops WireGuard tunnel at `10.7.0.63` — §5.1 / §5.3.1. The controller-side `10.10.10.2` door below stays the DS-laptop path.)*

```bash
ssh -L 11434:10.10.10.2:11434 ferdi@100.94.99.125     # leave this session OPEN; it IS the tunnel
curl http://localhost:11434/api/tags                    # expect the model list
```

Auth is key-only (§3.3.2): the laptop needs an ED25519 key in `ferdi`'s `~/.ssh/authorized_keys` (`ssh-keygen -t ed25519 -C "johan-hite"`; hand the `.pub` line to Ferdi). Without it, SSH closes at auth (`Connection closed by 100.94.99.125 port 22`, no password prompt). `agent.py` reads `OLLAMA_URL` (default `http://127.0.0.1:11434`), so with the tunnel up the default works. First Ollama call cold-loads the 4.7 GB model (20–60 s) — that is why `call_ollama` timeout is 300, not 45. This tunnel is a foreground dependency: closing the SSH session kills generation mid-run. For a reproducible/monitored pipeline the agent should eventually run on a WireGuard-side host (controller/Mac Mini) so the path stops depending on a laptop.

##### 1.3.8 Real tickets can "ground" in the wrong SOP

Retrieval returns the nearest neighbor, never nothing, so a ticket with no matching SOP still yields `grounded=True` — backed by an irrelevant SOP. The naive check can't tell. Real-data grounding needs a retrieval-correctness signal (or human review), not just `how != placeholder`. (Observed 2026-08-05: REAL-001/002 grounded in SOP-KIT-002 / SOP-DB-001 because no Ollama/Loki SOP exists; see §0 item 15.)

##### 1.3.9 Temperature/seed trade-offs (Metal)

temp 0.1 → synthesis JSON degeneration (45/55, 9 parse errors). temp 0.8 + fixed seed → reduces but does **not** eliminate run-to-run variance (49→48, different ticket sets) — llama.cpp on Metal isn't bit-deterministic. Report a band/threshold, not a single point. (2026-08-05 reproducibility runs; see §0 item 15.)

##### 1.3.10 Synthesis parse failures are the last loss bucket

With tool-selection and retrieval healthy (all `search_sop`, 0 empty), the remaining ungrounded tickets are synthesis JSON decode failures. Fix: retry synthesis on parse failure. (2026-08-05 clean run: 48/55, 7 synthesis parse failures; see §0 item 15.)

#### 1.4 What Needs to Be Done (TODOs)

**Platform Engineering Unblocks**
- [x] Open K0s network route to expose Rust API (port 8080) to allow Data Science inference testing. *(done 2026-07-31 — api-gateway now NodePort 30082, reachable at `http://100.106.122.68:30082`)*
- [x] Open K0s network route to expose Qdrant (port 6333) to allow Data Engineering SOP ingestion. *(done — ingestion ran via SSH port-forward tunnel; Qdrant intentionally not exposed permanently, see §3.2.3)*

**Data Engineering Unblocks**
- [x] Ingest the 18 SOPs from the RAG manual, generate embeddings, and populate the empty Qdrant vector database. *(done 2026-07-31 — `qti_knowledge_base` = 83 points, 384-dim/Cosine; expanded to **110 points / 24 SOPs** by 2026-08-05, §1.1) (re-ingested at 103 points 2026-08-06, §0 item 23)*
- [x] Wire the Rust backend (`/v1/query`) to actively call `clients::qdrant::search_sop` instead of returning the placeholder payload. *(done 2026-07-31 — commit `10898a1`, deployed as `a0b4bec`)*
- [x] Confirm `clients/inference.rs` is needed — confirmed NOT needed (joint DS+DE decision 2026-08-02); gateway is retrieval-only (§1.3.5 / §2.3.6), file not built.

**Data Science (Johan)**
- [x] Run full 5W1H evaluation suite against live data — done 2026-08-02 via the agent pipeline: 55/55 valid, 55/55 schema, 24/55 (43.6%) grounded (§1.6). *(committed to `QTI-MAGANG` `main` as `ef915cf`, 2026-08-02)*
- [x] Raise grounding above the 43.6% baseline — **DONE 2026-08-03 (treatment band 94.5–96.4%, ≈95%)**. Real root cause was the "Error"-substring veto in the synthesis gate, not the falsy gate; the `is not None` hypothesis was falsified first.
- [x] Spec the business metrics from data — Tier A/B/C, escape-hatch rate, confidence. *(Done 2026-08-03: Tier A = Complete + Grounded; Tier B = Complete + Ungrounded; Tier C = Incomplete/Escape. Handed off to Jep for dashboards).*
- [x] Add agent-side observability hooks (JSON-decode failure, Ollama timeout, empty-retrieval counters). *(Done 2026-08-03: exposed 5 custom `qti_*` metrics on `/metrics`, §4.1.4).*
- [x] Finalize methodology documentation — done 2026-08-06 (METHODOLOGY.md committed; §8 final).
- [ ] (IN PROGRESS) Stage 2 real-data validation: manual error→eval curation DONE (3 real tickets, coverage-gap found). Next: SOP knowledge-base expansion (author SOP-INF-003/004 → Farrel ingest → re-eval) + collect more real tickets. Automated Loki curator still pending Loki access.

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
#   Grounded (how != pending): 52–53/55 (94.5–96.4%)  # treatment band; a single run = 52 or 53

# --- one-shot cross-tab (diagnose WHY ungrounded; uses only guaranteed fields) ---
python -c "import json,collections as c; d=json.load(open('evaluation_results.json',encoding='utf-8')); g=lambda i: bool(((i.get('5w1h_output') or {}).get('how') or '').strip()) and 'pending sop search' not in (((i.get('5w1h_output') or {}).get('how') or '').lower()); print('grounded mean %.1fs ungrounded mean %.1fs'%(sum(i['inference_time_sec'] for i in d if g(i))/max(1,sum(1 for i in d if g(i))), sum(i['inference_time_sec'] for i in d if not g(i))/max(1,sum(1 for i in d if not g(i))))"

# --- commit (exclude .venv / __pycache__ per §1.3.4) ---
git add llm-inference/agent.py llm-inference/prompts.py llm-inference/tools.py llm-inference/test_run.py llm-inference/grade_result.py llm-inference/evaluation_results.json
git commit -m "feat(llm-inference): wire agent + first real-data 5W1H baseline (100% schema / 43.6% grounded)"
git push origin main
```

```powershell
# --- Grade the real-ticket set instead of the synthetic one ---
$env:DATASET = "..\data-pipeline\real_tickets.json"
python test_run.py
Remove-Item Env:DATASET

# --- Reproducibility knobs (set in the agent's terminal before uvicorn) ---
$env:OLLAMA_TEMPERATURE = "0.8"   # 0.1 degenerates synthesis JSON (see ablation)
$env:OLLAMA_SEED = "42"
```

---

#### 1.6 Evaluation Results — 2026-08-03 (Error-gate treatment; Stage 1 synthetic baseline)

| Metric | Value | Meaning |
| --- | --- | --- |
| Grounded (how ≠ pending) | 52–53/55 (94.5–96.4%) | treatment band; HEAD committed = 52/55, archived `3a03a55` = 53/55 |
| Tier A (complete + grounded) | 52 (HEAD) / 53 (archived) | |
| Tier B (complete, ungrounded) | 3 (HEAD: TKT-1016/1041/1047) / 2 (archived: TKT-1054/1055) | |
| Tier C (incomplete) | 0 | schema is 100%, so none |

**Note (2026-08-03, reconciled):** bimodality collapsed — synthesis now runs on 52–53/55. Grounding verified genuine (0 empty-preview grounded tickets). Pre-fix control band: 24–28/55 (43.6–50.9%). The 52-vs-53 delta across runs is stochastic (which tickets fell through), not a metric change; report the band (or mean ± range over N runs) as the stable headline, not a single point. Stage 1 (synthetic) result; real-data validation is Stage 2.

What the 2026-08-02 baseline proved: the retrieval→generation join works end-to-end on a real fraction; the shape problem that printed 0% for weeks is closed; and — because the grader measures grounding — the project can finally SEE that "schema-complete" ≠ "RAG-grounded." That distinction is the binding constraint going forward, not a bug.

> **Repo note (2026-08-03, reconciled):** the committed `evaluation_results.json` at HEAD (`0cde7e2`, written by `edb7dff`) grades 52/55; the archived `treatment2_errorgate_0803.json` (`3a03a55`) grades 53/55. Both are valid samples of the ~95% treatment band.

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
| `api-gateway` Deployment (ns `qti`) | 1/1 Running, Healthy | NodePort **30082**. Image rebuilt 2026-08-03 (adds `qti_*` metrics) and again for the `http_requests_total` middleware — **currently deployed `ghcr.io/merpatidove/qti-api-gateway:485024c`** (tag auto-managed by CI in `kustomization.yaml`). `envFrom: secretRef: api-gateway-secrets` mounted; dead `INFERENCE_URL` env removed. |
| `api-gateway/src/main.rs` | Working, deployed | Orchestrator; wires router; exposes `/metrics`; binds `0.0.0.0:8080`. **`http_requests_total` wired via a `count_requests` middleware (2026-08-05)**. |
| `api_contract.md` | **v2.0 (2026-08-05)** | Retrieval-only contract. Gateway returns `{ticket_metadata, remediation_payload}` only. |
| `api-gateway/src/models.rs` | Working, deployed | `QueryRequest`, `QueryResponse`, `TicketMetadata`, `RemediationPayload` (serde; matches `api_contract.md`). |
| `api-gateway/src/routes/mod.rs` | Working, deployed | Module table of contents: `pub mod health; pub mod query;` |
| `api-gateway/src/routes/health.rs` | Working, deployed | `GET /v1/health` → `{"status":"ok","version":"0.1.0"}`; counter `health_checks_total`. K8s liveness/readiness target. |
| `api-gateway/src/routes/query.rs` | Working, deployed | `POST /v1/query`; returns real `QueryResponse` from `search_sop`. **Now also instruments `qti_qdrant_match_total`, `qti_request_duration_seconds`, `qti_ticket_classification_total`** (2026-08-03), all exported on `/metrics`. |
| `api-gateway/src/clients/mod.rs` | Working, deployed | Module table of contents: `pub mod qdrant;` |
| `api-gateway/src/clients/qdrant.rs` | Written, compiled, **WIRED + deployed** | `search_sop(Vec<f32>) -> Result<...>` via `reqwest` (REST, :6333). Constants `QDRANT_URL = http://qdrant.qdrant.svc.cluster.local:6333`, `COLLECTION_NAME = qti_knowledge_base`. Called from `query.rs` since commit `10898a1` (2026-07-31). |
| `api-gateway/src/clients/inference.rs` | **NOT written** | Mac Mini inference client — possibly obsolete (see §2.3.6 / §2.4). |
| `api-gateway/Cargo.toml` | Working | axum 0.8, tokio `[full]`, serde `[derive]`, serde_json, reqwest 0.12 `[rustls-tls, json]`, tracing, tracing-subscriber `[env-filter]`, prometheus 0.13 `[process]`, lazy_static 1.4, anyhow 1.0, **fastembed 4**. |
| `api-gateway/Dockerfile` | Working | Multi-stage `rust:1-bookworm` → `debian:bookworm-slim`; was ~32 MB before `--download-only` baked the embedding model in (image `a0b4bec` is larger). |
| `api-gateway/k8s/deployment.yaml` | Working | Liveness/readiness on `/v1/health`; `EMBED_CACHE_DIR=/opt/fastembed`; `envFrom: secretRef: api-gateway-secrets` mounted; `INFERENCE_URL` and hardcoded `QDRANT_URL` removed (2026-08-05) — Secret is the single source of truth. |
| `api-gateway/k8s/service.yaml` | Working | NodePort 30082 (was ClusterIP on 8080; changed 2026-07-31). |
| `api-gateway/k8s/kustomization.yaml` | Working | `newTag: <sha>` managed by CI commit-back. |
| `api-gateway/k8s/servicemonitor.yaml` | Working | Prometheus scrapes `/metrics` every 15s. |
| `data-pipeline/src/main.rs` | Working (**v1.1 fence-tracking parser + smart reset**) | Reads `RAG_Manual.md` via a line-by-line state machine that tracks code fences (so `# ` bash comments never truncate Remediation blocks) and terminates SOPs on `---`. Extracts explicit `SOP_ID`, `Category`, `Confidence_Tier`, `Tags` (array). Smart reset (resets only when `points_count` > 0, §2.3.14) + batched upserts of 32. |
| `data-pipeline/Cargo.toml` | Working | Parser uses std only; staged for next step: `fastembed`, `qdrant-client`, `uuid`, `serde`, `serde_json`, `anyhow`, `tokio`. |
| `data-pipeline/RAG_Manual.md` | Present (data) | 18 structured SOPs — the ingestion source. |
| `data-pipeline/golden_datasets.json` | Present (data) | Sample tickets — DS evaluation harness; **not** consumed by the Rust code yet. |
| Collection `qti_knowledge_base` | Created, **green, 103 points** | **384-dim / Cosine**. 24 SOPs re-ingested 2026-08-06 (RAG Manual v1.1, fence-tracking parser; supersedes the 110-point 2026-08-05 run). Payload includes `tags` array for filtering. |
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

**2026-08-03:** **removed** — `INFERENCE_URL` no longer exists in `k8s/deployment.yaml` (§0 item 11).

##### 2.3.13 The `kube-router` ClusterIP Bug & 3-Terminal Ingestion Workaround

As of 2026-08-05, the controller VM (`DEBIAN13`) suffers from a `kube-router` networking bug where SSH tunnels targeting Qdrant's ClusterIP (`10.108.156.131`) or DNS (`qdrant.qdrant.svc.cluster.local`) time out or fail to resolve. Standard `ssh -L` commands will fail with `Connection timed out` or `Name or service not known`.

To ingest data locally, you must bypass the broken cluster network using a 3-terminal workaround:
1. **Terminal 1 (Controller Port-Forward):** Bypasses the network by talking directly to the K8s API.
   `ssh ferdi@100.94.99.125` then `kubectl port-forward -n qdrant svc/qdrant 6333:6333`
2. **Terminal 2 (SSH Tunnel to Laptop):** Maps local port to controller's localhost.
   `ssh -L 6333:127.0.0.1:6333 ferdi@100.94.99.125`
3. **Terminal 3 (Local Ingestion):** `cd data-pipeline && cargo run`

##### 2.3.14 Qdrant Raft consensus timeout on heavy writes (NFS WAL)

After the 2026-08-06 pod restart, `data-pipeline`'s delete+recreate began failing with `Service internal error: Waiting for consensus operation commit failed. Timeout set at: 10 seconds`. A single-node Qdrant should commit instantly; the timeout points to slow WAL fsync on the NFS-backed PVC (`10.20.20.201` share) or degraded Raft state post-restart. **DE workaround (2026-08-06):** the pipeline checks `points_count` first and skips the delete+create when the collection is already empty (smart reset), upserting directly in batches of 32. **Infra follow-up (Ferdi):** check `kubectl logs qdrant-0 -n qdrant` for Raft warnings + NFS latency; consider local (non-NFS) storage for the Qdrant WAL or a tuned consensus timeout. Until then, avoid delete+recreate against a slow cluster; prefer upsert-to-empty.

#### 2.4 What Needs to Be Done (TODOs)

**My lane — shipped**
- [x] Modularize `api-gateway`; `/v1/health` live; `/v1/query` returns real data; `clients/qdrant.rs::search_sop` wired; root `.gitignore`; `rag-service/` removed.
- [x] `data-pipeline` parser isolates all 18 SOPs; `qti_knowledge_base` created at 384/Cosine.
- [x] Reachable Qdrant path for `data-pipeline` (SSH port-forward); collection populated (83 points, later **110 points / 24 SOPs**, §1.1).
- [x] Wire `/v1/query` → `search_sop` (commit `10898a1`, deployed `a0b4bec`).
- [x] Confirm `clients/inference.rs` NOT needed (retrieval-only, §1.3.5); file not built.
- [x] Move hardcoded `QDRANT_URL` to env-read in `clients/qdrant.rs` (falls back to in-cluster DNS).

**My lane — done 2026-08-03**
- [x] Remove dead `INFERENCE_URL` env from `deployment.yaml` (§2.3.12).
- [x] Mount `api-gateway-secrets` via `envFrom: secretRef` (§3.4.1 / §2.3.9).
- [x] Instrument gateway-side business metrics: `qti_qdrant_match_total`, `qti_request_duration_seconds`, `qti_ticket_classification_total` — verified live on `/metrics`.

**My lane — done 2026-08-04 / 2026-08-05**
- [x] Remove hardcoded `QDRANT_URL` from `deployment.yaml` (Secret is source of truth).
- [x] Wire `http_requests_total` via Axum middleware (resolves §0 item 9).
- [x] Update `api_contract.md` to v2.0 (retrieval-only).
- [x] Upgrade `data-pipeline` to full v1.1 ingestion tool (robust parser, auto-reset, batched upserts, explicit metadata/tags extraction).
- [x] **Stage 2 KB Expansion:** Ingested 24 SOPs (RAG Manual v1.1). `qti_knowledge_base` = **110 points**.
- [x] **Stage 2 KB re-ingestion (2026-08-06):** finalized fence-tracking parser + smart reset; `qti_knowledge_base` = **103 points** (supersedes the lost 110-point run, §0 items 21/23).

**Minor / optional cleanups (all complete 2026-08-05)**
- [x] `http_requests_total` is registered but never incremented (§0 item 9) — wired via the `count_requests` middleware.
- [x] Optional: fully migrate `QDRANT_URL` into the Secret — done; hardcoded `env:` line removed, Secret is the single source of truth.
- [x] Optional: update `api_contract.md` wording to match retrieval-only (§1.3.5) — done (v2.0).

**Long-term (explicitly not this sprint)**
- [ ] `data-pipeline` CI/CD — Dockerfile + workflow + CronJob (§3.4.3).
- [ ] Embedding consistency guard — assert at startup that the embedder dimension equals the collection's vector size, so §2.3.1 fails loudly instead of per-upsert.

**NOT mine (re-scoped 2026-08-03)**
- `qti_confidence_tier_total`, `qti_routing_decision_total`, `qti_fact_coverage_score` → DS-agent-side observability (Johan §1.4). The retrieval-only gateway never produces these values.

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

**Date:** 2026-07-17 (Updated 2026-07-30 — ServiceMonitors, Argo CD TLS, Tailscale access documented; Updated 2026-07-31 — `/v1/query` wired, NodePort 30082, model baked into image; Updated 2026-08-06 — local registry rebuilt as ghcr pull-through proxy, ghcr mirror on workers, data-pipeline CI)
**Cluster:** k0s v1.36.2+k0s (Debian 13 trixie)
**Repo:** [Merpatidove/QTI-MAGANG](https://github.com/Merpatidove/QTI-MAGANG)

> This section covers the CI/CD pipeline, Argo CD, container registries, SSH deploy keys, and general cluster/security notes. The Prometheus/Grafana/Loki/AlertManager observability stack that was originally reported alongside this content now lives in **§4 (Jep's section)** — see the editor's note at the top of this document for why it was split.

#### 3.1 What's Running on the Cluster (CI/CD & Platform components)

| Component | Status | Access |
|---|---|---|
| **Argo CD** | 7/7 pods Running | `https://argocd.hite.local` (admin / `12qwaszx`) |
| **Qdrant** | 1/1 Running | `qdrant.qdrant.svc.cluster.local:6333`, NFS-backed PVC (10Gi) |
| **api-gateway** | 1/1 Running, Healthy | NodePort **30082** — `http://100.106.122.68:30082/v1/health` returns `{"status":"ok"}`; `POST /v1/query` returns real SOP data. (Was ClusterIP `api-gateway.qti.svc:8080`.) |
| **NFS CSI driver** | 5/5 controller, 2/2 node pods | k0s path: `/var/lib/k0s/kubelet` |
| **Ingress-NGINX** | 1/1 Running (LoadBalancer) | NodePort 31084 (HTTP), 30616 (HTTPS) — routes to Grafana, Prometheus, AlertManager, ArgoCD. *(Grafana/Prometheus/AlertManager themselves are Jep's — see §4.)* |
| **Local Registry** | Running on controller (HTTPS, self-signed cert) | `10.20.20.201:5000` — **pull-through proxy for ghcr.io** (recreated 2026-08-06, cert now at `/etc/docker/registry/`, was crash-looping on a wiped `/tmp` cert — §0 item 22) |
| **hite-prod Registry** | 1/1 Running (Deployment) | `private-registry-svc.hite-prod:5000` (NodePort 32000) — Kubernetes-native registry:2 |

> **Note:** the original combined report also listed Prometheus/Grafana, Loki, Promtail, and AlertManager in this same table — those rows are preserved in §4.1 (Jep's section) instead of being duplicated here.

> **Note (2026-08-06):** workers pull `ghcr.io/merpatidove/...` images **through the local registry** — `/etc/k0s/containerd/certs.d/ghcr.io/hosts.toml` mirrors ghcr.io → `https://10.20.20.201:5000` (both workers). Argo CD manifests still reference ghcr.io; the proxy fetch-through-caches and serves the blobs to the air-gapped workers (§0 item 22).

##### 3.1.1 CI/CD Pipeline (Working End-to-End)

```
Push to main (api-gateway/**)
  -> GitHub Actions builds Docker image (Rust multi-stage + fastembed model baked via --download-only)
  -> cargo check (added 2026-07-31)
  -> Pushes to ghcr.io/merpatidove/qti-api-gateway:<git-sha>
  -> Docker smoke test: /v1/health must return 200; POST /v1/query must return remediation_payload (extended 2026-07-31)
  -> git pull --rebase then commit updated image tag to kustomization.yaml [skip ci] (fix for run #7, see below)
  -> Argo CD auto-syncs to cluster
  -> Workers pull the new image THROUGH the local registry (ghcr.io mirror -> 10.20.20.201:5000 proxy -> ghcr.io; air-gapped workers can't reach ghcr.io directly, added 2026-08-06)
  -> Pod restarts with new image, health check passes
```

- **Concurrency gate:** enabled — only the latest push builds (old in-progress runs are cancelled).
- **Rollback:** manual via `rollback.yml` — specify a previous image SHA/tag to revert instantly.
- **Known failure (2026-07-31):** CI run #7 built the image and passed smoke for commit `10898a1` but **never deployed** — the commit-back push failed because `main` had moved mid-run, so Argo CD stayed on old tag `1f34091`. Fixed with `git pull --rebase` before the tag commit-back. Runs #8/#9 green.
- **Manual trigger:** `workflow_dispatch` added 2026-07-31 (was push-only).
- **Image distribution (2026-08-06):** workers' containerd mirrors `ghcr.io` → `10.20.20.201:5000` (which runs registry:2 in `REGISTRY_PROXY_REMOTEURL=https://ghcr.io` mode). A new build therefore reaches workers automatically. Verified live with a forced-pull test on worker-1 (§0 item 22). Fallback for non-mirrored images (e.g. qdrant): manual `docker save`/`load` sync per §0 item 21.
- **data-pipeline CI (2026-08-06):** `data-pipeline/**` push now also builds `ghcr.io/merpatidove/qti-data-pipeline:<sha>` (`cargo check` → build/push only; ingestion still runs manually via `cd data-pipeline && cargo run`, §2.3.10).

**Last successful run:** Image `ghcr.io/merpatidove/qti-api-gateway:a0b4bec` (bakes `all-MiniLM-L6-v2` under `/opt/fastembed`), deployed 2026-07-31; health check passing at NodePort 30082. *(Deployed tag has since moved forward with each CI run — current tag `485024c`, §2.1; `a0b4bec` is the historical embedding-model milestone.)*

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
| `.github/workflows/data-pipeline.yml` | `data-pipeline/**` push → `cargo check` → build+push `ghcr.io/merpatidove/qti-data-pipeline:<sha>` (added 2026-08-06; no cluster deploy) | Working |
| `data-pipeline/Dockerfile` | Multi-stage Rust build, bakes `all-MiniLM-L6-v2` via `--download-only` into `/opt/fastembed` (added 2026-08-06) | Working |
| `k8s/argocd/application.yaml` | Argo CD Application CRD (`qti-api-gateway`, repo `QTI-MAGANG`, path `api-gateway/k8s`, ns `qti`) | Working |
| `k8s/argocd/ingress.yaml` | Argo CD Ingress (nginx, `argocd.hite.local`, TLS `argocd-tls`) | Working |

> **Note (2026-08-06):** the `rag-service/*` rows below were **struck** — the folder was deleted from the repo long ago (§2.3.5); kept here as the original report recorded it.

##### 3.1.3 rag-service (New Component — DELETED)

A separate Rust/Axum service (`rag-service/`) was scaffolded as the RAG inference engine. **The `rag-service/` folder was deleted from the repo** (replaced by the retrieval-only gateway architecture, §1.3.5) — this subsection is preserved as the original report recorded it:

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

> **Note (2026-08-06, §0 item 20):** this `10.10.10.0/24` tunnel is the **controller-side** LLM path (verified live on the controller: `wg0` peer `10.10.10.2`, handshake active). The **workers** reach Mac Mini Ollama over the separate mac-mini-ops tunnel at `10.7.0.63` — the workers' air-gap firewall allows only `10.7.0.0/24`, so from a worker `10.10.10.2` is dropped (§5.3.1). The two tunnels serve different consumers; do not merge them.

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
| Ollama binding | `0.0.0.0:11434` (`OLLAMA_HOST=0.0.0.0:11434`; reachable on the WireGuard iface at `10.10.10.2:11434`. Updated 2026-08-04 — was `10.10.10.2:11434` only) |
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
- **Ollama binding** — *original guidance* was `OLLAMA_HOST=10.10.10.2:11434` (bind directly to the WireGuard interface; `0.0.0.0` on macOS binds to IPv6 `[::]` which may not accept IPv4 connections from the tunnel). **Rebound 2026-08-04 to `0.0.0.0:11434`** (§0 item 14) and verified reachable over WireGuard at `10.10.10.2:11434` and over Tailscale — see §3.2.2 table row / §1.3.7.
- **macOS auto-start** — LaunchDaemon installed at `/Library/LaunchDaemons/com.wireguard.wg0.plist` with `KeepAlive`. Load with: `sudo launchctl load /Library/LaunchDaemons/com.wireguard.wg0.plist`.
- **`wg-quick` on macOS** searches configs in order: `/etc/wireguard/`, `/usr/local/etc/wireguard/`, `/opt/homebrew/etc/wireguard/`. The config lives under the Homebrew prefix since that directory is user-writable.
- **Pre-existing tunnel** — the Mac Mini also has a separate WireGuard tunnel **"mac-mini-ops"** (utun4, IP `10.7.0.63`, endpoint `117.54.250.111:51820`) managed by the macOS WireGuard app — connects to the ops infrastructure controller at `10.20.20.201`. Recovery script at `~/wg-recover.sh`. Do not confuse or reuse keys between tunnels. *(2026-08-06, §0 item 20: this mac-mini-ops tunnel is now also the workers' LLM path to Ollama — the workers' firewall whitelists `10.7.0.0/24` (§5.3.1), while the controller-side `10.10.10.2` tunnel above stays the DS SSH door.)*

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

A local HTTPS Docker registry runs on the controller. **Since 2026-08-06 it is a pull-through proxy for ghcr.io** (rebuilt after the original crash-looped ~7 days on a wiped `/tmp` cert — see §0 item 22):
- **Address:** `10.20.20.201:5000` (HTTPS, self-signed cert for IP 10.20.20.201)
- **Cert files (PERSISTENT, not `/tmp`):** `/etc/docker/registry/registry.crt`, `/etc/docker/registry/registry.key` (regenerated 2026-08-06; the old `/tmp/certs/` location caused the outage)
- **Docker trust cert:** `/etc/docker/certs.d/10.20.20.201:5000/ca.crt` (updated to the new cert 2026-08-06)
- **Containerd trust:** workers via `/etc/k0s/containerd/certs.d/10.20.20.201:5000/hosts.toml` + `ca.crt` (updated 2026-08-06)
- **Container run:** `--restart=always -p 5000:5000`, env `REGISTRY_HTTP_TLS_CERTIFICATE/KEY=/certs/registry.crt|.key`, `REGISTRY_PROXY_REMOTEURL=https://ghcr.io`; data volume `0311d171…` → `/var/lib/registry`
- **ghcr.io mirror:** `/etc/k0s/containerd/certs.d/ghcr.io/hosts.toml` on both workers → `server = "https://10.20.20.201:5000"` (2026-08-06). Verified: worker-1 pulled `ghcr.io/merpatidove/qti-api-gateway:e40ba85` through the mirror (33 MB, 4.5 s).
- **Push is DISABLED in proxy mode** — registry:2 proxy ignores push (405). If you need a cluster push target, use `hite-prod` (`10.20.20.202:32000`), not this registry.

**Currently stored images** (legacy, from before proxy mode):

| Image |
|---|
| `busybox` |
| `curlimages/curl` |
| `grafana/loki` |
| `grafana/promtail` |
| `jaegertracing/jaeger` |
| `prometheus/alertmanager` |

**Note:** Worker nodes have DNS UDP (port 53) blocked — they cannot resolve external registries (Docker Hub, etc.). Non-ghcr images must come from the local registry/mirror or be pre-loaded (e.g. `docker save`/`load` for qdrant, §0 item 21).

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

- [x] **api-gateway skeleton** — `/v1/health`, `/v1/query`, `/metrics` endpoints implemented. Query returned placeholder initially; **returns real retrieved SOP text since 2026-07-31** (commit `10898a1`, deployed `a0b4bec`; current deployed tag `485024c`).
- [x] **rag-service scaffold** — `POST /api/v1/ticket` accepts JSON, returns dummy response. No Qdrant/Mistral integration. ***(Deleted from repo — superseded by the retrieval-only gateway, §1.3.5 / §3.1.3.)***
- [x] **Write actual Rust source code** — *(per Data Engineering §2.3.5 above)*: `models.rs`, `routes/query.rs`, `clients/qdrant.rs` are **written, wired, and deployed** (2026-07-31). Only `clients/inference.rs` remains, and may be unnecessary — see §2.3.6.
  - `routes/query.rs` — POST /v1/query handler, Qdrant query, inference forward
  - `clients/qdrant.rs` — Qdrant HTTP client
  - `clients/inference.rs` — Mac Mini inference client
  - `models.rs` — matching `api_contract.md`
- [x] **Create `qti_knowledge_base` collection** in Qdrant — **done** (size 384, Cosine). **Point count 2026-08-06: 103** — after the §0 item 21 empty-KB finding, Farrel re-ingested the RAG_Manual v1.1 set with the finalized fence-tracking parser (§0 item 23). The curl below is historical (note the size is already corrected to 384):
  ```bash
  kubectl port-forward -n qdrant svc/qdrant 6333:6333
  curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
    -H 'Content-Type: application/json' \
    -d '{"vectors": {"size": 384, "distance": "Cosine"}}'
  ```
- [x] **Set up the Mac Mini inference server** — Ollama live via WireGuard tunnel (§3.2.2). The gateway is retrieval-only (§2.3.6) so it does not consume `INFERENCE_URL`; the Python agent calls Ollama directly.
- [x] **Add secrets** — **done 2026-08-05 (§0 item 17).** `api-gateway-secrets` Secret (ns `qti`) is now the single source of truth via `envFrom: secretRef`; hardcoded `QDRANT_URL` removed from `deployment.yaml`. (`INFERENCE_URL` dropped 2026-08-02 — retrieval-only decision §1.3.5.)
- [x] **Add `.gitignore`** — **done (2026-08-02, §0 item 10).** `target/`/`__pycache__`/`venv/` excluded; `rag-service/` was deleted entirely (§3.1.3), so the original `rag-service/target/` concern is moot.

##### 3.4.2 Infrastructure Improvements (CI/CD & Argo CD ownership)

- [x] **Smoke test scope** — **done (2026-07-31).** CI now checks `/v1/health` must return 200 **and** `POST /v1/query` must return a `remediation_payload`. *(Qdrant-connectivity + response-time assertions remain a possible future extension.)*
- [x] **Change Argo CD admin password** — changed from default. Initial admin secret deleted.
- [x] **TLS for Argo CD** — done. Self-signed CA + server cert for `argocd.hite.local`. TLS terminates at nginx-ingress, HTTPS backend to Argo CD. See §3.3.6.
- [x] **Ingress** — nginx-ingress deployed, Ingress resources for Grafana, Prometheus, AlertManager on `.hite.local` hosts. See §3.3.6.
- [x] **CI concurrency gate** — done. Only the latest push builds; old in-progress runs are cancelled.

> The remaining infra-improvement checklist items from the original combined report (ServiceMonitors, AlertManager, centralized logging, Loki in Argo CD, Grafana Loki data source) are **Jep's** and now live in §4.2.

##### 3.4.3 Long-Term (CI/CD & Platform)

- [ ] **Multi-environment** — replicate for production (separate namespace or cluster, Git branch, ApplicationSet).
- [ ] **Qdrant backup strategy** — NFS snapshots or Qdrant's built-in snapshot API.
- [ ] **Network policies** — restrict pod-to-pod traffic (only api-gateway → Qdrant, api-gateway → inference).
- [x] **data-pipeline CI/CD** — **CI done 2026-08-06 (§0 item 22):** `data-pipeline/Dockerfile` + `.github/workflows/data-pipeline.yml` (push to `data-pipeline/**` → `cargo check` → build+push `ghcr.io/merpatidove/qti-data-pipeline:<sha>`). *(The pipeline is Rust, not Python — §2.3.5.)* Still open: **in-cluster deployment** — a CronJob/Job that mounts `RAG_Manual.md` and runs ingestion against an in-cluster-reachable `QDRANT_URL` (currently hardcoded `localhost:6333`, so it still runs from a dev laptop, §2.3.10).
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

# Test api-gateway — use the LOCAL registry copy of curl (Docker Hub is blocked
# on workers; `--image=curlimages/curl` would ImagePullBackOff — 2026-08-06 gotcha)
kubectl -n qti run test --rm -i --restart=Never --image=10.20.20.201:5000/curlimages/curl \
  -- curl -s http://api-gateway.qti.svc:8080/v1/health

# Qdrant health
kubectl -n qdrant exec qdrant-0 -- curl -s http://localhost:6333/healthz

# Create Qdrant collection
curl -X PUT http://localhost:6333/collections/qti_knowledge_base \
  -H 'Content-Type: application/json' \
  -d '{"vectors": {"size": 384, "distance": "Cosine"}}'

# Push image to local registry — NOTE: DISABLED since 2026-08-06 (proxy mode ignores push, §3.3.5).
# Use the hite-prod registry (10.20.20.202:32000) as the cluster push target instead:
docker tag <image>:<tag> 10.20.20.201:5000/<image>:<tag>   # do NOT docker push here
docker tag <image>:<tag> 10.20.20.202:32000/<image>:<tag>
docker push 10.20.20.202:32000/<image>:<tag>

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
| **Prometheus/Grafana** | All targets up, **28 dashboards** | `http://<node-ip>:30000` (admin / `8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx`) |
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

**Custom PrometheusRule `hite-infra-alerts`** (created 2026-07-22; **14 rules** — source of truth: `k8s/prometheus/hite-infra-alerts.yaml`):

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
| `HITE_MacMiniDown` | critical | macmini (AI Inference Server, `job="mac-mini-external"`) unreachable 2m |

**Business rule `qti-business-alerts`** (source of truth: `k8s/prometheus/qti-business-alerts.yaml`):

| Alert | Severity | Trigger |
|---|---|---|
| `QTI_OllamaTimeoutSpike` | critical | rate(qti_agent_ollama_timeouts_total[5m]) > 0 |
| `QTI_AgentParseErrorHigh` | Warning | rate(qti_agent_parse_errors_total[5m]) > 0.05 |

**AI-pipeline rule `hite-ai-pipeline-alerts`** (added 2026-08-04; source of truth: `k8s/prometheus/hite-ai-pipeline-alerts.yaml`):

| Alert | Severity | Trigger |
|---|---|---|
| `HITE_AIOllamaTimeoutHigh` | critical | rate(qti_agent_ollama_timeouts_total[5m]) > 0.02 for 2m |
| `HITE_AILLMLatencyHigh` | warning | LLM p95 latency > 30s for 3m |
| `HITE_AIParseJSONErrorsHigh` | critical | rate(qti_agent_parse_errors_total[5m]) > 0.05 for 2m |
| `HITE_AIRetrievalEmptyHigh` | warning | rate(qti_agent_empty_retrieval_total[5m]) > 0.1 for 5m |
| `HITE_AIAgentDown` | critical | `up{job="qti-agent"} == 0` for 1m |

> **Gotcha (2026-08-06):** `HITE_AIAgentDown` fires on `up{job="qti-agent"} == 0`, but no live scrape job carries that label — the agent is scraped as `qti-agent-johan` (additional-scrape config, §4.1.4). As written the rule can never fire; fix the job matcher (see §4.2 follow-up). The `QTI_*` business alerts overlap the `HITE_AI*` pipeline alerts (both fire on `qti_agent_*` metrics) — intentional, different severities/thresholds.

Alert annotations are in Indonesian (e.g., "CPU node tinggi di atas 90% selama 5 menit").

> **Security note:** The Telegram bot token is stored in plaintext in the AlertManager Secret and Helm values. Consider rotating if this token is used elsewhere.

#### 4.1.4 Agent-Side AI Pipeline Metrics (Added 2026-08-03)

The Data Science Python agent (`agent.py`) now exposes business and AI health metrics to unblock Phase 6, 7, and 8.

- **Scrape Target:** `http://100.126.65.74:8000/metrics` (Johan's Tailscale IP, agent bound to `0.0.0.0`).
- **Scrape mechanism:** the `qti-agent-johan` job in the `prometheus-additional-configs` secret (k8s/prometheus/prometheus-additional.yaml), `metrics_path: /metrics`, scraped every 15s. *(A `johan-qti-agent` ScrapeConfig CRD that duplicated this target was deleted 2026-08-06 — follow-up: remove the leftover repo file `k8s/prometheus/johan-agent-scrape.yaml`.)*
- **Mac Mini node-exporter:** `mac-mini-external` → `http://100.79.30.90:9100/metrics` (`k8s/prometheus/prometheus-additional.yaml`). **DOWN as of 2026-08-06** (host unreachable from Prometheus) — was verified live 2026-08-05 (§0 item 16). Follow-up: confirm Mac Mini / node-exporter is up, or this target will keep failing `HITE_MacMiniDown`.
- **Exported Metrics (`qti_*`):**
  - `qti_llm_request_duration_seconds` (Histogram): LLM latency by `phase={analysis,synthesis}`.
  - `qti_llm_tokens_total` (Counter): Token usage by `type={prompt,completion}`.
  - `qti_agent_parse_errors_total` (Counter): JSON decode failures.
  - `qti_agent_ollama_timeouts_total` (Counter): Ollama request timeouts.
  - `qti_agent_empty_retrieval_total` (Counter): `search_sop` calls with no actionable SOP returned.

### 4.2 What Needs to Be Done — Observability

- [x] **ServiceMonitor for api-gateway** — done. Prometheus scrapes `/metrics` every 15s via `servicemonitor.yaml`.
- [x] **ServiceMonitor for Qdrant** — done. `qdrant-monitor` deployed in `qdrant` namespace, scrapes `/metrics` every 30s.
- [x] **AlertManager** — deployed with Telegram notifications (2 receivers). Custom alerts in `hite-infra-alerts` (14), `qti-business-alerts` (2), and `hite-ai-pipeline-alerts` (5) PrometheusRules. See §4.1.3.
- [x] **Centralized logging (Loki + Promtail)** — done. Logs ship from both nodes to Loki. Loki data source already provisioned in Grafana via ConfigMap.
- [x] **Loki in Argo CD** — Application created (`k8s/loki/application.yaml`). Synced/Healthy. Old Helm release uninstalled.
- [x] **Grafana Loki data source** — provisioned via `loki-loki-stack` ConfigMap. Alertmanager data source also configured.
- [x] **Expose LLM token throughput, Qdrant latency, and JSON decode error metrics** to Prometheus/Grafana once the pipeline goes live (see Data Scientist §1.4).
- [x] (DS-side, proposed) Error→eval feedback loop: a curator pulls error-shaped tickets from Loki into `golden_datasets.json` so the 5W1H baseline is graded on production shapes. The error LOG is already in Loki (§4.1.1); the Telegram message is only the alert derived from it — never parse Telegram back into a store. Raw errors must NOT be auto-embedded into Qdrant (human-gated SOP authoring via §2.4 only). Status: PROPOSED, not built.
- [x] (DS-side) Agent error counters (JSON-decode / Ollama-timeout / empty-retrieval) are DS-owned and ride with the agent observability hooks (§1.4); the gateway-side `qti_*` business counters remain Farrel's (§4.4). Note for Phase 8: the Mac Mini *network* blocker is gone (WireGuard §3.2.2 + DS SSH tunnel §1.3.7); the only thing left blocking AI-pipeline monitoring is the metrics themselves (Farrel's `qti_*` + DS hooks). *(2026-08-03: agent error counters now live — `qti_agent_parse_errors_total` / `qti_agent_ollama_timeouts_total` / `qti_agent_empty_retrieval_total` on `100.126.65.74:8000/metrics`, §4.1.4; the metrics blocker is gone.)*

**Follow-ups opened 2026-08-06 (from the state reconciliation):**
- [ ] Fix `HITE_AIAgentDown` job matcher — `up{job="qti-agent"}` matches no live job (agent scraped as `qti-agent-johan`); change the expr so the alert can fire (§4.1.3).
- [ ] Confirm Mac Mini node-exporter — `mac-mini-external` target is DOWN; bring the host/endpoint back up (§4.1.4).
- [ ] Consolidate agent scrape mechanisms — delete the leftover repo file `k8s/prometheus/johan-agent-scrape.yaml` and `k8s/prometheus/qti-agent-monitor.yaml` (both replaced by `prometheus-additional.yaml`; cluster-side resources already removed).
- [ ] Consider overlapping `QTI_*` (qti-business-alerts) vs `HITE_AI*` (hite-ai-pipeline-alerts) rules — both alert on the same `qti_agent_*` metrics; confirm the threshold/severity split is intended (§4.1.3).

### 4.3 Quick Reference (Observability)

```bash
# ==============================================================================
# 1. GRAFANA DASHBOARD ACCESS
# ==============================================================================
# Web UI: http://<worker-node-ip>:30000
# Credentials: admin / 8fOwy3G9NWqtWqBfqvXZS5PijKGeADBVmuNQv2fx


# ==============================================================================
# 2. PODS & CLUSTER HEALTH STATUS
# ==============================================================================
# View all observability pods (Prometheus, Grafana, Loki, Promtail, Event-Exporter)
kubectl get pods -n monitoring -o wide

# Check K8s Event Exporter status
kubectl get pods -n monitoring -l app.kubernetes.io/name=event-exporter

# Check NFS Persistent Volume Claims for Observability Stack
kubectl get pvc -n monitoring


# ==============================================================================
# 3. LOKI & CENTRALIZED LOGGING
# ==============================================================================
# Test Loki API via port-forward
kubectl port-forward -n monitoring svc/loki 3100:3100 &
curl -G http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="monitoring"}' \
  --data-urlencode 'limit=10'

# Check Promtail log collectors status across all nodes
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail


# ==============================================================================
# 4. SERVICEMONITORS & PROMETHEUS SCRAPE TARGETS
# ==============================================================================
# List all active ServiceMonitors (api-gateway, qdrant, etc.)
kubectl get servicemonitor -A

# Verify custom scrapers config (Mac Mini Node Exporter & Johan Agent Metrics)
kubectl get secret -n monitoring prometheus-additional-configs \
  -o jsonpath='{.data.prometheus-additional\.yaml}' | base64 -d

# Check live scrape targets / health (mac-mini-external, qti-agent-johan, api-gateway, qdrant)
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090 &
curl -s 'http://localhost:9090/api/v1/targets?state=active' | jq '.data.activeTargets[] | {job: .labels.job, health}'


# ==============================================================================
# 5. ALERTMANAGER & TELEGRAM NOTIFICATIONS
# ==============================================================================
# Check AlertManager pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager

# View AlertManager configuration (Verify Telegram Bot Token & Chat ID)
kubectl get secret -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Verify loaded Prometheus Rules (Infra & Business & AI-pipeline Alerts)
kubectl get prometheusrule -n monitoring
kubectl get prometheusrule hite-infra-alerts qti-business-alerts hite-ai-pipeline-alerts -n monitoring -o yaml


# ==============================================================================
# 6. EXTERNAL METRICS ENDPOINTS CHECK (BUSINESS & MAC MINI)
# ==============================================================================
# Test Mac Mini Node Exporter scraping endpoint (Tailscale/WireGuard IP)
curl -I http://100.79.30.90:9100/metrics

# Test Johan Agent Business Metrics endpoint
curl -I http://100.126.65.74:8000/metrics

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
| **Phase 2** | Instalasi Stack Inti | ✅ | Prometheus, Grafana, Loki + Promtail, Alertmanager running. *Jaeger telah dihapus (removed).* |
| **Phase 3** | Konfigurasi Production | ✅ | Datasource & Ingress siap, Contact Point Telegram aktif. *Retention Loki sudah diatur menjadi 720 jam = 30 hari*. |
| **Phase 4** | Dashboard | ✅ | Dashboard Node Exporter, K8s, & AI Pipeline Section 2 terisi data metrik qti_* secara real-time. |
| **Phase 5** | Alert Rules | ✅ | 14 alert infra + 2 business alert (QTI_OllamaTimeoutSpike & QTI_AgentParseErrorHigh) + 5 AI-pipeline alert (`hite-ai-pipeline-alerts`, §4.1.3) aktif & berstatus Normal di Grafana. |
| **Phase 6** | Monitoring AI Pipeline | ✅ | *Selesai (2026-08-03)*. Checklist Application Layer:Mac Mini node-exporter: ✅ Selesai (100.79.30.90:9100) Ollama Remote Reachability: ✅ Selesai (Bound 0.0.0.0:11434, verified curl dari DEBIAN13) Metric Agent Johan: ✅ Selesai (Didaftarkan via additionalScrapeConfigs) Pod qti-agent: ~~1/1 Running, image-pull fixed, port 8080 up~~ — **tidak ada deployment `qti-agent` di cluster; manifes dipindah ke `k8s/archive/qti-agent/` (2026-08-06). Agent berjalan di laptop Johan (`100.126.65.74:8000`), bukan sebagai pod.** |
| **Phase 7** | Testing | ✅ | Testing infra E2E selesai. *Testing log E2E selesai. Log api-gateway (namespace qti) berhasil masuk ke Loki.*. |
| **Phase 8** | Production Checklist | ✅ | Audit infrastruktur, PVC storage, Pod health, GitOps sync, dan penyesuaian Contact Point Telegram Raw JSON. Stack observability Production-Ready.. |
| **Phase 9** | Troubleshooting | ⏳ | Berjalan reaktif, akan didokumentasikan di akhir. |

---

## BAGIAN 3 — Checklist Requirement Awal (Detail Observability Target)

### 1. Infrastructure & Node

- [x] CPU, RAM, Disk Monitoring
- [x] Network Monitoring (Dashboard / Alert)
- [x] Node Health, Availability, Pressure, Restart Status (Lengkap)

### 2. Container & Kubernetes Layer

- [x] Container CPU, Memory, Restart Count, Health (Lengkap)
- [x] Pod Status
- [x] Deployment & Replica Status
- [x] Namespace-specific View Dashboard (seperti Kubernetes / Compute Resources / Namespace (Pods))
- [x] Kubernetes Events (Perlu tool tambahan, helm repo sempat error 404)

### 3. Application Layer
 
- [x] Qdrant: Lengkap (ServiceMonitor + Alerting)
- [x] Rust API (api-gateway): Metric infra ✅, log ✅, alert Down ✅
- [x] Mac Mini / Inference / LLM: 
      - Mac Mini Node Exporter: Scraped via prometheus-additional.yaml (100.79.30.90:9100) ✅
      - Ollama Remote Reachability: Bound 0.0.0.0:11434, verified via Tailscale ✅
      - Agent Johan Metrics: Scraped via prometheus-additional.yaml (100.126.65.74:8000/metrics) ✅
      - Pod qti-agent Deployment: ~~1/1 Running in ns qti, port 8080 up~~ — **tidak ada pod/deployment seperti itu di cluster; manifes diarsipkan ke `k8s/archive/qti-agent/` (2026-08-06). Agent = proses laptop Johan (`100.126.65.74:8000`).**

### 4. Business Metrics

- [x] **Metrics Target**: Tier A/B/C, Confidence, Schema Validation, Escape Hatch, Inference/Retrieval Time, Request Rate, Latency, Error Rate
- [x] Metrik qti_llm_tokens_total, qti_agent_parse_errors_total, & qti_agent_ollama_timeouts_total mengalir ke Prometheus & terakses di Grafana Explore.
- [x] Business Alerting: PrometheusRule `qti-business-alerts` aktif terintegrasi dengan Telegram alert receiver.
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

- **Action Item 1 (metric bisnis):** ✅ Gateway-side DONE 2026-08-03 — `qti_qdrant_match_total`, `qti_request_duration_seconds`, `qti_ticket_classification_total` live di `/metrics`. `qti_confidence_tier_total` / `qti_routing_decision_total` / `qti_fact_coverage_score` di-re-scope ke observability agent DS (Johan, §1.4) karena gateway retrieval-only tidak pernah menghasilkan Tier/routing/fact-coverage.
- **Action Item 2 (log ke stdout):** ✅ Done — log api-gateway masuk Loki (§7 resolved).
- **Action Item 3 (api_contract.md):** Resolved-in-principle oleh keputusan retrieval-only (§1.3.5); update wording contract = cleanup dokumentasi minor.

### 👤 Johanes

- **Status:** Non-teknis. Cukup pastikan tetap berada di grup Telegram **"HITE QTI Logs"** untuk penerimaan alert log.

---

## §5. Platform Engineering (Owner: Hilmi)

### K0rdent (k0s) Infrastructure & Network — Infrastructure & State Report

**Date:** 2026-08-05
**Owner:** Hilmi
**Repo/Path:** On-Premises VM Cluster (worker-1: 10.20.20.202, worker-2: 10.20.20.200)

#### 5.1 What's Running / Current State

| Component / File | Status | Access / Details |
| :--- | :--- | :--- |
| **K0s Worker Nodes** | Active | `worker-1` (Local: 10.20.20.202 / Tailscale: 100.68.225.41) and `worker-2` (Local: 10.20.20.200 / Tailscale: 100.106.122.68). |
| **Firewall (Air-Gapped + VPN)** | Active | Managed by `iptables-persistent`. Blocks 100% of outbound public internet traffic. Explicitly allows routes to K0s subnets (`10.244.0.0/16`, `10.96.0.0/12`), local VM LAN (`10.20.20.0/24`), physical LAN (`192.168.0.0/16`), Tailscale (`100.64.0.0/10`), and WireGuard (`10.7.0.0/24`). |
| **WireGuard Tunnel (LLM Link)** | Active | Mac Mini interface (`mac-mini-ops`) is bound to `10.7.0.63/24`. Exposes Ollama API directly to workers without Reverse SSH Tunnel. |
| **Private Registry (`hite-prod`)** | Active | Runs Kubernetes-native `registry:2`. Accessible externally at `10.20.20.202:32000` and internally at `private-registry-svc.hite-prod:5000`. |
| `update-firewall-v2.sh` | Deployed | Shell script on `worker-1` and `worker-2` defining the `iptables` DROP/ACCEPT rules. |

#### 5.2 CI/CD & Automation (If applicable)

- **Firewall Persistence:** Rules established via `update-firewall-v2.sh` are permanently saved using `netfilter-persistent save`. The `iptables-persistent` package automates the reloading of these strict routing tables upon any VM reboot, ensuring the air-gap and VPN bypasses remain intact.
- **Network Tunneling:** The legacy Reverse SSH Tunnel (`autossh` / `com.hite.tunnel.plist` via `launchctl`) has been formally deprecated and removed. Connection resilience is now fully handled by Tailscale's mesh routing and WireGuard's internal handshake/keepalive.

#### 5.3 Notable Observations & "Gotchas"

##### 5.3.1 Networking & Firewall Quirks
- **WireGuard IP Mismatch (CRITICAL):** The Master Guidebook previously listed the Mac Mini Wireguard IP as `10.10.10.2`. However, the actual active `mac-mini-ops` interface is bound to `10.7.0.63`. Any `curl` to `10.10.10.2` will fail instantly with a connection refusal, whereas requests to `10.7.0.63` without proper firewall rules will freeze/timeout because the packets are silently dropped.
- **Firewall Rule Injection:** To fix the timeout issue to Ollama, the rule `sudo iptables -I OUTPUT 7 -d 10.7.0.0/24 -j ACCEPT` had to be injected manually on the workers to ensure the WireGuard subnet bypassed the final `DROP` rule.
- **`worker-2` Package Missing:** `worker-2` initially failed to save firewall rules because `iptables-persistent` was not installed. It required temporarily opening the firewall (`sudo iptables -P OUTPUT ACCEPT; sudo iptables -F OUTPUT`) to run `sudo apt update && sudo apt install iptables-persistent -y` before applying the air-gapped script.

##### 5.3.2 Cluster & RBAC Architecture
- **Promtail RBAC Limitation (Answering Jep):** Promtail requires explicit Role and RoleBinding configurations to access logs within specific namespaces. The initial restriction to `argocd`, `hite-prod`, `kube-system`, and `monitoring` was a baseline security posture to prevent cluster-wide log scraping by default. The `qti` namespace has now been whitelisted, allowing `api-gateway` logs to reach Loki.
- **The Role of `hite-prod` Namespace:** This namespace is dedicated to hosting the Kubernetes-native Docker registry (`registry:2`). It acts as the internal image store for the cluster, completely separate from the host-level registry on the controller.
- **XOA Hypervisor Status:** The physical topology remains unverified. We must confirm via Xen Orchestra (XOA) if the Mac Mini is the actual physical hypervisor hosting `worker-1`, `worker-2`, and the controller.

#### 5.4 What Needs to Be Done (TODOs)

**Infra Improvements & Verification**
- [x] Deprecate Reverse SSH Tunnel and shift Mac Mini LLM traffic to WireGuard / Tailscale.
- [x] Apply Air-Gapped Firewall (SOP-06) on `worker-1` and `worker-2` via `update-firewall-v2.sh`.
- [x] Whitelist Tailscale CGNAT (`100.64.0.0/10`) and WireGuard (`10.7.0.0/24`) subnets in `iptables`.
- [x] Install `iptables-persistent` on `worker-2` to ensure firewall rule survival across reboots.
- [ ] Confirm with Ferdi via XOA (`Home` → `Hosts`) if the Mac Mini is the physical hypervisor host for the cluster VMs.

**Long-Term**
- [ ] Coordinate with office network admin for a dedicated Network Bridge and DHCP Reservation (Static IP) once the Mac Mini is physically returned to the office.

#### 5.5 Quick Reference (Cheat Sheet)

```bash
# Verify Air-Gapped Firewall is actively blocking public internet (Expect: Timeout/100% Loss)
curl --connect-timeout 5 -I https://api.anthropic.com
ping -c 3 8.8.8.8

# Verify Tailscale Mesh routing is allowed (Expect: 0% packet loss)
ping -c 3 100.79.30.90

# Verify WireGuard Tunnel to Mac Mini Ollama is allowed and responsive (Expect: JSON model list)
curl -m 5 -s http://10.7.0.63:11434/api/tags

# View active iptables OUTPUT rules to ensure VPN bypasses are positioned before the DROP rule
sudo iptables -L OUTPUT -v -n --line-numbers

# Manually inject the WireGuard subnet bypass if it is missing (Place it before the final DROP)
sudo iptables -I OUTPUT 7 -d 10.7.0.0/24 -j ACCEPT
sudo netfilter-persistent save
```

---

## §6. Mac Mini Full-Hosting: Consolidated View (NEW)

> **This section is synthesis, not a verbatim report.** You said HITE is moving to fully host on the Mac Mini — here's every Mac-Mini-related fact scattered across the five reports, pulled into one place, plus the open questions they leave unanswered. Nothing here is a new decision; it's the source material you'll need to make one.

### 6.1 What each report says about the Mac Mini today

| Source | What it says |
|---|---|
| Platform Eng §5.1 / §5.3.2 | Mac Mini is currently **at an employee's home** (`192.168.1.18`), not the office (`192.168.20.163`). Connected to `worker-1` only via a **Reverse SSH Tunnel** (ports `19100`→Node Exporter, `11434`→Ollama). Office network bridge is currently impossible due to the network segment mismatch. SSH config uses mDNS `Qtis-Mac-mini.local` as a ProxyJump target to survive IP changes. *(Superseded 2026-08-05 — Reverse SSH Tunnel deprecated and removed; the Mac Mini now connects to the workers via the mac-mini-ops WireGuard tunnel `10.7.0.63` (§5.3.1); office-bridge/DHCP plan still open, §5.4.)* |
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
2. **Two different network paths to the Mac Mini exist simultaneously and neither team seems aware of the other's:** Platform Eng relies on a **Reverse SSH Tunnel** (`launchctl`-managed, ports 19100/11434) while Data Engineering relies on **Tailscale** (`100.79.30.90`). **Update 2026-07-30:** a third path has been added — **WireGuard over Tailscale** (`10.10.10.0/24`, §3.2.2) — which provides a dedicated encrypted tunnel between the VM and Mac Mini for LLM traffic (5x latency improvement over Tailscale DERP relay). This is now the recommended path for inference traffic. The Reverse SSH Tunnel remains in place for Node Exporter metrics and legacy access. *(Update 2026-08-05 — Reverse SSH Tunnel formally removed; workers reach Ollama via mac-mini-ops `10.7.0.63` (§5.1 / §5.3.1); the controller-side `10.10.10.0/24` tunnel remains as the DS SSH door, §3.2.2.)*
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
| Data Engineering `data-pipeline` ingestion (§2.4) | DevOps CI/CD (Ferdi) confirmation | `data-pipeline` runs outside the cluster and can't reach `qdrant.qdrant.svc.cluster.local` — needs a decided external path (Mac Mini, port-forward, or NodePort) — see §2.3.3. | ✅ **Resolved (2026-08-05)** — ingestion via the 3-terminal workaround (§2.3.13, bypasses the `kube-router` ClusterIP bug); `qti_knowledge_base` = **103 points** (2026-08-06 re-ingest) (24 SOPs). |
| DS Stage 2 Real-Data Eval (§1.4 / §8) | Data Engineering (Farrel) | Real tickets grounded in wrong SOPs due to coverage gaps; required new `SOP-INF-*` entries and v1.1 metadata parsing. | ✅ **Resolved 2026-08-05** — 24 SOPs ingested with explicit Categories, Tiers, and Tags. `qti_knowledge_base` = **103 points** (2026-08-06 re-ingest). DS fully unblocked for Stage 2 re-eval. |
| `/v1/query` returning real data (§2.4, §1.4) | Data Engineering (Farrel) | `clients::qdrant::search_sop` is written but not wired into the route handler yet, and the collection has 0 points until ingestion runs. | ✅ **Resolved** — wired (commit `10898a1`), deployed (`a0b4bec`), verified returning SOP text (e.g. SOP-GIT-003, SOP-DOC-001). |
| DevOps Prometheus/Grafana/Loki Phase 8 (AI pipeline monitoring) | Mac Mini networking resolution (Hilmi + Ferdi) & Johan (metrics) | DONE / LIVE — Blocker jaringan teratasi via Tailscale/WireGuard. Metrik Mac Mini Node Exporter (100.79.30.90:9100) dan Agent Johan (100.126.65.74:8000/metrics) resmi unblocked dan aktif dipantau oleh Prometheus & Loki. | ✅ **Resolved** — Infra blocker bypassed via Tailscale binding. Johan delivered agent metrics (`qti_*`) on `100.126.65.74:8000/metrics`. |
| DevOps business-metric dashboards (Phase 6/7) | Farrel (gateway metrics) + Johan (agent metrics & Tier spec) | DONE / LIVE — Instrumentasi metrik bisnis qti_* (gateway & agent, termasuk Tier-based Confidence) sudah tuntas 100%, divisualisasikan di Grafana Dashboard, serta terintegrasi ke Telegram Alerting (qti-business-alerts). | ✅ **Resolved** — Farrel delivered gateway metrics. Johan delivered agent hooks (`qti_*`) and official Tier A/B/C specs. Jep is fully unblocked to build dashboards. |
| Promtail logs for `api-gateway` (namespace `qti`) reaching Loki | RBAC debugging (Jep), explanation from Hilmi | Promtail currently only has RBAC for 4 namespaces (`argocd`, `hite-prod`, `kube-system`, `monitoring`) — `qti` is not among them; per §4.4 this needs Hilmi to explain the current allocation. | ✅ **Resolved** — `qti` namespace logs confirmed in Loki. |
| Any full-hosting decision (§6) | Ferdi + Hilmi | Needs the XOA hypervisor question answered and a single network path (tunnel vs. Tailscale) chosen. | ⏳ **Open** — XOA question still unanswered; WireGuard tunnel adopted as the LLM path. |
| `clients/inference.rs` (api-gateway) | Design confirmation from Data Science (Johan) | Possibly unnecessary if the Python agent calls Ollama/Qwen directly — see §2.3.6. | ✅ **Resolved 2026-08-02** — joint DS+DE decision: do NOT build it; gateway retrieval-only, generation in the DS agent (§1.3.5). Dead `INFERENCE_URL` env **removed 2026-08-03** (§0 item 11). |
| DS grounding rate / retrieval→synthesis join | Data Science (Johan) | Synthesis-gate "Error"-substring veto was the root cause (§1.4 / §1.6). | ✅ Resolved 2026-08-03 — grounding raised from the 43.6–50.9% control band to a 94.5–96.4% treatment band (≈95%) via the `_is_err` synthesis-gate fix; verified genuine. |
| **Ollama Remote Binding** | Jep / DevOps | Mac Mini (`100.79.30.90`) | ✅ **Resolved** | Rebound to `0.0.0.0:11434`; verified via `curl /api/tags`. |
| **Scrape Target Agent Johan** | Jep / DevOps | Johan / Data Science | ✅ **Resolved** | Added `100.126.65.74:8000` to `prometheus-additional.yaml`. |
| **`qti-agent` Deployment** | Ferdi / DevOps | K8s Cluster (ns `qti`) | ⏳ **Corrected 2026-08-06** | **No such deployment exists** — manifests archived to `k8s/archive/qti-agent/`; the agent is Johan's laptop process on `100.126.65.74:8000`, not a cluster pod. |
---

## §8. Data Science Methodology — 5W1H Triage Evaluation (FINAL)

Owned by Johan. Finalized 2026-08-06 (replaces the draft). Two-stage maturity model: Stage 1 = synthetic golden-set validation (can the pipeline produce a complete, grounded 5W1H?); Stage 2 = real-data validation (does it ground in the RIGHT SOP on production-shaped errors?). Stage 1 is necessary but not sufficient: the synthetic set has a matching SOP for every ticket, so it can never exercise the wrong-SOP failure mode that Stage 2 exposes.

**8.1 System under test**

ticket (raw_text + project_tags) → agent.py (FastAPI ReAct orchestrator, :8000 /process-ticket): analysis phase (Ollama Qwen2.5-Coder-7B Q4_K_M over the SSH tunnel → WireGuard, §1.3.7) produces a 5W1H + tool choice; if search_sop is chosen, tools.search_sop calls the gateway /v1/query (NodePort 30082) for RAG context; a synthesis call grounds why/how in the retrieved SOP. Output = flat 6-key 5W1H + confidence tier. The gateway is retrieval-only (§1.3.5); the 5W1H lives in the agent output, not the gateway response.

**8.2 Metrics (two-metric grader + tier)**

Complete 5W1H Schema = all six keys (Who/What/When/Where/Why/How, case-insensitive) present and non-empty — measures shape. Grounded = how ≠ the "Pending SOP search" placeholder — measures whether the RAG half reached the answer (§1.3.6). Tier mapping: A = complete + grounded; B = complete but ungrounded; C = incomplete / escape-hatch. Caveat (§1.3.8): the naive grounding check is blind to retrieval correctness — a nearest-neighbor SOP always returns, so wrong-SOP grounding still counts as grounded; honest grounding needs human review or a retrieval-correctness signal.

**8.3 Stage 1 — Synthetic validation (55 tickets)**

| Run | Schema | Grounded | Note |
| --- | --- | --- | --- |
| 2026-08-02 baseline | 55/55 | 24/55 (43.6%) | first real grounding measurement |
| 2026-08-03 treatment (_is_err gate) | 55/55 | 52–53/55 (94.5–96.4%) | root cause was the "Error"-substring veto |
| 2026-08-05 clean run | 55/55 | 48/55 | 7 synthesis JSON decode failures = last loss bucket (§1.3.10) |

Reproducibility (§1.3.9): temp 0.1 → 45/55 (JSON degeneration); seed 42 @ temp 0.8 → 49→48 (Metal/llama.cpp not bit-deterministic). Report a band/threshold, never a single point.

**8.4 Stage 2 — Real-data validation (8 tickets)**

Curation rules: one ticket per root cause (dedupe repeats); info/benign noise ≠ incident; sources = Loki log mining + real debugging sessions.

| Ticket | Root cause | Expected SOP |
| --- | --- | --- |
| REAL-001 | Ollama unreachable over WireGuard | SOP-INF-003 |
| REAL-002 | Loki datasource unreachable | SOP-INF-004 |
| REAL-003 | nginx bind() port in use | SOP-DOC-002 |
| REAL-004 | VolumeSnapshot CRDs missing (csi-snapshotter) | SOP-INF-005 |
| REAL-005 | Loki querier context-canceled bursts | SOP-INF-004 |
| REAL-006 | worker lost route to control plane | SOP-INF-006 |
| REAL-007 | etcd request timeout (leader election) | SOP-INF-007 |
| REAL-008 | Argo CD Redis unreachable | SOP-INF-008 |

Coverage-gap finding: initial 3-ticket run = 1/3 correctly grounded, 2/3 grounded-in-wrong-SOP. Remediation = SOP expansion SOP-INF-003..008 → RAG_Manual v1.1 (24 SOPs) → Farrel ingest → **103 points** (§2.4, re-ingested 2026-08-06). Real-set grounding 6/8 (75%); post-ingest correct-SOP re-eval pending.

**8.5 Key methodological findings**

(1) schema-complete ≠ RAG-grounded — conflating them hid the real state for weeks; (2) naive grounding is blind to retrieval correctness — nearest-neighbor always returns; (3) the last synthetic loss bucket is synthesis JSON decode failure, fixed by retry-on-parse-failure; (4) once real tickets are ingested they become "seen" — generalization needs fresh unseen tickets.

**8.6 Limitations & future work**

Automated error→eval curator from Loki (§4.2, PROPOSED, not built; raw errors must NOT auto-embed into Qdrant — human-gated SOP authoring only). Retrieval-correctness signal to automate honest grounding. grade_result.py into CI (threshold gate, e.g. grounded ≥ 45/55). Synthesis retry-on-parse-failure to close the last synthetic loss bucket.

---

*End of Master Guidebook. All five source reports are preserved in full above; only escaping artifacts were cleaned, the DevOps report was split by role (CI/CD vs. observability), and organizational headers/cross-references were added.*
