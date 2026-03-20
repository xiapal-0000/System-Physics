# Use Cases

## Scope

These are the concrete applications of the system physics layer. Each one describes a real problem, why existing tools don’t solve it, and what the physics layer adds. Each one also states what the physics layer cannot do in that domain.

## 1. Predictive Failure Detection

**The problem:** Production incidents are detected after they’ve started. Monitoring tools alert when a threshold is crossed, which means the system has already degraded by the time anyone knows. The response is reactive.

**Why existing tools don’t solve it:** Threshold-based alerting is structurally reactive. It waits for a metric to exceed a boundary. Some tools add anomaly detection (Datadog Watchdog, Dynatrace Davis), but these are statistical pattern matchers that identify “this looks different from normal.” They don’t explain why it’s different or predict what happens next.

**What system physics adds:** The engine computes the system’s trajectory from its current state variables. If the load factor is approaching the stability boundary and accelerating, the engine can predict that failure will occur before the threshold is crossed. More importantly, it can explain the causal mechanism: “memory pressure is causing GC pause increases, which are widening latency variance, which will trigger retry amplification within 15 minutes.”

**What it cannot do:** Predict failures caused by events it has no signal for. An external network outage, a cloud provider incident, or a zero-day exploit won’t show up in the system’s state variables until the impact propagates to observable metrics. The prediction window starts when the first observable metric shifts, not before.

## 2. Data Pipeline Quality Monitoring

**The problem:** Large data environments (1,000+ sourcetypes flowing through collection pipelines into analytics platforms) have silent quality failures. A host stops sending logs. A parsing rule breaks. A routing change duplicates events. Nobody notices until an audit or a downstream analysis produces wrong results.

**Why existing tools don’t solve it:** Most pipeline monitoring tracks volume only. If data is flowing, the dashboard is green. Tools like TrackMe for Splunk attempt per-sourcetype monitoring but at massive compute cost (potentially hundreds of thousands of dollars annually) because they poll each sourcetype independently.

**What system physics adds:** Quality measurement across five dimensions (accuracy, consistency, timeliness, validity, uniqueness) from normalized metrics, with per-channel regime detection that distinguishes volume health from quality health. A single firewall host stopping within a pool of 200 is detected by learned emission baselines, not by per-host alerting rules. See <DATA_QUALITY.md> for the full framework.

**What it cannot do:** Detect quality problems in sourcetypes that produce no measurable metadata. If a source sends data with no error counters, no upstream generation counts, and no timestamp metadata, the quality dimensions that depend on those signals cannot be computed. The framework is transparent about what it can measure in any given environment.

## 3. Change Validation

**The problem:** When a team upgrades a runtime, migrates a framework, or changes infrastructure configuration, the validation process is “run benchmarks, look at dashboards, decide if it looks better.” This misses systemic effects. A JavaScript engine upgrade might reduce average latency but increase tail latency under load because the new GC heuristics trade consistency for throughput. Dashboards comparing averages would show an improvement. Production under sustained load would reveal a regression.

**Why existing tools don’t solve it:** Benchmarking tools measure individual metrics in isolation. Load testing tools stress the system but evaluate results as metric comparisons rather than systemic analysis. Neither asks “did the system’s overall stability envelope change?”

**What system physics adds:** A thermodynamic profile comparison. Before the change: compute all state variables under representative load. After the change: compute them again. The comparison shows not just which metrics changed but whether the stability envelope expanded or contracted, whether the regime transition threshold moved, and whether the system can handle the same peak load with the same margin. The output is a provable statement: “this upgrade expanded the stability envelope by 11% at median load but narrowed the margin before GC-driven regime transition by 4% at peak load.”

**What it cannot do:** Validate changes for load profiles it hasn’t observed. If the change is tested under synthetic load that doesn’t represent production traffic patterns, the validation applies to the synthetic profile, not to production. The physics layer is explicit about which load profile the validation was computed against.

## 4. Demand Characterization for Orchestration

**The problem:** Kubernetes autoscalers react to metrics after they’ve changed. CPU hits 80%, scale up. Queue depth exceeds threshold, add consumers. This is structurally late. Worse, in some failure modes, reactive scaling makes things worse. A retry storm drives CPU up. HPA adds pods. The new pods join the retry storm. CPU rises further. The autoscaler is feeding the failure.

**Why existing tools don’t solve it:** Every autoscaler in the ecosystem operates on the same principle: observe a metric, compare to a threshold, react. None of them model the system’s physics. None can distinguish genuine demand increase from artificial amplification. None compute whether scaling up will help or hurt.

**What system physics adds:** True demand characterization. The engine computes the actual computational cost of the workload, including amplification from retries, feedback loops, and structural inefficiency. It emits a demand signal that existing autoscalers can consume as a custom metric. When the demand signal says “this is amplification, not genuine load,” the autoscaler can hold instead of scaling. When it says “this is genuine demand approaching capacity,” the autoscaler can scale proactively before the threshold is crossed.

**What it cannot do:** Replace the autoscaler. System physics computes the signal. The autoscaler acts on it. The physics layer does not manage pods, nodes, or infrastructure directly. It makes the existing orchestration layer smarter by giving it better inputs.

## 5. Governance and Compliance

**The problem:** Compliance today is performed manually. A compliance team reviews dashboards, checks reports, interviews engineers, and produces an attestation based on their interpretation of the evidence. This is labor-intensive, inconsistent across reviewers, and fundamentally based on opinion rather than proof.

**Why existing tools don’t solve it:** Observability tools produce dashboards, not proofs. A dashboard that shows green for the last 12 months doesn’t prove data was complete. It proves the dashboard was green. There may have been silent quality degradation that the dashboard wasn’t configured to detect.

**What system physics adds:** Continuous, computable attestation. For compliance-bound data flows, the physics layer maintains a running quality record across all five dimensions with per-interval scoring. Any quality gap is recorded with timestamps, affected dimensions, estimated impact, and causal attribution when available. The output is an auditable evidence chain: “this sourcetype maintained 99.97% DQI over the last 12 months, with two completeness gaps totaling 14 minutes, both attributed to upstream configuration changes at these specific times.”

AI translates this evidence chain into compliance documents that auditors can evaluate without understanding the underlying mathematics.

**What it cannot do:** Guarantee that auditors will accept DQI-based attestation. No regulatory framework currently recognizes mathematical quality scoring as a compliance standard. The physics layer produces stronger evidence than manual review, but the compliance team must evaluate whether that evidence satisfies their specific regulatory requirements. We believe mathematical attestation is more rigorous than dashboard review. The regulatory world has not yet caught up to that position.

## 6. AI Infrastructure Management

**The problem:** Organizations are rapidly scaling AI workloads (inference serving, model training, fine-tuning pipelines, RAG systems, and increasingly autonomous agents) without the same operational maturity they’ve built for traditional services. GPU utilization, inference latency, batch queue depth, memory pressure from large models, training pipeline reliability: these are system physics problems that most teams are monitoring with generic infrastructure metrics that don’t capture the dynamics of AI workloads.

This is not theoretical. In March 2026, an AI agent at Meta acted without authorization, posted responses an engineer hadn’t approved, gave incorrect guidance that another employee acted on, and inadvertently exposed sensitive company and user data to unauthorized engineers for two hours. Meta classified it as a Sev 1 incident. Separately, a Meta safety director reported that her AI agent deleted her entire inbox despite being instructed to confirm before taking any action. These are computable systems operating outside their declared behavioral boundaries with no computational layer detecting the deviation before the damage was done.

**Why existing tools don’t solve it:** Traditional monitoring tracks GPU utilization as a percentage and inference latency as a number. It doesn’t model the relationship between batch size, memory pressure, and latency distribution. It doesn’t predict when a model serving endpoint will degrade under increasing load. It doesn’t distinguish between a healthy inference queue and one that’s approaching a throughput cliff. And critically, for autonomous agents, it doesn’t model behavioral boundaries: what actions an agent is authorized to take, whether its outputs conform to its declared constraints, or whether its interaction pattern has deviated from its expected envelope.

**What system physics adds:** The same mathematical framework applied to AI workloads. Entropy production from inference requests, dissipation capacity from GPU compute and memory bandwidth, load factor trajectories that predict when serving endpoints will degrade. For training pipelines: flow analysis of data preprocessing stages, conservation equations for training data completeness, and stability validation of the training loop itself.

For autonomous agents specifically: behavioral envelope detection. An agent has a declared set of authorized actions, interaction constraints, and output boundaries. These are mathematically definable. When an agent takes an action outside its authorized set, that’s a conservation violation: it produced an output that shouldn’t exist given its constraints. When an agent ignores a confirmation requirement, that’s a regime shift: the agent’s behavioral state has left its stability envelope. The physics layer detects these deviations from the same state variables it uses for any other system, because an AI agent is a computable system with observable behavior, not a special case.

This creates a natural adoption path. Organizations scaling their AI footprint need to manage those workloads responsibly. System physics gives them the same mathematical rigor for AI infrastructure that it provides for traditional services. The layer that governs your applications also governs your AI.

**What it cannot do:** Evaluate model quality, training convergence, or inference correctness. System physics reasons about the infrastructure that runs AI, not about the AI itself. Whether a model is producing good outputs is a different problem. Whether the infrastructure serving that model is stable, efficient, and correctly scaled is a system physics problem.

## 7. Invisible Change Detection and Vendor Validation

**The problem:** Software changes happen without your knowledge. Cloud providers update underlying infrastructure without customer notification. SaaS vendors push backend changes that alter API behavior or performance characteristics. Runtime updates modify GC heuristics. Library patches change memory allocation patterns. The changelog says “bug fixes and improvements” or says nothing at all. Your system behaves differently and nobody knows why, because the change was invisible to your team.

This also happens in reverse. Vendors ship updates that they believe are safe, based on their internal benchmarks, but the update introduces regressions that only manifest under specific customer workload patterns that the vendor’s test suite didn’t cover.

**Why existing tools don’t solve it:** Monitoring tools detect that something changed in your metrics, but they can’t distinguish between an internal cause and an external one. If latency increased, was it your code, your traffic, your infrastructure, or something the vendor changed? Without a physics model that accounts for all internal state, the external signal can’t be isolated. And vendors have no standardized way to validate that their changes don’t degrade customer stability envelopes, because no tool computes stability envelopes.

**What system physics adds:**

For customers: invisible change detection. The adaptive stability envelope knows the system’s thermodynamic profile. When the profile shifts and there is no corresponding internal change event (no deployment, no config change, no traffic pattern shift), the physics layer can conclude that something external changed. The output is specific: “the system’s processing cost per request increased by 14% starting at 03:00 UTC with no internal change event. Dissipation capacity decreased by 8%. The stability envelope narrowed. The most likely explanation is an external change to the underlying platform or a dependency.” That gives the customer mathematical evidence to bring to their vendor, not a vague complaint.

For vendors before GA: a stability gate. Run the physics layer against a representative customer workload profile on the current version. Run it again on the candidate release. The thermodynamic profile comparison proves whether the stability envelope expanded or contracted, whether new failure modes were introduced, whether tail latency behavior changed under load, and whether dissipation capacity was affected. This catches regressions that benchmarks miss because benchmarks test throughput and latency in isolation, not the system’s holistic stability characteristics under realistic load.

For vendors after GA: continuous customer impact monitoring. If a vendor runs system physics against their own platform, they can detect when their own changes degrade the thermodynamic profile of the workloads they serve, before customers report it. The vendor discovers the regression from the math, not from a support ticket.

**What it cannot do:** Identify exactly what the external change was. The physics layer can prove that the thermodynamic profile shifted due to an external cause and quantify the impact. It cannot tell you which specific binary the cloud provider updated or which line of code the SaaS vendor changed. It proves that something changed and measures the effect. The vendor must identify the specific cause.

## 8. Security Breach Detection from Thermodynamic Deviation

**The problem:** Every security tool in the market works by matching patterns: known signatures, known behaviors, known attack vectors. They detect what they’ve been trained to detect. A novel attack that doesn’t match any known signature passes through undetected. Zero-day exploits exist precisely because signature-based detection can’t catch what it’s never seen. The entire security industry is in a reactive arms race: attackers develop new methods, defenders update their signatures, attackers adapt. The defender is always one step behind.

Meanwhile, breaches that go undetected for weeks or months are common. The attacker is inside the system, doing work (exfiltrating data, mining crypto, moving laterally, establishing persistence), and existing tools don’t flag it because the work doesn’t match a known pattern.

**Why existing tools don’t solve it:** Signature-based detection (antivirus, IDS/IPS) matches known threat patterns. Behavioral analytics (UEBA, EDR) learns user and process behavior patterns and flags deviations, but operates at the application and user level, not at the system physics level. SIEM platforms aggregate logs and correlate events but depend on detection rules written by humans who anticipated the attack vector. None of these approach the problem from first principles: is this system doing work it wasn’t designed to do?

**What system physics adds:** A fundamentally different detection model. Instead of asking “does this match a known threat?” system physics asks “is this system producing entropy that its legitimate workload doesn’t account for?”

An attacker or compromised process is doing work. That work is physical. It consumes CPU cycles, allocates memory, generates network traffic, performs disk operations. Every one of these is an irreversible operation that produces measurable entropy. The system’s thermodynamic profile shifts in ways that don’t correlate with any legitimate cause.

The detection works like this:

The adaptive stability envelope knows the system’s normal thermodynamic profile: how much entropy it produces per unit of legitimate work, what its resource utilization patterns look like across daily and weekly cycles, what its network flow characteristics are, what its memory allocation patterns are.

When a compromise occurs, the system starts doing work that isn’t explained by its legitimate inputs. Processing cost per request increases without a corresponding change in request complexity. Memory allocation patterns shift without a deployment. Network flow characteristics change without a traffic pattern change. CPU utilization rises without throughput increasing proportionally. The entropy production rate increases, but the useful work output doesn’t.

The physics layer detects this as a thermodynamic anomaly: “this system’s entropy production increased by 23% at 02:17 UTC with no deployment, no traffic change, no configuration event, and no external platform change. The additional work is not producing useful output. The ratio of entropy production to useful work has diverged from the learned baseline.”

This detection is agnostic to the attack method. It doesn’t matter whether the attacker is running a crypto miner, exfiltrating data, performing reconnaissance, or establishing persistence. All of these activities produce entropy that the system wasn’t designed to produce. The physics layer doesn’t need to know what the attack looks like. It needs to know what the system looks like when nothing is wrong, and detect when the system is doing work it shouldn’t be.

**How this differs from existing behavioral analytics:** UEBA and EDR operate at the user and process level: “this user logged in at an unusual time” or “this process spawned an unexpected child process.” System physics operates at the thermodynamic level: “this system is doing more irreversible work than its inputs justify.” These are complementary signals at different layers of abstraction. A sophisticated attacker can mimic normal user behavior and avoid triggering behavioral rules. They cannot perform computational work without producing entropy. The work is physical. The physics is measurable.

**The integration with security tools:** System physics doesn’t replace SIEM, EDR, or UEBA. It feeds them a signal they don’t currently have. When the physics layer detects unexplained entropy production, that signal is emitted to the security stack as a high-priority anomaly with mathematical evidence: the magnitude of the deviation, the time of onset, the specific state variables that shifted, and the absence of any legitimate explanation. The security team investigates from there. The physics layer narrows the search from “something might be wrong somewhere” to “this specific system started doing unexplained work at this specific time, and here’s exactly how much.”

**Why this is groundbreaking:** The security industry has spent decades trying to enumerate every possible attack and build a signature for each one. That approach can never be complete because the attacker has infinite creativity. System physics inverts the problem. Instead of trying to recognize every threat, it recognizes when a system’s physics are wrong. The set of possible attacks is infinite. The set of legitimate thermodynamic profiles for a given system is finite and learnable. Detecting deviation from a known good profile is a fundamentally more tractable problem than enumerating every possible bad pattern.

**What it cannot do:** Tell you what the attack is, who the attacker is, or how they got in. System physics detects that anomalous work is happening and quantifies it. It does not perform forensic investigation, malware analysis, or attribution. It is a detection layer that catches what signature-based tools miss, not a replacement for the investigation and response workflow that follows detection.

It also cannot detect attacks that produce negligible computational work. A configuration change made by a compromised account that doesn’t alter the system’s runtime behavior won’t shift the thermodynamic profile. Attacks that operate purely at the data level (modifying database records, altering configurations) without measurably changing the system’s processing characteristics are outside the physics layer’s detection capability. This is a real limitation, not a theoretical one.
