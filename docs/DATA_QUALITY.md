# Data Quality: The Five Dimensions

## Scope

This document defines how the system physics layer measures data quality. Data quality is one application of the layer, not the totality of it. The layer also addresses failure prediction, stability validation, demand characterization, and change validation. This document focuses specifically on how quality is measured across five dimensions.

## Principle

Data quality is not a single measurement. It is five independent dimensions, each capturing a different way that data can fail to represent reality. A data flow can be timely but inaccurate, complete but inconsistent, unique but invalid. Monitoring volume alone, which is what most tools do, captures none of these dimensions.

The system physics layer can measure these five dimensions through normalized metrics at ingestion boundaries, without storing event data. It derives quality signals from counters, rates, distributions, and conservation equations.

The depth of measurement depends on what the source systems expose. Where a pipeline publishes per-stage event counts, conservation equations are computable. Where a destination exports latency distributions, timeliness is measurable. Where upstream systems report generation counters, accuracy is verifiable. The framework defines what to measure and how. The available signals determine how much can be measured for any given environment.

## The Five Dimensions

### Accuracy

**What it measures:** Whether data values correctly represent the real-world events they describe. Did the system receive what the source actually sent?

**Mathematical basis:** Information theory: signal fidelity. The mutual information between the source signal and the received signal should be maximized. Any loss of fidelity is a measurable accuracy degradation.

**How it is measured without storing event data:** Count reconciliation at each stage of the data lineage. The source system reports how many events it emitted. The pipeline reports how many it received and forwarded. The destination reports how many it indexed. Deltas between stages indicate loss or corruption. Timestamp distributions reveal clock drift. Field cardinality tracking detects misrouting; if a sourcetype suddenly has a different set of source hosts than expected, data may be arriving from the wrong place.

**Honest constraint:** Count reconciliation requires the source system to report how many events it generated. Not all systems do this. Firewalls with SNMP counters do. Applications with custom metrics do. Systems with no external event counter can only be measured at the pipeline stage, which reduces the accuracy signal to "did the pipeline lose anything" rather than "did everything that was generated arrive."

**What degradation looks like:** The count reconciliation delta between stages grows. Timestamp distributions widen or shift. Field cardinality changes without a known configuration change.

### Consistency

**What it measures:** Whether data uses uniform formats, schemas, and naming conventions across all sources. Does the field called `host` in one sourcetype mean the same thing as `event_host` in another?

**Mathematical basis:** Information theory: informational entropy. Every inconsistent field name or schema divergence increases the ambiguity in the dataset. High consistency means low informational entropy across sources; a query against one sourcetype produces semantically compatible results with a query against another.

**How it is measured without storing event data:** Schema fingerprinting. Each sourcetype's field structure is represented as a fingerprint, a normalized description of field names, types, and relationships. Fingerprints are compared across sourcetypes to identify semantically equivalent fields with different names, structurally similar sourcetypes with divergent schemas, and format inconsistencies.

**Honest constraint:** Schema fingerprinting requires access to field-level metadata. Some systems export this natively (Splunk's `| metadata` and `| fieldsummary` commands produce this without returning event content). Others don't. Where field metadata isn't available, consistency measurement is limited to what can be inferred from pipeline configuration rather than observed data.

**What degradation looks like:** Schema fingerprint divergence increases over time as new sourcetypes are added without normalization. Cross-source analysis requires increasingly complex field mappings. This is a slow degradation, not an incident but a compounding tax on every downstream consumer.

### Timeliness

**What it measures:** Whether data arrives within an acceptable window from the moment the event occurred. The gap between when something happened and when the system knows about it.

**Mathematical basis:** Queueing theory: flow velocity through a processing pipeline. Each stage adds latency. When any stage's throughput drops below its input rate, queue depth grows and latency increases nonlinearly.

**How it is measured without storing event data:** Ingest latency computed from metadata that source systems already produce. The difference between event time and index time, aggregated as a distribution (p50, p95, p99) per sourcetype per interval. These are aggregate statistics exported from the pipeline or destination. A widening gap between p50 and p95 indicates the pipeline is degrading for a subset of events even if median performance looks healthy.

**Honest constraint:** This is the easiest dimension to measure because most pipelines and destinations already track latency in some form. The main limitation is granularity: some systems only report average latency rather than percentile distributions, which hides tail latency problems.

**What degradation looks like:** The latency distribution spreads; p95 and p99 diverge from p50. This signals the pipeline is approaching a capacity boundary. The distribution shape is an early warning that precedes visible throughput degradation.

### Validity

**What it measures:** Whether data conforms to expected formats, types, ranges, and business rules. Are required fields present? Are values within expected ranges?

**Mathematical basis:** Formal verification and type theory: structural integrity. Each sourcetype has an expected structure describable as a set of constraints. Validity is the degree to which incoming data satisfies those constraints. Invalid data consumes processing resources without producing useful analytical output; it is computational work that generates no value.

**How it is measured without storing event data:** Pipeline error counters and parsing failure rates. Every pipeline stage that transforms or validates data produces error metrics: events rejected, fields unparseable, transforms failed, events routed to error queues. These are pre-aggregated counts that the pipeline already tracks.

**Honest constraint:** This depends entirely on the pipeline having error tracking. Most mature data pipelines (Cribl, Logstash, Fluentd) do track these metrics. Simpler setups that just forward raw data without transformation may not produce error metrics because there's no validation step to fail.

**What degradation looks like:** Parsing error rates spike after an upstream system change: a firmware upgrade changed the log format, a code deployment altered the event structure. The validity failure rate jumps from a baseline of 0.1% to 12% within minutes. Throughput looks normal, but a growing fraction of events are structurally broken.

### Uniqueness

**What it measures:** Whether each event is delivered exactly once. No duplicates from retry logic or misconfigured routing. No phantom events created by transform errors. No silent loss where events disappear without record.

**Mathematical basis:** Conservation laws. In a correctly functioning pipeline, the number of events entering must equal the number exiting, adjusted for intentional filtering. This is a conservation equation: events are neither created nor destroyed in transit.

**How it is measured without storing event data:** Events-in versus events-out at each pipeline stage. If a stage outputs more events than it receives (beyond intentional fan-out), events are being duplicated. If it outputs fewer than it receives minus known drops, events are being silently lost. The conservation delta, the difference between expected and observed event counts at each stage, is the uniqueness signal.

**Handling intentional fan-out:** Real environments have legitimate many-to-many routing. A single event may intentionally be sent to three destinations. The conservation equation accounts for this by incorporating the declared routing topology. If a pipeline is configured to fan out 1:3, then three output events per input event is the expected ratio, not a violation. The physics layer must know the intended routing topology to distinguish intentional fan-out from accidental duplication. This topology is ingested from pipeline configuration or declared via the adapter interface. Where topology is unknown, the conservation equation operates at the aggregate level (total in vs. total out across all routes) which catches absolute loss but cannot distinguish duplication from fan-out.

**Honest constraint:** Conservation equations require per-stage event counters. If the pipeline only exposes total throughput at the entry and exit points without per-stage counts, the equation can identify that something is wrong but cannot localize where. Finer-grained pipeline instrumentation produces finer-grained conservation analysis.

**What degradation looks like:** The conservation delta goes positive after a routing change: a misconfigured rule is duplicating events. Or the delta goes negative without a corresponding increase in the drop counter: events are being silently lost. Either condition means the destination's event count no longer reflects reality.

## Composite Data Quality Index

The five dimensions are independent; a data flow can score well on some and poorly on others. A composite Data Quality Index (DQI) aggregates all five into a single score per flow channel (sourcetype, host, or pipeline).

The composite score is weighted by data classification:

**Compliance-bound data** (e.g., firewall logs with regulatory retention requirements) weights all five dimensions strictly. Missing events, inaccurate timestamps, and duplicate records are all compliance violations regardless of operational impact.

**Operational data** (e.g., application performance metrics) weights timeliness and completeness most heavily. Late or missing metrics impair operational decisions.

**Informational data** (e.g., debug logs) weights timeliness highest. The other dimensions are relaxed because the consequences of degradation are lower.

The specific weights within each classification are configurable and should be tuned to the organization's risk profile. The framework defines the structure. The organization defines the policy.

**Open question:** Whether a single composite score is the right abstraction or whether the five dimensions should always be presented independently is an active area of evaluation. A composite score is convenient for alerting and trending but can mask which dimension is degrading. The current design computes both: a composite DQI for summary views and per-dimension scores for investigation. Feedback on this trade-off is welcome.

**On the concern that configurable weights are just another form of threshold tuning:** This is a fair criticism. The difference is scope. Traditional alert thresholds are set per metric, per service, requiring hundreds or thousands of individual tuning decisions. DQI weights are set per data classification (compliance, operational, informational), requiring three configuration decisions that apply across all channels in that class. The tuning surface is smaller by orders of magnitude. It does not eliminate judgment, but it concentrates judgment where it matters most.

## How AI Makes This Accessible

The five dimensions, their mathematical foundations, and the DQI scoring are rigorous and can be complex. Not every team has someone who thinks in terms of conservation equations and information entropy.

AI bridges this gap without compromising the math:

**Plain language quality reports.** Instead of "conservation delta positive, 7.2% above expected fan-out ratio on pipeline fw-ingest," AI translates: "your firewall log pipeline is producing 7.2% more events than expected. This started after the routing change at 14:18 and is likely an accidental duplicate route. Here's which route to check."

**Guided remediation.** When a quality dimension degrades, AI generates specific recommendations tied to the actual pipeline configuration, not generic advice. The math identifies the problem. AI explains what to do about it in terms the team already understands.

**Compliance narratives.** Auditors need reports in natural language, not mathematical notation. AI generates attestation documents from the DQI evidence chain, translating "99.97% DQI with two completeness gaps totaling 14 minutes, both attributed to upstream configuration changes" into a format the compliance team can submit.

The math is the source of truth. AI is the translator. The engineer and the auditor interact with the translation. The derivation is always available for anyone who wants to inspect it.

## DQI and Regime Detection

The DQI integrates with the system physics regime model. A flow channel has two independent regime assessments:

**Volume regime:** Is the channel producing data at its expected rate? This is the traditional monitoring signal.

**Quality regime:** Is the data meeting quality standards across all five dimensions? This is what system physics adds.

A channel can be green on volume and red on quality. Data is flowing at the expected rate but it's full of duplicates, or the timestamps are drifting, or the schema changed and parsing is failing. Volume monitoring would show a healthy channel. Quality regime detection reveals the problem.

This separation is essential for governance. A compliance report that says "data was delivered continuously for 12 months" is incomplete if 3% of that data was duplicated and 0.5% had corrupted timestamps. The DQI provides the complete picture.

## Architectural Constraint: Normalize, Don't Store

System physics never stores event data. Every quality signal described above is derived from normalized metrics (counters, rates, distributions, conservation deltas, schema fingerprints) extracted at ingestion boundaries.

This constraint is deliberate and non-negotiable. It means:

- Storage cost scales with the number of channels, not with event volume. Monitoring 1,300 sourcetypes costs the same whether they produce 1,000 events per second or 1,000,000.
- There are no data residency concerns. System physics holds quality metrics, not regulated content.
- The approach is source-agnostic. Any system that can export aggregate metrics about its data flows can feed the quality engine.

The physics layer depends on other layers to export aggregate metrics about their data flows. This is not a limitation, it is the layer model working as designed. The data layer stores data. The observability layer collects signals. System physics derives meaning from those signals. Duplicating storage or collection would violate the principle that each layer has a distinct responsibility. System physics does the math. It trusts other layers to do theirs.
