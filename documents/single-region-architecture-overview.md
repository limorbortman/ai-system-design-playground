# Current AI Grading Platform - Architecture Overview

## System at a Glance

The AI-Powered Student Grading Platform is a single-region, centralized system designed to automate examination scoring across schools and districts. Teachers and administrators submit exams (single or in bulk), the system applies sophisticated AI-driven grading logic, and returns scored results with detailed feedback.

---

## Architecture Diagram

```
                           ┌─────────────────────────────────┐
                           │      End Users / Clients        │
                           │  (Teachers, Admins, Students)   │
                           └──────────────┬──────────────────┘
                                          │
                                          ▼
                        ┌─────────────────────────────────────┐
                        │               Web UI                │
                        │  • Submission & Status Dashboards   │
                        │  • Result & Feedback Viewer         │
                        └──────────────┬──▲───────────────────┘
                                       │  │
                                       ▼  │
                        ┌─────────────────┴───────────────────┐
                        │            API Gateway              │
                        │  • Request Routing & Validation     │
                        │  • Result & Status Retrieval        │
                        └──────────────┬──▲───────────────────┘
                                       │  │
                    ┌──────────────────┘  └─────────────────────────────┐
                    │                                                   │
                    ▼                                                   │
        ┌───────────────────────────┐                                   │
        │    Unified Task Queue     │                                   │
        │  • Priority-based         │                                   │
        └───────────┬───────────────┘                                   │
                    │                                                   │
                    ▼                                                   │
        ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐    │
        │ Config Store     │  │ Enrichment & │  │ Localization DB  │    │
        │                  │──▶ Rules Engine │◀──│                 │   │
        │ • Exam templates │  │              │  │ • Language packs │    │
        │ • Subject rules  │  │ • Data       │  │ • Rubric i18n    │    │
        │ • Grade scales   │  │   enrichment │  │ • Feedback i18n  │    │
        └──────────────────┘  └──────┬───────┘  └──────────────────┘    │
                                     │                                  │
                                     ▼                                  │
                    ┌──────────────────────────────────────┐            │
                    │   AI Grading Engine                  │            │
                    └──────────────┬───────────────────────┘            │
                                   │                                    │
                                   ▼                                    │
                    ┌──────────────────────────────────────┐            │
                    │   Primary Database (Single Region)   │            │
                    │                                      │            │
                    │  • Exam submissions & metadata       │            │
                    │  • User profiles & permissions       │            │
                    │  • Processing status & logs          │            │
                    │  • Regional configurations           │            │
                    │                                      │            │
                    │  (SQL Database - Single Replica)     │            │
                    └──────────────┬───────────────────────┘            │
                                   │                                    │
                                   ▼                                    │
                    ┌──────────────────────────────────────┐            │
                    │   Result Storage & Reporting         │────────────┘
                    │                                      │
                    │  • Graded results & scores           │
                    │  • Grade reports & PDF generation    │
                    │  • Email delivery service            │
                    │  • Score exports & Audit logs        │
                    └──────────────────────────────────────┘
```

---

## Key System Components

### 1. **Web UI**
- The primary interface for all users (Teachers, Admins, Students).
- **Dashboards**: Allows users to upload exams, monitor batch processing status, and view real-time grading progress.
- **Result Viewer**: Displays final grades (A++ to F--), partial credit rationale, and detailed feedback narratives.
- **Admin Tools**: Interface for managing regional rules and configurations.

### 2. **API Gateway**
- The central entry point for the Web UI and any external integrations.
- **Submission Path**: Validates incoming exam data and pushes tasks to the **Unified Task Queue**.
- **Retrieval Path**: Provides REST endpoints for the Web UI to fetch graded results and reports directly from **Result Storage & Reporting**, and processing status from the **Primary Database**.
- Handles authentication, authorization, and request rate limiting.

### 3. **Unified Task Queue**
- Replaces separate queues and the Job Scheduler with a single, priority-aware system.
- **High Priority**: Assigned to real-time, single-test submissions for immediate processing.
- **Normal/Low Priority**: Assigned to batch/bulk uploads to be processed as capacity allows.
- Relies on native queue system features (e.g., Redis/RabbitMQ priority) to manage execution order without a custom coordinator.

### 4. **AI Grading Engine**
The core processor that executes grading logic:
- **LLM Inference**: Runs AI model to evaluate exam answers using the enriched context.
- **Grading Logic**: Applies subject-specific rules and templates.
- **Partial Credit Evaluation**: Determines fractional scores based on methodology correctness.
- **Grade Assignment**: Maps scores to the A++ to F-- scale.
- **Feedback Generation**: Creates explanations for scores.

### 5. **Enrichment & Rules Engine**
- A unified component that prepares exam data for the AI Grading Engine.
- **Data Enrichment**: Uses the **Config Store** and **Localization DB** to add necessary metadata, templates, and language context to the submission.
- **Rule Application**: Injects regional grading standards and preferences.
- Ensures the AI model receives a complete, context-aware package for processing.

### 6. **Primary Database**
Single source of truth storing:
- Exam submissions and metadata
- User accounts and role-based access
- Processing status and event logs
- Regional configurations and preferences

### 7. **Result Storage & Reporting**
Persists and serves graded outcomes:
- **Result Store**: The authoritative source for graded results, scores, and feedback narratives.
- **Email Delivery**: Automatically sends graded reports to teachers and students.
- **Report Generation**: Creates PDF artifacts and feedback summaries.
- **Audit & Export**: Maintains logs for compliance and supports SIS exports.

---

## Data Flow: Exam to Grade

### Real-Time Path (Single Exam)
1. **Teacher** submits exam via **Web UI**.
2. **API Gateway** validates and pushes to **Unified Task Queue** with **High Priority**.
3. **Enrichment & Rules Engine** prepares the context, and **AI Grading Engine** processes the exam.
4. **Result** is persisted in **Result Storage & Reporting**.
5. **Web UI** (via the **API Gateway**) fetches the final grade and feedback from **Result Storage & Reporting**.
6. **Result Storage & Reporting** also triggers an **Email Notification** with the final grade.

### Batch Path (Bulk Upload)
1. **Admin** uploads 1,000+ exams via **Web UI**.
2. **API Gateway** pushes tasks to **Unified Task Queue** with **Normal Priority**.
3. **Admin** monitors progress via the **Web UI status dashboard**, which fetches updates from the **Primary Database** through the **API Gateway**.
4. Once complete, results are served to the **Web UI** from **Result Storage & Reporting**.
5. **Result Storage & Reporting** sends summary emails to the Admin and individual result emails to students/teachers.

---

## Operational Characteristics

### Single-Region Design
- All data resides in one geographic location
- All processing occurs in one central facility
- Users across multiple cities/districts connect to same endpoint

### Synchronous Grading
- Real-time submissions block until AI inference completes
- API response time directly tied to LLM latency
- User experience depends on model inference speed

### Central Processing
- All AI inference occurs on single compute server
- Tasks are pulled directly from the Unified Task Queue
- No external coordination layer; workers manage their own throughput

### Unified Storage
- One database handles all reads and writes
- No data replication or failover replicas
- All queries traverse same network path

---

## Typical Performance Characteristics

| Metric | Value |
|--------|-------|
| **Single Exam Response Time** | 2–5 seconds (LLM inference) |
| **Bulk Batch Throughput** | 10–50 exams/minute (GPU-bound) |
| **Database Write Latency** | 50–200ms |
| **Real-Time Queue Depth** | 0–100 (normal load) |
| **Max Concurrent Requests** | ~50–100 (single server) |
| **Cross-Region User Latency** | 100–500ms network round-trip |

---

## Summary

The current system is a **centralized, single-region architecture** optimized for initial launch and single-district operation. It consolidates all grading logic, configuration, and data storage into one unified system, making it straightforward to deploy, monitor, and maintain grading consistency across a focused user base.

As scale increases (multiple regions, thousands of concurrent users, end-of-semester peaks), the architecture will need enhancements to distribute load, replicate data across regions, and decouple processing components for resilience and performance.