# Spec Delta

## Purpose

Defines the syntax, function-shape, warning and test-style rules an agent follows when it writes Scala under the `virtuslab-scala-style` rules file, stated without naming any build tool, formatter or library so that they hold in every Scala repository, on Scala 3 and on Scala 2.13.

## ADDED Requirements

### Requirement: Repository conventions win on syntax
WHEN a syntax rule in this capability conflicts with a convention the repository's code already follows or its formatter configuration enforces, the agent SHALL follow the repository. The rules on structure, organisation, modelling, errors and tests SHALL apply regardless of the repository's existing habits.

#### Scenario: A braced Scala 3 codebase
- **WHEN** the repository's Scala 3 code uses braces throughout
- **THEN** the agent writes braced code consistent with the surrounding file instead of introducing braceless syntax

#### Scenario: The formatter rewrites the agent's code
- **WHEN** the repository's formatter changes the layout of code the agent wrote
- **THEN** the agent keeps the formatter's output and does not undo it

#### Scenario: A design rule against the repository's habit
- **WHEN** the repository's existing classes hold mutable fields
- **THEN** the agent still adds none in the code it writes, because that rule is not a syntax rule

### Requirement: Scala 3 code is braceless
In Scala 3 code the agent SHALL use the indentation-based syntax for every definition and control structure it writes and SHALL NOT introduce braces, subject to the precedence of repository conventions.

#### Scenario: A new Scala 3 file
- **WHEN** the agent creates a file in a braceless Scala 3 repository
- **THEN** no definition or control structure in it uses braces

#### Scenario: A Scala 2.13 module
- **WHEN** the module compiles with Scala 2.13
- **THEN** the agent uses braces, since the language has no other syntax, and follows the repository's layout

### Requirement: Public members declare their return types
Every public `def`, `val` and `given` (or `implicit` definition on Scala 2.13) SHALL carry an explicit return type.

#### Scenario: A new public method
- **WHEN** the agent adds a public method
- **THEN** its result type is written out, even when the compiler could infer it

#### Scenario: A refactor changes an implementation
- **WHEN** a refactor would change the inferred type of a public member
- **THEN** the explicit type makes the compiler report the change instead of silently altering the signature

#### Scenario: A local value
- **WHEN** the agent declares a private or local `val`
- **THEN** it may leave the type inferred

### Requirement: Each function handles one concern
Each function SHALL do exactly one thing: either a single logical operation or a short orchestration of named steps, never both mixed. Responsibilities SHALL be split between entities named after what they do.

#### Scenario: A function that parses and persists
- **WHEN** one function both parses input and writes the result to storage
- **THEN** the agent splits it into a parsing function and a persisting function, orchestrated by a third if needed

#### Scenario: An orchestration
- **WHEN** a function sequences several steps
- **THEN** each step is a call to a named function and the orchestration contains no inline logic of its own

#### Scenario: Naming a responsibility
- **WHEN** logic is extracted into a new entity
- **THEN** the entity is named after the responsibility it holds, not after where it is used

### Requirement: Single-use values are chained
WHEN a `val` is used exactly once, on the next line, only to feed the next expression, the agent SHALL chain the expressions with `pipe` or `tap` from `scala.util.chaining` instead of naming the value. WHEN the name documents intent, the agent SHALL keep the named `val`.

#### Scenario: A pass-through value
- **WHEN** `val parsed = parse(raw)` is followed only by `validate(parsed)`
- **THEN** the agent writes `parse(raw).pipe(validate)`

#### Scenario: A side effect in the middle of a chain
- **WHEN** a value must be logged and then passed on unchanged
- **THEN** the agent uses `tap` for the logging step

#### Scenario: A name that documents intent
- **WHEN** the value's name explains a non-obvious step, such as `val normalisedEmail`
- **THEN** the agent keeps the named `val`

### Requirement: Non-Unit results are discarded explicitly
WHEN an expression's non-Unit result is intentionally unused, the agent SHALL discard it explicitly, in the form the repository already uses, rather than leave a bare non-Unit statement.

#### Scenario: An unused return value
- **WHEN** a call returns a value the code does not need
- **THEN** the agent discards it explicitly, and no discarded-value warning is produced

#### Scenario: A value used later
- **WHEN** the result is needed further down
- **THEN** it is bound to a named `val` instead

### Requirement: Code compiles with zero warnings
Code the agent writes SHALL compile with zero warnings under the repository's configured compiler options. The agent SHALL fix the cause in code, SHALL suppress a warning only for generated code or an unfixable third-party issue, with a comment saying why, and SHALL NOT change compiler options to silence a warning.

#### Scenario: A warning in the agent's code
- **WHEN** the build reports an unused import, an unused parameter or a discarded value in code the agent wrote
- **THEN** the agent removes the cause and does not add a suppression

#### Scenario: A warning in generated code
- **WHEN** a warning comes from code a generator produced
- **THEN** the agent suppresses that warning as narrowly as possible, with a comment naming the generator

#### Scenario: A compiler flag would silence it
- **WHEN** removing or weakening a compiler option would make the warning go away
- **THEN** the agent does not touch the options and fixes the code instead

### Requirement: Unbounded data is streamed or paged
The agent SHALL NOT load a collection of unbounded size into memory. It SHALL process such data incrementally, as a stream or an iterator, or page through it.

#### Scenario: All rows of a table
- **WHEN** code must process every row of a table of unknown size
- **THEN** the agent streams the rows or fetches them page by page

#### Scenario: A bounded list
- **WHEN** the data is a configuration list or another collection with a known small bound
- **THEN** a plain in-memory collection is fine

### Requirement: Each test covers one scenario
Each test SHALL cover exactly one scenario. The agent SHALL NOT write tests that overlap with or repeat the coverage of another test.

#### Scenario: One behaviour with several assertions
- **WHEN** one scenario needs several assertions to describe its outcome
- **THEN** they stay in one test

#### Scenario: Two behaviours
- **WHEN** a test exercises two independent behaviours
- **THEN** the agent splits it into two tests

#### Scenario: A duplicate
- **WHEN** a new test would cover what an existing test already covers
- **THEN** the agent does not add it

### Requirement: Tests are deterministic
Tests SHALL control time, randomness and identifiers through the injected clock, random source and id generator, SHALL trigger batch or background work themselves, and SHALL NOT sleep or poll while waiting for a result.

#### Scenario: Time-dependent behaviour
- **WHEN** a test checks an expiry
- **THEN** it supplies a fixed clock and asserts on an exact instant

#### Scenario: Background work
- **WHEN** behaviour is normally driven by a periodic job
- **THEN** the test calls the job's body directly and asserts on the result

#### Scenario: A sleep in a test
- **WHEN** a test waits a fixed time for an effect to appear
- **THEN** the agent replaces the wait with a direct trigger

### Requirement: Scala 2.13 equivalents
WHEN the module compiles with Scala 2.13, the agent SHALL use the equivalents of the four Scala 3 constructs these rules name: braces for braceless syntax, a `sealed trait` with case objects and case classes in its companion for an `enum`, a value class or tagged type for an opaque type, and `implicit` definitions and implicit classes for givens and extension methods. Every other rule SHALL apply unchanged.

#### Scenario: A closed set on Scala 2.13
- **WHEN** the rules call for an enum and the module is on Scala 2.13
- **THEN** the agent declares a `sealed trait` with case objects in its companion

#### Scenario: A domain value on Scala 2.13
- **WHEN** the rules call for an opaque type and the module is on Scala 2.13
- **THEN** the agent declares a value class or a tagged type with the same smart constructor

#### Scenario: A cross-built module
- **WHEN** a module compiles for both Scala 2.13 and Scala 3
- **THEN** the agent uses the Scala 2.13 forms, which both versions accept
