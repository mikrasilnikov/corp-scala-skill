# scala-domain-modelling Specification

## Purpose

Defines how an agent types domain data under the `virtuslab-scala-style` rules file: immutable data with pure transitions, explicit dependencies for time and identity, and types under which invalid data cannot be constructed, so that the compiler rejects what a review would otherwise have to catch.

## Requirements

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

### Requirement: Proof types record a normalisation
WHEN a value must have been transformed before use, such as lower-cased, trimmed or hashed, the result SHALL have a distinct type, an opaque subtype on Scala 3, produced only by the function that performs the transformation, so that an untransformed value cannot be passed in its place.

#### Scenario: A lookup by email
- **WHEN** a lookup requires a lower-cased email
- **THEN** its parameter type is the lower-cased type, and a raw `String` does not compile

#### Scenario: Storing a password
- **WHEN** a repository method stores a password
- **THEN** it accepts only the hashed type, which only the hashing function produces

### Requirement: Generated and external types stop at the boundary
Types generated from schemas or wire formats, and types of other systems, SHALL be converted to the domain's own types at the boundary where they arrive, and domain code SHALL NOT import a generated package or name another system's type.

#### Scenario: A generated class with string fields
- **WHEN** a code generator produces classes whose fields are all `String`
- **THEN** the adapter that receives them builds the domain's types from those fields, giving each the type the domain declares for it, and domain code never imports the generated package

#### Scenario: A data transfer object
- **WHEN** a data transfer object carries an identifier or a quantity
- **THEN** the mapping at the adapter gives the value the type the domain declares for it, whether the DTO carried that type or the primitive its wire format uses

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

### Requirement: Enumerations looked up by name or value use enumeratum
A closed set of values whose members carry no data SHALL be declared as an enumeratum enumeration when it is looked up by name or by a wire value, enumerated, or serialised under custom names. Members SHALL be looked up with the non-throwing variants, and the library's lookup error SHALL be wrapped in the module's error type. A set that needs none of that SHALL stay a plain enum, and a hierarchy whose cases carry data SHALL stay a sealed hierarchy.

#### Scenario: A status parsed from its name
- **WHEN** a status is read from a string and must be found by its name
- **THEN** the agent declares it as an enumeratum enumeration and looks the name up with an `Option` or `Either` returning variant, never the throwing one

#### Scenario: A code with a wire value
- **WHEN** each member of a set maps to an integer or string code on the wire
- **THEN** the agent declares a value enumeration carrying that code

#### Scenario: An unknown name
- **WHEN** the lookup finds no member
- **THEN** the failure is returned as a case of the module's error type, not as the library's error or an exception

#### Scenario: A set that is only matched on
- **WHEN** a closed set is only pattern-matched and never looked up, enumerated or renamed
- **THEN** it stays a plain `enum`, or a `sealed trait` with case objects on Scala 2.13

#### Scenario: A data transfer type read by derived codecs
- **WHEN** a type is read or written by a serialisation library's derived codecs
- **THEN** it stays a plain `enum` or sealed trait, since the derived codecs do not know the enumeratum form

#### Scenario: Cases with data
- **WHEN** some cases of a closed set carry fields
- **THEN** the set is a sealed hierarchy, not an enumeratum enumeration

### Requirement: Mappings that are mostly field by field are derived with Chimney
WHEN a mapping between two types is mostly field by field, such as a data transfer object to its domain type, the agent SHALL derive it with Chimney and state only the fields that differ. An inbound mapping that validates SHALL be a partial transformer built from the smart constructors, with its errors wrapped in the module's error type at the adapter. A mapping in which most fields need logic of their own SHALL be written by hand.

#### Scenario: A DTO with matching fields
- **WHEN** a DTO and its domain type share most field names and types
- **THEN** the agent derives the transformer and overrides only the renamed or computed fields

#### Scenario: An inbound mapping with invariants
- **WHEN** the domain type's fields are opaque types with smart constructors returning `Either`
- **THEN** the agent derives a partial transformer whose field transformers come from those smart constructors, and the adapter wraps its errors in a case of the module's error type

#### Scenario: Instances for a domain type
- **WHEN** a domain type needs a transformer instance
- **THEN** it is declared in the type's companion, where implicit resolution finds it

#### Scenario: A mapping that is mostly logic
- **WHEN** most fields of the target are computed from several source fields or from other inputs
- **THEN** the agent writes the mapping by hand

### Requirement: Updates nested two or more levels deep go through Monocle
WHEN the agent updates a field nested two or more levels deep in an immutable case class, it SHALL do so through Monocle's focus syntax, not nested `copy` calls. A lens used in more than one place SHALL be a named value in the state type's companion. A one-level update SHALL stay `copy`.

#### Scenario: A nested field
- **WHEN** code must change a user's address's street number
- **THEN** the agent writes `user.focus(_.address.streetNumber).modify(_ + 1)`, not `user.copy(address = user.address.copy(streetNumber = ...))`

#### Scenario: A shared lens
- **WHEN** two places update the same nested path
- **THEN** the lens is declared once in the companion of the state type and both use it

#### Scenario: A one-level update
- **WHEN** code changes a direct field of a case class
- **THEN** `copy` is used

#### Scenario: Optional and indexed paths on Scala 2.13
- **WHEN** the path goes through an `Option` or a collection element in a Scala 2.13 module
- **THEN** the agent composes an optional or a traversal explicitly, since the focus syntax does not support those steps there
