# Adapters: Integration Architecture

## Principle

System physics is a layer, not a silo. Its value is proportional to what it connects to. The design principle: any system that produces metrics can feed the physics layer, and any system that consumes signals can act on physics-derived outputs.

The physics layer does not require any specific vendor, platform, or toolchain. It meets your infrastructure where it already is.

## What the Physics Layer Consumes

The physics layer needs observable state: metrics, rates, counts, distributions, topology, and change events. It does not need event data, log content, or trace payloads. It consumes normalized signals about system behavior, not the raw artifacts of that behavior.

Input sources fall into three categories:

**Application telemetry.** Runtime metrics like latency distributions, error rates, GC behavior, thread pool utilization, allocation rates. These provide the deepest view into application-level physics. OpenTelemetry is the natural protocol here, but any system that exports application metrics can serve as a source.

**Infrastructure metrics.** Resource utilization, pipeline throughput, queue depth, dropped events, network health, container state. These come from the platforms you already run: your monitoring stack, your orchestrator, your data pipeline tooling, your cloud provider's native metrics.

**Context signals.** Deployment events, configuration changes, topology shifts, incident status. These aren't metrics in the traditional sense but they're essential for causal attribution. When the physics layer detects a regime shift, context signals help it explain why.

## What the Physics Layer Emits

The physics layer produces derived state: mathematical conclusions about system behavior that other tools can consume and act on.

**Regime classification.** The current state of a service or flow channel (stable, degrading, critical) with the mathematical basis for that classification.

**Load factor and trajectory.** Where the system sits relative to its stability boundary and whether it's moving toward or away from that boundary.

**Recurrence potential.** The likelihood that a current condition will recur under load, derived from the gap between entropy production and dissipation capacity.

**Data quality index.** Composite quality scoring across five dimensions for data flow channels.

**Orchestration signals.** Physics-derived scaling and routing recommendations grounded in true demand characterization rather than raw metric thresholds.

**Change validation verdicts.** Whether a system change improved or degraded the stability envelope.

**Causal attribution.** When something degrades, the mathematical evidence chain explaining what changed first, what amplified the problem, and why the system could not recover.

These outputs are designed to be consumed by tools that already exist in your stack: your alerting system, your orchestrator, your governance tooling, your CI/CD pipeline. The physics layer reasons. Your existing tools act.

## Ingestion Tiers

Not every environment has the same instrumentation maturity. The physics layer works at three levels of depth:

**OTel-only.** Application-level telemetry provides the deepest view into application physics. Best for teams that have already invested in OpenTelemetry. Strongest application-level prediction and root cause analysis.

**Adapter-only.** Infrastructure metrics from existing tools provide broad coverage with lower application-internal depth. Best for teams that want value without deploying new instrumentation. Lower onboarding friction. Higher coverage breadth.

**Hybrid.** Both combined. Application telemetry correlated with infrastructure metrics provides the full picture. This is where the layer produces its strongest conclusions because it can trace causality across the full stack.

Each tier is useful independently. The hybrid tier is where system physics does things no other approach can, because it correlates signals that currently live in disconnected tools.

**Honest constraint on tiers:** The tier you can run depends on your current instrumentation. If you don't have OTel deployed, the OTel tier isn't available to you without investment. The adapter tier is designed to work with what you already have, but "what you already have" varies enormously across organizations. Some teams have rich Prometheus metrics on everything. Others have minimal cloud provider metrics and not much else. The physics layer's depth of analysis is directly proportional to the richness of its inputs.

## Adapter Architecture

An adapter bridges an external system and the physics layer. It has three responsibilities:

**Collect.** Retrieve metrics from the external system via its native API, export format, or event stream.

**Normalize.** Translate the external system's metric format into the physics layer's state variable model. A Prometheus counter becomes an entropy production signal. A cloud provider's CPU metric becomes a dissipation capacity input. The normalization step is where domain-specific metrics become physics-layer state.

**Emit.** Deliver normalized signals to the physics layer's intake.

**What building an adapter actually involves:** It is real engineering work. Each adapter must understand the source system's API, authenticate correctly, handle rate limits and failures, normalize metrics into the physics layer's model, and stay maintained as the source system evolves. The framework defines the normalization contract. The adapter developer must understand both the source system and the physics model well enough to map between them correctly.

The adapter contract, the specific interface definition, normalization requirements, and protocol specifications, will be published when the reference implementation exits validation. The architecture is described here so that the integration model is understood from the outset.

## Operational Reality

Every integration layer has ongoing operational cost. Adapters break when source systems change their APIs. Normalization logic needs updating when new metric types are introduced. Authentication tokens expire. Rate limits get hit.

The framework is designed to minimize this cost (adapters are stateless, normalization is declarative where possible, health checks are built in), but it does not eliminate it. Running system physics means running adapters, and running adapters means maintaining integrations. This is the same operational cost as any other integration in your stack, not more, not less.

## Ecosystem Intent

System physics is designed to integrate with every layer in the infrastructure stack through open, documented interfaces that any tool can implement.

The goal is not to own the integration surface. It is to define an interface clear enough that the community and vendors can build their own adapters without depending on us. The physics layer becomes more valuable as more sources feed it and more tools consume its outputs. That value should be open to everyone.

When the adapter contract is published, it will be open and permissively licensed. We want every observability, orchestration, and governance tool to integrate with the system physics layer, and we intend to make that integration as straightforward as possible.
