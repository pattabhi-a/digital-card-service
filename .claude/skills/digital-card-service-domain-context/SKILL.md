---
name: digital-card-service-domain-context
description: >
  Domain context for MOSIP Digital Card Service: ecosystem position, credential lifecycle,
  integration points, and business drivers. Establishes MOSIP identity platform context
  for downstream lens triggering and driver synthesis.
task_type: analysis
license: Haiintel
depends_on: 
  - profiling-system-identity
  - extracting-controller-endpoints
  - digesting-build-dependencies
---

## Purpose

Establish the Digital Card Service's position within the MOSIP identity platform ecosystem, define the credential lifecycle, and document integration touchpoints. This skill answers:

- **What role in MOSIP?** — Post-registration credential packaging service
- **What lifecycle stage?** — Issuance (generation, encryption, packaging, delivery)
- **Who integrates?** — Registration Processor, Credential Service, KeyManager, DataShare, PDF generators
- **What constraints?** — PII protection, credential non-repudiation, audit trails, regulatory compliance

## Inputs

```json
{
  "system_identity": {
    "artifact_id": "digital-card-service",
    "group_id": "io.mosip",
    "version": "1.2.0",
    "endpoints": [
      { "method": "POST", "path": "/cards/generate" },
      { "method": "GET", "path": "/cards/{id}" },
      { "method": "GET", "path": "/cards/{id}/download" },
      { "method": "POST", "path": "/cards/{id}/verify" },
      { "method": "POST", "path": "/cards/{id}/revoke" }
    ]
  },
  "dependencies": {
    "imports": [
      "io.mosip.registrationprocessor:registration-processor-core",
      "io.mosip.credentialservice:credential-service-adapter",
      "io.mosip.keymanager:key-manager-api",
      "io.mosip.datashare:data-share-service"
    ]
  }
}
```

## Output Contract

```json
{
  "skill": "digital-card-service-domain-context",
  "domain_context": {
    "platform": "MOSIP",
    "platform_version": "1.2.x",
    "service_name": "Digital Card Service",
    "service_role": "Credential Packaging & Issuance",
    "position_in_pipeline": "Post-registration flow",
    
    "business_context": {
      "business_drivers": [
        {
          "driver": "Credential Issuance Velocity",
          "description": "Scale digital card generation from 30-50 cards/min to 300-500 cards/min",
          "impact": "10x growth trajectory for identity lifecycle",
          "regulatory_backing": "MOSIP roadmap, government identity initiatives"
        },
        {
          "driver": "PII Protection",
          "description": "Mandatory encryption of credential data in transit and at rest",
          "impact": "Non-negotiable for citizen trust, legal compliance",
          "regulatory_backing": "GDPR (EU), DPDP Act (India), equivalent data residency laws"
        },
        {
          "driver": "Ecosystem Reliability",
          "description": "Integrate seamlessly with 6+ MOSIP services (eventual consistency model)",
          "impact": "Credential issuance not blocked by external service outages",
          "regulatory_backing": "MOSIP SLA contracts, service-level reliability expectations"
        }
      ],
      "stakeholders": [
        {
          "role": "Citizen",
          "interest": "Fast credential issuance, secure storage, verified authenticity"
        },
        {
          "role": "Registration Officer",
          "interest": "Transparent issuance workflow, retry mechanisms, audit trails"
        },
        {
          "role": "MOSIP Operator",
          "interest": "Scalability, multi-tenancy, compliance reporting, disaster recovery"
        },
        {
          "role": "Credential Verifier (Police, Banks)",
          "interest": "Tamper-proof credentials, revocation transparency, non-repudiation"
        }
      ]
    },

    "credential_lifecycle": {
      "stage_1_initiation": {
        "trigger": "Registration Processor → credential.created event (WebSub)",
        "action": "Listen for new registration completion",
        "actor": "Digital Card Service",
        "latency_target_ms": 500
      },
      "stage_2_credential_fetch": {
        "trigger": "Credential Service → credential.ready event",
        "action": "Fetch encrypted credential blob + metadata",
        "actor": "Digital Card Service calls Credential Service API",
        "latency_target_ms": 100
      },
      "stage_3_key_wrap": {
        "trigger": "Credential fetched; encryption key needed",
        "action": "Call KeyManager to wrap credential with current key version",
        "actor": "Digital Card Service calls KeyManager API",
        "latency_target_ms": 50,
        "constraint": "Key rotation (90-day mandate) must not block card generation"
      },
      "stage_4_pdf_generation": {
        "trigger": "Wrapped credential ready",
        "action": "Generate PDF card layout (currently BLOCKING operation)",
        "actor": "Digital Card Service or external PDF service",
        "latency_target_ms": 200,
        "constraint": "Blocking PDF generation is primary performance bottleneck"
      },
      "stage_5_packaging": {
        "trigger": "PDF + metadata ready",
        "action": "Package into encrypted container (QR code + metadata)",
        "actor": "Digital Card Service",
        "latency_target_ms": 50
      },
      "stage_6_delivery": {
        "trigger": "Card packaged",
        "action": "Store in DataShare service for citizen download",
        "actor": "Digital Card Service calls DataShare API",
        "latency_target_ms": 100,
        "constraint": "Non-blocking; eventual consistency acceptable"
      },
      "stage_7_validation": {
        "trigger": "Card delivered to citizen",
        "action": "Citizen scans QR; credential verification (off-device or with verifier)",
        "actor": "Citizen device + Verifier service",
        "latency_sla": "<1s for verification response",
        "constraint": "Non-repudiation required; signature validation must succeed"
      },
      "stage_8_revocation": {
        "trigger": "Citizen requests revocation OR registration cancelled",
        "action": "Mark credential as revoked; broadcast revocation.triggered event",
        "actor": "Digital Card Service",
        "latency_target_ms": 100,
        "constraint": "Revocation latency <1s; verifiers must reflect revocation within 5 seconds"
      }
    },

    "integration_points": [
      {
        "service": "Registration Processor",
        "protocol": "WebSub",
        "topic": "registration.completed",
        "direction": "inbound",
        "frequency": "high (peak: 300 events/min)",
        "sla": "Must not lose event notifications; queue-backed",
        "failure_mode": "Dead-letter queue; retry with exponential backoff"
      },
      {
        "service": "Credential Service",
        "protocol": "REST/HTTP",
        "endpoint": "GET /credentials/{registrationId}",
        "direction": "outbound",
        "frequency": "per registration (correlated with registration.completed)",
        "sla": "99.9% availability; timeout 5s",
        "failure_mode": "Retry 3x; escalate to monitoring if persistent"
      },
      {
        "service": "KeyManager Service",
        "protocol": "REST/HTTP",
        "endpoint": "POST /keys/wrap",
        "direction": "outbound",
        "frequency": "per credential (8 bytes avg, peak 2400 wraps/min for 300 cards/min)",
        "sla": "99.99% availability; timeout 1s",
        "failure_mode": "Key rotation transparent to caller; old keys auto-wrapped with current version"
      },
      {
        "service": "PDF Generation Service",
        "protocol": "REST/HTTP or sync library",
        "endpoint": "POST /generate-pdf",
        "direction": "outbound",
        "frequency": "per credential",
        "sla": "p99 <200ms; p99.9 <500ms",
        "failure_mode": "Async PDF generation (recommended future enhancement); sync currently blocks"
      },
      {
        "service": "DataShare Service",
        "protocol": "REST/HTTP",
        "endpoint": "POST /shares/{credentialId}",
        "direction": "outbound",
        "frequency": "per credential (non-blocking)",
        "sla": "eventual consistency (5s max); 99.5% availability",
        "failure_mode": "Async retry queue; DataShare handles eventual delivery"
      },
      {
        "service": "Audit Logging Service",
        "protocol": "Event (WebSub or Kafka)",
        "topic": "card.issued, card.revoked, card.verified",
        "direction": "outbound",
        "frequency": "per issuance/revocation",
        "sla": "fire-and-forget; no retry required",
        "failure_mode": "Non-critical; compliance dashboards may show slight lag (<1min)"
      }
    ],

    "regulatory_context": {
      "jurisdictions": [
        {
          "region": "India",
          "law": "Digital Personal Data Protection Act (DPDP), 2023",
          "requirements": [
            "Explicit consent for PII collection/processing",
            "Data residency (no cross-border transfer without consent)",
            "Right to erasure (credential deletion capability)",
            "Purpose limitation (credential use only for identity verification)"
          ]
        },
        {
          "region": "EU (if applicable)",
          "law": "General Data Protection Regulation (GDPR)",
          "requirements": [
            "Encryption at rest + in transit (mandatory)",
            "Data Processing Agreement (DPA) with all vendors",
            "Right to access, rectification, erasure (DSAR response <30 days)",
            "Data breach notification (<72 hours)"
          ]
        }
      ],
      "credentialing_standards": [
        {
          "standard": "W3C Verifiable Credentials Data Model",
          "requirement": "Credential structure must support portable verification (JSON-LD + linked data proofs)",
          "impact": "Allows off-device verification without MOSIP back-end"
        },
        {
          "standard": "OIDC Self-Issued Identity Provider (if mobile OIDC wallet planned)",
          "requirement": "Credential claims must map to OIDC standard claims (sub, aud, iss, exp, iat)",
          "impact": "Interoperability with third-party OIDC verifiers"
        }
      ]
    },

    "ecosystem_maturity": {
      "mosip_version": "1.2.0 (typical)",
      "digital_card_service_stability": "Production (1.2.0+)",
      "external_service_dependencies": {
        "registration_processor": "stable",
        "credential_service": "stable",
        "key_manager": "stable (3-year track record)",
        "data_share": "stable"
      },
      "known_constraints": [
        "WebSub event ordering not guaranteed (eventual consistency required)",
        "Key rotation does not trigger automatic credential re-issuance (citizen must re-download)",
        "PDF generation is synchronous bottleneck (no async path currently)",
        "No distributed tracing across MOSIP services (added via Jaeger in S01 findings)"
      ]
    }
  }
}
```

## Usage in S01

This skill is **conditionally triggered** when `assessing-evidence-coverage` detects:
- Endpoint patterns matching `(card|credential)` (e.g., `/cards/generate`)
- Dependencies matching `io.mosip` (e.g., `credential-service-adapter`, `key-manager-api`)
- WebSub or Kafka usage (event-driven architecture)

**Output feeds**:
1. `synthesizing-drivers` — Adds business drivers (credential velocity, PII protection, ecosystem reliability)
2. Lens triggering rules — Confirms security, distributed-systems, cloud-native, observability lenses
3. `hypothesizing-modernization-strategy` — Informs strategy hypothesis with domain-specific constraints

## Acceptance Criteria

- ✅ Output correctly identifies MOSIP position + credential lifecycle stages
- ✅ Integration points accurately map to actual dependencies (no hallucinated services)
- ✅ Latency targets grounded in evidence (e.g., PDF generation 200ms from code inspection)
- ✅ Regulatory requirements match declared jurisdiction (user-provided or inferred from data residency config)
- ✅ Ecosystem maturity aligned with version numbers (not generic "stable"/"immature")

---

**License**: Haiintel  
**Status**: Domain context skill for digital-card-service  
**Related Skills**: digital-card-service-api-context, digital-card-service-architecture-decisions, digital-card-service-nfr-context
