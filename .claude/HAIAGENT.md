# HAIAGENT.md — Digital Card Service Tenant Binding

**Organization**: MOSIP  
**Domain**: Identity Platform  
**Service**: Digital Card Service  
**Version**: 1.2.0  
**Status**: Accepted  

---

## Executive Summary

This document binds the HAI Intel Modernization Coordinator to the **MOSIP Digital Card Service** — a Spring Boot 3.2.3 microservice responsible for credential packaging and issuance post-registration.

The service operates in the **identity/credentialing** domain, which triggers specialized lens rules, domain context skills, and evidence traceability chains beyond the generic application-modernization blueprint.

---

## 1. Service Identity

### Core Attributes

| Attribute | Value |
|-----------|-------|
| **Service Name** | Digital Card Service |
| **Group ID** | io.mosip |
| **Artifact ID** | digital-card-service |
| **Current Version** | 1.2.0 |
| **Framework** | Spring Boot 3.2.3 |
| **Language** | Java 21 (OpenJDK) |
| **Persistence** | PostgreSQL 12+ |
| **Deployment** | Kubernetes 1.24+ (Helm 3, Istio) |
| **Architecture Style** | Layered + Event-Driven Hybrid |

### Ecosystem Position

- **Role in MOSIP**: Post-registration credential packaging and issuance
- **Lifecycle Stage**: Production (stable in MOSIP 1.2.0+)
- **Integration Points**: 6 major MOSIP services (Registration Processor, Credential Service, KeyManager, DataShare, PDF generator, Audit Logging)
- **Domain Classification**: **Identity/Credentialing** (non-payment, but regulated like payments: PII protection, audit trails, non-repudiation)

---

## 2. Domain Specialization

### Domain Classification

**This is an identity/credentialing service**, not a payment system, but shares key properties with payment systems:

| Property | Payment System | Digital Card Service |
|----------|----------------|----------------------|
| **PII Handling** | Account numbers, routing numbers | Biometric templates, demographic data |
| **Non-Repudiation** | Transaction signatures | Digital signatures on credentials |
| **Audit Trail** | Immutable transaction log | Immutable credential issuance/revocation log |
| **Key Rotation** | Periodic key updates | 90-day rotation mandate |
| **Compliance** | SOX, PCI-DSS | GDPR, DPDP Act |
| **Scaling Driver** | Transaction throughput | Credential issuance velocity (10x growth: 30→300 cards/min) |

### Always-Applied Lenses (like payment domain)

For Digital Card Service, treat these as **always triggered** (not conditional):

1. **Security Architecture Lens** (`applying-security-architecture-lens`)
   - Credential data is PII; encryption non-negotiable
   - WebSub signature verification required
   - Keycloak OAuth2 integration non-optional
   - Audit trail immutability critical

2. **Distributed Systems Lens** (`applying-distributed-systems-lens`)
   - WebSub event-driven; eventual consistency model
   - 6+ external service integrations (orchestration complexity)
   - Idempotency key handling (duplicate prevention)
   - Dead-letter queue for failed events

3. **Cloud-Native Lens** (`applying-cloud-native-lens`)
   - Kubernetes is deployment target (not optional)
   - Horizontal scaling mandatory (300-500 cards/min target)
   - Health probes + auto-recovery required
   - Zero-downtime deployment SLA

4. **Observability Lens** (`applying-observability-lens`)
   - End-to-end credential issuance tracing required
   - Encryption operation metrics mandatory
   - Revocation propagation SLA monitoring (<5s)
   - Alert on event loss (critical path)

5. **Data Architecture Lens** (`applying-data-architecture-lens`)
   - PII protection + data residency compliance
   - Encryption at rest + in transit (non-optional)
   - Audit trail immutability (write-once schema)
   - Database replication + backup SLA

---

## 3. Domain Context Skills (Tier 5: Client Context)

These 4 skills are loaded from `.claude/skills/` and provide domain-specific context:

### Skill 1: `digital-card-service-domain-context`
- **Purpose**: Establish MOSIP ecosystem position, credential lifecycle, integration points
- **Inputs**: System identity, endpoints, dependencies
- **Outputs**: Business drivers, credential lifecycle stages, integration matrix, regulatory context
- **Triggered in S01**: When endpoint patterns match `(card|credential)` and dependencies match `io.mosip.*`
- **Feeds**: `synthesizing-drivers`, lens triggering rules

### Skill 2: `digital-card-service-api-context`
- **Purpose**: Document REST API surface, WebSub subscriptions, data models, throughput contracts
- **Inputs**: Endpoints, event topics, JPA entity definitions
- **Outputs**: 5 REST endpoints + 5 event subscriptions, data model schema, throughput targets (current vs. target)
- **Triggered in S01**: When `extracting-controller-endpoints` finds card/credential endpoints
- **Feeds**: `applying-distributed-systems-lens`, `modeling-target-architecture` (S02)

### Skill 3: `digital-card-service-architecture-decisions`
- **Purpose**: Document ADRs for WebSub, PostgreSQL, Spring Boot, Kubernetes, encryption choices
- **Inputs**: System identity, framework, infrastructure, deployment
- **Outputs**: 5 ADRs with context, alternatives considered, trade-offs, evidence basis
- **Triggered in S01**: When `analyzing-architecture-style` identifies patterns
- **Feeds**: `hypothesizing-modernization-strategy`, `modeling-target-architecture` (S02)

### Skill 4: `digital-card-service-nfr-context`
- **Purpose**: Establish performance targets, scalability constraints, compliance requirements, observability SLAs
- **Inputs**: System identity, current metrics (30-50 cards/min), regulatory regime
- **Outputs**: Latency targets (p50, p95, p99), throughput targets (300-500 cards/min), SLA summary
- **Triggered in S01**: When `capturing-nfr-baseline` or `assessing-design-level-nfrs` runs
- **Feeds**: `applying-nfr-performance-lens`, `assessing-design-level-nfrs`

---

## 4. Enhanced Lens Triggering Rules (S01)

### Override: Promote Conditional Lenses to Always-Triggered

| Lens | Default Status | Digital Card Service | Rationale |
|------|---|---|---|
| `applying-cloud-native-lens` | Conditional (when cloud-target declared) | **Always triggered** | Kubernetes is deployment mandate; K8s maturity 7.5/10 indicates production usage |
| `applying-observability-lens` | Conditional (when SLA stated) | **Always triggered** | Credential issuance is business-critical; observability mandatory for debugging event loss, latency spikes |

### Add: New Conditional Lenses (identity-specific patterns)

| Lens | Trigger Condition | Rationale |
|------|---|---|
| **`credential-issuance-pattern-lens`** (NEW) | Triggered when: (1) `endpoint_pattern == (card\|credential)` AND (2) `event_subscription == WebSub` AND (3) `external_key_management == true` | Credential issuance is a distinct architectural pattern (async, multi-step, idempotent, key-dependent) |
| **`encryption-operations-lens`** (NEW) | Triggered when: (1) Code imports `CryptomanagerUtil` OR (2) Dependencies include `key-manager-api` | Key management + encryption performance critical for scale; latency directly impacts card generation SLA |
| **`event-driven-consistency-lens`** (NEW) | Triggered when: (1) Event subscriptions present AND (2) No strong ordering guarantee documented | WebSub introduces eventual consistency; idempotency + ordering semantics critical for credential correctness |

---

## 5. Evidence Traceability Chain

Each finding in S01 must trace through this chain (enforced in **anti-confabulation discipline**):

```
Evidence → Driver (business + technical) → Triggered Lens → Pattern Decision → Target Model → Action → Acceptance Criteria
```

### Example: Caching Disabled

| Step | Content |
|------|---------|
| **Evidence** | Hibernate L2 cache disabled in `application.properties` (observed in code) |
| **Driver** | Performance bottleneck: Credential metadata cached in memory would reduce Credential Service calls by 80%; cache miss latency ~50ms per card |
| **Triggered Lens** | `applying-nfr-performance-lens`, `credential-issuance-pattern-lens` |
| **Pattern Decision** | Implement distributed cache (Redis) for credential metadata (TTL: 5 minutes) |
| **Target Model** | Credential metadata cache layer between Digital Card Service ↔ Credential Service |
| **Action** | (S02/S03) Add Redis dependency, cache layer service, cache invalidation on credential update |
| **Acceptance Criteria** | p99 latency drops from 500ms → 200ms; cache hit rate >80%; zero data stale-access incidents |

---

## 6. Output Formats & Audience

### Deliverable Format by Stage

| Stage | Format | Audience | Location |
|-------|--------|----------|----------|
| **S01** | Markdown + diagrams (`.md` files) | Architects, tech leads | GitHub repository (source of truth) |
| **S02** | Modernization specification (Markdown) | Architects, product owners | GitHub + Confluence (for stakeholder review) |
| **S03-S07** | Code changes, runbooks, operational guides | Developers, DevOps | GitHub (code), Confluence (runbooks) |

### Markdown Emphasis

**Per MODERNIZATION_FRAMEWORK.md**: Markdown is preferred output (GitHub source of truth). HTML is generated on-demand for Confluence + stakeholder presentations.

---

## 7. Memory Model (3-Tier)

This service maintains memory at three scopes:

### Tier 1: User-Global (`~/.haiagent/memory/`)
- Cross-service learnings (e.g., "Spring Boot + PostgreSQL + WebSub is standard MOSIP pattern")
- Output preferences (e.g., "always include ADRs in S01 output")

### Tier 2: Tenant-Level (`digital-card-service/.claude/memory/`)
- **Cached MCP Context**
  - `postgres-context.md` (if SQL Server MCP enabled) — Digital Card Service schema
  - `confluence-context.md` (if Confluence MCP enabled) — Existing architecture pages
- **Cross-Session Patterns**
  - `mosip-ecosystem-patterns.md` (reused knowledge: WebSub topic conventions, KeyManager integration, RBAC roles)
  - `digital-identity-domain-decisions.md` (captured from S01 ADR output)

### Tier 3: Session-Level (`digital-card-service/.claude/session-<id>/memory/`)
- Per-S01 memory MDs (auto-generated by evidence collectors):
  - `codebase-patterns.md` (component breakdown, fan-in/out, package cycles)
  - `quality-findings.md` (PMD hotspots, exception swallowing, hardcoded magic strings)
  - `risk-findings.md` (security: credentials in config files, security: HTTP 200 for errors; performance: disabled cache)
  - `ddd-analysis.md` (domain shape; core aggregate: Card; value objects: Credential, EncryptionMetadata)
  - `test-coverage-map.md` (0% baseline; tests needed for card generation, revocation, encryption)
  - `nfr-baseline.md` (current: 30-50 cards/min, latency p99 2000-5000ms; target: 300-500 cards/min)

**MEMORY.md Index**: All three tiers maintain an index file listing memory artifacts (1-line pointers, <150 chars each).

---

## 8. Skill Ecosystem Binding

This service binds the following skills from the coordinator blueprint:

### Strategic Skills (Always Loaded)
- `profiling-system-identity` — Extract system identity (Spring Boot 3.2.3, PostgreSQL, Kubernetes, etc.)
- `analyzing-architecture-style` — Identify patterns (Layered + Event-Driven Hybrid)
- `analyzing-ddd-context` — Domain shape (Card, Credential, Encryption as core concepts)
- `assessing-design-level-nfrs` — Baseline: Scalability 60%, Availability 70%, Security 55%, Observability 50%
- `assessing-security-posture` — Security assessment: 4.3/10 (credentials in config files, HTTP 200 for errors, no input validation)
- `assessing-sustainability` — Codebase stability, deprecation track record, technical debt
- `hypothesizing-modernization-strategy` — Propose strategy hypothesis
- `selecting-modernization-strategy` (S02) — Make final call on modernization approach

### Architecture Lens Skills (Conditional + Domain Overrides)

**Always Triggered**:
- `applying-security-architecture-lens` (domain override)
- `applying-distributed-systems-lens` (domain override)
- `applying-cloud-native-lens` (domain override: promoted from conditional)
- `applying-observability-lens` (domain override: promoted from conditional)
- `applying-data-architecture-lens` (domain override)

**Conditionally Triggered**:
- `applying-ddd-lens` (when DDD output has bounded contexts)
- `applying-integration-architecture-lens` (6+ external integrations detected)
- `applying-nfr-performance-lens` (SLA stated: p99 < 500ms)
- `applying-application-architecture-lens` (non-trivial app)

**Domain-Specific Lens** (NEW):
- `credential-issuance-pattern-lens` (identity domain)
- `encryption-operations-lens` (key management detected)
- `event-driven-consistency-lens` (eventual consistency model)

### Evidence Collector Skills (26 existing)

All standard collectors apply:
- `digesting-pmd-violations` → identifies exception swallowing, large classes, hardcoded magic strings
- `digesting-checkstyle-violations` → identifies code style issues
- `classifying-npe-risks` → identifies null safety gaps
- `extracting-controller-endpoints` → identifies 5 REST endpoints (POST /cards/generate, GET /cards/{id}, GET /cards/{id}/download, POST /cards/{id}/verify, POST /cards/{id}/revoke)
- `digesting-entity-shape` → identifies Card, Credential, EncryptionMetadata entities
- `digesting-build-dependencies` → identifies spring-cloud-stream + RabbitMQ (WebSub), spring-security (Keycloak), spring-data-jpa (PostgreSQL)
- `narrating-ddd-analysis` → identifies domain shape
- `narrating-test-coverage` → shows 0% baseline
- Others as per standard pipeline

### Domain Context Skills (4 Custom Skills)

Loaded from `.claude/skills/`:
- `digital-card-service-domain-context` (custom)
- `digital-card-service-api-context` (custom)
- `digital-card-service-architecture-decisions` (custom)
- `digital-card-service-nfr-context` (custom)

---

## 9. HAIAGENT Directive Precedence

Directives in this file override lower tiers:

1. **Bundle-level** (HAI coordinator blueprint) — Lowest priority
2. **User-global** (`~/.haiagent/HAIAGENT.md`)
3. **Tenant-level** (This file: `digital-card-service/.claude/HAIAGENT.md`) — **Highest priority**

Example: This file specifies "always trigger cloud-native lens" (overrides blueprint default of "conditional").

---

## 10. Configuration & Hooks

### Environment Variables (Optional)

```bash
# SQL Server MCP (if database introspection needed)
HAIAGENT_MCP_SQLSERVER_HOST=postgres.default.svc.cluster.local
HAIAGENT_MCP_SQLSERVER_PORT=5432
HAIAGENT_MCP_SQLSERVER_USER=mosip_read
HAIAGENT_MCP_SQLSERVER_DB=digital_card_service

# Confluence MCP (if publishing to Confluence)
HAIAGENT_MCP_CONFLUENCE_BASE_URL=https://pattabhia.atlassian.net
HAIAGENT_MCP_CONFLUENCE_USERNAME=pattabhi.amperayani@gmail.com
HAIAGENT_MCP_CONFLUENCE_SPACE=SD
```

### Operational Hooks

(Optional: if using CI/CD automation)
- Post-S01 hook: Publish architectural assessment to Confluence
- Post-S02 hook: Create Architecture Decision Records as GitHub issues
- Pre-S03 hook: Validate test coverage (fail if <50% on new methods)
- Post-S07 hook: Archive session, update operational runbooks

---

## 11. Acceptance Criteria for Modernization

### Success Metrics

| Metric | Current | Target | Acceptance Criteria |
|--------|---------|--------|-------------------|
| **Throughput** | 30-50 cards/min | 300-500 cards/min | 10x improvement; p99 latency <500ms |
| **Latency p99** | 2-5 seconds | <500ms | PDF generation async (non-blocking) |
| **Test Coverage** | 0% | 80%+ | Unit + integration tests for critical paths |
| **Security Score** | 4.3/10 | 8/10 | Credentials in Vault, input validation, proper HTTP status codes |
| **Observability** | 50% maturity | 85%+ maturity | End-to-end tracing, encryption metrics, revocation SLA alerting |
| **Availability** | 99% (implied) | 99.9% SLA | Zero-downtime deployment, pod auto-recovery, database failover |

### Modernization Roadmap

**Phase 0 (Weeks 1-2): Critical Security Fixes**
- Move credentials from properties → Vault
- Fix HTTP status codes (not all 200)
- Add input validation
- Expected cost: $10K (2 weeks, 2 developers)

**Phase 1 (Weeks 3-6): Performance Foundation**
- Enable Hibernate L2 cache (Redis)
- Expand thread pool (2-5 → 50-100)
- Configure database connection pooling (HikariCP)
- Expected cost: $25K (4 weeks)

**Phase 2 (Weeks 7-10): Async Processing**
- Async PDF generation (non-blocking queue)
- Implement OpenTelemetry tracing
- Add encryption operation metrics
- Expected cost: $30K (4 weeks)

**Phase 3 (Weeks 11-13): Scalability + Observability**
- Configure HPA (3-10 replicas based on CPU)
- Implement SLA alerting (latency, event loss, revocation propagation)
- Add test coverage (80% target)
- Expected cost: $18K (3 weeks)

**Phase 4 (Weeks 14-16): Security Hardening**
- Implement API rate limiting
- Add request signing (prevent MITM attacks)
- Credential revocation SLA hardening (<5s propagation)
- Expected cost: $20K (3 weeks)

**Phase 5 (Weeks 17-20): Code Quality + Architecture**
- Refactor large service classes (SRP violation)
- Exception hierarchy cleanup (stop swallowing exceptions)
- Architecture documentation (C4 diagrams, ADRs in git)
- Expected cost: $25K (4 weeks)

**Total**: 20 weeks, ~$128K, 5-developer team (or 10 weeks with 10 developers)

---

## 12. Known Constraints & Assumptions

### Constraints

1. **WebSub Event Ordering**: Not guaranteed; eventual consistency model required (5-10s delivery latency acceptable)
2. **Key Rotation**: 90-day mandate; old keys retained in KeyManager; transparent to application
3. **PDF Generation Bottleneck**: Synchronous library call blocks card generation (improvement path: async service)
4. **Test Coverage**: Zero baseline; requires test-first approach in S03+

### Assumptions

1. **MOSIP Ecosystem Stability**: Credential Service, KeyManager, DataShare available (99.9% SLA assumed)
2. **PostgreSQL Cluster**: Primary-standby replication assumed; backup/restore SLA assumed
3. **Kubernetes Infrastructure**: Managed K8s cluster assumed (auto-updates, security patches applied)
4. **Keycloak Integration**: Already deployed + configured (OAuth2 scopes, roles available)

---

## 13. Contact & Governance

- **Product Owner**: MOSIP Platform Team
- **Architecture Lead**: Enterprise Architecture Board
- **Operations**: MOSIP SRE Team
- **Compliance**: MOSIP Legal + Data Governance

**Changes to this file** require approval from Architecture Lead (for lens triggers, NFR targets) and Product Owner (for roadmap, acceptance criteria).

---

**License**: Haiintel  
**Status**: Approved  
**Last Updated**: May 12, 2026  
**Version**: 1.0
