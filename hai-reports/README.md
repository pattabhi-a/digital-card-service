# HAI Enterprise Architecture Analysis — MOSIP Digital Card Service

**Branch:** `feat/hai-analysis-reports`
**Generated:** 2026-05-12
**Scope:** 79 Java files at `src/main/java/io/mosip/digitalcard/` validated against actual code.

## Layered Analysis Model

```
        +----------------------+
        |      Strategy        |  Executive view, ADRs, roadmap
        +----------+-----------+
                   ^
        +----------+-----------+
        |      Patterns        |  Synthesis, risk matrix, blueprints
        +----------+-----------+
                   ^
        +----------+-----------+
        |      Evidences       |  Atomic findings cited at file:line
        +----------------------+
```

**Reading order for executives:** start with `strategy/architectural-assessment.html`.
**Reading order for engineers:** start with `evidences/codebase-structure.html`.

## Directory Structure

```
hai-reports/
├── README.md
├── evidences/   (7 files)
├── patterns/    (7 files)
├── strategy/    (4 files)
└── diagrams/    (5 SVGs)
```

## File Index

### Evidences
| File | Description |
|------|-------------|
| evidences/codebase-structure.html | Package layout, LOC, fan-in/out, C4 context |
| evidences/quality-findings.html | Exception swallowing, magic strings, large classes |
| evidences/security-analysis.html | Secrets in config, HTTP 200 errors, sensitive logs |
| evidences/performance-profile.html | Sync PDF, no caching, thread-pool=5 |
| evidences/architecture-patterns.html | Layered + event-driven, adapter, DTO patterns |
| evidences/ddd-analysis.html | Anaemic entity, single bounded context |
| evidences/nfr-baseline.html | Maturity scores per NFR with citations |

### Patterns
| File | Description |
|------|-------------|
| patterns/business-capability-synthesis.html | Capability map → evidence trail |
| patterns/technical-patterns-synthesis.html | Recurring code patterns + improvements |
| patterns/risk-synthesis.html | Top-10 risks with heatmap |
| patterns/modernization-patterns.html | Async, Redis, Vault, OTel patterns |
| patterns/scalability-blueprint.html | 30→300 cards/min math |
| patterns/observability-strategy.html | Logging, metrics, tracing strategy |
| patterns/devops-maturity.html | CI/CD + Helm + Istio assessment |

### Strategy
| File | Description |
|------|-------------|
| strategy/architectural-assessment.html | KPI dashboard + top findings |
| strategy/modernization-strategy.html | 6-phase roadmap with cost/timeline |
| strategy/decision-records.html | 5 ADRs |
| strategy/success-metrics.html | KPI targets per phase |

### Diagrams
- `diagrams/c4-context-diagram.svg`
- `diagrams/credential-lifecycle-flow.svg`
- `diagrams/event-driven-architecture.svg`
- `diagrams/risk-heatmap.svg`
- `diagrams/modernization-roadmap-timeline.svg`

## Cross-Reference Convention

- Evidence → Pattern: `→ See pattern: X`
- Pattern → Evidence: `← Driven by evidence: Y`
- Strategy → either: explicit links to both layers

Every claim in evidence/pattern files is anchored to `file:line`.
