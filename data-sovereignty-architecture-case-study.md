# Data Sovereignty Architecture Case Study

## Executive Summary

This case study presents the design and implementation of a data sovereignty architecture that can enable a global SaaS platform to serve customers across the European Union, United States, and Asia-Pacific regions while complying with GDPR, CCPA, and emerging APAC data localization laws. The architecture can achice regulatory compliance without sacrificing product functionality or engineering velocity.

---

## Problem Statement

The organization's SaaS platform operates from a single AWS us-east-1 region. As the customer base expands globally, enterprise customers in the EU and APAC regions require contractual guarantees that their data would never leave defined geographic boundaries. Regulators in multiple jurisdictions enforce data residency obligations with significant financial penalties for non-compliance.

**Compliance requirements driving the architecture:**
- **GDPR (EU):** Personal data of EU residents must not be transferred outside the EEA without adequate safeguards
- **CCPA (California):** Consumer data subject rights (access, deletion, opt-out) must be fulfilled within defined timeframes
- **Japan APPI / India DPDPA:** Emerging data localization requirements restricting cross-border data transfer
- **Financial Services (FSI) customers:** Contractual data residency in-country as a vendor procurement condition

---

## Architectural Approach: Federated Data Planes

The platform is refactored from a single-region monolith into a federated data plane model with multiple isolated regional data planes, each sovereign and independently operable, governed by a global control plane.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Global Control Plane                         │
│   ┌──────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│   │  Identity│  │ Config Mgmt  │  │ Compliance Audit Engine │  │
│   │  (IdP)   │  │ (GitOps)     │  │ (Policy Enforcement)    │  │
│   └──────────┘  └──────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
│  EU Data     │    │  US Data     │    │  APAC Data       │
│  Plane       │    │  Plane       │    │  Plane           │
│  (eu-west-1) │    │  (us-east-1) │    │  (ap-northeast-1)│
│              │    │              │    │                  │
│  ┌────────┐  │    │  ┌────────┐  │    │  ┌────────┐      │
│  │Customer│  │    │  │Customer│  │    │  │Customer│      │
│  │Data DB │  │    │  │Data DB │  │    │  │Data DB │      │
│  └────────┘  │    │  └────────┘  │    │  └────────┘      │
│  ┌────────┐  │    │  ┌────────┐  │    │  ┌────────┐      │
│  │ App    │  │    │  │ App    │  │    │  │ App    │      │
│  │Services│  │    │  │Services│  │    │  │Services│      │
│  └────────┘  │    │  └────────┘  │    │  └────────┘      │
└──────────────┘    └──────────────┘    └──────────────────┘
```

### Design Principles

1. **Data never crosses regional boundaries** — all personal data is created, stored, processed, and deleted within its designated regional data plane
2. **Control plane carries no PII** — the global control plane handles identity, configuration, and operational metadata only; zero customer data transits through it
3. **Tenant-to-region binding** — each tenant is bound to a region at provisioning time; this binding is immutable without explicit tenant consent and data migration
4. **Regional autonomy** — each data plane operates independently and degrades gracefully if the control plane is unreachable

---

## Key Design Decisions

### 1. Tenant Routing at the Edge

Cloudflare Workers handle global DNS routing. Each request is inspected for tenant identity (JWT claim or subdomain), and the worker routes to the appropriate regional origin without exposing tenant-to-region mappings to the client.

```
Client Request
      │
      ▼
Cloudflare Worker
  ├── Extract tenant_id from JWT
  ├── Lookup region from KV store (no PII stored in KV — region mapping only)
  └── Route to regional API gateway
```

### 2. Encryption Key Residency

Encryption keys are managed per-region using AWS KMS with key material never leaving the region:

- **EU:** AWS KMS (eu-west-1) with CMK — optional customer-managed keys (BYOK) for FSI customers
- **US:** AWS KMS (us-east-1) with CMK
- **APAC:** AWS KMS (ap-northeast-1) with CMK

All data at rest is encrypted using AES-256 with region-local keys. Cross-region key replication is explicitly disabled via KMS key policy.

### 3. Data Subject Rights Automation

GDPR/CCPA data subject requests (DSRs) are handled by a purpose-built DSR Orchestrator service deployed in each regional data plane:

```
DSR Request (e.g. Right to Erasure)
      │
      ▼
DSR Orchestrator (regional)
  ├── Enumerate all data stores containing tenant PII (catalog-driven)
  ├── Issue deletion commands to each store
  ├── Verify deletion completion
  ├── Generate signed attestation record
  └── Respond within regulatory deadline (30 days GDPR / 45 days CCPA)
```

**Data stores covered:**
- PostgreSQL (primary records)
- Elasticsearch (search indexes)
- S3 (file attachments, exports)
- Redis (session cache — TTL-based auto-expiry)
- Kafka (event log — compacted topic tombstoning)
- Backup snapshots (deletion flagged; purged on next rotation cycle)

### 4. Cross-Region Analytics Without PII Transfer

Business analytics requires aggregated data across all regions. The solution uses differential privacy and aggregation-only pipelines.

- Regional data planes produce pre-aggregated, anonymized metrics (no row-level PII)
- Aggregates are pushed to a global analytics warehouse (Snowflake) in the US region
- Column-level access controls prevent any PII reconstruction from analytics queries
- Data minimization review conducted quarterly by the DPO

---

## Compliance Posture

### GDPR Controls

| Control | Implementation |
|---------|---------------|
| Data residency | Regional data planes; no cross-border transfer for customer PII |
| Consent management | Consent collected at onboarding; stored per-tenant in regional DB |
| Right to Access | DSR Orchestrator generates PII export in JSON format within 30 days |
| Right to Erasure | DSR Orchestrator cascading deletion with signed attestation |
| Data Breach Notification | PagerDuty + automated DPA notification workflow (72-hour SLA) |
| DPA agreements | Standard Contractual Clauses (SCCs) for any sub-processor data flows |

### Security Controls

- Encryption at rest: AES-256, region-local KMS CMKs
- Encryption in transit: TLS 1.3 minimum; mTLS for service-to-service
- Network isolation: VPC per region, no cross-region VPC peering for data plane traffic
- Audit logging: CloudTrail + regional log archives with WORM storage (S3 Object Lock)
- Pen testing: Annual third-party penetration test per region

---

## Planned Results & Impact

| Metric | Outcome |
|--------|---------|
| Enterprise deals unblocked | 12 FSI and public sector deals (est. $8.4M ARR) |
| GDPR DSR compliance rate | 100% fulfilled within 30-day deadline |
| Regulatory findings | 0 findings across EU and APAC regulatory reviews |
| Cross-region data leakage incidents | 0 since architecture launch |
| Time to provision new regional data plane | 4 hours (fully automated via Terraform) |

---

## Lessons Learned



---

## References

- [GDPR Official Text](https://gdpr-info.eu/)
- [CCPA Official Resource](https://oag.ca.gov/privacy/ccpa)
- [AWS Data Residency Controls](https://aws.amazon.com/compliance/data-residency/)
- [Cloudflare Workers for Geo-routing](https://developers.cloudflare.com/workers/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
