---
name: module-mall-architecture
description: Use when designing, changing, reviewing, or explaining module boundaries and dependencies so that change impact is small and discoverable, independent of programming language, framework, package manager, or deployment model.
---

# Module Mall Architecture

## Scope

Apply this context to source-code modules bounded by directories, packages, components, or equivalent structures. Use it to judge architecture during design, implementation, change analysis, review, testing, and explanation.

This context defines language- and tool-independent properties. Workspace layouts, manifests, export maps, linters, build tools, deployment units, and framework conventions are possible enforcement mechanisms, not the architecture itself. Do not require a module boundary to equal a process, service, repository, team, or deployment boundary.

## Core model

A **change request** describes a desired behavioral or design change without naming its code locations. Its **scope** is every part of the codebase that may need modification, inspection, or testing for the activity at hand.

For a modular system, scope contains:

1. Modules directly responsible for the change reasons in the request.
2. Modules potentially affected by changes to those modules, transitively.

Module Mall Architecture makes both sets small and discoverable. Optimize this only while preserving justified requirements such as performance, security, runtime constraints, and team ownership. Treat avoidable dispersion, propagation, and search cost as design debt; treat necessary exceptions as explicit tradeoffs whose dependencies remain traceable.

## Terms

- **Module**: Source code grouped behind an identifiable boundary.
- **Change reason**: A rule or concern whose change causes code to change.
- **Public API**: Functions, types, classes, events, or equivalent program elements usable by another module.
- **Public contract**: The public API plus its inputs, preconditions, guarantees, errors, side effects, and other externally observable behavior.
- **Internal implementation**: Code not promised to other modules.
- **Public contract artifact**: A compact, trustworthy description of the contract, such as responsibility documentation, public declarations, schemas, or contract tests.
- **Dependency**: Reliance by one module on another module's contract, including calls, types, events, data formats, configuration, resources, naming conventions, or runtime interactions.

## Required properties

### Change cohesion

Group code by change reason. One owner module defines each rule; consumers use the owner's public contract. Keep code that changes for the same reason together, and keep independently changing reasons separate.

A technical layer such as `utils`, `components`, `frontend`, or `backend` is not a sufficient boundary when it disperses one rule. A change request may legitimately involve multiple modules when it contains multiple independent change reasons; this does not violate cohesion.

### Implementation hiding

Separate the minimal public contract from internal implementation. Other modules may depend only on the public contract and must not bypass the boundary through internal paths, storage representations, private schemas, or equivalent mechanisms.

When the public contract remains stable, internal changes must not require dependent modules to change. Minimize the contract itself because every promised element expands the possible impact surface.

### Black-box usability

Make responsibility and externally observable behavior understandable without reading internal implementation. Public contract artifacts must reveal responsibility and non-responsibility; owned and excluded change reasons; available public API and its meaning; and relevant inputs, preconditions, guarantees, errors, and side effects.

Artifacts must be materially cheaper to inspect than the implementation, have an obvious location or navigation path, and remain consistent with the current contract. A change request's relationship to a module must be decidable from these artifacts before inspecting the module's internal implementation.

### Dependency traceability

Make every inter-module dependency explicit enough to identify direct and transitive impact candidates. Prefer statically analyzable symbols, declarations, schemas, or descriptors over duplicated strings, naming conventions, and independently redefined data shapes.

Dependency declarations must match actual dependencies. Detect or prohibit undeclared dependencies, boundary bypasses, and hidden dependency channels. When a dependency cannot be represented as a code-level reference, record it in a structured, discoverable form that identifies the dependent module, dependency owner, relied-upon contract or resource, and reason for the dependency.

## Invariants

All applicable modules must satisfy these observable conditions:

- **MMA-01 Single ownership**: Each change reason has one identifiable owner module; the same rule is not independently defined by multiple modules.
- **MMA-02 Separated reasons**: Code in a module can be explained by a coherent set of jointly changing reasons; unrelated reasons are not accumulated in generic containers.
- **MMA-03 Contract-only access**: Cross-module use reaches only declared public contracts; no cross-boundary reference reaches internal implementation.
- **MMA-04 Minimal contract**: Every public element has an external use consistent with the module's responsibility; implementation conveniences are not public.
- **MMA-05 Stable-contract containment**: A change that preserves a module's public contract can be completed without changing consumers solely because of that module's internal details.
- **MMA-06 Discoverable responsibility**: Responsibility, non-responsibility, owned change reasons, and excluded change reasons can be determined from public contract artifacts.
- **MMA-07 Observable behavior documented**: Inputs, preconditions, guarantees, errors, and side effects promised to consumers are represented in public contract artifacts.
- **MMA-08 Trustworthy artifacts**: Public contract artifacts agree with the implemented contract and are smaller and faster to inspect than internal implementation.
- **MMA-09 Complete direct dependencies**: Every semantic cross-module dependency has a declared edge from consumer to owner, including non-call and runtime dependencies.
- **MMA-10 Declaration fidelity**: Declared edges correspond to actual use, actual cross-module use is declared, and the declared dependency kind matches the use.
- **MMA-11 Traceable contracts**: Shared events, tokens, formats, schemas, and resources have an identifiable owner and a common reference when the technology permits static representation.
- **MMA-12 Transitive reachability**: Direct dependency information is sufficient to enumerate all modules that may be transitively affected by a public-contract change.
- **MMA-13 Request mapping before internals**: Whether a change request relates to a module can be determined from its public contract artifacts before reading internal implementation.
- **MMA-14 Complete non-symbol records**: Every dependency that cannot be represented by a code-level reference records the dependent module, dependency owner, relied-upon contract or resource, and dependency reason.

## Tradeoffs

A higher-priority runtime, performance, security, legal, or ownership constraint may justify weakening one or more properties. Treat a deviation as an intentional tradeoff only when all of these observable conditions hold:

- the prioritized constraint and the reason the normal rule cannot be satisfied are recorded;
- the remaining impact paths, affected ownership, and responsible owners are identifiable;
- the exception is limited to explicitly identified modules and contracts.

An undocumented exception, an exception whose remaining impact paths cannot be traced, or a deviation removable without harming the prioritized constraint is a violation of the affected invariant, not an intentional tradeoff.

## Judgment examples

| Situation | Judgment |
|---|---|
| A validation rule is repeated in UI and server modules | Violates change cohesion; one module must own the rule even if enforcement occurs in multiple places. |
| A request changes both expiration policy and its visual presentation | Two owner modules may change because the request contains two change reasons. |
| A consumer reads another module's database tables directly | Violates implementation hiding unless that schema is an intentional public contract owned and governed as such. |
| Two modules exchange an event using duplicated string names and shapes | Violates dependency traceability because the semantic dependency lacks a shared, owned reference. |
| Runtime constraints require duplicated logic | The duplication may be a justified tradeoff, but ownership, synchronization obligations, and dependency edges must remain explicit. |
| A module README lists files but not owned decisions | Insufficient black-box usability; inventory does not explain relation to a change request. |

## Relationship among the properties

Change cohesion reduces the number and size of directly responsible modules. Implementation hiding reduces propagation from internal changes. Black-box usability makes directly responsible modules discoverable without reading their internals. Dependency traceability makes affected consumers discoverable when public contracts change.

None substitutes for another: hidden implementation without understandable responsibility still causes broad search; documentation without enforced boundaries is not trustworthy; explicit dependencies do not compensate for dispersed ownership.
