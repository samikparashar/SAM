# SAM Algorithm Workbench Design

Date: 2026-11-21

## Intent

Extend the existing local-first forensic application with an interactive, backend-backed Algorithm Workbench for college project demonstrations. The workbench must explain the transformations and graph operations that actually ran, with evidence provenance and explicit uncertainty. The product name is **SAM — Strategic Attack Mapper**.

## Architecture report (current implementation)

- Evidence enters through `core/pipeline.py::parse_evidence_item`, shared by CLI, web, and capture paths. A parser streams normalized `Event` objects in batches of 5,000. A parser failure removes that evidence item's partial events.
- Parsers map input formats to `core/event.py::Event`; timestamps are best-effort parsed into UTC. The application does not currently retain raw log records as event fields. `raw_event_reference` is a provenance pointer, not the original record.
- `core/entities.py::extract_entities` maps selected normalized fields to entities. IDs are SHA-256-derived from lowercased, trimmed `(entity_type, value)`. `entity_events` records the source event and field; duplicate entity rows are upserted. Ports are indexed but excluded from relationship matching.
- `core/correlation.py::correlate_case` streams timestamped events in order and pairs events within a default 300-second window. A `related` link needs a shared correlating entity; `possible_relationship` needs only temporal proximity and is explicitly weak. The implementation prioritizes recent pairs, caps links at 50,000, limits each entity to 10% of that budget, and has a 2,000,000 scanned-pair CPU bound. This is deterministic rule-based correlation, not a causal attack-graph algorithm.
- The web `POST /api/cases/<id>/analyze` parses evidence, calls correlation, runs deterministic detections, and currently returns totals only. Other routes expose timelines, entities, and a one-hop entity graph.
- The current graph view is a radial canvas rendered by `web/static/app.js` from the entity endpoint. It does not use a graph library. BFS, DFS, path search, edge scoring, and intermediate execution traces are not present.
- The project uses a dependency-free JavaScript frontend, Flask API, DuckDB persistence, Python package `netforensicai`, and CLI command `netforensic`.

## Goals

- Preserve the existing application shell, palette, navigation, entity graph, APIs, and CLI behavior.
- Add a workbench that uses one real analysis context and reports actual event, entity, candidate, edge, provenance, and graph counts.
- Make candidate acceptance/rejection and any newly added temporal/scoring rules inspectable, with no implication of causation.
- Provide complete and stepwise execution views, BFS and DFS traces, Dijkstra path search with a documented nonnegative edge cost, graph metrics, and worked calculations tied to selected data.
- Map only genuinely implemented concepts in the optional syllabus section.
- Display SAM branding in product-facing UI and documentation while preserving import and command compatibility.

## Non-goals

- Replace the existing forensic product or introduce a separate app, graph database, frontend framework, or external graph library.
- Rename Python import paths, installed distribution, CLI executable, or remote repository URL in this compatibility-preserving change.
- Describe existing correlation as a causal or validated intrusion chain.
- Claim Bayesian inference, A*, backward chaining, or other algorithms unless they are implemented and used.
- Store or render full raw evidence records by default; show the existing evidence reference and normalized event unless a safe parser-level raw sample is available.

## Approaches considered

1. **Observable extension of the current pipeline (recommended).** Preserve the current DuckDB event/entity/correlation model and add structured trace output and workbench operations over that case. Add scoring as an explicit workbench validation layer and graph traversals as real algorithms. This best preserves compatibility and evidentiary wording.
2. **Replace correlations with a new attack-graph engine.** This would make a typed attack graph primary, but would change existing results, add migration and performance risk, and could imply causal semantics the evidence does not support.

The task explicitly prioritizes preserving the UI and current functionality, so approach 1 is the working design.

## Proposed design

### Execution and API

Extend the existing analysis path to produce a structured trace alongside its current result. One successful analysis atomically replaces the case's current trace and assigns it a run ID; the trace records the evidence IDs and analysis settings used. The workbench checks these against the current case and labels a trace stale after evidence changes, until analysis runs again. Step navigation reads this one recorded execution; it does not recompute results. Keep API additions scoped under the existing case routes. Include stage summaries, event/entity provenance, candidate decision details, configured limits, graph construction records, and graph statistics. Bound detailed candidate payloads and paginate them; report scanned, retained, suppressed, and truncated counts explicitly.

Represent the workbench graph as `G=(V,E)` with event nodes and entity nodes. Add an evidence edge from each event to each extracted entity, and event-to-event edges for correlation candidates accepted by the workbench's scoring threshold. Preserve the existing entity graph endpoint and display. Treat confidence labels from correlation as categorical labels, not numeric probabilities. The optional relationship score uses normalized components: `I=1` when the events share a correlating entity and otherwise `0`; `T=max(0, 1-Δt/Δt_max)` for nonnegative in-window pairs; `A` is the fraction of endpoints with a resolvable evidence reference; `R=1` for a shared-entity rule match and `0.25` for temporal-proximity-only. Defaults are `w=(0.40, 0.25, 0.20, 0.15)` and threshold `0.50`, with `S=wI*I+wT*T+wA*A+wR*R`. This is an uncalibrated relationship score, not a probability. Scoring is an optional workbench result; it does not overwrite stored correlations. Users may adjust weights/threshold and rerun scoring. State clearly which candidates the current bounded generator did not inspect.

### Interface

Add an Algorithm Workbench destination/panel using the existing shell and styles. It presents stage summaries, expandable mathematical explanations, event/entity/candidate tables with detail panes, accepted/rejected status, and an intermediate graph view that uses the existing canvas style. Complete mode runs the existing analysis flow and receives its trace. Step mode navigates the resulting recorded trace, with next, previous, reset, and complete-analysis controls. Node and edge selection displays backend provenance and calculations. Add a shared “How this was derived” disclosure on existing pages that render algorithmic results: overview correlation/detection metrics, normalized timeline, entity list/graph, deterministic detections, and ATT&CK rule mappings. Each explanation is specific to that page's actual backend method and data; pages without algorithmic output do not get generic filler. Keep current navigation and features available.

### Algorithms and concepts

- Implement BFS and DFS with explicit backend step records (frontier/stack, visited set, traversal order, discovered edges, and completion).
- Implement Dijkstra for path search. Event-to-entity evidence edges have cost `1`; event-to-event edges have cost `1-S(u,v)+0.001`, where `S` is the optional normalized relationship score. Document that these are investigation traversal costs, not probabilities. Do not label it A*.
- Calculate degree/in-degree/out-degree, connected components, reachability, and isolated-node counts from the same graph instance.
- Expose actual temporal window and shared-entity rules already used by correlation. Any new score must show each actual component, weight, threshold, final score, and decision. No hidden weights or fabricated trace rows.
- Syllabus entries are driven by implemented feature/module mappings; omit unsupported concepts such as Bayesian reasoning and backward chaining.

### Branding

Use “SAM” and “Strategic Attack Mapper” in the app title, visible brand, and project-facing docs. Retain `netforensicai` imports, `netforensic` command, distribution metadata, environment names, and existing API semantics for backward compatibility. Historical references and security-sensitive request headers should not be bulk-renamed without a compatibility reason.

## Success criteria

- Existing tests and the new focused tests pass.
- A case analyzed once yields the same final graph in complete and step modes; returned graph counts match returned nodes/edges.
- Every accepted/rejected displayed candidate has backend-generated evidence, rule, temporal details where applicable, and score details where scoring ran.
- BFS/DFS/Dijkstra output is computed from the same case graph shown in the workbench.
- Bounded/truncated correlation is visible and never presented as exhaustive.
- Existing UI routes, graph view, CLI, API behavior, and Python imports remain usable.
- Product-facing title and visible identity say SAM — Strategic Attack Mapper.
- Existing derived-result pages show a contextual algorithm explanation alongside their outputs, and every explanation names the actual backend rule/transformation used.

## Builder decisions after approval

- Choose the smallest trace persistence shape consistent with the case store and its locking model.
- Keep large case payloads paginated and limit samples rather than materializing all events in the browser.
- Use project-native Python and vanilla JavaScript; do not add a graph dependency.
- Replace the current case trace on each successful analysis and mark it stale when the current evidence IDs differ; do not retain historical trace archives in this change.
- Use the bipartite event/entity evidence graph above; apply relationship scoring only to event-pair candidates produced by the existing bounded temporal candidate generator. The temporal window is therefore a candidate-generation gate: report its configured value and results, and do not fabricate an exhaustive count of pairs the bounded generator never examined.
- Attempt a raw-record sample only where the parser's preserved source reference can resolve it safely; otherwise show the normalized event and its evidence/source locator.
- Verify via focused backend/frontend tests and the repository test suite, as requested in the pasted task.

