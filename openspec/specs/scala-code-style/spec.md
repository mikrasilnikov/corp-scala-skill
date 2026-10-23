# scala-code-style Specification

## Purpose

Defines the syntax, function-shape, naming, warning and test-style rules an agent follows when it writes Scala under the `virtuslab-scala-style` rules file, naming the four libraries the house style requires, mouse, enumeratum, Chimney and Monocle, and no build tool or formatter, so that they hold in every Scala repository that declares those libraries, on Scala 3 and on Scala 2.13.

## Requirements

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

### Requirement: Public members declare their return types
Every public `def`, `val` and `given` (or `implicit` definition on Scala 2.13) SHALL carry an explicit return type. Every `implicit val`, `lazy val` and `def`, private and local included, SHALL carry an explicit type; an implicit class or object declares no result type and is unaffected.

#### Scenario: A new public method
- **WHEN** the agent adds a public method
- **THEN** its result type is written out, even when the compiler could infer it

#### Scenario: A refactor changes an implementation
- **WHEN** a refactor would change the inferred type of a public member
- **THEN** the explicit type makes the compiler report the change instead of silently altering the signature

#### Scenario: A local value
- **WHEN** the agent declares a private or local `val` that is not `implicit`
- **THEN** it may leave the type inferred

#### Scenario: A private or local implicit definition
- **WHEN** the agent declares a private or local `implicit val`, `lazy val` or `def`, such as an ordering in a companion
- **THEN** it writes the type out, `private implicit val ordering: Ordering[Score] = ...`, since Scala 3 rejects an implicit member without one, and a local one whose right-hand side searches for implicits, and Scala 2.13 warns about the member

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
WHEN a `val` is used exactly once, on the next line, only to feed the next expression, the agent SHALL chain the expressions with `|>` from mouse instead of naming the value, and SHALL use `<|` for a side effect that passes the value on unchanged. WHEN the name documents intent, the agent SHALL keep the named `val`. The agent SHALL NOT use `pipe` or `tap` from `scala.util.chaining`.

#### Scenario: A pass-through value
- **WHEN** `val parsed = parse(raw)` is followed only by `validate(parsed)`
- **THEN** the agent writes `parse(raw) |> validate`

#### Scenario: A side effect in the middle of a chain
- **WHEN** a value must be logged and then passed on unchanged
- **THEN** the agent uses `<|` for the logging step, with a function that returns `Unit`

#### Scenario: A side effect that returns a value
- **WHEN** the side effect inside `<|` would return a value
- **THEN** the agent discards that value explicitly inside the lambda, so no discarded-value warning is produced

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
WHEN the module compiles with Scala 2.13, the agent SHALL use the equivalents of the four Scala 3 constructs these rules name: a `sealed trait` with case objects and case classes in its companion for an `enum`, a value class or tagged type for an opaque type, `implicit` definitions and implicit classes for givens and extension methods, and `_` for the `*` wildcard in an import. Every other rule SHALL apply unchanged.

#### Scenario: A closed set on Scala 2.13
- **WHEN** the rules call for an enum and the module is on Scala 2.13
- **THEN** the agent declares a `sealed trait` with case objects in its companion

#### Scenario: A domain value on Scala 2.13
- **WHEN** the rules call for an opaque type and the module is on Scala 2.13
- **THEN** the agent declares a value class or a tagged type with the same smart constructor

#### Scenario: A wildcard import on Scala 2.13
- **WHEN** a rule names an import such as `import mouse.all.*` and the module is on Scala 2.13
- **THEN** the agent writes `import mouse.all._`, which Scala 2.13 accepts without `-Xsource:3`

#### Scenario: A cross-built module
- **WHEN** a module compiles for both Scala 2.13 and Scala 3
- **THEN** the agent uses the Scala 2.13 forms, which both versions accept

### Requirement: Scala 3 brace style is the agent's choice, consistent within a file
In Scala 3 code the agent SHALL choose between the indentation-based and the braced syntax itself, matching the file it is in and, in a new file, the repository's prevailing style, and SHALL NOT mix the two styles within one file. WHEN the module compiles with Scala 2.13, the agent SHALL use braces.

#### Scenario: Editing an existing file
- **WHEN** the agent adds a definition to a Scala 3 file written in one style
- **THEN** the new definition is written in that file's style

#### Scenario: A new file
- **WHEN** the agent creates a Scala 3 file in a repository whose files share one style
- **THEN** the new file uses that style

#### Scenario: A repository with no prevailing style
- **WHEN** the repository's Scala 3 files use both styles and the agent creates a new file
- **THEN** the agent picks either style and uses it throughout the file

#### Scenario: Braces on Scala 2.13
- **WHEN** the module is on Scala 2.13, or cross-built for Scala 2.13 and Scala 3
- **THEN** the agent uses braces, since Scala 2.13 has no other syntax, and follows the repository's layout

### Requirement: Members are named in lowerCamelCase
Every `val`, `var`, `def`, parameter and case class field the agent declares SHALL be named in lowerCamelCase, constants included, with acronyms treated as words. Types, objects and enum cases SHALL be UpperCamelCase and packages lowercase. Another spelling SHALL be kept only for a member that overrides one the agent does not control.

#### Scenario: A constant
- **WHEN** the agent declares a constant such as the maximum number of retries
- **THEN** it is named `maxRetries`, not `MaxRetries` or `MAX_RETRIES`

#### Scenario: A constant in a pattern
- **WHEN** a lowerCamelCase constant is used in a `case` pattern
- **THEN** the agent back-quotes it, so the pattern compares against the constant instead of binding a new variable

#### Scenario: An acronym in a name
- **WHEN** a name contains an acronym such as ID or HTTP
- **THEN** it is written as `userId` or `httpClient`, not `userID` or `HTTPClient`

#### Scenario: A field that mirrors a wire name
- **WHEN** a case class field is read from or written to a wire format under a name that is not lowerCamelCase, such as `first_name`
- **THEN** the field is `firstName` and the wire name is carried by the serialisation library's key annotation, such as upickle's `@key("first_name")`

#### Scenario: An overridden member
- **WHEN** the agent overrides a member of a library or Java type whose name is not lowerCamelCase
- **THEN** the override keeps that name, since the agent cannot change it

### Requirement: Required libraries are declared or asked for
The libraries the rules file names as required SHALL be declared in every module the rules govern. WHEN a module lacks one that a rule needs for the code at hand, the agent SHALL stop and ask the user to add it, naming organisation, artifact, module and configuration, SHALL NOT add it itself, and SHALL NOT write the standard-library or hand-written form in its place.

#### Scenario: A chain in a module without mouse
- **WHEN** the agent would chain a single-use value in a module that does not declare mouse
- **THEN** it asks for mouse with its coordinates and does not write `pipe` from `scala.util.chaining` instead

#### Scenario: A nested update in a module without Monocle
- **WHEN** the agent would update a field nested two levels deep in a module that does not declare Monocle
- **THEN** it asks for Monocle and does not write nested `copy` calls instead

#### Scenario: The library's rule does not apply
- **WHEN** the code at hand needs none of a library's rules, such as a one-level `copy` or a plain enum that is only matched on
- **THEN** the agent writes it and asks for nothing

#### Scenario: The user has added the library
- **WHEN** the user confirms the dependency has been added
- **THEN** the agent continues with the library's form

### Requirement: A compiler option a required library needs is asked for
WHEN the code at hand declares an enumeratum value enumeration and a Scala 3 module, or the Scala 3 compilation of a cross-built module, lacks the compiler option `-Yretain-trees`, the agent SHALL stop and ask the user to add it, naming the option and the module, SHALL NOT change the module's compiler options itself, and SHALL NOT write a hand-written enumeration in its place.

#### Scenario: A value enumeration in a Scala 3 module without -Yretain-trees
- **WHEN** the agent would declare an enumeratum value enumeration in a Scala 3 module whose compiler options lack `-Yretain-trees`
- **THEN** it asks the user to add the option for that module, does not change the compiler options itself, and does not write a hand-written enumeration instead

#### Scenario: A value enumeration in a cross-built module
- **WHEN** the agent would declare an enumeratum value enumeration in a module cross-built for Scala 2.13 and Scala 3 that lacks `-Yretain-trees` on its Scala 3 compilation
- **THEN** the agent asks for the option on the Scala 3 compilation only, since Scala 2.13 rejects it

#### Scenario: A value enumeration on Scala 2.13 only
- **WHEN** the agent would declare an enumeratum value enumeration in a module that compiles with Scala 2.13 only
- **THEN** the agent asks for no compiler option, since only Scala 3 needs it

#### Scenario: A module that only looks a value enumeration up
- **WHEN** a Scala 3 module looks up or enumerates a value enumeration declared in another module, and declares none itself
- **THEN** the agent asks for no compiler option, since only the module that declares the enumeration needs it
