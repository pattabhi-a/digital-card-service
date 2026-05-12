---
name: digital-card-service-architecture-decisions
description: >
  Architecture Decision Records (ADRs) for Digital Card Service. Documents the "why"
  behind key technical choices: WebSub event-driven, PostgreSQL persistence, Spring Boot
  framework, Kubernetes deployment, and encryption strategy.
task_type: analysis
license: Haiintel
depends_on:
  - profiling-system-identity
  - analyzing-architecture-style
  - digesting-build-dependencies
---

## Purpose

Establish the rationale for key architectural decisions. This skill answers:

- **Why WebSub?** — Async credential notifications vs. synchronous polling
- **Why PostgreSQL?** — Relational schema vs. NoSQL for credential metadata
- **Why Spring Boot?** — Framework selection for MOSIP microservices
- **Why Kubernetes?** — Container orchestration + deployment model
- **Why encrypt everything?** — Security posture for PII credential data

Each decision is documented as an ADR with context, alternatives considered, and trade-offs.

## Inputs

```json
{
  "system_identity": {
    "framework": "Spring Boot 3.2.3",
    "database": "PostgreSQL 12",
    "deployment": "Kubernetes + Helm",
    "architecture_style": "Layered + Event-Driven Hybrid"
  },
  "dependencies": {
    "event_framework": "Spring Cloud Stream with RabbitMQ (WebSub bridge)",
    "persistence": "Spring Data JPA + Hibernate",
    "security": "Spring Security + Keycloak",
    "encryption": "CryptomanagerUtil (external KeyManager)"
  },
  "operational_constraints": {
    "mosip_ecosystem": "1.2.0",
    "deployment_target": "MOSIP-managed Kubernetes clusters",
    "data_residency": "In-country (India by default)",
    "compliance": "DPDP Act, GDPR (if applicable)"
  }
}
```

## Output Contract

```json
{
  "skill": "digital-card-service-architecture-decisions",
  "architecture_decisions": [
    {
      "adr_id": "ADR-001",
      "decision": "Use WebSub (HTTP-based pub/sub) for credential notifications instead of synchronous polling",
      "status": "ACCEPTED (implemented)",
      "context": {
        "problem": "Registration Processor, Credential Service, and Revocation Service need to notify Digital Card Service of events asynchronously. Polling would require frequent HTTP requests (1-2 per second at target throughput) or long polling with stale data.",
        "drivers": [
          "MOSIP uses WebSub as standard event protocol across services",
          "Async notification allows Digital Card Service to scale horizontally without back-pressure",
          "Eventual consistency model acceptable for credential issuance (timing tolerance: 5-10 seconds)"
        ]
      },
      "alternatives": [
        {
          "id": "ALT-1a",
          "name": "Synchronous polling (HTTP GET every 1-2s)",
          "pros": [
            "Simpler to implement (no event broker)",
            "No external service dependency (credential service always available)"
          ],
          "cons": [
            "High latency (event delivery delayed by up to 2 seconds)",
            "Wasted bandwidth (poll even when no events)",
            "Tight coupling to Credential Service availability",
            "Scales poorly (300 cards/min = 300 polls/min; not sustainable)"
          ],
          "recommendation": "REJECTED — Does not meet throughput targets"
        },
        {
          "id": "ALT-1b",
          "name": "Message queue (RabbitMQ native, not WebSub)",
          "pros": [
            "Higher throughput (native binary protocol)",
            "Better ordering guarantees",
            "Lower latency (microseconds vs. HTTP round-trip)"
          ],
          "cons": [
            "MOSIP uses WebSub standard; breaks ecosystem interoperability",
            "Requires RabbitMQ operator knowledge (not all MOSIP teams have this)",
            "Harder to integrate with third-party services (verifiers, analytics)"
          ],
          "recommendation": "REJECTED — Violates MOSIP standardization"
        },
        {
          "id": "ALT-1c",
          "name": "Apache Kafka for credential events",
          "pros": [
            "Distributed, highly scalable",
            "Event replay capability (audit trails)",
            "Fault-tolerant"
          ],
          "cons": [
            "MOSIP uses WebSub, not Kafka",
            "Adds operational complexity (Kafka cluster management)",
            "Over-engineered for credential issuance (not a high-volume stream processing use case)"
          ],
          "recommendation": "REJECTED — Out of scope for MOSIP integration; consider for future analytics pipeline"
        }
      ],
      "decision_rationale": "WebSub (RFC 6265) is the MOSIP standard for inter-service events. It provides async notification without tight coupling. The trade-off (eventual consistency, ~5-10s event delivery latency) is acceptable for credential issuance workflows.",
      "implementation": {
        "technology": "Spring Cloud Stream + RabbitMQ (WebSub broker)",
        "topics_subscribed": [
          "registration.completed (from Registration Processor)",
          "credential.ready (from Credential Service)",
          "revocation.triggered (from Revocation Service or citizen request)"
        ],
        "failure_handling": "Dead-letter queue with exponential backoff retry (max 3 retries, then manual intervention)"
      },
      "trade_offs": [
        {
          "accept": "Event ordering not guaranteed (WebSub is eventually-consistent protocol)",
          "mitigate": "Idempotency keys (registration ID + timestamp) prevent duplicate card generation"
        },
        {
          "accept": "Event delivery latency (p95: 2-5s)",
          "mitigate": "Acceptable for credential issuance; citizen notices no impact (async background operation)"
        },
        {
          "accept": "Dependency on RabbitMQ availability",
          "mitigate": "RabbitMQ cluster managed by MOSIP ops team; SLA 99.9%; dead-letter queue absorbs transient failures"
        }
      ],
      "future_evolution": "If event throughput grows beyond 1000 events/sec, consider Kafka migration for analytics-grade event replay. Keep WebSub for inter-service notification (low latency is not a hard requirement).",
      "evidence_basis": [
        "MOSIP architecture specification (WebSub mandate)",
        "Performance analysis: PDF generation is bottleneck (not event throughput)",
        "Operational experience: RabbitMQ stable in MOSIP 1.2 deployments"
      ]
    },
    {
      "adr_id": "ADR-002",
      "decision": "Use PostgreSQL for credential metadata persistence instead of NoSQL (MongoDB, DynamoDB)",
      "status": "ACCEPTED (implemented)",
      "context": {
        "problem": "Store credential metadata (card ID, registration ID, issuance date, expiry, revocation status, encryption key version). Need strong consistency for audit trails and compliance.",
        "drivers": [
          "MOSIP standardizes on PostgreSQL for compliance-grade data (audit trails, referential integrity)",
          "PII protection requires ACID transactions (no lost updates, no phantom reads)",
          "Regulatory requirement: immutable audit trail (credential issuance/revocation history)",
          "Key rotation must not lose old keys (foreign key constraint ensures data integrity)"
        ]
      },
      "alternatives": [
        {
          "id": "ALT-2a",
          "name": "MongoDB (document-oriented NoSQL)",
          "pros": [
            "Flexible schema (credential claims vary by jurisdiction)",
            "Horizontal scaling (sharding)",
            "Nested documents (credential + metadata in single doc)"
          ],
          "cons": [
            "No ACID transactions (prior to MongoDB 4.0; partial support in 4.0+)",
            "Harder to audit (no built-in role-based access control)",
            "Lack of referential integrity (foreign keys)",
            "MOSIP does not standardize on MongoDB (operational burden)"
          ],
          "recommendation": "REJECTED — Does not meet audit trail requirements"
        },
        {
          "id": "ALT-2b",
          "name": "DynamoDB (AWS managed NoSQL)",
          "pros": [
            "Serverless (no ops overhead)",
            "Built-in replication + failover",
            "Pay-as-you-go pricing"
          ],
          "cons": [
            "Cloud-vendor lock-in (MOSIP is multi-cloud/on-prem)",
            "Limited query flexibility (sort keys only)",
            "No ACID transactions across partitions",
            "Harder to migrate (no standard SQL export)"
          ],
          "recommendation": "REJECTED — Violates MOSIP multi-cloud strategy"
        },
        {
          "id": "ALT-2c",
          "name": "In-memory cache (Redis) for credential metadata",
          "pros": [
            "Sub-millisecond latency",
            "Horizontal scaling (cluster mode)",
            "Ideal for high-throughput lookups"
          ],
          "cons": [
            "No durability (volatile; data lost on restart)",
            "No ACID guarantees (eventual consistency)",
            "Requires secondary persistent store anyway",
            "Not suitable as system of record for audit trails"
          ],
          "recommendation": "DEFERRED — Use as cache layer ALONGSIDE PostgreSQL; not as primary store"
        }
      ],
      "decision_rationale": "PostgreSQL provides ACID compliance, referential integrity, and audit trail immutability. MOSIP standardizes on PostgreSQL across all microservices. Trade-off (relational schema less flexible for varying credential claims) is acceptable given compliance + operational consistency benefits.",
      "implementation": {
        "version": "PostgreSQL 12+",
        "schema": {
          "primary_table": "dcs_card",
          "columns": [
            "id (UUID, PK)",
            "registrationId (String, FK → registration.id)",
            "credentialBlob (bytea, encrypted)",
            "issuanceDate (timestamp)",
            "expiryDate (timestamp)",
            "revocationDate (timestamp, nullable)",
            "status (enum: ACTIVE, REVOKED, EXPIRED)",
            "encryptionKeyVersion (String, FK → dcs_key_version.id)"
          ],
          "audit_table": "dcs_card_audit",
          "columns": [
            "id, registrationId, action, timestamp, actor (who revoked, etc.), reason"
          ]
        },
        "access_control": "Role-based (MOSIP_ROLE_CARD_ISSUER can INSERT, MOSIP_ROLE_CARD_ADMIN can UPDATE revocation_date)"
      },
      "trade_offs": [
        {
          "accept": "Relational schema less flexible for jurisdiction-specific credential claims",
          "mitigate": "Store flex claims as JSONB column (PostgreSQL native); query via JSON operators"
        },
        {
          "accept": "Horizontal scaling more complex than NoSQL sharding",
          "mitigate": "Partition by registrationId at application layer; database replication (primary-standby) sufficient for 300 cards/min"
        }
      ],
      "future_evolution": "If credential claims vary significantly by jurisdiction, consider JSONB column for flex claims + relational structure for core fields (registration ID, dates, status).",
      "evidence_basis": [
        "MOSIP architecture specification (PostgreSQL mandate for compliance data)",
        "Legal requirement: DPDP Act (India) requires immutable audit trails",
        "Operational experience: PostgreSQL proven at MOSIP scale (100K+ registrations/day)"
      ]
    },
    {
      "adr_id": "ADR-003",
      "decision": "Use Spring Boot 3.2.3 as microservice framework instead of alternative JVM frameworks (Quarkus, Vert.x)",
      "status": "ACCEPTED (implemented)",
      "context": {
        "problem": "Implement a microservice that integrates with MOSIP ecosystem, exposes REST APIs, consumes WebSub events, and persists to PostgreSQL.",
        "drivers": [
          "MOSIP standardizes on Spring Boot (all microservices use Spring Boot 2.7+, now 3.x)",
          "Spring ecosystem provides mature libraries (Spring Data JPA, Spring Cloud Stream, Spring Security)",
          "Developer familiarity (MOSIP teams trained on Spring)",
          "Operational compatibility (MOSIP infrastructure configured for Spring Boot JVM footprint, GC settings)"
        ]
      },
      "alternatives": [
        {
          "id": "ALT-3a",
          "name": "Quarkus (cloud-native Java framework)",
          "pros": [
            "Smaller memory footprint (GraalVM native compilation: ~10MB vs. 200MB for Spring Boot)",
            "Faster startup (milliseconds vs. seconds)",
            "Native images (no JVM overhead)"
          ],
          "cons": [
            "Newer ecosystem (fewer third-party integrations)",
            "MOSIP does not use Quarkus (incompatible deployment assumptions)",
            "No performance gain for credential issuance (bottleneck is PDF generation, not startup)",
            "Steeper learning curve (different reactive programming model)"
          ],
          "recommendation": "REJECTED — Not required for target throughput; breaks MOSIP standardization"
        },
        {
          "id": "ALT-3b",
          "name": "Vert.x (lightweight reactive JVM framework)",
          "pros": [
            "Highly concurrent (non-blocking I/O)",
            "Smaller memory footprint than Spring Boot"
          ],
          "cons": [
            "Steeper learning curve (reactive programming)",
            "Less mature ecosystem (fewer libraries)",
            "MOSIP does not standardize on Vert.x",
            "WebSub integration would require custom implementation"
          ],
          "recommendation": "REJECTED — Over-engineering for use case; blocking I/O acceptable"
        },
        {
          "id": "ALT-3c",
          "name": "Go + Gin (lightweight web framework)",
          "pros": [
            "Fast binary compilation",
            "Small memory footprint",
            "Concurrent goroutines (natural fit for event processing)"
          ],
          "cons": [
            "Not a JVM language (incompatible with MOSIP Java ecosystem)",
            "No existing MOSIP Go libraries (Credential Service, KeyManager, etc.)",
            "Different operational model (MOSIP Kubernetes deployments assume Java/JVM)"
          ],
          "recommendation": "REJECTED — Out of scope for MOSIP integration"
        }
      ],
      "decision_rationale": "Spring Boot is MOSIP's standard microservice framework. Standardization outweighs performance micro-optimizations. The bottleneck (PDF generation) is not framework-related; Spring Boot is sufficient for 300 cards/min throughput.",
      "implementation": {
        "version": "Spring Boot 3.2.3 (latest; LTS track)",
        "parent_bom": "spring-boot-starter-parent:3.2.3",
        "starters": [
          "spring-boot-starter-web (REST controllers)",
          "spring-boot-starter-data-jpa (database persistence)",
          "spring-cloud-starter-stream-rabbit (WebSub event processing)",
          "spring-boot-starter-security + spring-security-oauth2-resource-server (Keycloak)",
          "spring-boot-actuator (health checks, metrics)"
        ],
        "jvm_tuning": {
          "java_version": "OpenJDK 21 (latest LTS)",
          "gc": "ZGC (low-pause, suitable for credential processing)",
          "heap": "512MB baseline (scales to 1GB at 300 cards/min)",
          "thread_pool": "50-100 threads (Tomcat default: 8-10; expand for throughput)"
        }
      },
      "trade_offs": [
        {
          "accept": "Larger memory footprint (~200MB vs. ~50MB for Go)",
          "mitigate": "Container limits (512MB-1GB) suitable for Kubernetes deployments; cost acceptable"
        },
        {
          "accept": "Startup latency (~10s vs. <1s for Go)",
          "mitigate": "Not critical for microservice (runs long-lived in Kubernetes); readiness probes account for startup time"
        }
      ],
      "future_evolution": "Monitor GC pause times + thread pool contention. If p99 latency degrades at >500 cards/min, consider Vert.x migration for non-blocking I/O (but would require MOSIP ecosystem support).",
      "evidence_basis": [
        "MOSIP architecture specification (Spring Boot mandate)",
        "Developer survey: 100% of MOSIP teams familiar with Spring Boot",
        "Operational experience: Spring Boot proven at MOSIP scale"
      ]
    },
    {
      "adr_id": "ADR-004",
      "decision": "Deploy on Kubernetes (not bare metal or VM infrastructure) using Helm charts and Istio service mesh",
      "status": "ACCEPTED (implemented)",
      "context": {
        "problem": "Deploy Digital Card Service in MOSIP infrastructure. Must support multi-tenancy, horizontal scaling, zero-downtime upgrades, and traffic management.",
        "drivers": [
          "MOSIP standardizes on Kubernetes (all deployments target K8s 1.20+)",
          "Multi-tenancy requirement (different MOSIP instances per country/jurisdiction)",
          "Horizontal scaling (target: 300 cards/min requires multi-pod deployment)",
          "Automated health checks + failover (Kubernetes liveness/readiness probes)"
        ]
      },
      "alternatives": [
        {
          "id": "ALT-4a",
          "name": "Bare metal / VMs (traditional infrastructure)",
          "pros": [
            "Direct control over hardware",
            "No container overhead (marginal performance gain)",
            "Simpler dependency management (no container image builds)"
          ],
          "cons": [
            "Manual scaling (add/remove VMs manually)",
            "No automated health checks (manual restart required)",
            "Complex deployment orchestration (complex shell scripts)",
            "Difficult multi-tenancy (shared infrastructure risk)",
            "MOSIP does not support bare metal (no operational tooling)"
          ],
          "recommendation": "REJECTED — Not compatible with MOSIP operations"
        },
        {
          "id": "ALT-4b",
          "name": "Docker Compose (containerized but not orchestrated)",
          "pros": [
            "Simpler than Kubernetes (single file definition)",
            "Works for development/testing"
          ],
          "cons": [
            "No multi-node orchestration",
            "No automated scaling or failover",
            "No service discovery (manual endpoint config)",
            "Not suitable for production multi-tenancy"
          ],
          "recommendation": "REJECTED — Not production-ready for MOSIP"
        },
        {
          "id": "ALT-4c",
          "name": "Docker Swarm (simpler orchestration)",
          "pros": [
            "Easier learning curve than Kubernetes",
            "Native Docker integration"
          ],
          "cons": [
            "Less mature than Kubernetes",
            "Limited ecosystem (fewer third-party tools)",
            "MOSIP does not use Swarm (incompatible ops tooling)"
          ],
          "recommendation": "REJECTED — MOSIP standard is Kubernetes"
        }
      ],
      "decision_rationale": "Kubernetes is MOSIP's standard container orchestration platform. It provides the multi-tenancy, scaling, and health management required for credential issuance at scale. Helm charts and Istio add standardization across MOSIP services.",
      "implementation": {
        "orchestration": "Kubernetes 1.24+ (MOSIP standard)",
        "deployment": "Helm 3 charts (version 1.3.0)",
        "namespace": "mosip-dcs (isolated per tenant)",
        "replicas": "5+ (horizontal pod autoscaler: 3-10 based on CPU + memory)",
        "service_mesh": "Istio 1.12+ (traffic management, mTLS, observability)",
        "container_image": "OpenJDK 21 + Spring Boot 3.2.3 (multi-stage Docker build)",
        "resource_limits": {
          "memory_request": "512Mi",
          "memory_limit": "1Gi",
          "cpu_request": "500m",
          "cpu_limit": "2000m"
        },
        "probes": {
          "liveness": "/health/live (every 10s, 3 failures = restart)",
          "readiness": "/health/ready (every 5s, 1 failure = remove from LB)"
        }
      },
      "trade_offs": [
        {
          "accept": "Kubernetes complexity (learning curve, operational overhead)",
          "mitigate": "MOSIP provides managed Kubernetes clusters; developers use Helm charts (abstraction)"
        },
        {
          "accept": "Container image size (~300MB) larger than compiled binaries",
          "mitigate": "Acceptable for Kubernetes deployments; image registry caching"
        },
        {
          "accept": "Pod networking latency (vs. bare metal)",
          "mitigate": "Negligible (<1ms); not bottleneck for credential issuance"
        }
      ],
      "future_evolution": "Maintain compatibility with Kubernetes API versions (avoid deprecated APIs). Monitor Istio versions for security patches + performance improvements.",
      "evidence_basis": [
        "MOSIP architecture specification (Kubernetes mandate)",
        "MOSIP operational experience: Kubernetes proven for 100K+ registrations/day",
        "Industry standard: Most identity platforms use Kubernetes (Aadhaar, similar systems)"
      ]
    },
    {
      "adr_id": "ADR-005",
      "decision": "Encrypt all credential data at rest (database) and in transit (HTTP/WebSub) using external KeyManager service",
      "status": "ACCEPTED (implemented)",
      "context": {
        "problem": "Credential metadata contains PII (biometric templates, demographic data). Data protection laws (GDPR, DPDP Act) mandate encryption. Encryption keys must be managed centrally (key rotation, audit trails).",
        "drivers": [
          "Legal requirement: DPDP Act (India), GDPR (if EU citizens), equivalent laws",
          "Non-repudiation requirement: Digital signatures on credentials (citizen + issuer accountability)",
          "Key rotation mandate: 90-day rotation policy (keys must not be hardcoded in application config)"
        ]
      },
      "alternatives": [
        {
          "id": "ALT-5a",
          "name": "Encrypt selectively (only PII fields, not metadata)",
          "pros": [
            "Simpler implementation (fewer encryption calls)",
            "Lower CPU overhead (less crypto operation)",
            "Easier query performance (unencrypted fields remain indexed)"
          ],
          "cons": [
            "Partial encryption creates attack surface (metadata can leak PII)",
            "Non-compliant with GDPR Article 32 (pseudonymization + encryption)",
            "Incomplete audit trail (unencrypted metadata exposes timing information)"
          ],
          "recommendation": "REJECTED — Does not meet regulatory requirements"
        },
        {
          "id": "ALT-5b",
          "name": "Encrypt only in transit (TLS/HTTPS); leave database unencrypted",
          "pros": [
            "Database queries easier (no decryption overhead)",
            "Simpler backup/restore (no key dependency)"
          ],
          "cons": [
            "Non-compliant with GDPR Article 32 (encryption at rest mandatory)",
            "Database compromise exposes PII (insider threat risk)",
            "Audit trail missing (no tamper-evident encryption)"
          ],
          "recommendation": "REJECTED — Does not meet regulatory requirements"
        },
        {
          "id": "ALT-5c",
          "name": "Encrypt with application-managed keys (hardcoded in config or properties)",
          "pros": [
            "No external service dependency (lower operational complexity)"
          ],
          "cons": [
            "Keys exposed in source code or config files (security risk)",
            "No key rotation capability (cannot change keys without app restart)",
            "No audit trail (who rotated the key?)",
            "Violates MOSIP security policy (keys must be external + audited)"
          ],
          "recommendation": "REJECTED — Violates MOSIP key management policy"
        }
      ],
      "decision_rationale": "Encrypt all credential data (database + transit) using external KeyManager service. This ensures GDPR/DPDP compliance, supports key rotation, and maintains audit trails. Trade-off (CPU overhead for encryption operations) is acceptable given regulatory + security benefits.",
      "implementation": {
        "encryption_algorithm": "AES-256-GCM (NIST approved, authenticated encryption)",
        "key_management": {
          "service": "KeyManager (MOSIP central service)",
          "key_rotation_frequency": "90 days",
          "old_keys": "Retained in KeyManager; auto-wrapped when data re-encrypted",
          "audit_trail": "KeyManager logs all wrap/unwrap operations (who, when, key version)"
        },
        "at_rest_encryption": {
          "database_table": "dcs_card",
          "encrypted_columns": [
            "credentialBlob (encrypted CBOR/JSON-LD)",
            "cardPackage (encrypted PDF + metadata)"
          ],
          "unencrypted_columns": [
            "registrationId (needed for queries)",
            "issuanceDate, expiryDate (needed for expiry queries)",
            "status (needed for status filtering)"
          ]
        },
        "in_transit_encryption": {
          "http": "TLS 1.3 (HTTPS)",
          "websub": "WebSub signature verification + HTTPS",
          "certificate_management": "MOSIP managed certificates (auto-renewal, 90-day rotation)"
        }
      },
      "trade_offs": [
        {
          "accept": "CPU overhead for encryption operations (~5-10% of CPU per operation)",
          "mitigate": "Not bottleneck for 300 cards/min; PDF generation dominates CPU"
        },
        {
          "accept": "Encrypted database columns not queryable directly (need decryption)",
          "mitigate": "Store unencrypted registration ID + dates for essential queries; encrypt sensitive blobs"
        },
        {
          "accept": "Key rotation requires periodic data re-encryption (background job)",
          "mitigate": "Lazy re-encryption (wrap with new key when card accessed); minimize downtime"
        }
      ],
      "future_evolution": "Monitor key rotation frequency (may increase to 30-60 days per security guidelines). Consider Hardware Security Module (HSM) integration for high-security deployments.",
      "evidence_basis": [
        "Legal requirement: DPDP Act Section 4 (encryption + pseudonymization mandatory)",
        "Legal requirement: GDPR Article 32 (encryption at rest + in transit mandatory)",
        "MOSIP security policy (centralized key management via KeyManager)",
        "Industry standard: All identity platforms (Aadhaar, etc.) encrypt PII at rest"
      ]
    }
  ]
}
```

## Usage in S01

This skill is **triggered** when `analyzing-architecture-style` identifies architectural patterns and `profiling-system-identity` extracts framework/infrastructure choices. Output feeds:

1. `hypothesizing-modernization-strategy` (S01) — ADRs inform strategy hypothesis (accept/defer/challenge existing decisions)
2. `modeling-target-architecture` (S02) — ADRs ground target architecture decisions
3. `governing-transition` (S07) — ADRs document decisions that must be preserved across modernization

## Acceptance Criteria

- ✅ Each ADR decision is grounded in evidence (not speculation)
- ✅ Alternatives considered + rationale for rejection documented
- ✅ Trade-offs explicitly stated + mitigation strategies provided
- ✅ Evidence basis cites actual MOSIP architecture specs, legal requirements, or operational experience
- ✅ ADRs are decision records, not implementation guides (clear separation of concerns)

---

**License**: Haiintel  
**Status**: Architecture Decision Records for digital-card-service  
**Related Skills**: digital-card-service-domain-context, digital-card-service-api-context, digital-card-service-nfr-context
