# SAM Algorithm Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the existing local forensic app into SAM — Strategic Attack Mapper — with a functional, evidence-backed algorithm workbench and contextual explanations on existing derived-result pages.

**Architecture:** Keep the current Flask, DuckDB, and vanilla JavaScript application. Record one bounded, backend-produced analysis trace per case; build a bipartite event/entity graph from persisted records and score only the existing correlation candidates. Add BFS, DFS, and Dijkstra over that graph, expose them through case APIs, then extend the existing rail UI with a workbench and inline explanation disclosures.

**Tech Stack:** Python 3, Flask, DuckDB, Pydantic, vanilla JavaScript, existing HTML/CSS, pytest.

**Spec:** `docs/superpowers/specs/2026-11-21-sam-algorithm-workbench-design.md`

## Global Constraints

- Preserve the existing UI structure, color palette, routes, entity graph, current results, Python import path, CLI command, and API behavior.
- Product-facing identity is `SAM — Strategic Attack Mapper`; retain `netforensicai` and `netforensic` for compatibility.
- Correlation means proximity/association; do not represent it as causation or a calibrated probability.
- Candidate/link generation remains bounded by the existing 300-second window, 50,000 link budget, 10% per-entity share, and 2,000,000 scanned-pair cap; expose those bounds.
- Relationship score defaults: `S=0.40I+0.25T+0.20A+0.15R`, threshold `0.50`, with `I`, `T`, `A`, `R` defined in the approved design.
- Dijkstra costs: event-to-entity evidence edge `1`; event-to-event edge `1-S+0.001`.
- Do not add a graph library, UI framework, or fabricated trace/output. Paginate large case data.

## Review Focus

- Empty cases and cases with no timestamped events: show a useful empty state and zero/unknown values without failing. Test in the workbench API/UI task.
- Invalid, missing, or naive timestamps: use the project's UTC normalization and keep un-timestamped events out of temporal pairing. Test in trace/scoring tasks.
- Dense captures hitting correlation limits: mark the trace truncated and report limits/suppressed counts. Test in trace task.
- Evidence uploaded after analysis or analysis errors: mark the saved trace stale or keep the prior successful trace; never label it current. Test in API task.
- Invalid weights, thresholds, node IDs, and disconnected path endpoints: return validation errors or a clear no-path result without changing stored correlations. Test in scoring/API tasks.

---

### Task 1: Produce an honest, bounded analysis trace

**Files:**
- Modify: `netforensicai/core/pipeline.py`
- Modify: `netforensicai/core/correlation.py`
- Create: `netforensicai/core/workbench.py`
- Test: `tests/test_workbench.py`
- Test: `tests/test_pipeline.py`
- Test: `tests/test_correlation.py`

**Interfaces:**
- Produce `build_analysis_trace(store, analysis_results, evidence_ids, run_id) -> dict` and `save_analysis_trace(case_dir, trace) -> None`, `load_analysis_trace(case_dir) -> dict | None`.
- Extend correlation execution with optional trace collection while preserving `correlate_case(...)`'s default return and stored-link behavior.
- Record raw/parser counts only when the parser can provide them; represent unavailable counters explicitly instead of substituting zero. Record normalized event counts/types, entity extraction/provenance, actual candidate links, caps, and actual accepted/rejected decisions.

- [ ] Add tests for observed normalized event/entity counts, source evidence IDs, preserved correlation results, dense-case cap visibility, missing timestamps, and atomic trace replacement.
- [ ] Run focused tests and confirm failures before implementation.
- [ ] Instrument parse/correlation stages and write the trace atomically to the case directory only after a successful analysis.
- [ ] Run `pytest tests/test_workbench.py tests/test_pipeline.py tests/test_correlation.py -q`.

### Task 2: Implement graph scoring, metrics, and traversals

**Files:**
- Modify: `netforensicai/core/workbench.py`
- Test: `tests/test_workbench.py`

**Interfaces:**
- `score_candidates(candidates, weights, threshold) -> dict` returns each candidate's component values, weighted terms, score, threshold, and decision.
- `build_case_graph(store, weights, threshold) -> dict` returns event/entity nodes, provenance and scored edges, stage counts, degree metrics, connected components, isolated nodes, and reachability inputs.
- `run_bfs(graph, start_id) -> dict`, `run_dfs(graph, start_id) -> dict`, and `run_dijkstra(graph, start_id, target_id) -> dict` return actual step traces and final result.

- [ ] Add tests for score arithmetic/defaults, threshold decisions, evidence-edge provenance, node/edge/count consistency, connected components, BFS/DFS order and state, Dijkstra costs/optimal path, disconnected nodes, and unknown IDs.
- [ ] Run `pytest tests/test_workbench.py -q` and confirm new tests fail.
- [ ] Implement the typed event/entity graph and deterministic algorithms without third-party graph dependencies.
- [ ] Run `pytest tests/test_workbench.py -q`.

### Task 3: Expose trace, graph, scoring, and algorithms through the existing case API

**Files:**
- Modify: `netforensicai/web/app.py`
- Modify: `docs/api.md`
- Test: `tests/test_web_workbench.py`
- Test: `tests/test_web.py`

**Interfaces:**
- `GET /api/cases/<case_id>/workbench?offset=N&limit=N` returns run/stale status, stage summaries, graph counts, and a paginated node/edge slice.
- `POST /api/cases/<case_id>/workbench/score` accepts optional `weights` and `threshold` and returns recomputed workbench decisions without changing `correlation_links`.
- `POST /api/cases/<case_id>/workbench/traversal` accepts `algorithm` (`bfs`, `dfs`, or `dijkstra`), `start_id`, and optional `target_id`.
- Successful `/analyze` adds `run_id` and summary to its existing response. Failed or stale execution cannot be represented as current.

- [ ] Add API tests for absent/stale/current traces, pagination, scoring validation and isolation from stored links, all algorithms, invalid nodes, and no-path results.
- [ ] Run the focused API tests and confirm expected failures.
- [ ] Add routes using existing case loading, `locked_store`, and error conventions; document payloads in `docs/api.md`.
- [ ] Run `pytest tests/test_web_workbench.py tests/test_web.py -q`.

### Task 4: Add the Algorithm Workbench and contextual explanations

**Files:**
- Modify: `netforensicai/web/static/app.js`
- Modify: `netforensicai/web/static/style.css`
- Modify: `netforensicai/web/static/index.html`
- Test: `tests/test_web_workbench.py` (route payload coverage)
- Test: `tests/test_web_home.py` (brand/navigation rendering)

**Interfaces:**
- Add a `workbench` case route and rail entry in the existing app navigation.
- Workbench supports complete analysis and recorded step navigation (next, previous, reset), candidate/entity/event tables, selected node/edge details, math disclosures, graph metrics, BFS/DFS/Dijkstra controls, scoring weights/threshold, and clear empty/stale/truncated states.
- Add a reusable `How this was derived` disclosure for overview metrics, timeline, entities/graph, detections, and ATT&CK mapping; content names the actual method and backend values.

- [ ] Add UI assertions for workbench navigation/title and contextual explanations on the derived-result routes.
- [ ] Implement rendering from API payloads only; do not hardcode trace counts, sample edges, scores, or traversals.
- [ ] Preserve existing pages, graph renderer, and responsive styles; test navigation, step controls, selection, and error/empty states in the browser where available.

### Task 5: Apply SAM branding, demo walkthrough, and full verification

**Files:**
- Modify: `README.md`
- Modify: `docs/architecture.md`
- Modify: `docs/walkthrough.md`
- Modify: `netforensicai/web/static/index.html`
- Modify: `netforensicai/web/static/app.js`
- Modify: `netforensicai/web/static/style.css`
- Modify: `docs/api.md`

- [ ] Replace product-facing NetForensicAI branding with `SAM — Strategic Attack Mapper`; leave technical package/CLI/import identifiers unchanged.
- [ ] Update the architecture and walkthrough for actual current algorithms, score formula, limits, unsupported syllabus concepts, and demo steps.
- [ ] Run focused tests for all changed backend/API/UI files, then the full repository suite with `pytest -q`.
- [ ] Run the synthetic incident demo, open the web UI, and verify the full analysis → workbench → selected evidence → BFS/DFS/Dijkstra path in a browser. Capture a screenshot or provide a reproducible walkthrough.
- [ ] Inspect `git diff --check`, `git status`, and the final changed-file list; report test/demo results and known parser-specific raw-record limitations.

