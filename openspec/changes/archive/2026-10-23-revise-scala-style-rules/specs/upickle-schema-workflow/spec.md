# Spec Delta

## ADDED Requirements

### Requirement: Mirrored names follow the consumer's naming rules
A field or case name mirrored from the producer's source SHALL be reproduced in meaning but spelled as `upickle-schema-mapping` requires, lowerCamelCase for a field and UpperCamelCase for a case, with `@key("<producer name>")` carrying the producer's spelling where it differs, so that the consumer reads what the producer writes.

#### Scenario: A producer field in another naming style
- **WHEN** the producer declares `first_name: String` with no `@key`
- **THEN** the consumer's field is `@key("first_name") firstName: String`, which reads the `first_name` property the producer writes

#### Scenario: A producer field already annotated
- **WHEN** the producer declares `@key("first-name") firstName: String`
- **THEN** the consumer's field is the same `@key("first-name") firstName: String`

#### Scenario: A producer case in another naming style
- **WHEN** the producer declares `case object in_progress` in a sealed family
- **THEN** the consumer's case is `@key("in_progress") case object InProgress`, or the equivalent enum case

#### Scenario: A producer name already in the required style
- **WHEN** the producer declares `totalCount: Int`
- **THEN** the consumer's field is `totalCount: Int`, with no annotation
