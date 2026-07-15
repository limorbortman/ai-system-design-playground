# Multi-Region Scaling Architecture (Option A)
## Distributed Worker Pools + Regional Database Replication

**Design Date**: 2026-07-12  
**Author**: System Design Collaboration (Claude Haiku 4.5)  
**Status**: Ready for Engineering Review  

---

## 1. Context & Problem Statement

### Current State
The AI-Powered Student Grading System is a **centralized, single-region architecture** designed for initial launch. It operates at:
- **Compute bottleneck**: 10–50 exams/minute (single GPU server)
- **Database bottleneck**: Single primary node, 50–200ms write latency
- **Geographic coverage**: One region only; users across multiple cities experience high latency

### Business Problem
The system must scale to handle:
- **333 events/sec** burst traffic (from product spec)
- **Multiple geographic regions** while maintaining GDPR compliance (data residency per region)
- **3-hour processing SLA** for all submissions
- **Zero cross-region data movement** (student data stays in region of origin)

### Why This Matters
Without this redesign:
- Burst traffic will overwhelm the single GPU server (queuing delays, dropped exams)
- Database writes will become a bottleneck (contention, high latency)
- Regional latency will hurt user experience (100–500ms round-trip for distant users)
- GDPR compliance risk (data in unintended regions)

---

## 2. Requirements & Constraints

### Functional Requirements
| Requirement | Details |
|---|---|
| **Multi-Region Deployment** | Each region (US, EU, APAC, etc.) runs independent instance; zero cross-region data movement |
| **Burst Throughput** | Handle 333 events/sec for up to 1 hour continuously |
| **Async Processing** | No real-time SLA; all submissions processed within 3 hours |
| **Grading Consistency** | Apply same grading logic across all regions; results are deterministic |
| **Result Retrieval** | Teachers/admins can query results with <500ms latency from their region |

### Non-Functional Requirements
| Requirement | Details |
|---|---|
| **Data Residency** | Student exam data, results, and logs never cross region boundary |
| **Strong Consistency** | Within a region: immediate visibility of graded results |
| **Availability** | Single region can tolerate 1 node failure without serving queued tasks; failover time <10 min |
| **Audit Trail** | All exam submissions and grading decisions logged within region for compliance |

### Technical Constraints
| Constraint | Details |
|---|---|
| **Processing Latency SLA** | P95 latency ≤ 180 minutes (3 hours) from submission to result availability |
| **Database Write Throughput** | Must sustain ≥100 writes/sec per region during burst |
| **Queue Throughput** | Must sustain ≥333 messages/sec per region |
| **Consistency Model** | Strong within region (no eventual consistency across regions) |
| **Network Overhead** | Regional latency within region <50ms; inter-region traffic zero |

### Assumptions
1. **Regional independence**: No shared services between regions (each is self-contained)
2. **Batch-friendly workload**: Async processing is acceptable; no requirement for sub-second feedback
3. **GDPR compliance**: Data residency is non-negotiable; region boundaries map to legal jurisdictions
4. **Homogeneous regions**: Each region has similar compute capacity and scale requirements

---

## 3. Current Status

### Existing Architecture
```
Single-Region Centralized System:
  API Gateway → Unified Task Queue (Redis/RabbitMQ)
    → Enrichment & Rules Engine → AI Grading Engine
    → Primary Database (single node)
    → Result Storage & Reporting
```

### Identified Bottlenecks
1. **Grading Compute**: Single GPU server processes 10–50 exams/min; cannot handle 333 events/sec (~20,000 exams/min)
2. **Database Writes**: Single primary node; write contention limits throughput to ~100 writes/sec
3. **Queue**: Redis/RabbitMQ lacks consumer group balancing; workers manually compete for tasks
4. **Geographic Coverage**: All users route to single endpoint; cross-region latency is unavoidable

### Performance Baseline
| Metric | Current | Target |
|---|---|---|
| **Throughput** | 10–50 exams/min | 333 events/sec = ~20,000 exams/min |
| **Write Latency** | 50–200ms | <100ms (P95) |
| **Failover Time** | N/A (single region) | <10 min per region |
| **Data Residency** | N/A (single region) | 100% within-region for each region |

---

## 4. High-Level Target Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MULTI-REGION DEPLOYMENT                           │
│         (Each region is independent; zero cross-region data flow)           │
└─────────────────────────────────────────────────────────────────────────────┘

                    REGION 1 (e.g., EU)        REGION 2 (e.g., US)
                    ─────────────────         ─────────────────

                  ┌─────────────────┐      ┌─────────────────┐
                  │   Web UI (EU)   │      │   Web UI (US)   │
                  │   API Gateway   │      │   API Gateway   │
                  └────────┬────────┘      └────────┬────────┘
                           │                        │
                           ▼                        ▼
                  ┌─────────────────┐      ┌─────────────────┐
                  │  Kafka Cluster  │      │  Kafka Cluster  │
                  │   (3 brokers)   │      │   (3 brokers)   │
                  └────────┬────────┘      └────────┬────────┘
                           │                        │
          ┌────────────────┼────────────────┐      │
          ▼                ▼                ▼      │
    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
    │ Worker 1 │    │ Worker 2 │    │ Worker N │  │
    │ (GPU)    │    │ (GPU)    │    │ (GPU)    │  │
    └────┬─────┘    └────┬─────┘    └────┬─────┘  │
         │               │               │         │
         └───────────────┼───────────────┘         │
                         ▼                         ▼
                  ┌─────────────────────────────────────┐
                  │   Primary Database (EU)             │
                  │   + Read Replicas (EU)              │
                  │                                     │
                  │  - Exam submissions                 │
                  │  - Results & feedback               │
                  │  - Audit logs                       │
                  └──────────────┬──────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
            ┌────────────────┐      ┌────────────────┐
            │ Primary (Write)│      │  Read Replica  │
            │                │      │   (Reporting)  │
            └────────────────┘      └────────────────┘
                    │                        │
         (sync replication)       (async replication)
                    │                        │
                    └────────────────────────┘
                           ▼
                  ┌──────────────────────┐
                  │ Result Storage       │
                  │ (for fast retrieval) │
                  └──────────────────────┘
```

### Key Design Decisions

#### 1. **Queue: Kafka (replacing Redis/RabbitMQ)**
- **Why**: Kafka supports consumer groups (multiple workers auto-balance), higher throughput (500k+ msgs/sec)
- **Configuration per region**:
  - 3-broker cluster (HA)
  - Replication factor: 3 (fault tolerance)
  - 10+ partitions (allows parallelism across 10+ workers)
  - Retention: 24 hours (buffer for slow processing)

#### 2. **Compute: Horizontal Worker Pool**
- **Worker Design**:
  - Each worker: Python/Node service pulling from Kafka
  - Stateless: can be killed/restarted without data loss (offset managed by Kafka)
  - Processes 1–5 exams/minute per worker (GPU-dependent)
- **Scaling**:
  - Start: 6–10 workers per region (333 events/sec ÷ 5 exams/min per worker = ~67 workers needed for sustained load; burst only needs 10–20)
  - Auto-scale: Trigger new worker if queue depth > 1000 events for >5 minutes
  - Max per region: 50–100 workers (cloud provider limits, cost control)

#### 3. **Database: Primary + Read Replicas (per region)**
- **Primary Node**: Handles all writes (exam submissions, results, logs)
  - Instance type: High-memory, SSD-backed (e.g., AWS RDS db.r5.2xlarge)
  - Connection pool: 100–200 connections (worker threads)
  - Estimated writes: ~333 events/sec → ~100–150 writes/sec (batched)
  
- **Read Replicas** (minimum 1 per region):
  - Serves reporting queries (Web UI dashboard, result retrieval)
  - Async replication lag: <1 second (acceptable for "eventual consistency" within region)
  - Can have multiple replicas for geographic distribution within region (if needed)

#### 4. **Result Storage**
- **Primary store**: SQL database (same as exam submissions)
- **Cache layer** (optional): Redis for frequently accessed results (teacher dashboards)
  - TTL: 1 hour (results are stable after grading completes)
  - Size: ~10GB per region (estimated)

#### 5. **API Gateway & Web UI**
- **Routing**: Traffic from region X automatically routes to region X infrastructure
  - Use GeoIP routing (CloudFlare, AWS Route53)
  - No cross-region API calls
  - Regional endpoint: `api-eu.grading.com`, `api-us.grading.com`, etc.

- **Authentication & Authorization**:
  - Regional identity provider (per region)
  - No shared session store across regions
  - User accounts replicated to each region (one-way initial sync; local edits after)

#### 6. **Data Isolation & GDPR Compliance**
- **Isolation Strategy**:
  - Exam submissions tagged with `region_id` at ingestion
  - All queries filtered by `region_id` (cannot query across regions)
  - Database backups per region; no cross-region restore
  - Audit logs per region; deletion/retention per local jurisdiction

- **Cross-Region Data Movement Policy**:
  - ✅ Allowed: Config/rule templates (grading rubrics shared globally, but applied locally)
  - ❌ Forbidden: Student data, exam answers, results, user accounts, logs

---

## 5. Alternative Analysis

### Alternative 1: Option B (Event Stream Processing + Sharded Database)
**Why rejected**:
- Requires Kafka Streams expertise (team ramp-up time: 6–8 weeks)
- Sharding complexity (shard key design, rebalancing, hot-shard monitoring)
- Higher operational burden; not recommended for first multi-region rollout
- Suitable for **Phase 2** if write throughput exceeds 1000 writes/sec

### Alternative 2: Stick with Single-Region (Vertical Scaling)
**Why rejected**:
- GDPR compliance impossible (data cross-region)
- Max throughput limited by single GPU server (~100 exams/min even with best hardware)
- Cannot achieve 333 events/sec burst without multi-region distribution
- Unfeasible for next phase of growth

### Alternative 3: Global Database with Regional Read Replicas
**Why rejected**:
- Violates GDPR (writing to non-local primary)
- Replication lag across regions creates audit trail inconsistency
- Cross-region network latency for writes (unacceptable)
- Compliance risk: data leaves region during normal operation

---

## 6. Testability & Operations

### Testing Strategy

#### Unit Tests (QE Responsibility)
- **Worker Logic**:
  - Test exam enrichment, grading rule application, score mapping
  - Coverage: 100% of grading branches (A++ to F--)
  - Mock Kafka consumer; verify offset handling
  
- **API Gateway**:
  - Test region routing (GeoIP resolution)
  - Test authentication per region
  - Coverage: Happy path + auth failures

- **Database Queries**:
  - Test `region_id` filtering (ensure no cross-region leakage)
  - Test bulk insert (batch submission)
  - Coverage: All GDPR-critical queries

#### Integration Tests
- **End-to-End (single region)**:
  1. Submit exam via Web UI → stored in regional DB
  2. Exam reaches Kafka → worker picks it up
  3. Worker enriches, grades, writes result
  4. Result visible in Web UI within <3 hours
  5. Audit log created for exam

- **Multi-Region**:
  1. Submit exam in region A
  2. Submit identical exam in region B
  3. Verify results are isolated (no cross-region query)
  4. Verify results are deterministic (same exam, same grade)

#### Load Tests (QE + Monitor)
- **Per Region**:
  - Burst: 333 events/sec for 1 hour
  - Monitor: Queue depth, worker latency, database write latency, CPU/memory
  - Success criteria: P95 latency <180 minutes

- **Failover Simulation**:
  - Kill 1 Kafka broker → verify remaining brokers take over
  - Kill 1 worker → verify auto-replacement triggered
  - Kill 1 database replica → verify reads still work (primary takes over)

### Operational Observability (Monitor Responsibility)

#### Key Metrics per Region
| Metric | Alert Threshold | Log Level |
|--------|---|---|
| Queue Depth (Kafka) | >5000 events | INFO (log event) |
| Queue Depth | >10000 events | WARN (page on-call) |
| Worker Latency (P95) | >600 seconds | WARN |
| Database Write Latency | >500ms | WARN |
| Replication Lag (async) | >10 seconds | WARN |
| Kafka Broker Down | Any broker | CRITICAL (page) |
| Database Primary Down | N/A | CRITICAL (failover trigger) |

#### Logs & Traces
- **Exam Submission**: Log `exam_id`, `region_id`, `timestamp`, `status`
- **Grading Start/Complete**: Log `exam_id`, `worker_id`, `latency_ms`, `grade`
- **Database Write**: Log `table`, `region_id`, `row_count`, `latency_ms`
- **Kafka Consumer Lag**: Log `consumer_group`, `partition`, `lag_messages`, `age_seconds`

#### Alerts & Escalation
| Alert | Action | Escalation |
|---|---|---|
| Queue Depth >10k | Scale up 5 new workers | On-call engineer |
| Write Latency >500ms | Check DB CPU, connections | Database team + on-call |
| Kafka Broker Down | Auto-replace broker | Platform team |
| Replication Lag >10s | Investigate network | Database team |

### Operational Playbooks
1. **Add a new region**:
   - Provision Kafka cluster, database primary, 1 read replica
   - Route regional traffic via API Gateway
   - Smoke test: Submit exam → verify result
   
2. **Scale workers up**:
   - Spin N new workers
   - Kafka consumer group auto-rebalances
   - Monitor: Verify queue depth decreases, latency drops
   
3. **Database primary failure**:
   - Promote read replica to primary
   - Verify all writes succeed
   - Re-provision replica
   - Estimated time: ~5 minutes

4. **Kafka broker failure**:
   - Auto-replace via container orchestration
   - Kafka rebalances partitions
   - No manual intervention if quorum remains
   
---

## 7. Timeline & Milestones

### Phase 1: Foundation (Weeks 1–4)
**Goal**: Deploy Option A in one region (EU); production-ready

| Week | Deliverable | Owner | Acceptance Criteria |
|---|---|---|---|
| 1–2 | Kafka cluster setup + monitoring | Platform Eng | 3 brokers running, metrics in dashboard |
| 2–3 | Worker pool scaffold (stateless service) | Coder | 10 workers deployed; can process 1 exam/min |
| 3 | Database replication setup | Database Eng | Primary + 1 read replica, <1s lag |
| 4 | End-to-end integration test | QE | Exam ingestion → result retrieval; P95 latency recorded |
| 4 | Load test (burst to 100 events/sec) | QE + Monitor | Sustained throughput, no dropped events |

**Deployment**: EU production launch; monitoring live

---

### Phase 2: Expansion (Weeks 5–8)
**Goal**: Deploy to US region; multi-region operational procedures validated

| Week | Deliverable | Owner | Acceptance Criteria |
|---|---|---|---|
| 5–6 | Replicate EU infrastructure to US | Platform Eng | US cluster operational; independent from EU |
| 6–7 | GeoIP routing + API Gateway regional endpoints | Coder | Traffic from US routes to US; no cross-region calls observed in logs |
| 7 | Data isolation validation (GDPR audit) | Security + QE | No cross-region queries; audit trail per region; GDPR checklist signed off |
| 8 | Failover drills (both regions) | Monitor + Platform Eng | Documented runbooks; <10 min failover time verified |

**Deployment**: US + EU live; regional independence verified

---

### Phase 3: Optimization (Weeks 9–12)
**Goal**: Production hardening; roadmap to Phase 2 (sharding)

| Week | Deliverable | Owner | Acceptance Criteria |
|---|---|---|---|
| 9 | Load test (burst to 333 events/sec; 1-hour sustained) | QE + Monitor | P95 latency <180 min; no queue overflow |
| 10 | Cost optimization (worker sizing, DB instance type) | Platform Eng + Monitor | Recommendation for optimal cost/performance per region |
| 11 | Runbook documentation + on-call training | Ops + Monitor | Runbooks peer-reviewed; team trained on escalation |
| 12 | Performance baseline + roadmap to Phase 2 | Coder + QE | Metrics collected; triggers for migration to Option B (sharding) defined |

**Deployment**: Production-grade multi-region system live; ready for next phase

---

### Critical Path Dependencies
- Week 1: Kafka must be operational before workers can be deployed
- Week 3: Database replication must be stable before load testing
- Week 4: Single-region load test must pass before multi-region expansion
- Week 6: US infrastructure must mirror EU before GeoIP routing
- Week 7: Data isolation must be validated before claiming GDPR compliance

---

## 8. Risks, Assumptions & Open Questions

### High-Risk Items

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Kafka consumer group rebalancing causes processing lag | Medium | High | Pre-test rebalancing behavior; provision 20% spare workers |
| Database primary CPU bottleneck before burst ends | Medium | High | Monitor CPU during Phase 1 load test; scale to larger instance if needed |
| Cross-region data leak via buggy query | Low | Critical | Code review all queries; automated GDPR compliance tests (no region_id = auto-reject) |
| Regulatory requirement to separate auth infrastructure | Low | Medium | Plan for regional identity provider (separate work) |

### Critical Assumptions

1. **GeoIP routing is accurate**: Assumes users route to nearest region; mismatch could break GDPR
   - **Validation**: Test with VPN from different regions; confirm traffic routes correctly
   
2. **Workers can process 1–5 exams/min consistently**: Assumes LLM inference time is stable
   - **Validation**: Benchmark worker throughput with real exam data; account for inference variance
   
3. **Kafka replication (RF=3) provides sufficient HA**: Assumes 2-broker failure is acceptable downtime
   - **Validation**: Test 2-broker failure scenario; measure recovery time
   
4. **Regional databases can sustain 100 writes/sec**: Assumes batching and connection pooling work
   - **Validation**: Load test with concurrent worker connections; monitor write latency

### Open Questions

1. **How is user account replication handled?**
   - Currently undefined: Do admins exist in all regions or only their home region?
   - Action: Clarify before Phase 2; impacts auth design
   
2. **What is the exact sharding strategy if we migrate to Option B?**
   - Currently undefined: Shard by `school_id`, `region_id`, or `exam_id`?
   - Action: Document shard key rationale before Phase 3
   
3. **Can exam config/templates be shared globally, or must they be per-region?**
   - Currently assumed: Shared globally; applied locally (no GDPR violation)
   - Action: Confirm with product team before Phase 1
   
4. **Is 3-hour SLA hard or soft?**
   - Currently assumed: Soft; exceeding it triggers investigation but not an incident
   - Action: Confirm SLA classification; impacts alert thresholds

### Known Limitations

- **Single-region failover**: If a region's database goes down, all users in that region are blocked (no automatic cross-region failover due to GDPR)
  - Mitigation: Backup strategy to restore within <1 hour; on-call response time <10 minutes
  
- **Query performance at scale**: As result set grows, reporting queries (Web UI dashboard) may slow
  - Mitigation: Implement read replica; add caching layer in Phase 2
  
- **No global audit trail**: Compliance audits must be run per-region
  - Mitigation: Document audit procedures per region; central aggregation (if needed) is off-site analysis, not live data movement

---

## 9. Implementation Roadmap (Phased Delivery)

### Phase 1: Single-Region Deployment (4 weeks)
- Deploy EU infrastructure (Kafka, workers, DB primary + replica)
- Achieve 100 events/sec burst sustainably
- Validate end-to-end grading pipeline
- **Release**: Migrate 10% of EU users; monitor

### Phase 2: Multi-Region Expansion (4 weeks)
- Deploy US infrastructure (parallel to Phase 1)
- Implement GeoIP routing
- Validate data isolation
- **Release**: Migrate 100% of EU + 100% of US

### Phase 3: Production Hardening & Scale Testing (4 weeks)
- Load test to 333 events/sec (full burst)
- Failover drills; runbook documentation
- Cost optimization
- **Release**: Production-grade system; ready for 2x growth

### Phase 4 (Future): Horizontal Sharding (Optional)
- Migrate from Option A → Option B if write throughput exceeds 500 writes/sec
- Estimated effort: 8–12 weeks
- Trigger: Monitor write latency; if P95 > 1 second for 2+ weeks, begin migration planning

---

## 10. Success Metrics

### Delivery Success
- ✅ Option A deployed in EU region (Week 4)
- ✅ US region operational (Week 8)
- ✅ 333 events/sec sustained for 1 hour (Week 12)

### Operational Success
- ✅ P95 latency <180 minutes under burst load
- ✅ Queue depth stays <5000 during normal operations
- ✅ Database write latency <200ms (P95)
- ✅ Replication lag <1 second

### Compliance Success
- ✅ Zero cross-region data movement detected in logs
- ✅ GDPR audit checklist 100% complete
- ✅ Audit trail captures all exams + results per region

---

## 11. Appendix: Example Kafka Configuration

```yaml
# Kafka Cluster (per region)
broker_count: 3
replication_factor: 3
partitions_per_topic: 12

topics:
  - name: "exam_submissions"
    partitions: 12
    replication_factor: 3
    retention_ms: 86400000  # 24 hours
    cleanup_policy: "delete"
    compression_type: "snappy"

  - name: "grading_results"
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000  # 7 days (audit trail)
    cleanup_policy: "delete"
    compression_type: "snappy"

# Consumer Group (worker pool)
consumer_group: "grading_workers_{region}"
enable_auto_commit: true
auto_offset_reset: "earliest"
max_poll_records: 5  # Process 5 exams per poll
session_timeout_ms: 30000
heartbeat_interval_ms: 10000
```

---

## 12. Appendix: Example Worker Service (Pseudo-code)

```python
class GradingWorker:
    def __init__(self, region_id, kafka_bootstrap_servers):
        self.region_id = region_id
        self.consumer = KafkaConsumer(
            'exam_submissions',
            group_id=f'grading_workers_{region_id}',
            bootstrap_servers=kafka_bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode())
        )
        self.db = DatabaseConnection(region_id)
    
    def run(self):
        for exam_msg in self.consumer:
            exam_data = exam_msg.value
            
            # Validate region_id to prevent cross-region processing
            assert exam_data['region_id'] == self.region_id
            
            try:
                # Enrich + Grade
                enriched = self.enrich(exam_data)
                grade = self.grade(enriched)
                
                # Write result (to primary DB)
                self.db.save_result(exam_data['exam_id'], grade, self.region_id)
                
                # Commit offset (Kafka confirms processing)
                self.consumer.commit()
                
            except Exception as e:
                # Log failure; retry on next run
                logger.error(f"Exam {exam_data['exam_id']} failed: {e}")
                # Do NOT commit; offset resets after session timeout
```

---

## 13. Appendix: Database Schema (Per Region)

```sql
-- Exams (read-only after ingestion)
CREATE TABLE exams (
    exam_id UUID PRIMARY KEY,
    region_id VARCHAR(50) NOT NULL,  -- GDPR: Always present
    school_id UUID NOT NULL,
    subject VARCHAR(100) NOT NULL,
    exam_content JSONB NOT NULL,
    submitted_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX (region_id, created_at),  -- Fast queries per region
    CONSTRAINT check_region_matches CHECK (region_id = current_region)  -- Database-level GDPR guard
);

-- Results (written by workers)
CREATE TABLE results (
    result_id UUID PRIMARY KEY,
    exam_id UUID NOT NULL UNIQUE,
    region_id VARCHAR(50) NOT NULL,  -- GDPR: Always present
    grade VARCHAR(3) NOT NULL,  -- A++, A+, ..., F--
    score_numeric FLOAT NOT NULL,
    feedback_narrative TEXT,
    partial_credit_breakdown JSONB,
    graded_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX (region_id, graded_at),  -- Fast queries per region
    CONSTRAINT check_region_matches CHECK (region_id = current_region),
    FOREIGN KEY (exam_id) REFERENCES exams(exam_id) ON DELETE CASCADE
);

-- Audit Log (immutable)
CREATE TABLE audit_log (
    log_id UUID PRIMARY KEY,
    region_id VARCHAR(50) NOT NULL,  -- GDPR: Always present
    exam_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,  -- 'SUBMITTED', 'GRADED', 'ACCESSED'
    actor_id UUID,  -- User ID if applicable
    timestamp TIMESTAMP NOT NULL,
    details JSONB,
    INDEX (region_id, timestamp),  -- Fast queries per region
    CONSTRAINT check_region_matches CHECK (region_id = current_region)
);
```

---

## End of Design Document

**Next Steps**:
1. **Coder**: Begin Phase 1 infrastructure setup (Kafka provisioning, worker scaffold)
2. **QE**: Prepare integration test suite; define load test scenarios
3. **Monitor**: Set up Kafka + database monitoring dashboards; alert rules
4. **Security**: Conduct GDPR compliance review; data isolation validation
5. **Product**: Confirm shared config assumptions; finalize multi-region user model