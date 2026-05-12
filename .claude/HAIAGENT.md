# HAIAGENT.md — Digital Card Service Tenant Binding

**Service**: MOSIP Digital Card Service  
**Domain**: Identity/Credentialing  
**Framework**: Spring Boot 3.2.3  
**Deployment**: Kubernetes + Helm + Istio  
**Current Version**: 1.2.0  
**Status**: Production  

---

## 1. Service Identity

| Attribute | Value |
|-----------|-------|
| **Group ID** | io.mosip |
| **Artifact ID** | digital-card-service |
| **Database** | PostgreSQL 12+ |
| **Event Framework** | WebSub (Spring Cloud Stream + RabbitMQ) |
| **Authentication** | OAuth2 + Keycloak |
| **Deployment Target** | Kubernetes 1.24+ |

---

## 2. Domain Specialization

**Classification**: Identity/Credentialing (not payment, but similar constraints)

**Key Properties**:
- PII handling (biometric + demographic data)
- Non-repudiation (digital signatures on credentials)
- Audit trails (immutable issuance/revocation logs)
- Key rotation (90-day mandate)
- Compliance (GDPR, DPDP Act)
- Eventual consistency (WebSub ordering)

---

## 3. Generic Skills to Use

Reference skills from `hai-domain-configs/blueprints/coordinator/skills/`:

1. **`identity-credentialing-domain-context`**
   - Apply to: MOSIP ecosystem, credential lifecycle (8 stages), 6 integration points
   - Override: Regulatory context = DPDP Act + GDPR (if EU)

2. **`credential-issuance-api-context`**
   - Apply to: 5 REST endpoints, 5 WebSub subscriptions
   - Override: Throughput targets = 300-500 cards/min (not generic)

3. **`identity-service-architecture-patterns`**
   - Apply to: WebSub + PostgreSQL + Spring Boot + Kubernetes choices
   - Override: None (decisions already made; document rationale)

4. **`credential-issuance-nfr-context`**
   - Apply to: Performance, scalability, compliance, observability
   - Override: Latency p99 = <500ms (not generic), thread pool = 50-100 (not generic)

5. **`identity-service-implementation-patterns`**
   - Apply to: Design patterns, best practices, anti-patterns
   - Override: Async PDF generation pattern (deferred improvement)

6. **`generating-architecture-diagrams`**
   - Generate: C4 context + container + component diagrams
   - Output: SVG inline in HTML

7. **`generating-flow-diagrams`**
   - Generate: Credential lifecycle flow, event-driven architecture, revocation propagation
   - Output: SVG inline in HTML

8. **`generating-html-reports`**
   - Generate: Beautiful HTML artifacts from markdown findings
   - Output: Self-contained HTML files with Tailwind CSS, interactivity, export buttons

9. **`generating-interactive-dashboards`**
   - Generate: Strategy recommendations dashboard
   - Output: HTML with tabs, cards, metrics, drill-down links

---

## 4. Enhanced Lens Triggering

### Always-Applied Lenses (never conditional for this service)

- ✅ `applying-security-architecture-lens` — PII data; encryption mandatory
- ✅ `applying-distributed-systems-lens` — WebSub eventual consistency; 6+ integrations
- ✅ `applying-cloud-native-lens` — Kubernetes deployment; auto-scaling mandatory
- ✅ `applying-observability-lens` — Credential issuance critical path; event loss alerting mandatory
- ✅ `applying-data-architecture-lens` — PII protection; audit trail immutability

### Conditionally-Triggered Lenses

- ✅ `applying-ddd-lens` (when DDD bounded contexts detected)
- ✅ `applying-integration-architecture-lens` (6+ external services)
- ✅ `applying-nfr-performance-lens` (SLA: p99 <500ms stated)
- ✅ `applying-application-architecture-lens` (non-trivial app)

### Domain-Specific Lenses (from generic skills)

- ✅ `identity-service-implementation-patterns` — Design patterns specific to credential issuance
- ✅ `credential-lifecycle-pattern-lens` — 8-stage issuance workflow
- ✅ `encryption-key-management-lens` — 90-day rotation, key wrapping performance

---

## 5. NFR Targets (Service-Specific Overrides)

| NFR | Target | Rationale |
|-----|--------|-----------|
| **Latency p99** | <500ms per card | PDF generation bottleneck identified |
| **Throughput** | 300-500 cards/min | 10x growth from current 30-50 |
| **Scalability** | 50-100 threads | From current 2-5 (thread pool expansion) |
| **Availability** | 99.9% SLA | Credential issuance critical path |
| **Test Coverage** | 80%+ | From current 0% baseline |
| **Security Score** | 8/10 | From current 4.3/10 |
| **Observability** | 85%+ maturity | End-to-end tracing + alerting |

---

## 6. Compliance Requirements

- **DPDP Act (India)**: Encryption at rest/transit, audit trails, consent, data residency
- **GDPR (if EU)**: Data protection, DSAR response <30 days, breach notification <72 hours
- **OIDC Standard (future)**: Credential claims mapping for verifier interoperability

---

## 7. Output Preferences

- **Format**: HTML-first artifacts (self-contained, interactive, exportable)
- **Diagrams**: SVG inline (zoomable, legend toggles)
- **Reports**: Cross-referenced (evidences → patterns → strategy)
- **Citation**: file:line pointers for all findings
- **Export**: Markdown, JSON, CSV buttons on every dashboard

---

## 8. Modernization Goals

**5-Phase Roadmap** (documented in strategy/success-metrics.html):
- Phase 0: Critical security fixes (2 weeks, $10K)
- Phase 1: Performance foundation (4 weeks, $25K)
- Phase 2: Async processing (4 weeks, $30K)
- Phase 3: Observability (3 weeks, $18K)
- Phase 4: Security hardening (3 weeks, $20K)
- Phase 5: Code quality (4 weeks, $25K)

**Total**: 20 weeks, ~$128K, 5-developer team

---

## 9. Memory Model (3-Tier)

- **User-global**: `~/.haiagent/memory/` (cross-service learnings)
- **Tenant-level**: `digital-card-service/.claude/memory/` (MOSIP patterns, cached MCP context)
- **Session-level**: `digital-card-service/.claude/session-*/memory/` (per-analysis findings)

---

## 10. Analysis Methodology

**Approach**: Option B (fresh code analysis with generic skills)

**Output**: `hai-reports/` directory with:
- **evidences/** (L3 - Atoms): Detailed findings from code analysis
- **patterns/** (L2 - Molecules): Synthesis + patterns from findings
- **strategy/** (L1 - Organism): Strategic recommendations + decision records

**Deliverables**:
- 7 HTML evidence files (C4 diagrams, quality, security, performance, architecture, DDD, NFR)
- 7 HTML pattern synthesis files (business, technical, risk, modernization, scalability, observability, devops)
- 4 HTML strategy files (assessment, strategy, ADRs, metrics)
- 5 SVG diagram files (C4, credential lifecycle, event flow, risk heatmap, roadmap timeline)
- Cross-references between all layers
- Evidence citations in every finding

---

## 11. Contact & Governance

- **Product Owner**: MOSIP Platform Team
- **Architecture Lead**: Enterprise Architecture Board
- **Operations**: MOSIP SRE Team

---

**License**: Haiintel  
**Status**: Ready for S01 Comprehend  
**Last Updated**: May 12, 2026
