# HAI Modernization Coordinator — Enhancement Proposal for Digital Card Service

**Date**: May 12, 2026  
**Status**: Analysis & Recommendation  
**Scope**: Review of base coordinator prompts + skills; proposal for digital-card-service domain context enhancements

---

## 1. Review of Existing Infrastructure ✅

### 1.1 Coordinator Base Preamble (`base.md`)

**Current State**: ✅ Solid foundation
- Clear identity and role definition
- Traceability chain well-articulated: Evidence → Driver → Triggered Lens → Pattern Decision → Target Model → Action → Acceptance Criteria
- Skill ecosystem structure (5 layers) defined
- HAIAGENT.md hierarchy and precedence clear
- Memory model (3-tier) properly scoped
- Anti-confabulation discipline explicit
- Output discipline aligned to evidence citations

**Fit for digital-card-service**: ✅ **As-is sufficient**
- Coordinator can ingest any Java Spring Boot service
- Skill ecosystem supports payment-adjacent use cases (microservice with security, performance, scalability constraints)
- Evidence collection via existing 26 collector skills covers all digital-card-service evidence types (dependencies, exceptions, endpoints, build errors, test coverage)

---

### 1.2 Dispatcher Mode (`dispatcher.md`)

**Current State**: ✅ Operationally sound
- FSM workflow well-defined (Greet → Detect branch → Confirm → Run → Handle gates)
- Q&A protocol clear for operator queries between gates
- Anti-confabulation rules enforced (read first, never guess, cite file:line)
- Tool surface constraints clear (only `run_shell_command` for git ls-remote, `read_file`, `grep_search`, `glob_files`, MCP tools)

**Fit for digital-card-service**: ✅ **As-is sufficient**
- Workflow applies to any git repo
- Q&A protocol works for any modernization session

---

### 1.3 S01 Comprehend Stage (`s01_comprehend.md`)

**Current State**: ✅ Evidence-driven assessment design
- Strategic-skill chain well-sequenced
- Evidence coverage assessment explicit (8-category scoring)
- Lens triggering rules clear (always vs. conditional)
- Payment-domain specialization recognized (ledger, compliance, settlement, payout idempotency)

**Fit for digital-card-service**: ⚠️ **Partial fit — needs domain lens triggers**
- Generic lens triggers apply (distributed-systems, security, data, observability, devops)
- **Missing**: Domain-specific triggers for identity/credentialing context (e.g., credential service integration, WebSub event-driven pattern, encryption key management)
- **Missing**: Digital identity domain terminology (card, credential, issuance, encryption, revocation, OIDC, WebSub)

---

### 1.4 Evidence Collection Skills (26 existing)

**Current State**: ✅ Comprehensive coverage
- Collectors for: PMD violations, Checkstyle, NPE risks, endpoints, entities, dependencies, build errors, codebase patterns, DDD analysis, test coverage, NFR baseline

**Fit for digital-card-service**: ✅ **As-is sufficient**
- All 26 collectors apply to Spring Boot microservices
- Will surface concrete evidence on:
  - Code quality issues (exceptions, magic strings, large classes) — detected
  - Architecture patterns (layered + event-driven) — detectable via endpoint + dependency analysis
  - Security concerns (credentials in config, HTTP 200 for errors, sensitive logging) — may need custom pattern rules in PMD/Checkstyle
  - Performance issues (disabled caching, small thread pools) — may need custom NFR rules
  - Test coverage gaps (0 test files currently) — will be detected

---

## 2. Gap Analysis: Current vs. Digital Card Service Domain

| Dimension | Current State | Gap | Required Enhancement |
|-----------|---------------|-----|----------------------|
| **Domain Identity** | Generic microservice framework | Missing MOSIP identity context | Create `digital-card-service-domain-context` skill |
| **Domain Terminology** | No specialized vocabulary | Missing terms (card, credential, issuance, encryption, WebSub) | Create `digital-card-service-terminology` skill |
| **Architecture Decisions** | Generic patterns (DDD, layered, event-driven) | Missing explanation of *why* WebSub, PostgreSQL, Kubernetes | Create `digital-card-service-architecture-decisions` skill |
| **API Surface** | Generic endpoint extraction | Missing credential/issuance workflow documentation | Create `digital-card-service-api-context` skill |
| **Domain-Specific Drivers** | Business + technical drivers generic | Missing identity platform context (registration pipeline, credential service integration) | Enhance `synthesizing-drivers` skill with domain triggers |
| **Lens Triggering** | 8 generic lens rules | Missing identity/credentialing-specific lens rules | Add triggering conditions for security (key management), integration (WebSub, external services) |
| **NFR Baseline** | Generic SLA patterns | Missing card issuance throughput targets (30→300 cards/min), encryption performance, credential validation latency | Create `digital-card-service-nfr-context` skill |

---

## 3. Proposed Enhancements

### 3.1 New Domain Context Skills (4 skills)

These skills are **triggered conditionally** in S01 when `assessing-evidence-coverage` detects this is a digital identity service. They feed `synthesizing-drivers` and lens triggering logic.

#### A. `digital-card-service-domain-context` (Tier 5: Client Context)
**Purpose**: Establish MOSIP ecosystem position and credential lifecycle
**Inputs**: 
- System identity (via `profiling-system-identity`)
- Endpoints (via `extracting-controller-endpoints`)
- Dependencies (via `digesting-build-dependencies`)

**Outputs** (memory/digital-card-service-domain.md):
- MOSIP identity pipeline role: Post-registration, automated credential packaging
- Credential lifecycle: issuance → storage → delivery → validation
- Integration points: Registration Processor, Credential Service, KeyManager, DataShare, WebSub subscribers
- Ecosystem maturity: MOSIP 1.2.x context, related services versions
- Business drivers: Scale, PII protection, regulatory compliance

#### B. `digital-card-service-api-context` (Tier 5: Client Context)
**Purpose**: Document REST API and event subscription patterns
**Inputs**:
- Endpoints (via `extracting-controller-endpoints`)
- Dependencies (via `digesting-build-dependencies`)
- Code patterns (via `narrating-codebase-patterns`)

**Outputs** (memory/digital-card-service-api.md):
- REST endpoints: `/cards/generate`, `/cards/{id}`, `/cards/{id}/download`, `/cards/verify`, `/cards/revoke`
- Event subscriptions: WebSub topics (credential.issued, card.generated, revocation.triggered)
- Data models: Card, Credential, Encryption metadata, Event payload
- Integration patterns: WebSub signature verification, external service calls (PDF generation, encryption)
- Throughput contracts: Expected traffic (30-50 cards/min currently), target (300-500/min)

#### C. `digital-card-service-architecture-decisions` (Tier 5: Client Context)
**Purpose**: Explain architectural choices and constraints
**Inputs**:
- System identity, dependencies, build metadata

**Outputs** (memory/digital-card-service-decisions.md):
- **Why WebSub**: Asynchronous credential notifications from external services; eventual consistency acceptable
- **Why PostgreSQL**: Relational integrity for credential metadata, compliance audit trails, existing MOSIP ecosystem choice
- **Why Spring Boot**: Microservice interoperability within MOSIP, standard Java ecosystem
- **Why Kubernetes**: Multi-tenant deployment, horizontal scaling, existing MOSIP infrastructure
- **Why Docker + OpenJDK 21 + ZGC**: GC tuning for credential processing latency, cloud-native packaging
- **Constraints**: Synchronous PDF generation (blocking), no distributed tracing, no cache layer

#### D. `digital-card-service-nfr-context` (Tier 5: Client Context)
**Purpose**: Establish domain-specific performance and compliance baselines
**Inputs**:
- System identity, test coverage, endpoints, NFR baseline

**Outputs** (memory/digital-card-service-nfrs.md):
- **Performance**: Card generation latency (target: <200ms per card), throughput (target: 300-500 cards/min), PDF generation blocking time
- **Scalability**: Horizontal scaling (multi-replica deployment), no single replica constraint, thread pool sizing (currently 2-5 threads)
- **Availability**: 99.9% uptime SLA (credential issuance blocking), zero-downtime deployments required
- **Security**: PII encryption mandatory, credential revocation latency <1s, WebSub signature verification enforced, audit trail completeness
- **Compliance**: GDPR/data residency (per MOSIP policy), credential audit retention, key rotation cadence (90-day policy)
- **Observability**: Credential issuance trace (end-to-end), encryption operation metrics, WebSub subscription health, failure alerting for revocation

---

### 3.2 Enhanced Lens Triggering Rules (S01)

**Add to `s01_comprehend.md` Step 3 (lens triggering)**:

For **identity/credentialing services** (detected via endpoint naming + dependency analysis):

| Lens | Trigger Condition | Rationale |
|------|-------------------|-----------|
| `applying-security-architecture-lens` | **Always** (already triggered by any endpoint) — **Enhance** with identity-specific checks: credential encryption, key management, revocation, audit trails | Credential data is PII; encryption + key rotation are non-negotiable |
| `applying-distributed-systems-lens` | **Always** (already triggered by any network call) — **Enhance** with event-driven specifics: WebSub subscriptions, eventual consistency, idempotency, clock skew tolerance | WebSub is async; credential issuance must handle out-of-order notifications |
| `applying-cloud-native-lens` | **Conditional** (currently "when cloud-target declared") — **Promote to Always** if Kubernetes deployment detected | Digital Card Service targets Kubernetes; scaling + auto-recovery are critical |
| `applying-data-architecture-lens` | **Always** (already triggered by any persistent store) — **Enhance** with identity-specific: PII residency, replication strategy, backup/recovery SLAs, audit trail immutability | Credential data requires compliance-grade data governance |
| `applying-observability-lens` | **Always** (already triggered by any production-bound system) — **Enhance** with identity-specific: credential issuance tracing, encryption operation metrics, revocation SLA monitoring | Credential operations are business-critical; observability is non-optional |
| **`NEW: credential-issuance-pattern-lens`** | **Conditional** (triggered when `endpoint_pattern == credential\|card` AND `event_subscription == WebSub`) | Credential issuance is a distinct architectural pattern (async, multi-step, idempotent) |
| **`NEW: encryption-operations-lens`** | **Conditional** (triggered when code imports CryptomanagerUtil OR KeyManager dependencies) | Key management + encryption operation performance are critical for scale |

---

### 3.3 Enhanced `synthesizing-drivers` Skill

**Add domain-specific driver detection** for digital identity services:

**Business Drivers** (current: market timing, compliance, cost):
- **Credential issuance velocity**: Scale from 30→300 cards/min (10x growth)
- **PII protection**: Mandatory encryption + audit trails (GDPR, MOSIP policy)
- **Ecosystem reliability**: WebSub integration with 6+ external services (Registration Processor, Credential Service, KeyManager, DataShare, PDF generator, encryption service)

**Technical Drivers** (current: code quality, performance, architecture):
- **Async credential notifications**: WebSub subscriptions introduce eventual consistency + ordering challenges
- **Encryption bottleneck**: Synchronous key wrapping blocks PDF generation (currently no async)
- **Thread pool starvation**: 2-5 threads insufficient for 300 cards/min (need 50-100 threads)
- **Key rotation frequency**: 90-day rotation mandate requires seamless key rollover without re-issuance

---

## 4. Recommended Implementation Plan

### Phase A: Create New Skills (Week 1)

1. **Create** `.claude/skills/digital-card-service-domain-context/SKILL.md`
   - YAML frontmatter (name, description, task_type: "analysis", license: "Haiintel")
   - Purpose, Inputs, Output Contract sections
   - Integration points table
   - Lifecycle diagram (registration → issuance → delivery → validation)

2. **Create** `.claude/skills/digital-card-service-api-context/SKILL.md`
   - REST endpoint inventory (6 endpoints with method/path/controller)
   - WebSub subscription patterns (5 topics)
   - Data model definitions (Card, Credential, EncryptionMetadata)
   - Throughput contracts (current vs. target)

3. **Create** `.claude/skills/digital-card-service-architecture-decisions/SKILL.md`
   - 8 "Why" decisions documented with evidence citations
   - Constraints articulated
   - Trade-offs captured

4. **Create** `.claude/skills/digital-card-service-nfr-context/SKILL.md`
   - Performance baselines (latency targets, throughput targets, thread pool sizing)
   - Compliance requirements (GDPR, audit trail, key rotation)
   - Observability SLAs

### Phase B: Enhance Base Prompts (Week 1-2)

1. **Enhance** `base.md` (Modernization Coordinator preamble)
   - Add section: "Digital Identity Specialization" (optional context for identity-domain services)
   - List the 4 new domain context skills
   - Note: "For identity/credentialing services, domain-context skills feed lens triggering and driver synthesis"

2. **Enhance** `s01_comprehend.md` (Stage S01 orchestration)
   - Add Step 1.5: "For identity/credentialing services, dispatch domain-context skills in parallel with evidence collectors"
   - Add to Step 3 (lens triggering): Table of domain-specific lens triggers (see section 3.2 above)
   - Add Step 3.5: "For identity services, always trigger cloud-native lens (Kubernetes mandatory) and promote observability lens"

3. **Create** `.claude/HAIAGENT.md` (Digital Card Service tenant binding)
   - Define service identity: "MOSIP Identity Platform → Digital Card Service (credential issuance)"
   - Domain specialization: "identity/credentialing"
   - Required skills: List the 4 new domain context skills + standard strategic/lens skills
   - Lens triggers: Document the enhanced rules
   - Output format: "Markdown first, HTML on-demand for Confluence publishing"
   - Memory bindings: Point to `memory/digital-card-service-*.md`

---

## 5. Alignment with MODERNIZATION_FRAMEWORK.md

The enhancements directly implement **Phase 2** (Enhanced Structure) and **Phase 4** (Enhanced Prompts & Skills) from the framework:

| Framework Item | Enhancement Proposal |
|---|---|
| Gap 1: Evidence Traceability | New domain-context skills feed evidence → driver → lens → pattern chains |
| Gap 4: Decision Records | `digital-card-service-architecture-decisions` skill documents "why" decisions with evidence |
| Gap 5: Multi-lens Analysis | Enhanced lens triggering rules explicitly label each finding with trigger lens |
| Gap 6: Acceptance Criteria | `digital-card-service-nfr-context` skill defines measurable performance/compliance AC |
| Phase 4: Domain Context Skills | Creates the 4 domain skills outlined in framework |
| Phase 6: Evidence Traceability Matrix | New skills populate the Evidence → Driver → Lens rows with real digital-card-service examples |

---

## 6. What Does NOT Need Enhancement

✅ **Coordinator Base Preamble** (`base.md`) — Generic; applies to any microservice  
✅ **Dispatcher Mode** (`dispatcher.md`) — Generic workflow; applies to any git repo  
✅ **Evidence Collectors** (26 existing skills) — Generic; will surface digital-card-service evidence correctly  
✅ **Strategic Skills** (profiling, analyzing, assessing, hypothesizing) — Generic; work for any system  
✅ **Architecture Lens Skills** (16 existing) — Generic; digital-card-service will trigger appropriate subsets  
✅ **S02-S07 Stages** (specify, test, architect, generation, delivery, operations) — Generic; no domain specialization needed

---

## 7. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|-----------|
| New skills introduce confabulation if domain facts are hallucinated | HIGH | Skills use only evidence-grounded outputs (endpoint extraction, dependency analysis); validate domain assumptions against actual code |
| Lens triggering rules too narrow, miss relevant patterns | MEDIUM | Include "always trigger for identity services" for observability + security; add "conditional" rules for specific patterns |
| Domain terminology skills conflict with generic skill vocabulary | LOW | Domain skills are supplementary; base.md already has vocabulary directive precedence model |
| Enhanced triggers add overhead to S01 | MEDIUM | Skills are dispatched in parallel; minimal additional latency |

---

## 8. Next Steps

### For Approval:
1. **User Review**: Does the proposal align with your modernization vision?
2. **Domain Validation**: Are the 4 new skills capturing the right domain context?
3. **Lens Triggering**: Does the enhanced triggering correctly prioritize identity-domain patterns?

### For Implementation (with approval):
1. **Create feature branch** `feat/hai-domain-enhancements` in digital-card-service repo
2. **Create `.claude/skills/` directory** with 4 new domain-context SKILL.md files
3. **Create `.claude/HAIAGENT.md`** with tenant binding and enhanced directives
4. **Create `.claude/ENHANCEMENT_LOG.md`** documenting all changes
5. **Push branch** and merge to main (or keep as feature branch for pilot)

---

**License**: Haiintel  
**Status**: Proposal ready for review  
**Estimated Implementation Time**: 2-3 days (skills + prompt enhancements)
