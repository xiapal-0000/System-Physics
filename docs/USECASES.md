# Use Cases

## Scope

These are the concrete applications of the system physics layer. Each one describes a real problem, why existing tools don't solve it, and what the physics layer adds. Each one also states what the physics layer cannot do in that domain.

## 1. Predictive Failure Detection

**The problem:** Production incidents are detected after they've started. Monitoring tools alert when a threshold is crossed, which means the system has already degraded by the time anyone knows. The response is reactive.

**Why existing tools don't solve it:** Threshold-based alerting is structurally reactive. It waits for a metric to exceed a boundary. Some tools add anomaly detection (Datadog Watchdog, Dynatrace Davis), but these are statistical pattern matchers that identify "this looks different from normal." They don't explain why it's different or predict what happens next.

**What system physics adds:** The engine computes the system's trajectory from its current state variables. If the load factor is approaching the stability boundary and accelerating, the engine can predict that failure will occur before the threshold is crossed. More importantly, it can explain the causal mechanism: "memory pressure is causing GC pause increases, which are widening latency variance, which will trigger retry amplification within 15 minutes."

**What it cannot do:** Predict failures caused by events it has no signal for. An external network outage, a cloud provider incident, or a zero-day exploit won't show up in the system's state variables until the impact propagates to observable metrics. The prediction window starts when the first observable metric shifts, not before.

## 2. Data Pipeline Quality Monitoring

**The problem:** Large data environments (1,000+ sourcetypes flowing through collection pipelines into analytics platforms) have silent quality failures. A host stops sending logs. A parsing rule breaks. A routing change duplicates events. Nobody notices until an audit or a downstream analysis produces wrong results.

**Why existing tools don't solve it:** Most pipeline monitoring tracks volume only. If data is flowing, the dashboard is green. Tools like TrackMe for Splunk attempt per-sourcetype monitoring but at massive compute cost (potentially hundreds of thousands of dollars annually) because they poll each sourcetype independently.

**What system physics adds:** Quality measurement across five dimensions (accuracy, consistency, timeliness, validity, uniqueness) from normalized metrics, with per-channel regime detection that distinguishes volume health from quality health. A single firewall host stopping within a pool of 200 is detected by learned emission baselines, not by per-host alerting rules. See [DATA_QUALITY.md](DATA_QUALITY.md) for the full framework.

**What it cannot do:** Detect quality problems in sourcetypes that produce no measurable metadata. If a source sends data with no error counters, no upstream generation counts, and no timestamp metadata, the quality dimensions that depend on those signals cannot be computed. The framework is transparent about what it can measure in any given environment.

## 3. Change Validation

**The problem:** When a team upgrades a runtime, migrates a framework, or changes infrastructure configuration, the validation process is "run benchmarks, look at dashboards, decide if it looks better." This misses systemic effects. A JavaScript engine upgrade might reduce average latency but increase tail latency under load because the new GC heuristics trade consistency for throughput. Dashboards comparing averages would show an improvement. Production under sustained load would reveal a regression.

**Why existing tools don't solve it:** Benchmarking tools measure individual metrics in isolation. Load testing tools stress the system but evaluate results as metric comparisons rather than systemic analysis. Neither asks "did the system's overall stability envelope change?"

**What system physics adds:** A thermodynamic profile comparison. Before the change: compute all state variables under representative load. After the change: compute them again. The comparison shows not just which metrics changed but whether the stability envelope expanded or contracted, whether the regime transition threshold moved, and whether the system can handle the same peak load with the same margin. The output is a provable statement: "this upgrade expanded the stability envelope by 11% at median load but narrowed the margin before GC-driven regime transition by 4% at peak load."

**What it cannot do:** Validate changes for load profiles it hasn't observed. If the change is tested under synthetic load that doesn't represent production traffic patterns, the validation applies to the synthetic profile, not to production. The physics layer is explicit about which load profile the validation was computed against.

## 4. Demand Characterization for Orchestration

**The problem:** Kubernetes autoscalers (HPA, KEDA, Karpenter) react to metrics after they've changed. CPU hits 80%, scale up. Queue depth exceeds threshold, add consumers. This is structurally late. Worse, in some failure modes, reactive scaling makes things worse. A retry storm drives CPU up. HPA adds pods. The new pods join the retry storm. CPU rises further. The autoscaler is feeding the failure.

**Why existing tools don't solve it:** Every autoscaler in the ecosystem operates on the same principle: observe a metric, compare to a threshold, react. None of them model the system's physics. None can distinguish genuine demand increase from artificial amplification. None compute whether scaling up will help or hurt.

**What system physics adds:** True demand characterization. The engine computes the actual computational cost of the workload, including amplification from retries, feedback loops, and structural inefficiency. It emits a demand signal that existing autoscalers can consume as a custom metric. When the demand signal says "this is amplification, not genuine load," the autoscaler can hold instead of scaling. When it says "this is genuine demand approaching capacity," the autoscaler can scale proactively before the threshold is crossed.

**What it cannot do:** Replace the autoscaler. System physics computes the signal. The autoscaler acts on it. The physics layer does not manage pods, nodes, or infrastructure directly. It makes the existing orchestration layer smarter by giving it better inputs.

## 5. Governance and Compliance

**The problem:** Compliance today is performed manually. A compliance team reviews dashboards, checks reports, interviews engineers, and produces an attestation based on their interpretation of the evidence. This is labor-intensive, inconsistent across reviewers, and fundamentally based on opinion rather than proof.

**Why existing tools don't solve it:** Observability tools produce dashboards, not proofs. A dashboard that shows green for the last 12 months doesn't prove data was complete. It proves the dashboard was green. There may have been silent quality degradation that the dashboard wasn't configured to detect.

**What system physics adds:** Continuous, computable attestation. For compliance-bound data flows, the physics layer maintains a running quality record across all five dimensions with per-interval scoring. Any quality gap is recorded with timestamps, affected dimensions, estimated impact, and causal attribution when available. The output is an auditable evidence chain: "this sourcetype maintained 99.97% DQI over the last 12 months, with two completeness gaps totaling 14 minutes, both attributed to upstream configuration changes at these specific times."

AI translates this evidence chain into compliance documents that auditors can evaluate without understanding the underlying mathematics.

**What it cannot do:** Guarantee that auditors will accept DQI-based attestation. No regulatory framework currently recognizes mathematical quality scoring as a compliance standard. The physics layer produces stronger evidence than manual review, but the compliance team must evaluate whether that evidence satisfies their specific regulatory requirements. We believe mathematical attestation is more rigorous than dashboard review. The regulatory world has not yet caught up to that position.

## 6. AI Infrastructure Management

**The problem:** Organizations are rapidly scaling AI workloads (inference serving, model training, fine-tuning pipelines, RAG systems) without the same operational maturity they've built for traditional services. GPU utilization, inference latency, batch queue depth, memory pressure from large models, training pipeline reliability: these are system physics problems that most teams are monitoring with generic infrastructure metrics that don't capture the dynamics of AI workloads.

**Why existing tools don't solve it:** Traditional monitoring tracks GPU utilization as a percentage and inference latency as a number. It doesn't model the relationship between batch size, memory pressure, and latency distribution. It doesn't predict when a model serving endpoint will degrade under increasing load. It doesn't distinguish between a healthy inference queue and one that's approaching a throughput cliff.

**What system physics adds:** The same mathematical framework applied to AI workloads. Entropy production from inference requests, dissipation capacity from GPU compute and memory bandwidth, load factor trajectories that predict when serving endpoints will degrade. For training pipelines: flow analysis of data preprocessing stages, conservation equations for training data completeness, and stability validation of the training loop itself.

This creates a natural adoption path. Organizations scaling their AI footprint need to manage those workloads responsibly. System physics gives them the same mathematical rigor for AI infrastructure that it provides for traditional services. The layer that governs your applications also governs your AI.

**What it cannot do:** Evaluate model quality, training convergence, or inference correctness. System physics reasons about the infrastructure that runs AI, not about the AI itself. Whether a model is producing good outputs is a different problem. Whether the infrastructure serving that model is stable, efficient, and correctly scaled is a system physics problem.
