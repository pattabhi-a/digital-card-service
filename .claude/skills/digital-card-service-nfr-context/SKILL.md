---
name: digital-card-service-nfr-context
description: >
  Non-functional requirements (NFRs) and SLA baselines for Digital Card Service.
  Establishes performance targets, scalability constraints, compliance requirements,
  and observability SLAs for credential issuance operations.
task_type: analysis
license: Haiintel
depends_on:
  - profiling-system-identity
  - capturing-nfr-baseline
  - assessing-design-level-nfrs
---

## Purpose

Establish domain-specific non-functional requirements and SLA baselines. This skill answers:

- **What performance targets?** — Card generation latency (p50, p95, p99), throughput (cards/min)
- **What scalability constraints?** — Thread pool sizing, connection pooling, horizontal pod scaling
- **What compliance obligations?** — GDPR, DPDP Act, audit trail requirements, key rotation frequency
- **What observability SLAs?** — Credential issuance tracing, encryption operation metrics, revocation propagation time

## Inputs

```json
{
  "system_identity": {
    "artifact_id": "digital-card-service",
    "endpoints": 5,
    "dependencies": 6,
    "event_topics": 5
  },
  "nfr_baseline": {
    "current_throughput": "30-50 cards/min",
    "current_p99_latency": "2000-5000ms (PDF generation blocking)",
    "test_coverage": "0 tests (baseline: 0%)",
    "deployment_readiness": "7.5/10 (K8s maturity)"
  },
  "operational_context": {
    "deployment_target": "Kubernetes multi-tenant",
    "user_base": "100M+ citizens (Aadhaar-scale)",
    "regulatory_regime": "DPDP Act (India), GDPR (if EU)",
    "uptime_requirement": "99.9% (credential issuance critical path)"
  }
}
```

## Output Contract

```json
{
  "skill": "digital-card-service-nfr-context",
  "nfr_baselines": {
    "performance": {
      "latency_targets": {
        "card_generation": {
          "p50_ms": 100,
          "p95_ms": 200,
          "p99_ms": 500,
          "p99_9_ms": 1000,
          "max_allowed_ms": 10000,
          "sla": "99% of cards generated within 500ms (p99)"
        },
        "credential_fetch": {
          "description": "Time to fetch encrypted credential from Credential Service",
          "p50_ms": 50,
          "p95_ms": 100,
          "p99_ms": 200,
          "bottleneck": "Network latency to Credential Service (typically <100ms in same region)"
        },
        "key_wrap": {
          "description": "Time to call KeyManager and wrap credential with encryption key",
          "p50_ms": 25,
          "p95_ms": 50,
          "p99_ms": 100,
          "constraint": "Key rotation (90-day cycle) must not block operation; auto-wrap with current key"
        },
        "pdf_generation": {
          "description": "Time to generate PDF card (CURRENTLY BLOCKING BOTTLENECK)",
          "p50_ms": 150,
          "p95_ms": 300,
          "p99_ms": 500,
          "p99_9_ms": 1200,
          "constraint": "This is the limiting factor for 300 cards/min throughput",
          "improvement_path": "Async PDF generation via external service (non-blocking queue)"
        },
        "card_packaging": {
          "description": "Time to encrypt PDF + metadata into card container",
          "p50_ms": 30,
          "p95_ms": 50,
          "p99_ms": 100
        },
        "datasource_publish": {
          "description": "Time to publish card to DataShare (non-blocking async)",
          "p50_ms": 100,
          "p95_ms": 300,
          "p99_ms": 1000,
          "note": "Non-critical path; eventual consistency acceptable (5s max delivery)"
        }
      },
      "throughput_targets": {
        "current_baseline": "30-50 cards/min",
        "target_state": "300-500 cards/min (10x growth)",
        "peak_load": "600 cards/min (for burst handling)",
        "sustained_load": "300 cards/min for 8+ hours daily",
        "calculation": "At p99 latency of 500ms per card: 1000ms / 500ms = 2 sequential requests. With 50 threads: 50 * 2 = 100 cards/sec = 6000 cards/min capacity (headroom provided)"
      },
      "resource_utilization": {
        "cpu_per_card": "~10-20ms (PDF generation: 80%, encryption: 10%, REST: 10%)",
        "memory_per_card": "~5-10MB (PDF buffer, credential blob, card package)",
        "database_connection_per_card": "~100-200ms (store to dcs_card table)",
        "network_bandwidth_per_card": "~50-100KB (credential blob encrypted, PDF ~50KB)"
      }
    },

    "scalability": {
      "horizontal_scaling": {
        "pod_replica_strategy": "Horizontal Pod Autoscaler (HPA)",
        "target_metric": "CPU utilization",
        "cpu_threshold": "70% average (scale up), 30% (scale down)",
        "min_replicas": 3,
        "max_replicas": 10,
        "scale_up_cooldown": "30s (quick response to spike)",
        "scale_down_cooldown": "300s (avoid thrashing)"
      },
      "thread_pool_sizing": {
        "current": "2-5 threads (Spring Tomcat default)",
        "required_at_300_cards_min": "50-100 threads (to achieve p99 < 500ms)",
        "rationale": "At 300 cards/min = 5 cards/sec; each card takes ~100-500ms (p50-p99). With 100 threads: 500ms latency / 1000ms per thread-second = 0.5 threads per request. Headroom for 10x spikes."
      },
      "database_connection_pool": {
        "current": "Not configured (default: limited)",
        "required": "20-30 connections (HikariCP recommended)",
        "calculation": "At 300 cards/min: ~5 cards/sec × 200ms DB latency = 1 connection per card × 2s = 2 connections for nominal. With HPA scaling to 10 pods: 10 pods × 3 connections per pod = 30 total.",
        "monitoring": "Alert if pool exhaustion >80%"
      },
      "event_processing_parallelism": {
        "websub_subscription_threads": "10-20 (per registration.completed topic)",
        "deadletter_queue": "Exponential backoff retry (max 3 retries, then manual intervention)",
        "ordering_guarantee": "None (WebSub eventual consistency); use idempotency keys (registration ID + timestamp)"
      }
    },

    "availability": {
      "uptime_sla": "99.9% (30 minutes downtime per month)",
      "rpo_target": "5 minutes (Recovery Point Objective — max data loss on disaster)",
      "rto_target": "15 minutes (Recovery Time Objective — time to restore service)",
      "deployment_strategy": {
        "zero_downtime_deployment": true,
        "rolling_update": "One pod at a time; old pods drained gracefully (30s timeout)",
        "health_check_readiness": "Delayed 30s (allow Spring Boot startup)",
        "liveness_restart": "Automatic pod restart on liveness probe failure (>3 consecutive failures)"
      },
      "failover_strategy": {
        "pod_failure": "Kubernetes ReplicaSet automatically restarts failed pod",
        "node_failure": "Pods evicted + rescheduled to healthy node (5-10s recovery)",
        "database_failure": "Connection pool detects DB unavailability; circuit breaker prevents cascade",
        "keymanger_outage": "Retry with exponential backoff; fail requests after timeout (5s); alert ops"
      }
    },

    "security": {
      "authentication": {
        "protocol": "OAuth2 + OIDC via Keycloak",
        "token_validation": "Signed JWT validation at every endpoint",
        "token_expiry": "15 minutes (typical); refresh token valid for 8 hours",
        "mfa_requirement": "Optional (per MOSIP policy; recommended for admin endpoints)"
      },
      "authorization": {
        "rbac_roles": [
          "MOSIP_ROLE_CARD_ISSUER (can POST /cards/generate, GET /cards/{id})",
          "MOSIP_ROLE_CARD_ADMIN (can POST /cards/{id}/revoke)",
          "MOSIP_ROLE_CARD_VIEWER (can GET /cards/{id} read-only)"
        ],
        "scope_requirement": "cards:read, cards:write, cards:admin (OIDC scopes)"
      },
      "encryption": {
        "credential_data": {
          "algorithm": "AES-256-GCM",
          "key_management": "External KeyManager service",
          "key_rotation_frequency": "90 days",
          "rekey_on_rotation": "Lazy (wrap with new key when card accessed)"
        },
        "tls": {
          "version": "TLS 1.3",
          "cipher_suites": "Only strong suites (ECDHE + AESGCM)",
          "certificate_validity": "90 days (auto-renewal via cert-manager)"
        }
      },
      "input_validation": {
        "requirement": "Validate all inputs (registrationId format, credentialBlob size, etc.)",
        "current_status": "MISSING (identified as security gap)",
        "improvement": "Add @Valid annotations + custom validators"
      },
      "error_handling": {
        "http_status_codes": {
          "current": "HTTP 200 for all responses (SECURITY GAP — leaks error info)",
          "target": "Use correct status codes (201 for creation, 400 for validation, 401 for auth, 404 for not found, 500 for server error)",
          "improvement": "Never return stack traces or SQL errors to client"
        },
        "logging": {
          "requirement": "Log security events (auth failures, authorization denials, revocations)",
          "constraint": "Never log credential blobs or encryption keys",
          "audit_trail": "Immutable log in dcs_card_audit table (who, when, what action)"
        }
      },
      "compliance": {
        "gdpr_requirements": [
          "Right to access (DSAR): Can user request export of their credential data?",
          "Right to erasure (right to be forgotten): Can user request credential deletion?",
          "Data processing agreement: All vendors must sign DPA",
          "Data breach notification: <72 hours if PII exposed"
        ],
        "dpdp_act_requirements_india": [
          "Purpose limitation: Credential only for identity verification",
          "Data minimization: Collect only biometric + demographic data needed",
          "Consent: Explicit consent from citizen before credential issuance",
          "Grievance redressal: Mechanism to address citizen complaints"
        ]
      }
    },

    "observability": {
      "logging": {
        "log_level": "INFO (application logs), DEBUG (development only)",
        "log_format": "JSON (structured logging for parsing by observability platform)",
        "fields_per_log_entry": [
          "timestamp",
          "level (INFO, WARN, ERROR)",
          "logger_name (e.g., io.mosip.digitalcard.service.CardService)",
          "request_id (trace ID for distributed tracing)",
          "user_id (authenticated user)",
          "event (what happened: card.generated, card.revoked, etc.)",
          "status (success, failure)",
          "latency_ms (time taken for operation)",
          "error (if status=failure)"
        ],
        "sensitive_data_masking": "Never log credential blobs, encryption keys, tokens"
      },
      "metrics": {
        "prometheus_exposition": "/actuator/prometheus (Spring Boot actuator)",
        "key_metrics": [
          {
            "name": "digital_card_generation_total",
            "type": "Counter",
            "labels": ["status (success, failure)", "endpoint (/cards/generate)"],
            "description": "Total cards generated (cumulative)"
          },
          {
            "name": "digital_card_generation_duration_seconds",
            "type": "Histogram",
            "buckets": "[0.01, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0]",
            "description": "Card generation latency (p50, p95, p99 computed from histogram)"
          },
          {
            "name": "jvm_memory_used_bytes",
            "type": "Gauge",
            "labels": ["area (heap, non-heap)"],
            "description": "JVM memory consumption"
          },
          {
            "name": "jvm_threads_live",
            "type": "Gauge",
            "description": "Active threads (monitor thread pool starvation)"
          },
          {
            "name": "db_connection_pool_active",
            "type": "Gauge",
            "description": "Active database connections (alert if >80% of pool)"
          },
          {
            "name": "websub_event_processing_duration_seconds",
            "type": "Histogram",
            "description": "Time to process WebSub event (registration.completed → card.issued)"
          },
          {
            "name": "keymanger_wrap_duration_seconds",
            "type": "Histogram",
            "description": "Time to wrap credential with KeyManager"
          }
        ]
      },
      "tracing": {
        "framework": "OpenTelemetry (not currently implemented; recommended improvement)",
        "span_context": [
          {
            "span_name": "POST /cards/generate",
            "child_spans": [
              "fetch_credential (call Credential Service)",
              "wrap_credential (call KeyManager)",
              "generate_pdf (call PDF service or library)",
              "package_card (encrypt + assemble)",
              "publish_datasource (call DataShare)"
            ]
          }
        ],
        "sla": "End-to-end tracing latency <50ms overhead (span recording + export)",
        "sampling": "100% for production (low overhead)"
      },
      "alerting": {
        "critical_alerts": [
          {
            "alert": "CardGenerationLatencyHigh",
            "condition": "p99 latency > 1000ms (5 minute window)",
            "action": "Investigate PDF generation bottleneck; scale up pods if CPU >70%"
          },
          {
            "alert": "WebSubEventLoss",
            "condition": "Dead-letter queue size > 100 (5 minute window)",
            "action": "Investigate registration.completed topic; check RabbitMQ broker health"
          },
          {
            "alert": "DatabaseConnectionPoolExhaustion",
            "condition": "Active connections > 25 of 30 (1 minute window)",
            "action": "Scale up replicas; increase connection pool size"
          },
          {
            "alert": "KeyManagerLatencyHigh",
            "condition": "wrap operation p99 > 200ms (5 minute window)",
            "action": "Check KeyManager SLA; possible key rotation in progress"
          },
          {
            "alert": "UptimeBelowSLA",
            "condition": "Monthly uptime < 99.9% (rolling 30-day calculation)",
            "action": "Post-mortem; identify failures; implement mitigation"
          }
        ]
      }
    },

    "maintainability": {
      "code_quality_targets": {
        "test_coverage": "Target: 80% (unit + integration)",
        "current": "0% (baseline)",
        "path": "Add tests incrementally (critical paths first: card generation, revocation)"
      },
      "documentation": {
        "api_documentation": "OpenAPI / Swagger spec (auto-generated from Spring annotations)",
        "architecture_documentation": "C4 context + component diagrams (see MODERNIZATION_FRAMEWORK.md)",
        "operational_runbooks": "Troubleshooting guides for common issues (high latency, event loss, database connection pool exhaustion)"
      },
      "dependency_management": {
        "framework_versions": "Keep Spring Boot within LTS track (3.2.x until 3.3 LTS released)",
        "security_patches": "Update weekly (automated via Dependabot or similar)",
        "major_version_upgrades": "Plan quarterly (test thoroughly before production rollout)"
      }
    }
  },

  "sla_summary": {
    "uptime": "99.9% (30 min/month downtime)",
    "latency_p99": "<500ms per card generation",
    "throughput": "300-500 cards/min sustained; 600 cards/min peak",
    "revocation_propagation": "<5 seconds (verifiers must reflect revocation within SLA)",
    "encryption_overhead": "<10% CPU (encryption not bottleneck)",
    "compliance_audit_trail": "100% of credential issuance/revocation events logged immutably"
  }
}
```

## Usage in S01

This skill is **triggered** when `capturing-nfr-baseline` or `assessing-design-level-nfrs` identifies performance + compliance requirements. Output feeds:

1. `synthesizing-drivers` (S01) — Domain-specific drivers: throughput 10x growth, encryption compliance, zero-downtime deployment
2. `assessing-design-level-nfrs` (S01) — Baseline maturity assessment against domain targets
3. `modeling-target-architecture` (S02) — NFR targets inform architecture decisions (async PDF, caching, thread pooling)
4. `applying-nfr-performance-lens` (S01) — Performance patterns: bottleneck analysis, scaling strategy

## Acceptance Criteria

- ✅ Performance targets grounded in evidence (e.g., PDF generation p99 from actual code analysis)
- ✅ Scalability math documented (thread count, connection pool sizing justified)
- ✅ SLA achievable with proposed improvements (not aspirational)
- ✅ Compliance requirements mapped to laws (GDPR Article X, DPDP Act Section Y)
- ✅ Observability SLAs realistic (tracing overhead <50ms, not 1-2x latency increase)

---

**License**: Haiintel  
**Status**: NFR context for digital-card-service  
**Related Skills**: digital-card-service-domain-context, digital-card-service-api-context, digital-card-service-architecture-decisions
