# REQUIREMENTS

## Required Requirement Inputs

- `Project purpose: Logs analyzer that is hosted in k8s. Any type of log-based notifications that would be sent in a MS Teams, disregarding if they are based on logs from JAVA, Python or any other app, should be translated into human-readeble text with a possible issue and possible solutions.`
- `Primary users / actors: Members of an existing MS Teams group`
- `Core workflows: Take existing logs from Grafana (VictoriaLogs datasource), analyze them (using app dedicated Github repos and web search) and sending notification with a summury of an issue and possible ways to resolve it.`
- `Business objects / data entities: Existing log collection stack is: VictoriaMetrics, Vector, Grafana. All the application, who's logs are being collected and sent to Grafana are hosted in k8s cluster`
- `External integrations: With the log source, Ideally from Grafana, but if it's easier to take it directly from k8s-based VictoriaLogs - than from it. Also, integration with MS Teams`
- `Authentication / roles: The app would be hosted in k8s so all the people from k8s.auth`
- `Regulatory or privacy constraints: no specific requirements`

## Functional Requirements

### FR-1: Log Ingestion

- **FR-1.1** — The system MUST retrieve logs from a log data source, preferring Grafana (VictoriaLogs datasource) as the primary source.
- **FR-1.2** — The system MAY retrieve logs directly from VictoriaLogs (bypassing Grafana) when this is functionally simpler, provided the resulting log content is equivalent.
- **FR-1.3** — The system MUST support logs originating from applications written in any language (explicitly including Java and Python), without requiring language-specific ingestion logic.
- **FR-1.4** — The system MUST be able to identify which application/service a given log entry originated from. Exact mechanism (label, source field, namespace, etc.) is `UNKNOWN`.
- **FR-1.5** — The trigger condition for "a log-based notification should be sent" (e.g. error-level threshold, specific pattern match, alert rule from Grafana) is `TO BE DECIDED`.

### FR-2: Log Analysis

- **FR-2.1** — When a qualifying log/notification is detected, the system MUST analyze the log content to determine the nature of the issue.
- **FR-2.2** — The system MUST be able to consult application-specific GitHub repositories as part of analysis, in order to ground the analysis in the actual source/config of the affected app. Mechanism for mapping a log source to its corresponding GitHub repo is `UNKNOWN`.
- **FR-2.3** — The system MUST be able to use web search as an additional analysis input (e.g. for known error messages, library-specific issues, common fixes).
- **FR-2.4** — The system MUST produce, per analyzed issue: (a) a human-readable summary of the issue, and (b) one or more possible solutions/resolution steps.
- **FR-2.5** — The system MUST NOT forward raw, untranslated log dumps as the notification content — output MUST be human-readable text, regardless of the source log's original format or language-specific stack trace conventions.
- **FR-2.6** — Behavior when analysis cannot determine a likely cause (i.e., no confident summary/solution can be produced) is `TO BE DECIDED`.

### FR-3: Notification Delivery

- **FR-3.1** — The system MUST send a notification to an existing MS Teams group for each analyzed issue.
- **FR-3.2** — Each notification MUST contain a human-readable summary of the issue and at least one possible resolution.
- **FR-3.3** — Each notification SHOULD identify the source application/service the issue relates to, so MS Teams group members can triage by affected system.
- **FR-3.4** — Notification format/structure within MS Teams (e.g. plain message vs. card/adaptive card) is `TO BE DECIDED`.
- **FR-3.5** — Deduplication/rate-limiting behavior for repeated or recurring instances of the same underlying issue is `UNKNOWN`.

### FR-4: Access & Authorization

- **FR-4.1** — Access to the system (where applicable, e.g. any admin/config surface) MUST be governed by the existing k8s cluster authentication/authorization (`k8s.auth`); the system MUST NOT introduce a separate, parallel auth mechanism.
- **FR-4.2** — Whether any role differentiation exists among MS Teams group members (e.g. who can configure rules vs. who only receives notifications) is `UNKNOWN`.

## Open Questions

- What specific condition(s) qualify a log entry as notification-worthy (severity threshold, alert rule, anomaly detection, manual trigger)?
- How is a log entry/source mapped to its corresponding application-specific GitHub repository for analysis context?
- What should happen if analysis is inconclusive — suppress notification, send a partial/best-effort notification, or escalate differently?
- Is there a need for deduplication or rate-limiting of repeated notifications for the same recurring issue?
- What MS Teams notification format is expected (plain text message vs. Adaptive Card vs. other)?
- Is there any role distinction among MS Teams group members (consumers vs. configurators of the system)?
- Are there any latency/freshness expectations for how quickly a notification should follow the originating log event? (Not currently treated as in-scope per provided constraints, but worth confirming.)
