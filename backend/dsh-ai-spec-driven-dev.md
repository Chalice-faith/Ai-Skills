# Skill: dsh-ai-spec-driven-dev

**Location:** `.agents/skills/dsh-ai-spec-driven-dev/SKILL.md`

---

## Purpose

This skill codifies the principles of **AI programming** and **specification-driven development** used in the DeepSeek Harness codebase.

Its purpose is to make AI-assisted changes:

- **Predictable** — behavior follows explicit contracts rather than hidden assumptions.
- **Auditable** — important decisions and model-visible state can be reconstructed later.
- **Maintainable** — architecture, configuration, lifecycle, and tests remain understandable over time.
- **Reversible** — registrations and side effects have explicit cleanup paths.
- **Verifiable** — important contracts are enforced mechanically by tests and CI.

Apply this skill whenever you are:

- Adding a new feature or capability.
- Modifying model-visible inputs or outputs.
- Introducing a new plugin, service, or tool.
- Designing or changing configuration schemas.
- Changing persistence or wire formats.
- Modifying error handling that affects observable behavior.
- Writing tests for product-, agent-, or user-visible behavior.

---

# 1. Classify the Change First

Before editing code, classify the requested change.

The classification determines which parts of this skill are mandatory.

## Level A — Mechanical

Examples:

- Typo fixes.
- Formatting.
- Comment corrections.
- Mechanical renames with no semantic change.
- Refactors that provably preserve externally observable behavior.

Requirements:

- Keep the change minimal.
- Run relevant static checks/tests.
- No Agent Note is required unless a non-obvious decision is introduced.
- No snapshot update is required if observable behavior is unchanged.

## Level B — Behavioral

Examples:

- Changing business logic.
- Changing configuration.
- Changing a public API.
- Changing error behavior.
- Changing persistence behavior.
- Changing user-visible output without changing the agent/model boundary.

Requirements:

- Apply relevant core principles.
- Create an Agent Note when the change contains a design decision, trade-off, new contract, or migration concern.
- Add or update appropriate tests.
- Add/update snapshots when observable product behavior changes.
- Review documentation and public contracts.

## Level C — Architectural or Model-Visible

Examples:

- Adding a new capability.
- Adding a tool.
- Adding a plugin or provider.
- Modifying prompts.
- Modifying model-visible context.
- Adding fields to model requests.
- Changing tool results visible to the model.
- Changing session/event formats.
- Changing agent-loop behavior.

Requirements:

- Create an Agent Note.
- Define capability boundaries explicitly.
- Audit model-visible logging and replayability.
- Add/update behavioral snapshots.
- Review configuration resolution.
- Review lifecycle/disposal.
- Update relevant architecture documentation.
- Run the full applicable verification suite.

When uncertain between two levels, choose the higher level only when the change actually crosses that boundary. Do not inflate scope merely because a nearby subsystem is complex.

---

# 2. Minimal Change Surface

## Rule

Implement the **smallest coherent change** that fully satisfies the requested specification.

Do not expand the task merely because adjacent code could also be improved.

## Do Not

Do not automatically:

- Refactor unrelated code.
- Rename unrelated symbols.
- Reformat unrelated files.
- Replace working abstractions.
- Introduce abstractions for hypothetical future requirements.
- Change public contracts unless required by the task.
- Fix unrelated bugs unless they prevent correct implementation.
- Perform opportunistic cleanup across neighboring modules.

## Adjacent Problems

If you discover an unrelated issue:

1. Determine whether it blocks correctness.
2. If it blocks correctness, make the smallest necessary fix and explain why it is required.
3. Otherwise, record or report it separately.
4. Do not silently expand the current change.

## Final Scope Check

Before completing the task, inspect the final diff and ask:

> Would every changed line be reasonably expected from the requested task or a necessary consequence of it?

If not, remove or separate the unrelated change.

---

# 3. Immutable Knowledge with Agent Notes

## Rule

For every non-trivial architectural or design decision, create an **Agent Note** under `.agents/notes/`.

Agent Notes are durable project memory.

Do not rely on conversational history or the current agent context to preserve important decisions.

## Why

AI memory and conversation context are temporary.

Important decisions must survive:

- New conversations.
- Different agents.
- Context-window truncation.
- Team-member changes.
- Refactors months later.

## Agent Note Lifecycle

An Agent Note has two conceptual states:

### Active

An active note may be edited while the decision is still being developed in the current change.

### Archived

Once the decision is accepted and the note is archived/merged as historical project knowledge, treat it as **immutable**.

Do not rewrite history by modifying an archived decision.

If an archived decision changes later:

1. Create a new Agent Note.
2. Reference the previous note.
3. Explain what changed and why.
4. Mark the new note as superseding the old decision where appropriate.

Conceptually:

```text
decision-v1
     ↓
superseded by
     ↓
decision-v2
```

The old note remains available as historical context.

## Skip Agent Notes For

Agent Notes are normally unnecessary for:

- Typo fixes.
- Formatting.
- Purely mechanical refactors.
- Changes with no semantic or architectural decision.

## Decision Rule

```text
IF change contains a non-obvious design decision
OR changes public API/config/persistence contracts
OR introduces a capability/tool/plugin
OR changes model-visible behavior
THEN create an Agent Note.
```

## Checklist

- [ ] Does this change contain a design decision or trade-off?
- [ ] Does it introduce a new contract?
- [ ] Does it affect API, configuration, persistence, tools, or model behavior?
- [ ] If yes, does an Agent Note capture the rationale?
- [ ] If replacing an older decision, does the new note reference it instead of rewriting history?

---

# 4. Model-Visible Inputs Must Be Logged

## Rule

Any data that reaches or influences a model request must be reconstructable from the session log.

Examples include:

- System prompts.
- User messages.
- Tool results.
- Model-visible context.
- Dynamically injected instructions.
- Capability descriptions.
- Model-visible configuration.
- Fields that influence model behavior.

Do not introduce hidden inputs that affect the model without recording them.

## Why

Given a historical session, a developer should be able to answer:

> What exactly did the model see?

Without this property, model behavior cannot be reliably replayed or debugged.

## Decision Rule

```text
IF a new value influences the model request
THEN
    identify its session event representation
    record it
    verify that replay reconstructs it.
```

## Required Property

Conceptually:

```text
Session Log
     ↓
   Replay
     ↓
Equivalent Model Request
```

The reconstructed request must contain all behaviorally relevant model-visible information.

## Checklist

- [ ] Does this change introduce or modify model-visible information?
- [ ] Is that information represented in the session log?
- [ ] Can replay reconstruct the model request?
- [ ] Are ordering and relevant metadata preserved?
- [ ] Is any behaviorally relevant input hidden outside the event history?

---

# 5. Snapshot Tests for Behavioral Changes

## Rule

For changes that alter **model outputs**, **agent behavior**, or **user-visible product behavior**, add or update a **keyless snapshot test** using a real runnable example.

Do not rely solely on mocks to validate observable behavior.

## Why

Unit tests prove local logic.

Snapshots prove what the product actually does.

A behavioral snapshot acts as:

- Regression protection.
- Executable documentation.
- Reviewable evidence of intentional behavior changes.

## Test Selection

Use the smallest appropriate test layer.

### Unit Test

Use for:

- Pure functions.
- Parsing.
- Resolution logic.
- Edge cases.
- Internal transformations.

### Integration Test

Use for:

- Provider/service boundaries.
- Persistence boundaries.
- Component integration.
- Serialization contracts.

### Snapshot Test

Use for:

- Agent-visible behavior.
- Model-visible behavior.
- User-visible output.
- Tool interaction behavior.
- Important observable product flows.

### End-to-End Test

Use when correctness depends on multiple real components working together and a smaller test cannot adequately verify the contract.

## Important

Snapshot tests do **not** replace unit or integration tests.

Use snapshots for observable behavior and focused tests for internal invariants.

## Decision Rule

```text
IF observable model/agent/user behavior changes
THEN add or update a behavioral snapshot.

IF only pure internal logic changes
THEN prefer focused unit/integration tests.

IF both change
THEN use both.
```

## Snapshot Requirements

Snapshots should:

- Use real runnable examples where practical.
- Avoid mocks as the sole behavioral verification.
- Be deterministic enough to review.
- Replay on supported platforms.
- Avoid unnecessary platform-specific normalization.

## Checklist

- [ ] Is observable behavior changing?
- [ ] Is a snapshot required?
- [ ] Are important internal invariants covered separately?
- [ ] Does the snapshot represent a realistic execution path?
- [ ] Does it work across supported environments?

---

# 6. Tool UI Intent Is a First-Class Design Decision

## Rule

When designing a tool, explicitly define its presentation intent.

Examples include:

- `generic`
- `terminal`
- `diff`
- `locations`

The rendering method must be a **pure function of its declared inputs**, normally `args`.

## Why

A tool is both:

```text
Tool
 ├─ Capability
 └─ Presentation
```

How a tool is rendered affects:

- User understanding.
- Trust.
- Debuggability.
- Replayability.
- Testability.

Presentation behavior must therefore be explicit rather than accidental.

## Decision Rule

```text
IF adding or changing a tool
THEN
    define renderIntent
    verify rendering purity
    verify presentation behavior with tests where relevant.
```

## Purity Requirement

Given the same rendering inputs:

```text
render(args) → same presentation
```

Rendering must not depend on hidden mutable global state.

## Checklist

- [ ] Does every new tool declare its `renderIntent`?
- [ ] Does the selected intent match the tool's semantic output?
- [ ] Is rendering deterministic?
- [ ] Is rendering free from unrelated side effects?
- [ ] Does rendering avoid hidden external state?

---

# 7. Capability Seam: Service Definition / Provider / Consumer

## Rule

Every significant capability must be separated into three roles:

1. **Service Definition** — interface and contract.
2. **Provider** — implementation of the capability.
3. **Consumer** — code that depends on the capability.

Examples of capabilities:

- Shell.
- Filesystem.
- Web search.
- Storage.
- Model access.
- External services.

## Structure

```text
Consumer
   ↓
Service Definition
   ↑
Provider A
Provider B
Provider C
```

Consumers depend on the service contract, not a concrete provider.

## Why

This enforces dependency inversion and allows providers to be replaced without rewriting consumers.

It also makes capability boundaries:

- Explicit.
- Testable.
- Documentable.
- Replaceable.

## Anti-Patterns

Avoid:

- Provider-specific logic inside consumers.
- One plugin containing definition, implementation, and consumption without a boundary.
- Ad-hoc global singletons.
- Consumers reaching around the service interface to access provider internals.

## Decision Rule

```text
IF introducing a new capability
THEN
    define its service contract
    implement provider(s)
    consume through the contract.
```

## Checklist

- [ ] Is this a new capability?
- [ ] Is the Service Definition explicit?
- [ ] Is the Provider separate from the Consumer?
- [ ] Can another provider be introduced without rewriting consumers?
- [ ] Is the public service contract documented?
- [ ] Are relevant event contracts documented?

---

# 8. Explicit Resolution Over Implicit Defaults

## Rule

Configurable values must be resolved through an explicit:

```text
resolve(request) → Spec
```

step at the package or subsystem boundary.

Do not hide configuration defaults inside execution methods such as `run()`.

## Preferred Flow

```text
Raw Configuration
       ↓
    resolve()
       ↓
Validated Spec
       ↓
Service Construction
       ↓
      run()
```

## Responsibilities of `resolve()`

Resolution may:

- Apply documented defaults.
- Resolve references.
- Validate configuration.
- Normalize representations.
- Detect incompatible combinations.
- Produce a complete runtime specification.

After resolution, downstream code should consume the resolved specification rather than repeatedly interpreting raw configuration.

## Anti-Pattern

Avoid:

```text
run()
 ├─ discover missing config
 ├─ apply hidden default
 ├─ silently repair config
 └─ continue
```

## Decision Rule

```text
IF behavior depends on configurable input
THEN resolve it before execution.

IF a required value cannot be resolved
THEN fail before starting the affected service whenever possible.
```

## Checklist

- [ ] Are defaults applied explicitly during resolution?
- [ ] Is raw configuration converted into a validated runtime spec?
- [ ] Does service execution avoid hidden fallback logic?
- [ ] Are configuration errors detected before execution when possible?

---

# 9. Trust Static Types, Validate at Boundaries

## Rule

Inside a typed process, trust invariants already guaranteed by the static type system.

Do not repeatedly validate values whose types already guarantee the required property.

Validate where external or untrusted data enters the typed system.

## Validation Boundaries

Examples include:

- Parser/config loader.
- Model JSON deserialization.
- Tool JSON deserialization.
- Durable file reads.
- Database reads/writes.
- Worker/process boundaries.
- Spawn boundaries.
- Network wire protocols.

Conceptually:

```text
External World
      ↓
 Validation
      ↓
Typed Internal System
```

## Why

Redundant defensive checks:

- Clutter internal logic.
- Obscure the actual contract.
- Duplicate validation rules.
- Encourage silent fallback.
- Make ownership of invariants unclear.

## Exception

If the type system cannot express an invariant, validate it at the nearest appropriate boundary.

Examples:

- Sorted arrays.
- Cross-field relationships.
- Semantic identifier formats.
- Numeric relationships.
- State-machine invariants.

## Decision Rule

```text
IF value comes from an external/untyped boundary
THEN validate it.

ELSE IF the static type system already guarantees the property
THEN trust the type.

ELSE IF the invariant cannot be represented statically
THEN validate it at the nearest owning boundary.
```

## Checklist

- [ ] Am I validating genuinely external data?
- [ ] Is validation happening at the earliest appropriate boundary?
- [ ] Am I duplicating checks already guaranteed by static types?
- [ ] Is a runtime guard enforcing an invariant the type system cannot express?

---

# 10. Fail Loud — No Silent Ignoring

## Rule

Configuration and dependency errors must fail at the earliest point where they can be conclusively detected.

Never silently ignore invalid configuration, missing referents, or required dependencies.

## Preferred Behavior

```text
Load
 ↓
Resolve
 ↓
Validate
 ↓
FAIL with actionable error
```

rather than:

```text
Load
 ↓
Run
 ↓
Discover problem
 ↓
Fallback / Ignore
 ↓
Undefined behavior later
```

## Error Quality

Errors should explain:

1. What is wrong.
2. Which configuration/dependency caused it.
3. What the user or developer can do to fix it.

## Exception Handling

Do not write broad silent catches such as:

```text
try
    ...
catch
    ignore
```

unless the ignored failure is explicitly expected, narrowly scoped, and documented.

## Decision Rule

```text
IF invalid state is self-contained and knowable during load/resolve
THEN fail during load/resolve.

ELSE fail at the earliest point where the missing information becomes resolvable.
```

## Checklist

- [ ] Is invalid configuration rejected?
- [ ] Are missing required dependencies rejected?
- [ ] Is failure occurring as early as possible?
- [ ] Is the error actionable?
- [ ] Is any exception being silently swallowed?

---

# 11. Effects Registration and Disposal

## Rule

Registrations and long-lived side effects must use the project's lifecycle mechanisms such as:

- `ctx.effect()`
- `ctx.on()`

and must have a corresponding disposal path.

Examples include:

- Event listeners.
- Plugin registrations.
- Service registrations.
- Commands.
- Watchers.
- Subscriptions.
- Timers.
- Long-lived resources.

## Lifecycle

Every registration should conceptually have:

```text
Register
   ↓
Active
   ↓
Dispose
   ↓
Clean State
```

## Why

Explicit disposal enables:

- Hot reload.
- Test isolation.
- Graceful shutdown.
- Plugin unloading.
- Resource cleanup.

## Anti-Patterns

Avoid:

- Registering global listeners without cleanup.
- Starting timers that cannot be cancelled.
- Registering services outside lifecycle management.
- Assuming process exit is sufficient cleanup.
- Cleanup paths that require hidden external state.

## Decision Rule

```text
IF code registers or starts a long-lived effect
THEN
    register through the lifecycle mechanism
    provide a disposer
    verify teardown behavior.
```

## Checklist

- [ ] Does the change register anything?
- [ ] Is registration lifecycle-managed?
- [ ] Does it return or define cleanup?
- [ ] Is cleanup executed during teardown?
- [ ] Can the component be initialized, disposed, and initialized again safely?

---

# 12. Specification as Code — Verifiable Contracts

## Rule

Important contracts should be mechanically enforceable by CI whenever practical.

Documentation alone is not sufficient for rules that can be checked automatically.

## Existing Mechanisms

Use project mechanisms such as:

- `verify-export-jsdoc`
- `test:coverage`
- `hygiene`
- `knip`
- `publint`
- workspace constraints
- `doc-sync`

## Principle

Prefer:

```text
Specification
     ↓
Types / Tests / Static Checks
     ↓
CI
     ↓
Pass / Fail
```

over:

```text
Documentation
     ↓
Developer remembers rule
     ↓
Maybe correct
```

## Examples

If:

> Public exports require JSDoc.

Then CI should reject undocumented exports.

If:

> Documentation must match generated contracts.

Then `doc-sync` should detect drift.

If:

> A behavior is a product contract.

Then tests or snapshots should detect unintentional changes.

## Decision Rule

```text
IF a new important contract can be mechanically checked
THEN prefer encoding that check in types, tests, lint, or CI.

Do not create unnecessary infrastructure for trivial one-off invariants.
```

## Checklist

- [ ] Does a new public export have required documentation?
- [ ] Is new functionality covered?
- [ ] Can the new contract be checked automatically?
- [ ] Is documentation synchronized with code?
- [ ] Are dependency/export constraints respected?

---

# 13. Operational Decision Matrix

Use this matrix before implementation.

| Change | Agent Note | Unit / Integration | Snapshot | Replay Audit | Capability Seam | Docs Review |
|---|---|---|---|---|---|---|
| Typo / formatting | No | Usually no | No | No | No | If affected |
| Pure internal logic | If design decision | Yes | Usually no | No | If applicable | If contract changes |
| Configuration | Usually | Yes | If behavior changes | If model-visible | If applicable | Yes |
| Public API | Yes | Yes | If observable | If model-visible | If applicable | Yes |
| Persistence format | Yes | Yes | If observable | If model-visible | If applicable | Yes |
| New provider | Yes | Yes | If observable | If model-visible | Yes | Yes |
| New capability | Yes | Yes | If observable | If model-visible | Yes | Yes |
| New tool | Yes | Yes | Yes | Yes | Usually | Yes |
| Prompt/model context | Yes | Focused tests | Yes | Yes | If applicable | Yes |
| Agent loop | Yes | Yes | Yes | Yes | Review | Architecture docs |

This table is guidance. The actual semantic impact of the change takes precedence over the label.

---

# 14. Standard Execution Workflow

When starting a task:

## Step 1 — Understand the Request

Identify:

- Requested behavior.
- Explicit constraints.
- Files/subsystems likely involved.
- What is explicitly out of scope.

Do not start by refactoring.

## Step 2 — Classify the Change

Choose:

```text
Level A
Level B
Level C
```

Use the classification to determine required artifacts and verification.

## Step 3 — Read Existing Project Knowledge

For non-trivial changes:

- Read relevant Agent Notes.
- Read relevant architecture documentation.
- Inspect existing nearby implementations.
- Prefer established project patterns over inventing new ones.

Do not rely on conversation memory when project documentation exists.

## Step 4 — Define the Contract

Before implementation, determine:

- Inputs.
- Outputs.
- Errors.
- Configuration.
- Ownership.
- Lifecycle.
- Observable behavior.

For new capabilities, define:

```text
Service Definition
Provider
Consumer
```

## Step 5 — Resolve Configuration

If configuration is involved:

```text
Raw Config
    ↓
 resolve()
    ↓
Validated Spec
```

Apply defaults and validation before execution.

## Step 6 — Identify Boundaries

Determine where external data enters:

- Config.
- JSON.
- Network.
- Storage.
- Model/tool payloads.
- Worker/process communication.

Validate there.

Trust internal static types afterward.

## Step 7 — Design Lifecycle

If introducing registrations or long-lived resources:

- Define initialization.
- Define ownership.
- Define disposal.
- Verify reinitialization is safe.

## Step 8 — Implement the Minimum Necessary Change

Follow the existing architecture.

Do not expand scope.

Do not introduce speculative abstractions.

## Step 9 — Verify Model Visibility

If anything affects the model:

- Add/update session events.
- Verify model-visible values are logged.
- Verify replayability.

## Step 10 — Select Tests

Use:

```text
Pure logic          → Unit
Component boundary  → Integration
Observable behavior → Snapshot
Critical full flow  → E2E
```

Use multiple layers when the change affects multiple contracts.

## Step 11 — Update Project Knowledge

If required:

- Create the Agent Note.
- Update relevant documentation.
- Do not rewrite archived decisions.
- Create a superseding note when a decision changes.

## Step 12 — Run Verification

Run applicable project checks, including:

```bash
pnpm run hygiene
pnpm run test:coverage
```

and relevant snapshot/documentation checks.

Do not claim a check passed unless it was actually run successfully.

## Step 13 — Review the Diff

Before finishing:

- Inspect changed files.
- Look for accidental edits.
- Remove unrelated formatting/refactors.
- Confirm every changed line belongs to the requested task or is a necessary consequence.

## Step 14 — Report Completion

Summarize:

- What changed.
- Important design decisions.
- Tests/checks performed.
- Any unresolved issue or follow-up discovered.

Do not hide failed checks.

---

# 15. Definition of Done

A task is complete only when all **applicable** items below are satisfied.

## Scope

- [ ] Requested behavior is implemented.
- [ ] No unrelated changes were introduced.
- [ ] The final diff has been reviewed.

## Architecture

- [ ] New capabilities use an explicit Service Definition / Provider / Consumer seam.
- [ ] Existing project patterns were followed unless intentionally superseded.
- [ ] Significant architectural decisions are documented.

## Knowledge

- [ ] Required Agent Note exists.
- [ ] Archived Agent Notes were not rewritten.
- [ ] Superseding decisions reference previous decisions.

## Configuration

- [ ] Configurable values are explicitly resolved.
- [ ] Defaults are visible in the resolution step.
- [ ] Invalid configuration fails early.
- [ ] Error messages are actionable.

## Type and Boundary Safety

- [ ] External inputs are validated at appropriate boundaries.
- [ ] Internal code trusts valid static types.
- [ ] Non-type-expressible invariants are checked at the nearest owning boundary.

## Model Behavior

- [ ] Model-visible inputs are represented in session logs.
- [ ] Historical model requests can be reconstructed sufficiently for replay/debugging.
- [ ] Model-visible behavior changes have appropriate snapshots.

## Lifecycle

- [ ] Registrations use lifecycle mechanisms.
- [ ] Long-lived effects have cleanup/disposal paths.
- [ ] Teardown does not leak listeners/resources.

## Testing

- [ ] Pure logic has focused tests where appropriate.
- [ ] Boundary behavior has integration coverage where appropriate.
- [ ] Observable behavior has snapshot coverage where required.
- [ ] Critical cross-component behavior has E2E coverage where justified.

## Contracts and Documentation

- [ ] New public exports satisfy documentation requirements.
- [ ] Relevant architecture/product documentation is synchronized.
- [ ] Mechanically checkable contracts are enforced by types/tests/static checks where practical.

## Verification

- [ ] Relevant tests pass.
- [ ] Relevant snapshots pass.
- [ ] `hygiene` passes.
- [ ] `test:coverage` passes.
- [ ] Documentation synchronization checks pass where applicable.
- [ ] No verification failure is being silently ignored.

Only applicable checks are required. Do not add unnecessary work solely to satisfy a checklist item that does not apply to the change.

---

# 16. Red Flags

Stop and reconsider the implementation if you are:

- Writing comments that only restate the code.
- Adding `TODO`, `FIXME`, or `XXX` without explaining why it exists and what resolves it.
- Using `any` without justification.
- Catching an exception without identifying what is intentionally swallowed and why.
- Adding hidden fallback behavior.
- Defining defaults inside `run()` instead of resolution.
- Validating internal values already guaranteed by static types.
- Introducing a provider dependency directly into consumers.
- Adding a capability without defining its boundary.
- Adding a tool without defining its render intent.
- Adding model-visible information without session logging.
- Changing observable behavior without considering snapshots.
- Registering listeners/services without disposal.
- Modifying archived Agent Notes.
- Refactoring unrelated code while implementing a focused task.
- Introducing an abstraction solely because it may be useful someday.
- Changing the agent loop without updating relevant architecture documentation.
- Claiming tests or checks passed without actually running them.
- Silently ignoring a failing test because it appears unrelated.

---

# 17. Compact Agent Decision Procedure

For every requested code change, execute this reasoning procedure:

```text
1. What exactly is being requested?
          ↓
2. Is it Mechanical, Behavioral, or Architectural/Model-visible?
          ↓
3. What is the smallest coherent change?
          ↓
4. Is there an existing Agent Note or project pattern?
          ↓
5. Does this introduce/change a capability?
   YES → Definition / Provider / Consumer
          ↓
6. Does this use configuration?
   YES → resolve() → validated Spec
          ↓
7. Does external data enter here?
   YES → validate at boundary
          ↓
8. Can invalid state be detected now?
   YES → fail loud now
          ↓
9. Does this create a long-lived effect?
   YES → registration + disposer
          ↓
10. Does this affect the model?
    YES → session event + replay audit
          ↓
11. Does observable behavior change?
    YES → snapshot
          ↓
12. What focused unit/integration tests are needed?
          ↓
13. Does this introduce an important decision?
    YES → Agent Note
          ↓
14. Can the contract be mechanically verified?
    YES → encode in types/tests/CI
          ↓
15. Run verification
          ↓
16. Review git diff
          ↓
17. Complete only when applicable Definition of Done items pass.
```

---

# Final Principle

The goal of this skill is **not** to maximize the amount of architecture, documentation, validation, or testing added to every change.

The goal is to make important behavior **explicit and verifiable** while keeping each change as small as reasonably possible.

Prefer:

> explicit contracts, narrow boundaries, reproducible behavior, durable decisions, reversible effects, and mechanically enforced specifications.

Avoid:

> hidden assumptions, silent fallback, speculative abstraction, undocumented architectural decisions, unreplayable model state, and unrelated refactoring.

When these principles conflict, optimize first for **correctness and explicit contracts**, then for **minimal change surface**, while preserving the existing architecture unless the specification requires changing it.