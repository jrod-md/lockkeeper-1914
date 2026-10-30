# Requirements Document

## Introduction

The interlocking engine is a generic, domain-agnostic TypeScript library that lives in `engine/`. It evaluates whether an action on a control handle is allowed given the current state and a set of declarative rules, and it exhaustively searches the reachable state space to determine whether any hazardous state can be reached. The engine is shared by the web app, CLI, and MCP server. It has no UI, no Node.js-specific dependencies, no Panama Canal logic, and no cloud infrastructure logic. Allow/deny decisions are made exclusively by the deterministic engine; no LLM, prompt, or heuristic participates.

## Glossary

- **Engine**: The pure TypeScript library in `engine/` that evaluates rules and explores state spaces. Has no runtime npm dependencies and no UI or Node.js-specific imports.
- **Model**: The complete, immutable description of a system: its Controls, initial State, Rules, Hazards, and active Keys. Constructed by a trusted human caller before any query is made.
- **Control**: A named handle with a finite, ordered list of named Positions and a designated initial Position.
- **Position**: A named value that a Control can occupy at any point in time.
- **State**: A compact encoding of the current Position of every Control in a Model, represented as a single JavaScript safe integer (≤ 53 bits).
- **Action**: A triple (control id, from position, to position) that moves one Control from its current Position to another.
- **Rule**: A declarative constraint object — either a `requires` rule or an `atMost` rule — that carries an id, description, source, status, and an enabled flag. Rules are never mutated at runtime.
- **Literal**: A (control id, position) pair that is true when that Control occupies that Position in the given State. A Literal may be negated, meaning it is true when that Control does NOT occupy that Position.
- **Requires_Rule**: A Rule that, when all its `when` Literals hold in the resulting State, requires all its `then` Literals to also hold.
- **AtMost_Rule**: A Rule with an optional `when` guard that requires at most `k` Literals from a specified set to hold in the resulting State. `k` may be zero, meaning none of the Literals may hold when the guard is active.
- **Key**: A configuration object, supplied only by a trusted human caller when constructing the Model, that disables one or more named Rules for the lifetime of that Model evaluation. Keys are not accepted at query time.
- **Hazard**: An object with a unique id, description, source string, and a pure boolean function `isSatisfied(state)` that the caller supplies to express a domain-specific dangerous condition.
- **Checker**: The breadth-first search procedure inside the Engine that determines whether any Hazard is reachable from the initial State under the active rule set.
- **Counterexample_Path**: The shortest sequence of allowed Actions from the initial State to a State that satisfies a Hazard predicate, as returned by the Checker.
- **CheckResult**: The value returned by `check(model, options?)`: one of `"no-hazard-reachable"`, `"hazard-reachable"` (with a Counterexample_Path), or `"inconclusive"` (search limit reached).
- **ExploreResult**: The value returned by `explore(model, options?)`: either `"complete"` (with the full set of reachable States) or `"inconclusive"` (search limit reached).
- **Test_Suite**: The set of Vitest + fast-check property-based tests that verify the Engine's correctness properties using randomly generated small models. Generated models MUST include negated Literals, `atMost` rules both with and without `when` guards, and Keys.

## Requirements

### Requirement 1: Model Definition

**User Story:** As a rule-set author, I want to define a model with controls, positions, and an initial state, so that the engine can evaluate actions and explore reachable states.

#### Acceptance Criteria

1. THE Engine SHALL accept a Model containing one or more Controls, where each Control has a unique id, a non-empty list of named Positions, and an initial Position that is a member of that list.
2. THE Engine SHALL derive the initial State of a Model from the initial Position of every Control.
3. WHEN two Controls in the same Model share the same id, THE Engine SHALL reject the Model with a descriptive error.
4. WHEN a Control's initial Position is not a member of its declared Position list, THE Engine SHALL reject the Model with a descriptive error.
5. THE Engine SHALL encode State as a single JavaScript number (a 53-bit safe integer). WHEN a Model's state space requires more bits than can be represented as a JS safe integer, THE Engine SHALL reject the Model with a descriptive error naming the offending controls and required bit width.
6. WHEN the initial State violates any enabled Rule (after applying active Keys), THE Engine SHALL reject the Model with a descriptive error listing the violated rules.

---

### Requirement 2: Action Definition

**User Story:** As a rule-set author, I want to define actions that move a single control to a new position, so that the checker can enumerate all possible transitions.

#### Acceptance Criteria

1. THE Engine SHALL represent an Action as a triple of (control id, from position, to position), where `from` and `to` are distinct Positions of that Control.
2. WHEN an Action references a control id that is not present in the Model, THE Engine SHALL reject it with a descriptive error.
3. WHEN an Action specifies a `to` position that is not in the Control's declared Position list, THE Engine SHALL reject it with a descriptive error.
4. THE Engine SHALL enumerate all valid Actions for a given State as every (control, current_position, other_position) triple derivable from the Model definition.
5. THE Engine SHALL enumerate Actions in declared control order (the order controls appear in the Model), then in declared position order within each control.

---

### Requirement 3: Interlock Rule — Requires

**User Story:** As a rule-set author, I want to express "when these handle positions hold, those other handle positions must also hold" as a declarative rule, so that the engine can block actions that would produce a forbidden state.

#### Acceptance Criteria

1. THE Engine SHALL support a `requires` rule with an id, description, source, status (`"documented"` or `"inferred"`), enabled flag, a non-empty `when` list of Literals, and a non-empty `then` list of Literals.
2. WHEN all `when` Literals of an enabled `requires` rule hold in the resulting State of an Action, THE Engine SHALL require all `then` Literals to hold in that same State.
3. IF any `then` Literal of an enabled `requires` rule does not hold in the resulting State, THEN THE Engine SHALL classify the Action as blocked by that rule.
4. WHEN the `when` condition of a `requires` rule does not hold in the resulting State, THE Engine SHALL not apply that rule's `then` constraint.
5. WHEN a `requires` rule has `enabled` set to `false`, THE Engine SHALL ignore it entirely during allow/deny evaluation.
6. WHEN a `requires` rule references a control id or position name not present in the Model, THE Engine SHALL reject the Model with a descriptive error naming the unknown reference.
7. WHEN two Rules in the same Model share the same `id`, THE Engine SHALL reject the Model with a descriptive error.

---

### Requirement 4: Interlock Rule — AtMost

**User Story:** As a rule-set author, I want to express "at most k of these handle positions may hold at the same time" as a declarative rule, so that mutually exclusive positions are enforced structurally.

#### Acceptance Criteria

1. THE Engine SHALL support an `atMost` rule with an id, description, source, status, enabled flag, a non-negative integer `k` (where `k = 0` means none of the Literals may hold when the guard is active), a non-empty set of Literals to count, and an optional `when` guard list of Literals.
2. WHEN an `atMost` rule has no `when` guard, or when all its `when` guard Literals hold in the resulting State, THE Engine SHALL count how many Literals in its set hold in that State.
3. IF the count exceeds `k`, THEN THE Engine SHALL classify the Action as blocked by that rule.
4. WHEN an `atMost` rule has a `when` guard and the guard does not hold in the resulting State, THE Engine SHALL not apply that rule's count constraint.
5. WHEN an `atMost` rule has `enabled` set to `false`, THE Engine SHALL ignore it entirely during allow/deny evaluation.
6. WHEN an `atMost` rule references a control id or position name not present in the Model, THE Engine SHALL reject the Model with a descriptive error naming the unknown reference.

---

### Requirement 5: Rule Metadata and Source Attribution

**User Story:** As a rule-set author, I want every rule to carry source attribution and a status flag, so that historical claims can be verified and the distinction between documented and inferred rules is always visible.

#### Acceptance Criteria

1. THE Engine SHALL require every Rule to carry a `source` field (article, page, and figure if applicable) and a `status` field with value `"documented"` or `"inferred"`.
2. WHEN `isAllowed` returns a blocked result, THE Engine SHALL include the `source` and `status` of every blocking Rule in the response, so callers can explain why an action was blocked.
3. THE Engine SHALL store rule metadata as read-only data and SHALL NOT modify it at runtime.

---

### Requirement 6: Keys (Human Override Configuration)

**User Story:** As a human operator, I want to configure named keys that disable specific rules, so that interlocks can be removed explicitly and visibly, triggering a re-check.

#### Acceptance Criteria

1. THE Engine SHALL support Key objects, each with a unique id, a description, and a non-empty list of rule ids that the Key disables while active.
2. WHEN a Key is applied to a Model, THE Engine SHALL treat every rule id listed by that Key as having `enabled = false` for the duration of that Model evaluation.
3. THE Engine SHALL accept active Keys as part of the Model configuration only (embedded in the Model object by the trusted human caller). There is no query-time parameter, no API flag, and no MCP tool that accepts Keys or activates overrides.
4. THE Engine SHALL NOT expose any API that allows an agent or automated process to activate a Key. Key activation is exclusively a configuration act performed by a human caller constructing the Model.
5. WHEN a Key lists a rule id that does not exist in the Model's rule set, THE Engine SHALL reject the Model with a descriptive error.
6. WHEN any Key is active during a `check` or `explore` call, THE Engine SHALL include the active Key ids in the result so callers can log them.

---

### Requirement 7: isAllowed API

**User Story:** As a caller (agent, UI, or CLI), I want to ask whether a specific action is allowed in the current state, so that I can present accurate feedback and enforce the interlocking policy.

#### Acceptance Criteria

1. THE Engine SHALL expose a pure function `isAllowed(state, action, model)` that returns `{ allowed: true }` when no enabled rule blocks the resulting state, or `{ allowed: false, blockedBy: Rule[] }` listing every rule that blocks it.
2. WHEN `isAllowed` is called with a State in which the current state already violates one or more enabled rules, THE Engine SHALL return an explicit error result `{ error: "inconsistent-state", violatedRules: Rule[] }` rather than evaluating the action.
3. THE Engine SHALL evaluate `isAllowed` without mutating the State, the Model, or any Rule.
4. THE Engine SHALL evaluate `isAllowed` in time proportional to the number of enabled rules, not the number of reachable states.
5. WHEN an action is blocked, THE Engine SHALL return all blocking rules, not just the first one found.
6. THE Engine SHALL NOT accept Keys or any override parameter in the `isAllowed` call signature. The active rule set is determined solely by the Model passed in.

---

### Requirement 8: Exhaustive Checker API

**User Story:** As a rule-set author or operator, I want to exhaustively verify whether any hazardous state is reachable from the initial state under the current rule set, so that I can assess the safety of the model within its encoded constraints.

#### Acceptance Criteria

1. THE Engine SHALL expose a pure function `check(model, options?)` where `options` may include `maxStates: number` (default 5,000,000).
2. THE Engine SHALL perform breadth-first search over all States reachable from the initial State via allowed Actions, up to the `maxStates` limit.
3. WHEN the initial State itself satisfies any Hazard predicate, THE Engine SHALL return `{ status: "hazard-reachable", hazard: Hazard, path: [], activeKeys: string[], statesExplored: 1 }` with an empty path.
4. WHEN no reachable State satisfies any Hazard predicate and the search completes before `maxStates` is reached, THE Engine SHALL return `{ status: "no-hazard-reachable", statesExplored: number, activeKeys: string[] }`.
5. WHEN a reachable State satisfies a Hazard predicate, THE Engine SHALL return `{ status: "hazard-reachable", hazard: Hazard, path: Action[], statesExplored: number, activeKeys: string[] }` where `path` is the shortest sequence of allowed Actions from the initial State to that hazardous State.
6. WHEN the `maxStates` limit is reached before the search completes and no hazardous state has been found, THE Engine SHALL return `{ status: "inconclusive", statesExplored: number, activeKeys: string[] }`. An incomplete search SHALL NEVER be reported as `"no-hazard-reachable"`.
7. THE Engine SHALL use BFS to guarantee that the returned path is the shortest path to the hazardous State.
8. WHEN multiple hazardous States exist at the same BFS depth, THE Engine SHALL return the first found under declared control order then declared position order.
9. Result status values (`"no-hazard-reachable"`, `"hazard-reachable"`, `"inconclusive"`) and any user-facing messages returned by the Engine SHALL NOT claim or imply that the system or its state is safe. Ordinary technical terms used in code or comments — such as "safe integer" or "type-safe" — are not restricted by this criterion.
10. THE Engine SHALL be callable from a Web Worker: it SHALL NOT use any DOM API or Node.js-specific API.

---

### Requirement 9: Checker with Disabled Rules (Interlock Removal)

**User Story:** As a demo operator, I want to run the checker with one or more rules disabled, so that the system shows the shortest path to a hazard that the removed interlock was preventing.

#### Acceptance Criteria

1. THE Engine SHALL accept a Model in which individual Rules have `enabled = false` (or have been disabled via active Keys), and SHALL run `check` as if those rules do not exist.
2. WHEN a rule is disabled and the checker is re-run, THE Engine SHALL re-explore all reachable states from the initial State under the reduced rule set.
3. WHEN removing a rule exposes a previously unreachable hazardous state, THE Engine SHALL return `{ status: "hazard-reachable" }` with the shortest counterexample path under the new rule set.
4. THE Engine SHALL NOT cache results across calls with different enabled rule sets unless the caller explicitly manages that caching.

---

### Requirement 10: Explore API

**User Story:** As a developer or test author, I want to enumerate all reachable states from the initial state without stopping at the first hazard, so that I can verify monotonicity of reachability and other global properties of the model.

#### Acceptance Criteria

1. THE Engine SHALL expose a pure function `explore(model, options?)` where `options` may include `maxStates: number` (default 5,000,000).
2. WHEN the search completes before `maxStates` is reached, THE Engine SHALL return `{ status: "complete", states: Set<State>, statesExplored: number, activeKeys: string[] }`.
3. WHEN the `maxStates` limit is reached before the search completes, THE Engine SHALL return `{ status: "inconclusive", statesExplored: number, activeKeys: string[] }` without a partial state set.
4. THE Engine SHALL enumerate states and actions in the same deterministic order used by `check`.
5. THE Engine SHALL NOT use any DOM API or Node.js-specific API, so `explore` is also callable from a Web Worker.

---

### Requirement 11: Hazard Predicate Contract

**User Story:** As a rule-set author, I want to supply hazard predicates as opaque functions, so that the engine can evaluate domain-specific danger conditions without coupling to any domain.

#### Acceptance Criteria

1. THE Engine SHALL accept Hazards as objects each containing a unique id, a description, a source string, and a pure boolean function `isSatisfied(state)`.
2. THE Engine SHALL evaluate Hazards in declared order (the order they appear in the Model's hazards list).
3. THE Engine SHALL call `isSatisfied` only with valid States derived from the Model, and SHALL NOT mutate the State passed to it.
4. WHEN a Hazard's `isSatisfied` throws an exception, THE Engine SHALL propagate the error to the caller of `check` or `explore` rather than silently ignoring it.
5. THE Engine SHALL treat the semantics of a Hazard as opaque: it SHALL NOT inspect or infer meaning from the hazard description or function body.

---

### Requirement 12: No Runtime Dependencies and Isolation

**User Story:** As a developer integrating the engine, I want zero runtime dependencies and strict isolation from domain rule sets, so that the library is portable, auditable, and free of circular imports.

#### Acceptance Criteria

1. THE Engine SHALL have no runtime npm dependencies. Vitest and fast-check are permitted as devDependencies only.
2. THE Engine SHALL use only TypeScript language features and standard ECMAScript built-ins available in modern browsers and Node.js 18+.
3. THE Engine SHALL compile under TypeScript strict mode with no type errors.
4. THE Engine SHALL NOT import from `app/`, `cli/`, `mcp/`, `rules/`, or any path outside `engine/`. Rule sets import the engine; the engine never imports rule sets.

---

### Requirement 13: Property-Based Test Coverage

**User Story:** As a developer, I want property-based tests using fast-check on randomly generated small models, so that the engine's correctness properties hold across the space of possible inputs.

#### Acceptance Criteria

1. THE Test_Suite SHALL verify that every Action in a path returned by `check` is allowed from the preceding State, and that the final State satisfies the reported Hazard, for all randomly generated models where `check` returns `status: "hazard-reachable"`.
2. THE Test_Suite SHALL verify that the path returned by `check` is the shortest path to a hazardous State, by comparing it against a naive brute-force search on randomly generated small models.
3. THE Test_Suite SHALL verify that random walks of allowed Actions from the initial State never reach a hazardous State, for all randomly generated models where `check` returns `status: "no-hazard-reachable"`.
4. THE Test_Suite SHALL verify monotonicity of reachability: disabling any single enabled rule in a model never reduces the set of States returned by `explore()` (when both the original and modified model return `status: "complete"`), for randomly generated models.
5. THE Test_Suite SHALL verify that calling `check` twice on the same Model returns identical results (determinism), for randomly generated models.
6. THE Test_Suite SHALL generate models using fast-check arbitraries with at least 3 controls and at least 2 positions per control. Generated models MUST include negated Literals, `atMost` rules both with and without `when` guards, and Keys.
7. THE Test_Suite SHALL run each property with at least 100 examples.
