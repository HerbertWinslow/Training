# ARCHITECTURE

## Required Architecture Inputs

- `Requirements source: REQUIREMENTS.md`
- `System purpose: A logs analyzer hosted in k8s that ingests log-based signals from any application (Java, Python, or other), regardless of source language, and translates them into human-readable MS Teams notifications describing the issue and possible solutions.`
- `Primary use cases: (1) Detect a qualifying log/notification event from the cluster's log stack. (2) Analyze the underlying issue using app-specific GitHub repos and web search. (3) Send a human-readable MS Teams notification summarizing the issue and possible fixes.`
- `Target users / actors: Members of an existing MS Teams group (notification recipients); no distinct admin/configurator role currently defined (FR-4.2 UNKNOWN).`
- `Runtime environment: web application`
- `Server framework: preferred Django, but open to a better-suited option`
- `Client framework: React — explicitly questioned whether a client framework is needed at all (TO BE DECIDED, see below)`
- `API style and integration model: REST`
- `Authentication and session model: TO BE DECIDED`
- `Data model expectations: derived from REQUIREMENTS.md — log collection stack is VictoriaMetrics, Vector, Grafana (VictoriaLogs datasource); all monitored applications run in the k8s cluster; no further entity detail defined yet`
- `Deployment model: k8s, preferably Helm chart`
- `Scale expectations: number of pods required — TO BE DECIDED`
- `Security expectations: TO BE DECIDED, to be addressed in a follow-up pass`

## Initial Architecture (Provisional)

This is a provisional architecture, intentionally light on commitments where REQUIREMENTS.md leaves things `UNKNOWN`/`TO BE DECIDED`. It is meant to be revisited once those gaps close.

### Whether a UI is needed at all

REQUIREMENTS.md describes no user-facing workflow other than *receiving a notification in MS Teams* (FR-3.1–FR-3.3). There is no requirement implying a dashboard, configuration screen, or any other human-operated interface. Given that:

- **No frontend client (no React) is currently justified by the requirements.** The "Client framework" input asks whether one is needed — based on REQUIREMENTS.md, the answer is provisionally **no**, unless a future requirement introduces a UI-facing workflow (e.g. a config screen for FR-1.5/FR-2.6/FR-3.4 once those are decided).
- This should be revisited if "Required Architecture Inputs" or future requirements introduce an explicit UI need (e.g. for rule configuration, viewing notification history, or role management once FR-4.2 is resolved).

### High-level shape

A single backend service, deployed as one or more pods in the k8s cluster, with three logical stages matching the FR groups in REQUIREMENTS.md:

1. **Ingestion stage** (→ FR-1.x): Pulls or receives log data, preferring the Grafana/VictoriaLogs path, with a direct VictoriaLogs path allowed as a simplification (FR-1.2). Must handle logs from any application/language uniformly (FR-1.3) and identify the originating app per log entry (FR-1.4 — exact mechanism `UNKNOWN`).
2. **Analysis stage** (→ FR-2.x): Given a qualifying log event, produces a human-readable issue summary and candidate solutions (FR-2.4), using two enrichment inputs: application-specific GitHub repos (FR-2.2) and web search (FR-2.3). Must never forward a raw log dump as-is (FR-2.5).
3. **Notification stage** (→ FR-3.x): Delivers the analysis result to the MS Teams group (FR-3.1–FR-3.3), in a format `TO BE DECIDED` (FR-3.4).

### Server framework

- Preferred: Django, per stated input.
- Given the system is primarily a backend job/service (ingest → analyze → notify) exposed via REST (per "API style and integration model"), Django is a workable default but is heavier than strictly necessary for a service with no first-party UI. A lighter REST-only framework (e.g. FastAPI) would also satisfy the requirements as written.
- **Decision: deferred.** REQUIREMENTS.md does not mandate one over the other; this choice has no bearing on functional requirements and is left open per the instruction to exclude framework-choice rationale from requirements-driven decisions. Noted here only because it was an explicit input.

### REST API surface (provisional, minimal)

Only what's needed to satisfy traceable requirements below — not a full API spec:

- An internal endpoint/trigger to receive or pull a qualifying log event (supports FR-1.1–FR-1.5).
- An internal call path from ingestion → analysis → notification (supports FR-2.1–FR-2.6).
- No externally consumed API is implied by REQUIREMENTS.md beyond what's needed to talk to Grafana/VictoriaLogs (inbound) and MS Teams (outbound). No public-facing REST API for third parties is required.

### Authentication and session model

- `TO BE DECIDED`. REQUIREMENTS.md only states that access governance should ride on existing k8s cluster auth (`k8s.auth`) rather than a separate auth system (FR-4.1). No session model is implied because there is currently no first-party UI (see above) — sessions would only become relevant if a UI is later introduced.

### Data model

- No persistent business entities are mandated by REQUIREMENTS.md beyond the log stream itself flowing through VictoriaMetrics → Vector → Grafana (VictoriaLogs datasource).
- Whether the system needs to persist anything (e.g. dedup state for FR-3.5, history of past notifications) is `UNKNOWN` — FR-3.5 marks deduplication/rate-limiting as unresolved, which is the main candidate driver for needing a persistent store. Until that's decided, no datastore is assumed beyond what's required to call out to Grafana/VictoriaLogs and MS Teams.

### Deployment model

- Target: k8s, packaged as a Helm chart (per stated input).
- Scale (pod count) is `TO BE DECIDED` — no load/throughput requirement exists in REQUIREMENTS.md to size this from.

### Security

- `TO BE DECIDED` per stated input; intentionally out of scope for this pass beyond FR-4.1 (reuse `k8s.auth`, no parallel auth system).

## Requirement Traceability

| Requirement | Architecture Element |
|---|---|
| FR-1.1 | Ingestion stage — Grafana (VictoriaLogs datasource) as primary log source |
| FR-1.2 | Ingestion stage — optional direct VictoriaLogs path |
| FR-1.3 | Ingestion stage — language-agnostic log handling (no per-language ingestion logic) |
| FR-1.4 | Ingestion stage — per-entry source identification (mechanism `UNKNOWN`) |
| FR-1.5 | Ingestion stage — notification trigger condition (`TO BE DECIDED`) |
| FR-2.1 | Analysis stage — core analysis step |
| FR-2.2 | Analysis stage — GitHub repo lookup/enrichment (mapping mechanism `UNKNOWN`) |
| FR-2.3 | Analysis stage — web search enrichment |
| FR-2.4 | Analysis stage — output: human-readable summary + solution(s) |
| FR-2.5 | Analysis stage — output constraint: no raw log passthrough |
| FR-2.6 | Analysis stage — inconclusive-analysis behavior (`TO BE DECIDED`) |
| FR-3.1 | Notification stage — MS Teams delivery |
| FR-3.2 | Notification stage — required notification content |
| FR-3.3 | Notification stage — source app/service identification in notification |
| FR-3.4 | Notification stage — message format (`TO BE DECIDED`) |
| FR-3.5 | Notification stage / data model — dedup/rate-limit (`UNKNOWN`, drives persistence question) |
| FR-4.1 | Deployment/access — reuse `k8s.auth`, no parallel auth system |
| FR-4.2 | Authentication and session model — role differentiation (`UNKNOWN`) |

## Dependency Rules

- The **Ingestion stage** MUST NOT depend on the Notification stage; it MAY be depended on by the Analysis stage.
- The **Analysis stage** MUST NOT call MS Teams directly; all outbound notification delivery MUST go through the Notification stage.
- The **Notification stage** MUST NOT reach back into Grafana/VictoriaLogs or perform analysis; it only formats and sends what the Analysis stage produces.
- The system MUST NOT introduce a standalone authentication/authorization mechanism that bypasses `k8s.auth` (per FR-4.1).
- No component MUST depend on a frontend/client framework unless a future requirement introduces a genuine UI workflow; until then, client-framework choice (e.g. React) has no architectural dependency to satisfy.
- External dependencies are limited to: (a) Grafana/VictoriaLogs (inbound, log data), (b) application-specific GitHub repositories (inbound, analysis enrichment), (c) web search (inbound, analysis enrichment), (d) MS Teams (outbound, notification delivery). No other third-party integration is implied by current requirements.
