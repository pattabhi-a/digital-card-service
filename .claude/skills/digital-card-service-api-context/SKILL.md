---
name: digital-card-service-api-context
description: >
  API surface documentation for Digital Card Service: REST endpoints, WebSub subscriptions,
  data models, and throughput contracts. Extracted from codebase analysis and integration
  specifications.
task_type: analysis
license: Haiintel
depends_on:
  - extracting-controller-endpoints
  - digesting-build-dependencies
  - narrating-codebase-patterns
---

## Purpose

Document the Digital Card Service's external API surface (REST + events), data models, and throughput commitments. This skill answers:

- **What REST endpoints exist?** — Methods, paths, request/response models, authentication
- **What events does it subscribe to?** — WebSub topics, payload schemas, processing SLAs
- **What data models underlie the APIs?** — Card, Credential, EncryptionMetadata, EventPayload
- **What throughput contracts?** — Current (30-50 cards/min), target (300-500 cards/min), bottlenecks

## Inputs

```json
{
  "endpoints": [
    {
      "method": "POST",
      "path": "/cards/generate",
      "controller": "CardController.generate()",
      "auth": "Bearer token (Keycloak)",
      "consumes": "application/json",
      "produces": "application/json"
    },
    {
      "method": "GET",
      "path": "/cards/{id}",
      "controller": "CardController.getCard()",
      "auth": "Bearer token",
      "produces": "application/json"
    }
  ],
  "dependencies": {
    "event_framework": "Spring Cloud Stream + RabbitMQ (WebSub)",
    "data_models": [
      "io.mosip.digitalcard.entity.Card",
      "io.mosip.digitalcard.entity.Credential",
      "io.mosip.digitalcard.dto.GenerateCardRequest",
      "io.mosip.digitalcard.dto.CardResponse"
    ]
  },
  "code_metrics": {
    "endpoint_count": 5,
    "controller_count": 1,
    "service_count": 3,
    "dto_count": 23,
    "exception_count": 10
  }
}
```

## Output Contract

```json
{
  "skill": "digital-card-service-api-context",
  "api_surface": {
    "rest_endpoints": [
      {
        "id": "EP-001",
        "method": "POST",
        "path": "/cards/generate",
        "summary": "Generate a new digital card",
        "description": "Accepts a credential blob and metadata; returns a generated card with QR code and encrypted payload",
        "authentication": "Bearer token (Keycloak OIDC)",
        "authorization": "MOSIP_ROLE_CARD_ISSUER",
        "request_model": {
          "type": "io.mosip.digitalcard.dto.GenerateCardRequest",
          "fields": [
            { "name": "registrationId", "type": "String", "required": true, "description": "Unique registration ID from Registration Processor" },
            { "name": "credentialBlob", "type": "byte[]", "required": true, "description": "Encrypted credential blob from Credential Service" },
            { "name": "credentialMetadata", "type": "CredentialMetadata", "required": true, "description": "Credential attributes (issuer, issuance_date, expiry)" },
            { "name": "encryptionKeyVersion", "type": "String", "required": true, "description": "Current key version from KeyManager" }
          ]
        },
        "response_model": {
          "type": "io.mosip.digitalcard.dto.CardResponse",
          "fields": [
            { "name": "cardId", "type": "String", "description": "Unique card identifier (UUID)" },
            { "name": "qrCode", "type": "String", "description": "Encoded QR payload (Base64)" },
            { "name": "cardPackage", "type": "byte[]", "description": "Encrypted card container (PDF + metadata)" },
            { "name": "sharingUrl", "type": "URL", "description": "DataShare URL for citizen download" },
            { "name": "expiryDate", "type": "LocalDateTime", "description": "Card expiry (inherited from credential)" }
          ]
        },
        "status_codes": [
          { "code": 201, "description": "Card created successfully; card details in response" },
          { "code": 400, "description": "Invalid request (missing fields, bad payload format)" },
          { "code": 401, "description": "Unauthorized (invalid token or revoked privileges)" },
          { "code": 500, "description": "Internal error (Credential Service call failed, PDF generation timeout)" }
        ],
        "performance_target": {
          "p50_ms": 100,
          "p95_ms": 200,
          "p99_ms": 500,
          "timeout_ms": 10000
        }
      },
      {
        "id": "EP-002",
        "method": "GET",
        "path": "/cards/{id}",
        "summary": "Retrieve card metadata",
        "description": "Get card details (not encrypted payload) for status checking",
        "authentication": "Bearer token (Keycloak)",
        "request_model": {
          "path_params": [
            { "name": "id", "type": "String", "description": "Card ID (UUID)" }
          ]
        },
        "response_model": {
          "type": "io.mosip.digitalcard.dto.CardResponse",
          "subset": ["cardId", "expiryDate", "status", "issuanceDate"]
        },
        "status_codes": [
          { "code": 200, "description": "Card found" },
          { "code": 404, "description": "Card not found" },
          { "code": 401, "description": "Unauthorized" }
        ],
        "performance_target": {
          "p50_ms": 10,
          "p95_ms": 20,
          "p99_ms": 50,
          "timeout_ms": 5000
        }
      },
      {
        "id": "EP-003",
        "method": "GET",
        "path": "/cards/{id}/download",
        "summary": "Download card (encrypted payload)",
        "description": "Stream the encrypted PDF card for citizen download (via DataShare redirect)",
        "authentication": "Bearer token",
        "response_model": {
          "type": "application/octet-stream",
          "description": "Encrypted PDF card binary"
        },
        "status_codes": [
          { "code": 200, "description": "Card binary stream" },
          { "code": 404, "description": "Card not found" },
          { "code": 401, "description": "Unauthorized" }
        ],
        "performance_target": {
          "p50_ms": 50,
          "p95_ms": 100,
          "p99_ms": 200
        }
      },
      {
        "id": "EP-004",
        "method": "POST",
        "path": "/cards/{id}/verify",
        "summary": "Verify card authenticity",
        "description": "Validate card signature and metadata; used by offline verifiers",
        "authentication": "None (public verification endpoint)",
        "request_model": {
          "path_params": [
            { "name": "id", "type": "String", "description": "Card ID" }
          ],
          "body": {
            "type": "io.mosip.digitalcard.dto.VerifyCardRequest",
            "fields": [
              { "name": "qrPayload", "type": "String", "description": "Scanned QR content" },
              { "name": "signature", "type": "String", "description": "Digital signature (Base64)" }
            ]
          }
        },
        "response_model": {
          "type": "io.mosip.digitalcard.dto.VerificationResult",
          "fields": [
            { "name": "isValid", "type": "boolean", "description": "Signature validation result" },
            { "name": "issuerCertificate", "type": "X.509", "description": "Certificate chain for verification" },
            { "name": "isRevoked", "type": "boolean", "description": "Revocation status" }
          ]
        },
        "status_codes": [
          { "code": 200, "description": "Verification result (valid or invalid)" },
          { "code": 404, "description": "Card/signature not found" },
          { "code": 400, "description": "Malformed QR or signature" }
        ],
        "performance_target": {
          "p50_ms": 50,
          "p95_ms": 100,
          "p99_ms": 200
        }
      },
      {
        "id": "EP-005",
        "method": "POST",
        "path": "/cards/{id}/revoke",
        "summary": "Revoke a card",
        "description": "Mark card as revoked; broadcast revocation event to verifiers",
        "authentication": "Bearer token (MOSIP_ROLE_CARD_ADMIN)",
        "authorization": "MOSIP_ROLE_CARD_ADMIN",
        "request_model": {
          "path_params": [
            { "name": "id", "type": "String", "description": "Card ID" }
          ],
          "body": {
            "type": "io.mosip.digitalcard.dto.RevokeCardRequest",
            "fields": [
              { "name": "reason", "type": "String", "enum": ["CITIZEN_REQUEST", "REGISTRATION_CANCELLED", "CREDENTIAL_COMPROMISED"], "description": "Revocation reason" },
              { "name": "effectiveDate", "type": "LocalDateTime", "description": "When revocation takes effect" }
            ]
          }
        },
        "response_model": {
          "type": "io.mosip.digitalcard.dto.RevokeCardResponse",
          "fields": [
            { "name": "cardId", "type": "String" },
            { "name": "revokedAt", "type": "LocalDateTime" },
            { "name": "reason", "type": "String" }
          ]
        },
        "status_codes": [
          { "code": 200, "description": "Card revoked; event broadcasted" },
          { "code": 404, "description": "Card not found" },
          { "code": 401, "description": "Unauthorized" },
          { "code": 409, "description": "Card already revoked" }
        ],
        "performance_target": {
          "p50_ms": 100,
          "p95_ms": 200,
          "p99_ms": 500
        }
      }
    ],

    "event_subscriptions": [
      {
        "id": "EVT-001",
        "topic": "registration.completed",
        "source": "Registration Processor",
        "protocol": "WebSub",
        "direction": "inbound",
        "payload_schema": {
          "type": "io.mosip.common.event.RegistrationCompletedEvent",
          "fields": [
            { "name": "registrationId", "type": "String" },
            { "name": "timestamp", "type": "Instant" },
            { "name": "status", "type": "String", "enum": ["APPROVED", "REJECTED"] },
            { "name": "registrantBio", "type": "Object", "description": "Biometric + demographic data" }
          ]
        },
        "processing_sla": {
          "p50_ms": 500,
          "p95_ms": 2000,
          "p99_ms": 5000,
          "max_latency_ms": 10000
        },
        "failure_handling": "Dead-letter queue with retry (exponential backoff, max 3 retries)",
        "frequency": "peak: 300 events/min (at 300 cards/min throughput)"
      },
      {
        "id": "EVT-002",
        "topic": "credential.ready",
        "source": "Credential Service",
        "protocol": "WebSub",
        "direction": "inbound",
        "payload_schema": {
          "fields": [
            { "name": "registrationId", "type": "String" },
            { "name": "credentialBlob", "type": "byte[]", "description": "Encrypted credential (CBOR)" },
            { "name": "keyVersion", "type": "String", "description": "Key version used for encryption" },
            { "name": "timestamp", "type": "Instant" }
          ]
        },
        "processing_sla": {
          "p50_ms": 100,
          "p95_ms": 500,
          "p99_ms": 1000
        }
      },
      {
        "id": "EVT-003",
        "topic": "card.issued",
        "source": "Digital Card Service",
        "protocol": "WebSub or Kafka",
        "direction": "outbound",
        "payload_schema": {
          "fields": [
            { "name": "cardId", "type": "String" },
            { "name": "registrationId", "type": "String" },
            { "name": "issuanceTime", "type": "Instant" },
            { "name": "expiryDate", "type": "LocalDate" }
          ]
        },
        "subscribers": ["Audit Logging Service", "Analytics Service", "Notification Service"],
        "delivery_guarantee": "At-least-once (eventual consistency)"
      },
      {
        "id": "EVT-004",
        "topic": "card.revoked",
        "source": "Digital Card Service",
        "protocol": "WebSub or Kafka",
        "direction": "outbound",
        "payload_schema": {
          "fields": [
            { "name": "cardId", "type": "String" },
            { "name": "revocationTime", "type": "Instant" },
            { "name": "reason", "type": "String" }
          ]
        },
        "subscribers": ["Revocation Service", "Audit Logging Service", "Verifier Broadcast"],
        "delivery_guarantee": "At-least-once; verifiers must reflect revocation within 5 seconds"
      },
      {
        "id": "EVT-005",
        "topic": "revocation.triggered",
        "source": "Revocation Service or Citizen Request",
        "protocol": "WebSub",
        "direction": "inbound",
        "payload_schema": {
          "fields": [
            { "name": "credentialId", "type": "String" },
            { "name": "reason", "type": "String" },
            { "name": "timestamp", "type": "Instant" }
          ]
        },
        "processing_sla": {
          "p50_ms": 100,
          "max_latency_ms": 1000,
          "requirement": "Must not lose revocation event"
        }
      }
    ],

    "data_models": {
      "core_entities": [
        {
          "class": "io.mosip.digitalcard.entity.Card",
          "table": "dcs_card",
          "fields": [
            { "name": "id", "type": "UUID", "pk": true, "description": "Card ID" },
            { "name": "registrationId", "type": "String", "fk": "registration.id", "description": "Reference to registration" },
            { "name": "credentialBlob", "type": "byte[]", "description": "Encrypted credential (CBOR or JSON-LD)" },
            { "name": "cardPackage", "type": "byte[]", "description": "Encrypted PDF + metadata container" },
            { "name": "qrPayload", "type": "String", "description": "QR code content (Base64)" },
            { "name": "signature", "type": "String", "description": "Digital signature (Base64, RSA-2048)" },
            { "name": "issuanceDate", "type": "LocalDateTime", "description": "When card was issued" },
            { "name": "expiryDate", "type": "LocalDateTime", "description": "When credential expires" },
            { "name": "revocationDate", "type": "LocalDateTime", "nullable": true, "description": "When card was revoked (null if active)" },
            { "name": "status", "type": "Enum", "values": ["ACTIVE", "REVOKED", "EXPIRED"], "description": "Current status" },
            { "name": "encryptionKeyVersion", "type": "String", "description": "Key version used to wrap credential" },
            { "name": "createdAt", "type": "LocalDateTime", "description": "Record creation timestamp" },
            { "name": "updatedAt", "type": "LocalDateTime", "description": "Last update timestamp" }
          ],
          "indexes": [
            { "name": "idx_card_registration", "columns": ["registrationId"] },
            { "name": "idx_card_status", "columns": ["status"] },
            { "name": "idx_card_expiry", "columns": ["expiryDate"] }
          ]
        },
        {
          "class": "io.mosip.digitalcard.entity.Credential",
          "table": "dcs_credential",
          "fields": [
            { "name": "id", "type": "UUID", "pk": true },
            { "name": "cardId", "type": "UUID", "fk": "dcs_card.id" },
            { "name": "credentialBlob", "type": "byte[]", "description": "Original encrypted blob from Credential Service" },
            { "name": "metadata", "type": "JSON", "description": "Credential attributes (issuer, claims, etc.)" },
            { "name": "issuanceDate", "type": "LocalDateTime" },
            { "name": "expiryDate", "type": "LocalDateTime" }
          ]
        }
      ],
      "encryption_metadata": {
        "algorithm": "AES-256-GCM",
        "key_management": "External KeyManager service",
        "key_rotation_frequency": "90 days",
        "old_keys": "Retained for decryption of historical cards; auto-wrapped on re-encryption"
      }
    },

    "throughput_contracts": {
      "current_baseline": {
        "cards_per_minute": "30-50",
        "bottleneck": "Synchronous PDF generation (p99: 500ms per card)",
        "thread_pool_size": "2-5 threads",
        "concurrent_requests_served": "5-10"
      },
      "target_state": {
        "cards_per_minute": "300-500 (10x growth)",
        "target_p95_latency_ms": 200,
        "target_p99_latency_ms": 500,
        "required_improvements": [
          "Async PDF generation (non-blocking)",
          "Distributed cache for credential metadata (Redis)",
          "Thread pool expansion (50-100 threads)",
          "Database connection pooling (20-30 connections)",
          "Distributed tracing for bottleneck identification"
        ]
      }
    }
  }
}
```

## Usage in S01

This skill is **triggered** whenever `extracting-controller-endpoints` finds card/credential endpoints. Output feeds:

1. `modeling-target-architecture` (S02) — Informs API design decisions (async vs. sync, caching strategy)
2. `applying-distributed-systems-lens` — WebSub subscription ordering + eventual consistency patterns
3. `applying-nfr-performance-lens` — Throughput targets, latency SLAs, bottleneck identification

## Acceptance Criteria

- ✅ Endpoints match actual `@RequestMapping` annotations (no hallucination)
- ✅ Event topics match actual WebSub topic subscriptions in code
- ✅ Data models match actual JPA entity definitions + database schema
- ✅ Throughput contracts grounded in evidence (e.g., PDF generation p99 from performance analysis)
- ✅ Performance targets achievable within identified resource constraints

---

**License**: Haiintel  
**Status**: API context skill for digital-card-service  
**Related Skills**: digital-card-service-domain-context, digital-card-service-nfr-context
