# Spec Delta

## MODIFIED Requirements

### Requirement: Generated and external types stop at the boundary
Types generated from schemas or wire formats, and types of other systems, SHALL be converted to the domain's own types at the boundary where they arrive, and domain code SHALL NOT import a generated package or name another system's type.

#### Scenario: A generated class with string fields
- **WHEN** a code generator produces classes whose fields are all `String`
- **THEN** the adapter that receives them builds the domain's types from those fields, giving each the type the domain declares for it, and domain code never imports the generated package

#### Scenario: A data transfer object
- **WHEN** a data transfer object carries an identifier or a quantity
- **THEN** the mapping at the adapter gives the value the type the domain declares for it, whether the DTO carried that type or the primitive its wire format uses

## REMOVED Requirements

### Requirement: Domain concepts are not raw primitives
**Reason**: It made every identifier, quantity and configuration value an opaque type whatever the value's meaning, which adds a wrapper, an accessor and a conversion to values that carry no invariant and contradicts any repository that passes them as primitives, and the preamble's precedence clause, which covers syntax rules only, did not let the agent follow that repository.
**Migration**: A value takes the type the surrounding code uses for it, a plain `String`, `Int`, `Long` or `Boolean` included. A value with an invariant still gets a validated constructor under "Invalid data cannot be constructed" and "Smart constructors return Either"; a closed set of values is still an enum under "Invalid data cannot be constructed"; a Boolean whose meaning a call site cannot read is still replaced under "Two-case enums replace Booleans"; a value that must have been transformed before use still gets its own type under "Proof types record a normalisation". Wrapper types that existing code already declares stay, and the agent keeps using them.

### Requirement: Identifiers are phantom-typed
**Reason**: It gave every entity's identifier a distinct opaque type, which is the identifier case of "Domain concepts are not raw primitives" under another name; kept, it would hold that case in force after the removal.
**Migration**: An identifier takes the type the repository uses for that entity's identifiers, whether a type it declares, such as `OrderId` or `Id[Order]`, or a plain `String` or `Long`; where a repository declares such types, the agent keeps using them. An identifier with a format to validate still gets a validated type under "Invalid data cannot be constructed".
