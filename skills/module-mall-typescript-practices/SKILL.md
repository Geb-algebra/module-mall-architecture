---
name: module-mall-typescript-practices
description: Use when implementing, changing, reviewing, or explaining Module Mall Architecture boundaries in a TypeScript repository that uses npm-compatible workspace packages, package.json exports, and static import analysis.
---

# Module Mall TypeScript Practices

## Scope

Apply these rules when each architectural module can be represented by an npm-compatible workspace package. A workspace package is a code comprehension and change boundary; it need not be a deployment, process, repository, or team boundary.

These rules cover TypeScript/npm mechanisms that preserve change cohesion, implementation hiding, black-box usability, and dependency traceability. They do not prescribe unrelated compiler strictness, module format, test framework, package publishing, versioning policy, directory naming, or an absolute ban on dependency cycles.

## Workspace package boundary

Represent each module as a workspace package with its own `package.json`, root-level `README.md`, source, root-level `contract-tests/`, and any internal tests. Put a dependency in the manifest of the package that uses it. Root dependencies are only repository-wide tooling and do not replace package-local declarations.

The package boundary follows ownership of change reasons. Do not create packages solely from technical categories such as `utils`, `logics`, or `components` when that disperses a rule, and do not force independent change reasons into one package merely because they deploy together.

## Package README

Use the `README.md` at each package root as the source of truth for responsibility, non-responsibility, meaning, and intent. It must make these facts discoverable without reading implementation:

- responsibility and non-responsibility;
- rules and judgments owned by the package;
- changes that cause the package to change;
- changes that do not cause it to change;
- public entrypoints and externally observable errors or side effects when declarations alone are insufficient;
- non-import dependencies that cannot be represented as TypeScript symbols.

Describe owned decisions, not a file inventory or internal algorithm. Every responsibility-bearing package change must be explainable by a documented change reason. The README, exported API, and contract tests are separate views of one current public contract and must agree. An external architecture catalog may index or summarize packages, but it does not replace the package-root README.

## Public API and `exports`

List every supported entrypoint explicitly in `package.json#exports`. Export only APIs that consumers need and the package intends to maintain.

```json
{
  "name": "@app/todo",
  "exports": {
    ".": "./src/index.ts",
    "./events": "./src/events.ts"
  }
}
```

Point `exports` directly to intentional TypeScript source entrypoints so the source contract is the workspace contract. Use explicit named exports. Avoid wildcard subpath exports and broad `export *` barrels when they make internal additions public accidentally. Do not expose internal persistence shapes, helper functions, implementation errors, or third-party details merely for convenience.

Pointing `exports` to built JavaScript or declaration files is permitted only when publishing the package to npm, executing it in an environment that cannot consume TypeScript, or another documented non-MMA constraint requires build artifacts. Record that reason and the affected entrypoints in the package README. Without such a reason, built-output-only `exports` violate this rule.

Separate entrypoints by consumer purpose, such as operations and event contracts, only when each subpath is an intentional contract.

## Import boundary

Cross-package imports may use only the target package name and a declared `exports` entrypoint.

```ts
// Allowed
import { TodoBecameOverdue } from "@app/todo/events";

// Violations
import { TodoBecameOverdue } from "@app/todo/src/internal/events.js";
import { TodoBecameOverdue } from "../../todo/src/internal/events.js";
```

Within one package, relative imports of internal files remain valid. Across packages, prohibit:

- deep imports into unexported paths;
- relative, absolute, or file-URL imports into another package;
- TypeScript `paths` aliases or package import aliases that reach another package's internals;
- re-exports, dynamic imports, or CommonJS loads that bypass the same boundary.

Check the resolver's actual target path and its owning workspace package. String-pattern linting is useful feedback but cannot be the sole guarantee because aliases and alternate specifiers can hide the destination.

## Contract tests

Place all public contract tests in `contract-tests/` directly under the workspace package root. Do not place them under `src/` or mix them with internal implementation tests. Cover every promised part of the public contract: public type structure, inputs, boundary and failure conditions, guarantees, public errors, state changes, events, and side effects. Contract tests must import through the same exported specifiers available to consumers.

```text
packages/todo/
├── package.json
├── README.md
├── src/
├── contract-tests/
└── test/                 # optional internal implementation tests
```

```ts
import { createTodo } from "@app/todo";

it("rejects an empty title", () => {
  expect(() => createTodo({ title: "" })).toThrow();
});
```

Test compile-time contracts as consumer TypeScript code, including accepted and rejected usage. Runtime tests alone cannot verify promised input types. TypeScript source tests through the public package entrypoint are sufficient for MMA contract verification; emitted `.d.ts`, packed packages, and built-consumer tests are outside this rule even when another publishing or runtime policy requires them.

Internal tests may import internal code, but they do not count as contract coverage. Keep them outside `contract-tests/` and distinguishable from public contract tests.

## Manifest fidelity

Declare every directly used workspace package in the consumer package's manifest. Do not rely on root installation or a transitive dependency. Compare resolved imports with manifests in both directions:

- a cross-package import has a direct declaration;
- a declaration claimed as architectural dependency has corresponding use;
- runtime use is not available only as a development dependency;
- a dependency exposed by the public TypeScript API is declared and resolvable by consumers;
- test-only or tooling-only use is classified separately from runtime use.

Use the verified workspace manifests to construct the package dependency graph and trace reverse direct and transitive consumers of a changed public contract.

## Implicit dependency channels

Turn semantic dependencies expressed only by duplicated strings, shapes, or conventions into references to symbols owned by the provider package whenever possible.

```ts
// @app/todo/events
export const TodoBecameOverdue = defineEvent<{
  todoId: string;
  dueAt: Date;
}>("todo.became-overdue");

// Both publisher and subscriber import the same descriptor.
import { TodoBecameOverdue } from "@app/todo/events";
```

Apply the same ownership rule to event payloads, DI tokens, schemas, parsers, queue or resource descriptors, and registries. Prefer explicit handler imports and registration over discovery by glob or naming convention. Validate data arriving across runtime boundaries with an owned runtime schema when TypeScript types are unavailable at runtime.

Do not mistake adding an import for creating a dependency: if a consumer already relies on an event's meaning or data shape, the symbol makes the existing semantic dependency observable.

When a dependency cannot become an import—external APIs, environment variables, queues, shared files, or startup order—record its consumer, owner, contract or resource, and reason in the package-root README. Documentation is a fallback, not a substitute for a statically checkable symbol.

## Static enforcement

Continuously enforce workspace boundaries and dependency declarations with static checks covering every workspace package. Resolve each reference to its actual file and owning package; do not rely only on import-string patterns.

The checks must reject undeclared cross-package dependencies, deep imports, filesystem imports, aliases and alternate loading forms that bypass package boundaries, manifest dependencies that do not match use or dependency kind, and external access not listed in `exports`. No package may be excluded from these checks. Do not permit a manual-review-only exception path; a justified architectural tradeoff must still be represented in a form the checks can validate.

## Invariants

- **TS-MMA-01 Package ownership**: Every workspace package's code and exported API are explainable by change reasons documented in its package-root README.
- **TS-MMA-02 Source exports**: Every supported external specifier is intentionally listed in `exports` and points directly to a TypeScript source entrypoint, unless a documented non-MMA publishing or runtime constraint requires build artifacts; no internal path is externally reachable as a supported API.
- **TS-MMA-03 Resolved boundary**: Every resolved cross-package code reference lands on a declared public entrypoint, regardless of specifier syntax or aliasing.
- **TS-MMA-04 Contract agreement**: The package-root README, exported source entrypoints, and contract tests describe the same current public contract.
- **TS-MMA-05 Isolated consumer-view tests**: Every promised runtime and type-level behavior has a test through public entrypoints in the package-root `contract-tests/` directory; tests under `src/` and internal tests are not counted as contract evidence.
- **TS-MMA-06 Manifest/import agreement**: Resolved workspace imports and direct manifest declarations agree, including dependency kind.
- **TS-MMA-07 Graph reachability**: Verified manifests can enumerate direct and transitive consumer packages for any changed workspace package contract.
- **TS-MMA-08 Owned shared symbols**: Events, tokens, formats, schemas, and resources shared across packages have one owner and a common imported symbol whenever TypeScript can express them.
- **TS-MMA-09 Recorded runtime edges**: Semantic dependencies that imports cannot express identify their consumer, owner, relied-upon resource or contract, and dependency reason.
- **TS-MMA-10 Universal static enforcement**: Static boundary and dependency checks cover every workspace package and every supported loading form, with no excluded package or manual-review-only exception path.
