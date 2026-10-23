# Spec Delta

## ADDED Requirements

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
