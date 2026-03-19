# Validation: How to Prove the Layer Works

## Why This Document Exists

A framework that can't be tested is an opinion. System physics makes specific, falsifiable claims about its ability to derive system behavior, predict failure, and validate stability. This document defines how those claims are tested.

These tests are designed to be run by anyone, against any system, with results that are objectively scorable. If the layer works, the tests will show it. If it doesn't, the tests will show that too.

## Current Status

These tests have not all been completed. We are publishing the methodology before the results so that the community can evaluate whether the tests are fair, comprehensive, and rigorous. When results are available, they will be published in this repository with full data.

## The Five Tests

### Test 1: Replay

**What it proves:** The physics engine can predict failures from real metrics before the failure occurs.

**Method:** Obtain telemetry data from before a real production incident. Feed the metrics into the physics engine in chronological order, stopping at intervals before the failure. At each interval, record the engine's regime classification, predicted failure mode, predicted time-to-impact, and confidence score. Compare predictions against what actually happened.

**Scoring:**
- Prediction lead time: how many minutes before the failure did the engine first classify the system as degrading?
- Failure mode accuracy: did the predicted failure mode match the actual failure?
- Time-to-impact accuracy: how close was the predicted time to the actual time of impact?
- Confidence calibration: when the engine said 80% confidence, was it right roughly 80% of the time?

**What a pass looks like:** The engine predicts the failure with meaningful lead time (10+ minutes) and the predicted failure mode is correct or directionally correct. Perfect time-to-impact accuracy is not expected. Being within 50% of the actual time is a reasonable bar for early validation.

**What a fail looks like:** The engine doesn't detect the regime shift until after the failure, or it predicts the wrong failure mode, or it produces high-confidence predictions that are wrong. Any of these indicate the model needs work.

### Test 2: Comparison

**What it proves:** The physics layer catches things that existing monitoring tools miss.

**Method:** Take the same incident used in Test 1. Document what the existing monitoring stack (Datadog, Grafana, PagerDuty, whatever the team uses) alerted on and when. Compare the physics engine's prediction timeline against the existing tooling's alert timeline.

**Scoring:**
- Lead time difference: how many minutes earlier did the physics engine detect the problem compared to existing alerts?
- Signal quality: did the physics engine provide a causal explanation that existing alerts didn't? (e.g., "retry amplification causing cascade" vs. "CPU high")
- False positive comparison: in the period leading up to the incident, did the physics engine produce fewer or more false alarms than existing monitoring?

**What a pass looks like:** The physics engine detects the problem before existing monitoring fires, and provides a more specific or accurate explanation of the cause.

**What a fail looks like:** Existing monitoring catches the issue at the same time or earlier, with comparable signal quality. This would suggest the layer isn't adding value over well-configured threshold alerts.

### Test 3: Misdiagnosis

**What it proves:** The physics layer identifies the structural root cause, not just the symptom, when existing tools get it wrong.

**Method:** Find an incident where the monitoring tools or the incident response team initially pursued the wrong remediation. Examples: "add more memory" when the real cause was retry amplification, "scale up" when the real cause was a configuration change that increased per-request cost. Feed the pre-incident metrics into the physics engine and evaluate whether it would have identified the correct root cause.

**Scoring:**
- Root cause accuracy: did the physics engine identify the structural cause that the team eventually discovered?
- Remediation quality: did the physics engine's recommended remediation match what actually fixed the problem?
- Misdirection avoidance: did the physics engine avoid recommending the wrong fix that the team initially pursued?

**What a pass looks like:** The physics engine identifies the structural root cause and recommends remediation that matches what ultimately resolved the incident. It does not recommend the initial wrong fix.

**What a fail looks like:** The physics engine recommends the same wrong fix that the team initially tried, or identifies a root cause that doesn't match what actually resolved the incident.

### Test 4: Autoscaler Failure

**What it proves:** Orchestration tools make better decisions with physics-derived signals than with raw metric thresholds.

**Method:** Find or construct a scenario where reactive autoscaling worsened an incident. Common pattern: retry storm causes CPU spike, HPA scales up, new pods join the retry storm, CPU spikes again, HPA adds more pods. Feed the metrics into the physics engine and evaluate what signal it would have emitted to the autoscaler.

**Scoring:**
- Signal correctness: did the physics engine emit a signal that would have prevented the escalation? (e.g., "don't scale, apply backpressure" instead of "scale up")
- Demand accuracy: did the engine correctly characterize the demand as amplification rather than genuine load increase?

**What a pass looks like:** The physics engine correctly identifies that scaling up would worsen the problem and recommends an alternative action.

**What a fail looks like:** The physics engine emits a scaling signal that would have produced the same escalation as the reactive autoscaler. This would indicate the demand characterization isn't distinguishing genuine load from amplification artifacts.

### Test 5: Governance

**What it proves:** The physics layer detects data quality degradation that no one noticed, and can produce evidence suitable for compliance.

**Method:** For a data pipeline with compliance requirements, review historical data for periods where quality was silently degraded: a host stopped sending logs without anyone noticing, duplicate events inflated counts, latency degraded without alerting. Feed the historical metrics into the physics engine and evaluate whether it would have detected and reported the degradation.

**Scoring:**
- Detection accuracy: did the engine detect the quality degradation?
- Detection speed: how quickly after onset would the engine have alerted?
- Evidence quality: does the engine's output (DQI scores, gap records, causal attribution) constitute evidence an auditor could use?

**What a pass looks like:** The engine detects degradation that the team missed, within minutes of onset, and produces an evidence chain with timestamps, affected dimensions, and estimated impact.

**What a fail looks like:** The engine misses the degradation, or detects it but cannot produce sufficient evidence for compliance purposes.

## Aggregate Scoring

The five tests address different stakeholders:

| Test | Primary audience | What it validates |
|------|-----------------|-------------------|
| Replay | Engineering team | The math predicts correctly |
| Comparison | SRE team | Current tools have a blind spot |
| Misdiagnosis | VP of Engineering | The gap costs real incidents |
| Autoscaler | Platform team | Orchestration is operating blind |
| Governance | CISO / Compliance | Audit readiness requires this layer |

The layer thesis is validated if the physics engine demonstrably outperforms threshold-based monitoring on at least 3 of 5 tests. If it matches existing tools on all 5, the layer adds complexity without value. If it underperforms on any test, that test identifies where the model needs improvement.

## How to Contribute Validation Data

The most valuable contribution to this project right now is real incident data. If you have anonymized telemetry from a production incident, including the metrics from before the failure, what your monitoring detected and when, and what the actual root cause turned out to be, we would welcome the opportunity to run these tests against your data.

No event content is needed. Only aggregated metrics: rates, counts, latency distributions, error counts, and timeline of what happened. Contact information is in [CONTRIBUTING.md](../CONTRIBUTING.md).
