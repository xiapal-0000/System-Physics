# Model: Mathematical Framework

## What This Document Is

This document defines the mathematical state variables and reasoning framework that the system physics layer uses. It is not a textbook. It is a specification: what is computed, from what inputs, and what the outputs mean.

The math draws from multiple disciplines: information theory, control theory, queueing theory, graph theory, conservation laws, and statistical inference. No single branch of mathematics is sufficient. The framework uses whatever math is correct for the problem.

## Primary State Variables

### Entropy Production Rate (Ṡ)

**What it represents:** The rate at which a system produces disorder through its operations. Every computation, every network call, every state mutation, every retry is an irreversible operation that contributes to entropy production. Under healthy operation, entropy production is stable and proportional to useful work. Under degraded operation, entropy production grows faster than useful work because the system is doing more and more work to accomplish less and less.

**How it is estimated:** From observable metrics that correlate with irreversible work: request rate weighted by processing cost, error rate (failed work produces entropy without useful output), retry rate (repeated work multiplies entropy production), allocation rate (memory churn produces GC work that is pure entropy). The specific mapping from metrics to Ṡ depends on what the system exposes. More instrumentation produces a more accurate estimate.

**Honest limitation:** Ṡ is an estimate, not a direct measurement. Physical entropy production could theoretically be measured in watts of heat dissipation, but that's not practical for distributed software systems. The estimate is based on proxy metrics that correlate with entropy production. The correlation is strong for well-instrumented systems and weaker for poorly instrumented ones.

### Dissipation Capacity (D)

**What it represents:** The rate at which a system can absorb and dissipate entropy to maintain stability. This is determined by the system's structural properties: available CPU and memory headroom, backpressure mechanisms (circuit breakers, rate limiters), garbage collection capacity, queue buffer sizes, and connection pool limits. A system with high dissipation capacity can absorb load spikes without regime change. A system with low dissipation capacity enters degradation quickly.

**How it is estimated:** From resource utilization headroom and structural capacity metrics. Available CPU capacity above current utilization, available memory above current working set, queue buffer space remaining, connection pool availability. Also from observed recovery behavior: after a perturbation, how quickly does the system return to its baseline state? Fast recovery indicates high dissipation capacity. Slow recovery indicates the system is operating near its capacity limit.

**Honest limitation:** Some dissipation capacity is invisible to external metrics. A JVM's internal GC heuristics, a kernel's memory management behavior, a network switch's buffer management: these contribute to dissipation capacity but aren't directly observable. The estimate captures what's measurable. The gap between estimated and actual dissipation capacity is one reason the model's predictions are sometimes wrong.

### Thermodynamic Load Factor (λ = Ṡ / D)

**What it represents:** The ratio of entropy production to dissipation capacity. This is the single most important state variable. When λ is well below 1, the system is producing entropy slower than it can dissipate: stable operation. When λ approaches 1, the system is near its stability boundary. When λ exceeds 1, the system cannot dissipate entropy fast enough and degradation is mathematically inevitable.

**Why it matters:** λ unifies all the individual metrics into a single question: is this system operating within its capacity to remain stable? A system can have high CPU utilization (Ṡ is high) and still be stable if its dissipation capacity is proportionally high. Another system can have low CPU utilization but be approaching failure because its dissipation capacity is low (small buffers, no backpressure, no circuit breakers).

### First and Second Derivatives (dλ/dt, d²λ/dt²)

**What they represent:** The trajectory of the load factor. dλ/dt tells you whether the system is improving or degrading. d²λ/dt² tells you whether degradation is accelerating.

A system at λ = 0.7 with dλ/dt = 0 is stable. A system at λ = 0.7 with dλ/dt = +0.05/min is degrading. A system at λ = 0.7 with dλ/dt = +0.05/min and d²λ/dt² > 0 is degrading and accelerating, meaning the rate of degradation is itself increasing. The trajectory matters as much as the current position.

## Regime Classification

Regime is a function of λ and its derivatives. It is not a separate computation, it is derived from the state variables above.

**Green (Stable):** λ well below 1, dλ/dt approximately 0. The system is in steady state with margin. Normal operation.

**Amber (Degrading):** λ approaching 1, or dλ/dt positive and sustained, or d²λ/dt² positive (degradation accelerating). The stability envelope is narrowing. The system may recover on its own if the perturbation is transient, or it may continue toward failure.

**Red (Critical):** λ at or above 1, or the trajectory makes crossing 1 inevitable within the prediction horizon given current acceleration. Failure is mathematically certain without intervention. The prediction horizon is configurable and depends on how far ahead the engine extrapolates from current derivatives.

**Honest limitation on regime classification:** The thresholds between green, amber, and red are partially learned from the system's own history (the adaptive stability envelope) and partially set by configuration. Different systems have different effective thresholds. A system with robust backpressure mechanisms might tolerate λ = 0.95 comfortably. A system with no backpressure might cascade at λ = 0.85 because a discrete internal threshold was crossed. The engine learns these per-system boundaries over time, but early in observation the thresholds are approximate.

## Conservation Equations

For data flows, conservation laws apply: events entering a pipeline stage must equal events exiting plus known drops. Violations indicate duplication (more out than in) or silent loss (fewer out than in, minus drops).

The conservation delta at each stage is:

```
Δ = events_out - (events_in - known_drops - intentional_filtering)
```

Where Δ should be zero for a correctly functioning stage. Intentional fan-out is accounted for by incorporating the declared routing topology. A stage configured for 1:3 fan-out has an expected ratio of 3:1 events_out to events_in, which is not a conservation violation.

Persistent non-zero Δ indicates a pipeline defect. The sign indicates the type: positive Δ means duplication, negative Δ means unaccounted loss.

## Adaptive Stability Envelopes

Each service or channel develops a learned stability envelope over time. The envelope captures:

- Normal operating range for each state variable under typical load.
- Daily and weekly patterns (a service that's busier on weekdays has a different weekday envelope than weekend envelope).
- Recovery characteristics: how quickly the system returns to baseline after perturbation.
- Effective regime transition thresholds observed from past behavior.

The envelope is learned from observed behavior using statistical methods. This is the one place where learned inference (rather than mathematical derivation) is appropriate in the framework, because you're establishing what "normal" looks like for a specific system, which requires observation rather than derivation.

**Honest limitation:** Envelopes take time to converge. A new service or a service that was recently restructured won't have a reliable envelope for approximately 7 days of observation. During this period, regime classification uses default thresholds that may not match the system's actual behavior. The engine is explicit about when it's operating on learned envelopes versus defaults.

## Where the Model is an Approximation

**Hidden state.** Real systems have internal state that isn't visible in external metrics. JVM GC heuristics, kernel scheduler decisions, network buffer management. The model reasons over what it can observe. When the model predicts stability and the system fails, the gap usually indicates hidden state that the metrics don't capture.

**Discrete thresholds in continuous models.** The state variables are computed as continuous values, but real systems have discrete thresholds: a connection pool has exactly 0 available connections, a process is either running or crashed, a circuit breaker is either open or closed. These create cliff-edge behaviors that smooth mathematical models don't capture perfectly. The model treats these as known phase boundaries where possible (if the connection pool size is known, that's a hard boundary in the stability envelope) and as unknown risks where the boundary isn't observable.

**Emergent behavior.** Composed distributed systems can exhibit emergent behavior that isn't derivable from individual component analysis. Two services that are individually stable can interact in ways that produce instability at the system level. The cross-service causal chain analysis addresses this partially, but complex emergent behaviors may not be captured until they're observed once and incorporated into the adaptive envelope.

The model does not claim to be a perfect representation of system behavior. It claims to be a rigorous, inspectable, improvable approximation that produces better conclusions than threshold-based alerting or human intuition at scale. When it's wrong, the discrepancy is informative: it tells you something about the system that the current metrics don't capture.
