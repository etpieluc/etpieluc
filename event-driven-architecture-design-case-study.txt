# Event-Driven Architecture Design Case Study

## Executive Summary

This case study covers the architectural transformation of a tightly coupled, synchronous request-response system into a resilient, event-driven platform. The migration eliminates cascade failures, enables independent service deployments, and unlocks real-time data processing capabilities that were previously unattainable with the synchronous model.

---

## Problem Statement

The legacy platform relies on synchronous REST calls between 25+ services, creating a web of runtime dependencies. Any downstream service degradation causes upstream failures to propagate instantly, resulting in full-platform outages during peak traffic periods. Deployment coupling meant all services have to be released together, slowing delivery cadence to bi-weekly releases.

**Key challenges:**
- Cascade failures bringing down unrelated services
- Deployment coupling requiring coordinated, risky releases
- No ability to replay or audit historical event streams
- Inability to support real-time analytics and downstream consumers
- P99 latency spikes during synchronous fan-out operations

---

## Architectural Principles

The new architecture is designed around four foundational principles:

1. **Autonomy** — each service owns its data and publishes events; never calls into another service's database
2. **Durability** — all domain events are persisted and replayable
3. **Idempotency** — all event consumers handle duplicate delivery safely
4. **Observability** — every event carries a correlation ID traceable end-to-end

---

## Solution Design

### Event Topology

```
┌──────────────────────────────────────────────────────────────┐
│                     Producers                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Orders   │  │ Payments │  │ Inventory│  │ Shipping │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │              │              │         │
├───────▼──────────────▼──────────────▼──────────────▼────────┤
│                  Apache Kafka (MSK)                         │
│   ┌──────────────────────────────────────────────────┐      │
│   │  Topics: orders.placed, payments.processed,      │      │
│   │          inventory.updated, shipping.dispatched  │      │
│   └──────────────────────────────────────────────────┘      │
├────────────────────────────────────────────────────────────-┤
│                     Consumers                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Notif.   │  │ Analytics│  │ Audit Log│  │ Search   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Message Broker:** Apache Kafka (Amazon MSK, 3-broker cluster)
- **Schema Registry:** Confluent Schema Registry (Avro schemas)
- **Stream Processing:** Apache Flink (real-time aggregations)
- **Event Store:** Kafka with 90-day retention + S3 Tiered Storage for archival
- **Dead Letter Queue:** Dedicated DLQ topics per consumer group with alerting
- **Observability:** OpenTelemetry traces on all producers/consumers, Grafana dashboards
- **Outbox Pattern:** Transactional outbox via Debezium CDC on PostgreSQL

---

## Proposed Implementation Strategy

### Strangler Fig Migration

Rather than a big-bang rewrite, apply the Strangler Fig pattern to incrementally route traffic through the event bus:

**Step 1 — Dual Write**
Services simultaneously write to their database and publish to Kafka. Consumers read from both REST and Kafka, with a feature flag controlling which path was authoritative.

**Step 2 — Consumer Migration**
Each downstream consumer migrates to Kafka one by one. Legacy REST calls are deprecated with sunset dates communicated to consumers.

**Step 3 — Producer Cutover**
Once all consumers are event-driven, REST endpoints are decommissioned.

### Outbox Pattern for Guaranteed Delivery

To eliminate the dual-write race condition, all producers adopt the transactional outbox pattern:

```
┌─────────────┐    ┌──────────────────────┐    ┌─────────┐
│  Service A  │───▶│  DB Transaction:     │    │  Kafka  │
│  (Producer) │    │  1. Write to table   │    │  Topic  │
└─────────────┘    │  2. Write to outbox  │    └────▲────┘
                   └──────────────────────┘         │
                              │                      │
                   ┌──────────▼──────────┐           │
                   │   Debezium CDC      │───────────┘
                   │   (Kafka Connect)   │
                   └─────────────────────┘
```

---

## Proposed Event Schema Governance

All events are defined in Avro schemas and registered in Confluent Schema Registry. Schema evolution rules enforce backward compatibility:

- **Allowed:** Adding optional fields with defaults
- **Allowed:** Removing fields with defaults
- **Blocked:** Renaming fields (breaking change — requires new topic version)
- **Blocked:** Changing field types

Schema changes require a PR review by the Platform Architecture team and automated Confluent compatibility checks in CI.

---

## Planned Results & Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Cascade failure incidents | 8/quarter | 0/quarter | **Eliminated** |
| Deployment frequency | Bi-weekly (coordinated) | Daily (independent) | **7x faster** |
| P99 fan-out latency | 4,200ms | 180ms | **96% reduction** |
| Time to new consumer onboarding | 2 weeks | 1 day | **90% faster** |
| Real-time analytics lag | N/A (batch) | < 500ms | **New capability** |

---

## Lessons Learned


---

## References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Debezium CDC Connector](https://debezium.io/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
