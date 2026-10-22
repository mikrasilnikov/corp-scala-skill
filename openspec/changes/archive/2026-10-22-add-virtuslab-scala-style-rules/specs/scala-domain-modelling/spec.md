# Spec Delta

## Purpose

Defines how an agent types domain data under the `virtuslab-scala-style` rules file: immutable data with pure transitions, explicit dependencies for time and identity, and types under which invalid data cannot be constructed, so that the compiler rejects what a review would otherwise have to catch.

## ADDED Requirements

### Requirement: State is immutable and transitions are pure
The agent SHALL model state as immutable case classes and each transition as a pure function from state to state, and SHALL NOT share mutable state between components.

#### Scenario: A processing loop
- **WHEN** a loop accumulates state over items
- **THEN** the per-item transition is a pure function that can be tested without the loop, and the loop threads the state through it

#### Scenario: Two components sharing data
- **WHEN** two components need the same evolving data
- **THEN** one owns it and passes immutable snapshots, rather than both mutating a shared object

### Requirement: A var lives only in a method body
A `var` SHALL appear only inside a method body, confined to the smallest scope that needs it, and never as a class or object field.

#### Scenario: A loop variable
- **WHEN** a method runs a loop that accumulates a value
- **THEN** the `var` is declared inside that method and is not visible outside it

#### Scenario: A mutable field
- **WHEN** existing or proposed code holds a `var` as a field
- **THEN** the agent writes none and models the state as an immutable value returned from the operation instead

### Requirement: Collections are immutable
Production code SHALL use immutable collections only. Mutable collections SHALL appear only in test fakes.

#### Scenario: A mutable map in a service
- **WHEN** a service would keep a mutable map
- **THEN** the agent uses an immutable map held in an immutable state value

#### Scenario: An in-memory test fake
- **WHEN** a test fake stores what it receives
- **THEN** it may use a mutable collection

### Requirement: Time, randomness and identity are injected
A clock, a random source and an id generator SHALL be passed as parameters or constructor arguments, and SHALL NOT be obtained inside a method, so that results are reproducible and a test can assert on them.

#### Scenario: The current time
- **WHEN** a method needs the current instant
- **THEN** it reads it from a clock it was given, not from a static call

#### Scenario: A fresh identifier
- **WHEN** a method creates an entity that needs an id
- **THEN** the id comes from an injected generator

#### Scenario: In a test
- **WHEN** a test exercises such a method
- **THEN** it supplies a fixed clock and a predictable generator and asserts on exact values

### Requirement: Invalid data cannot be constructed
The agent SHALL design types so that invalid data cannot be represented: enums or sealed hierarchies for closed sets, `Option` for absence and never `null`, and validated constructors for values with invariants.

#### Scenario: An optional field
- **WHEN** a value may be absent
- **THEN** its type is `Option`, and `null` is never assigned or checked for

#### Scenario: A status with a fixed set of values
- **WHEN** a field can take one of a known set of values
- **THEN** it is an enum, not a `String`

#### Scenario: A value with an invariant
- **WHEN** a value must satisfy a condition to be meaningful
- **THEN** the only way to obtain one is a validated constructor

### Requirement: Domain concepts are not raw primitives
Identifiers, quantities and configuration values SHALL be wrapped in opaque types, or value classes or tagged types on Scala 2.13, or in enums, and SHALL NOT be passed around as bare `String`, `Int`, `Long` or `Boolean`.

#### Scenario: An identifier parameter
- **WHEN** a method takes an order identifier
- **THEN** the parameter type is `OrderId`, not `String`

#### Scenario: A configuration value
- **WHEN** a port or a topic name is read from configuration
- **THEN** it is typed as `Port` or `TopicName`, not `Int` or `String`

#### Scenario: Two identifiers swapped
- **WHEN** a product code is passed where an order identifier is expected
- **THEN** the code does not compile

### Requirement: Smart constructors return Either
WHEN a value has invariants, its constructor SHALL return `Either` with a typed error and SHALL NOT use `require` or throw. The companion SHALL expose no public way to build an invalid value.

#### Scenario: An invalid input
- **WHEN** a quantity of zero is constructed
- **THEN** the result is a `Left` carrying a typed error that names the offending value

#### Scenario: A valid input
- **WHEN** a positive quantity is constructed
- **THEN** the result is a `Right` holding the value

#### Scenario: A require in a constructor
- **WHEN** existing code enforces an invariant with `require`
- **THEN** the agent does not copy the pattern into new types

### Requirement: Two-case enums replace Booleans
A Boolean parameter or return value whose meaning a call site cannot read SHALL be replaced by a two-case enum that names both outcomes.

#### Scenario: A flag parameter
- **WHEN** a method takes `success: Boolean`
- **THEN** the agent replaces it with an enum such as `FlushOutcome.Success` and `FlushOutcome.Failure`

#### Scenario: A predicate
- **WHEN** a method such as `isEmpty` answers a yes-or-no question that its name states
- **THEN** its Boolean result stays

### Requirement: Identifiers are phantom-typed
Identifiers of different entities SHALL be distinct types, either one opaque identifier type parameterised by the entity it identifies or one opaque type per entity, so that an identifier of one entity cannot be passed for another.

#### Scenario: A user id where an order id is expected
- **WHEN** code passes a user's identifier to a method that takes an order's
- **THEN** the code does not compile

#### Scenario: Shared identifier plumbing
- **WHEN** many entities need the same identifier representation
- **THEN** one parameterised opaque type serves them all without losing the distinction

### Requirement: Proof types record a normalisation
WHEN a value must have been transformed before use, such as lower-cased, trimmed or hashed, the result SHALL have a distinct type, an opaque subtype on Scala 3, produced only by the function that performs the transformation, so that an untransformed value cannot be passed in its place.

#### Scenario: A lookup by email
- **WHEN** a lookup requires a lower-cased email
- **THEN** its parameter type is the lower-cased type, and a raw `String` does not compile

#### Scenario: Storing a password
- **WHEN** a repository method stores a password
- **THEN** it accepts only the hashed type, which only the hashing function produces

### Requirement: Generated and external types stop at the boundary
Types generated from schemas or wire formats, and types of other systems, SHALL be converted to domain types at the boundary where they arrive, and the raw strings and numbers they carry SHALL NOT reach the domain.

#### Scenario: A generated class with string fields
- **WHEN** a code generator produces classes whose fields are all `String`
- **THEN** the adapter that receives them builds domain values with opaque types, and domain code never imports the generated package

#### Scenario: A data transfer object
- **WHEN** a DTO carries an identifier or a quantity
- **THEN** it uses the domain's opaque type rather than falling back to a primitive

### Requirement: Configuration is typed per module and masks secrets
Each module's configuration SHALL be a case class of typed values, composed into one top-level configuration. A secret SHALL be wrapped in a type whose string form masks it. Invalid configuration SHALL fail at startup, which is the one place a constructor may throw.

#### Scenario: A password in a log line
- **WHEN** the configuration is logged at startup
- **THEN** the password prints as a mask, not its value

#### Scenario: A module's settings
- **WHEN** a module needs settings
- **THEN** it declares its own configuration case class, and the top-level configuration holds it as a field

#### Scenario: An invalid setting
- **WHEN** a configured pattern is malformed
- **THEN** the application fails at startup with a message naming the setting, not at the first request that uses it
