# Spec Delta

## MODIFIED Requirements

### Requirement: Objects with properties become case classes
An `object` schema with `properties` SHALL map to a case class with one constructor field per property, in the schema's property order. A property name that is already lowerCamelCase SHALL be the field name verbatim; any other SHALL get a lowerCamelCase field name annotated with `@key("<property>")`. `additionalProperties: false` SHALL add `@allowUnknownKeys(false)` to the class, and `true` SHALL add nothing. An object with no properties SHALL be an empty case class.

#### Scenario: Property names kept verbatim
- **WHEN** an object has the properties `firstName` and `totalCount`, which are already lowerCamelCase
- **THEN** the fields are `firstName` and `totalCount`, with no `@key` annotation

#### Scenario: Property names in another style
- **WHEN** an object has the properties `first_name` and `total_count`
- **THEN** the fields are `@key("first_name") firstName` and `@key("total_count") totalCount`

#### Scenario: Property name that is not an identifier
- **WHEN** an object has the property `content-type`
- **THEN** the field is `@key("content-type") contentType`

#### Scenario: Unknown keys rejected by the producer
- **WHEN** an object schema has `"additionalProperties": false`
- **THEN** the case class carries `@allowUnknownKeys(false)`

#### Scenario: Referenced object
- **WHEN** a property reads `{"$ref": "#/$defs/example.Address"}`
- **THEN** the field's type is the definition generated for that key, `Address`

#### Scenario: Empty object
- **WHEN** a `$defs` entry reads `{"type": "object", "properties": {}, "required": [], "additionalProperties": true}`
- **THEN** it becomes an empty case class

### Requirement: Family cases are named by their tags
Each case of a sealed family SHALL be named as its tag when the tag is already UpperCamelCase, and otherwise SHALL be named in UpperCamelCase and carry `@key("<tag>")`. A tag property other than `$type` SHALL be declared with `@key("<property>")` on the family. No other handling of the tag SHALL be written.

#### Scenario: Family with a renamed tag property
- **WHEN** the tag property in every `allOf` entry is `kind` instead of `$type`
- **THEN** the family carries `@key("kind")`, and its cases are otherwise unchanged

#### Scenario: Tag that is not a legal name
- **WHEN** a `oneOf` entry reads `{"const": "reddit.com"}`
- **THEN** the case is `@key("reddit.com") case object RedditCom`, or the equivalent enum case

#### Scenario: Case name differs from the tag
- **WHEN** a `oneOf` entry reads `{"const": "in_progress"}`, a tag that is not UpperCamelCase
- **THEN** the case is `@key("in_progress") case object InProgress`, or `@key("in_progress") case InProgress` in an `enum`

#### Scenario: Tag already in UpperCamelCase
- **WHEN** a `oneOf` entry reads `{"const": "End"}`
- **THEN** the case is `case object End`, or `case End` in an `enum`, with no annotation
