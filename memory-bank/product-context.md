---
title: Product context
---

(moved from `documents/memory-bank/product-context.md`)
# Product Context

## Vision
The **AI-Powered Student Grading System** is an enterprise-grade platform designed to automate examination scoring for schools across cities and districts. It aims to maintain fairness, consistency[...]

## Target Users & Personas
- **District Assessment Director**: Needs reliable scoring consistency across cities/districts and alignment with local educational standards.
- **School Administrator / Principal**: Needs streamlined grading workflows for whole school cohorts and dependable bulk processing.
- **Teacher / Examiner**: Needs accurate and fair grading, clear feedback on partial credit decisions, and support for both objective and subjective assessments.
- **Student**: Needs timely scores with meaningful, understandable outcomes and constructive feedback.
- **Regional Curriculum Specialist**: Needs scoring aligned with local curricular and assessment standards.

## Core Functional Requirements
1. **Multi-City and Regional Workflows**: Support localized grading expectations, language preferences, and curriculum-specific rules while preserving comparability.
2. **Localization & Regional Standards**: Full multi-language grading for exam content, instructions, and score output.
3. **Test Ingestion Features**: Support both single-test uploads (real-time) and asynchronous bulk uploads (batch processing).
4. **Grading Logic Rules & Scoring Guardrails**:
   - **Correctness**: Objective factual accuracy for objective questions; conceptual/argument accuracy for subjective responses.
   - **Depth and Detail**: Subject-specific evaluation (e.g., step-by-step reasoning in STEM, narrative structure/depth in Humanities).
   - **Partial Credit Rules**: Fractional deductions for minor execution errors/typos; larger deductions for conceptual errors.
   - **Custom Grading Scale**: Extended alphabetical scale from **A++** (exceptional mastery) to **F--** (failing/no valid approach).
   - **Score Output Expectations**: Clear score labels, partial credit explanations, and constructive narratives tailored for students and teachers.

## Performance Constraints

### Scroll Processing SLA (Added: 2026-07-12)
| Metric | Current | 
| :--- | :--- | 
| Scroll Processing Latency | 1 day (24 hours) | 
| Load Profile | ~100,000 events per 5 minutes | 
| Throughput (QPS) | ~333 events/sec | 

