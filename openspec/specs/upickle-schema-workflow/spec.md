# upickle-schema-workflow Specification

## Purpose

Defines how an agent carries out a schema-to-case-class conversion with upickle: asking once for the producer's source and mirroring it when it exists, the precondition that upickle is already declared, the derivation syntax for the module's Scala version, where the definitions go, the unit test that verifies them through the repository's build, and the report of the defaults and ambiguities it resolved and of the generator defects it followed.

## Requirements

### Requirement: The agent asks once whether the producer's source is available
Before generating anything from a schema, the agent SHALL ask the user once whether the Scala source the schema was generated from is available, and SHALL wait for the answer. It SHALL NOT ask again for the same schema in the same task, and SHALL use only the source the user points it at, without searching for it elsewhere.

#### Scenario: Source available
- **WHEN** the user points the agent at the producer's repository or files
- **THEN** the agent reads those definitions and mirrors them, as the next requirement describes

#### Scenario: Source not available
- **WHEN** the user answers that the source is not available
- **THEN** the agent derives names, nesting, type parameters, and defaults from the schema, and from a sample payload only where `upickle-schema-mapping` says so, and does not ask again

#### Scenario: Source available for some types only
- **WHEN** the user supplies definitions for part of the schema
- **THEN** the agent mirrors those, derives the rest from the schema, and says in its report which definitions came from where

### Requirement: Mirroring the producer's definitions
When the producer's source is available, the agent SHALL reproduce its definitions for every needed `$defs` entry: class, field, and case names, nesting, type parameters, default values, and `@key` and `@allowUnknownKeys` annotations, translated to the consumer's Scala version. Any custom pickler for a type the schema reaches SHALL be reproduced too. The schema SHALL decide which definitions are needed and, except as the next requirement and `upickle-schema-mapping` say, what each shape is.

#### Scenario: Scala 3 enum into a Scala 2.13 consumer
- **WHEN** the producer declares a Scala 3 `enum` with parameterless and parametrised cases and the consumer module is on Scala 2.13
- **THEN** the agent declares a `sealed trait` with case objects and case classes of the same names in its companion

#### Scenario: Defaults carried over
- **WHEN** the producer declares `s: String = "x"` and the schema leaves `s` out of `required`
- **THEN** the consumer's field is `s: String = "x"`

#### Scenario: Source and schema disagree
- **WHEN** a producer field's type or default does not match the schema's shape for that property, other than in what `upickle-schema-mapping` takes from a type the generator misdescribes
- **THEN** the agent follows the schema, and names the disagreement in its report

#### Scenario: Producer uses a custom pickler
- **WHEN** the producer reads a type the schema reaches through a custom `ReadWriter`
- **THEN** the agent reproduces that pickler for the consumer and names it in the report

### Requirement: The producer's definition decides the shapes the generator misdescribes
Where the producer declares a type whose schema the generator misdescribes, as `upickle-schema-mapping` names them, its definition SHALL decide only what that capability takes from it, such as a named tuple's names or a map's key type, and the agent SHALL list the property among the generator defects, not as a disagreement. A sample payload that contradicts the definition SHALL decide instead. Item and value types, and any other mismatch, SHALL be compared as for any other field.

#### Scenario: Source shows a shape the generator misdescribes
- **WHEN** the producer declares `point: (x: Int, y: Int)` and the schema shows an inline two-item `prefixItems` array for `point`
- **THEN** the agent follows the source, generating `case class Point(x: Int, y: Int)`, and lists `point` among the generator defects in its report, not among its disagreements with the source

#### Scenario: Another type at a misdescribed shape
- **WHEN** the producer declares `point: Seq[Int]` where the schema shows an inline two-item `prefixItems` array of `integer`s
- **THEN** the agent follows the schema, generating `point: (Int, Int)`, and names the disagreement in its report

#### Scenario: Value types differ at a misdescribed type
- **WHEN** the producer declares `counts: Map[Color, Long]` where the schema shows `{"type": "object", "additionalProperties": {"type": "integer"}}`
- **THEN** the field is `counts: Map[Color, Int]`, `counts` is listed among the generator defects for its key type, and the `Long` values are named as a disagreement with the source

#### Scenario: Payload contradicts a named pair in the source
- **WHEN** the producer declares `point: (x: Int, y: Int)` and a sample payload has `"point": [1, 2]`
- **THEN** the field is `point: (Int, Int)`, and the report names the disagreement with the source

### Requirement: upickle must already be declared
Before writing any definition, the agent SHALL confirm that upickle is declared for the target module, by reading the build definition or running a read-only build query. If it is not, the agent SHALL stop and ask the user to add it, stating the coordinates, module, configuration, and reason, and SHALL NOT add it, any other library, or a JSON Schema validator itself.

#### Scenario: upickle declared
- **WHEN** the module declares `com.lihaoyi::upickle`, directly or through a module it depends on
- **THEN** the agent proceeds

#### Scenario: upickle absent
- **WHEN** no declaration of upickle reaches the module
- **THEN** the agent stops and asks the user to add it, with coordinates, module, configuration, and reason, and the task pauses there

#### Scenario: A validator would make checking easier
- **WHEN** a JSON Schema validator library would let the agent check the generated classes against the schema
- **THEN** the agent does not add one and verifies by unit test instead

### Requirement: Derivation syntax follows the module's Scala version
The agent SHALL read the module's Scala version from the build definition. On Scala 3 it SHALL derive with `derives ReadWriter` and declare sealed families as `enum`, except both families of a sub-family; on Scala 2.13 or 2.12, or in a module cross-built with Scala 2, it SHALL add `implicit val rw: ReadWriter[X] = macroRW` to each companion and declare families as `sealed trait`, using `implicit lazy val` for recursive types. A module with its own pickler configuration SHALL derive from it.

#### Scenario: Scala 3 module
- **WHEN** the module's Scala version is 3
- **THEN** each case class and family carries `derives ReadWriter`, and families are `enum`s, except the two families of a sub-family, which are `sealed trait`s

#### Scenario: Scala 2.13 module
- **WHEN** the module's Scala version is 2.13
- **THEN** each companion holds `implicit val rw: ReadWriter[X] = macroRW`, and families are `sealed trait`s

#### Scenario: Cross-built module
- **WHEN** the module's `crossScalaVersions` include a Scala 2 version
- **THEN** the agent uses the Scala 2 form, which compiles on both

#### Scenario: Recursive type on Scala 2
- **WHEN** a definition refers to itself, directly or through another definition, on Scala 2
- **THEN** its `ReadWriter` is an `implicit lazy val`

#### Scenario: Module with its own pickler configuration
- **WHEN** the module already defines an object extending `upickle.AttributeTagged` or another `upickle.Api`
- **THEN** the agent derives the `ReadWriter`s from that object rather than from `upickle.default`

### Requirement: Generated definitions follow the module's layout
The agent SHALL generate one definition per `$defs` entry reachable from the document's root, other than an entry that `upickle-schema-mapping` gives no definition of its own, and nothing for an unreachable entry, place the definitions in the package and directory where the module keeps its models, and name the file after the root type. If the module has no such place, the agent SHALL ask the user where to put them.

#### Scenario: Existing models package
- **WHEN** the module keeps its case classes in one package
- **THEN** the generated definitions go into that package, in a file named after the root type

#### Scenario: No obvious place
- **WHEN** the module has no package or directory that holds models
- **THEN** the agent asks the user where to put the definitions before writing them

#### Scenario: Entry with no definition of its own
- **WHEN** a reachable property refers to `scala.util.Either[scala.Int, scala.Predef.String]`, whose `oneOf` refers to `scala.util.Left[...]` and `scala.util.Right[...]` entries
- **THEN** no definition is generated for any of the three entries

#### Scenario: Unreachable definition
- **WHEN** a `$defs` entry is not referenced, directly or indirectly, from the document's root
- **THEN** no definition is generated for it

### Requirement: Generated definitions are verified by a unit test through the repository's build
The agent SHALL add a test in the module's declared test framework that reads sample documents into the root type: for each field declared `Option[T]`, `T` not an `Option`, with `optionsAsNulls` on, a document with a value and one with `null`, plus one without the key where the field is not required; for each sealed family one document per case; and any sample payload the user supplies. The agent SHALL run the build and tests the way the repository requires before reporting the change done.

#### Scenario: Optional field states
- **WHEN** a generated definition read with `optionsAsNulls` on has a field declared `Option[T]`, `T` not an `Option`, that is not required, including a generic `Option[T]` instantiated with an `Option`
- **THEN** the test reads three documents for it, with a value, with `null`, and without the key, and expects `Some`, `None`, and the field's default, which is `None` unless the producer's source gives another, such as `Some(30)`

#### Scenario: Sealed family cases
- **WHEN** a generated family has a singleton case and a case class
- **THEN** the test reads a document with the bare string for the singleton and one with the tagged object for the case class

#### Scenario: Sample payload from the user
- **WHEN** the user supplies a real payload from the producer
- **THEN** the test reads it as well, and a failure to read it is treated as a defect in the generated definitions

#### Scenario: No test framework declared
- **WHEN** the module declares no test framework
- **THEN** the agent asks the user to add one as a dependency, and does not add it or skip the test

#### Scenario: Build not run
- **WHEN** the build or tests have not been run, or a test fails
- **THEN** the agent does not report the conversion as done

### Requirement: Test documents follow what upickle writes
For each field declared `Option[Option[T]]` read with `optionsAsNulls` on, the test SHALL read the boxed documents upickle writes, `[x]`, `[null]`, and `[]`, plus one without the key where the field is not required, and SHALL NOT read a bare `null` there, which upickle reads as a Scala `null`, not `None`. For each definition or property the report lists among the generator defects, the test SHALL read a document in the form upickle writes.

#### Scenario: Nested optional field states
- **WHEN** a generated definition read with `optionsAsNulls` on has a field declared `Option[Option[T]]` that is not required
- **THEN** the test reads four documents for it, `[x]`, `[null]`, `[]`, and one without the key, expects `Some(Some(x))`, `Some(None)`, `None`, and `None`, and reads no document with a bare `null` there

#### Scenario: Generator defect documents
- **WHEN** the agent generated `case class Pair(_1: Int, _2: String)` for a positional entry and `e: Either[Int, String]` for an `Either` entry
- **THEN** the test reads `{"_1": 1, "_2": "a"}` for `Pair` and `[0, 1]` and `[1, "s"]` for `e`, and no document in the array or tagged form the schema shows for them

### Requirement: Boxed optional fields are tested with arrays
When the generated types are read through a pickler with `optionsAsNulls = false`, the test SHALL read each `Option[T]` field, `T` not an `Option`, from `[x]` and `[]`, and each `Option[Option[T]]` field from `[[x]]`, `[[]]`, and `[]`, plus one document without the key where the field is not required, and SHALL NOT read a bare value or `null` there, which that pickler rejects or reads as a Scala `null`.

#### Scenario: Boxed optional field states
- **WHEN** a definition read through such a pickler has an `Option[Int]` field that is not required
- **THEN** the test reads `[1]`, `[]`, and a document without the key, expects `Some(1)`, `None`, and `None`, and reads no bare `1` or `null` there

### Requirement: The report names every default and ambiguity
The agent's report SHALL list the files written, the test it added, and the build it ran, and SHALL name by path every property resolved by a default or a fallback: `String` for a plain string, `Int` for an integer, `Long` for integral map keys, in upickle-jsonschema output `Map[String, T]` for a map without key information and `(A, B)` for an unsettled pair, `Option[T] = None` for an unknown default, `ujson.Value` or `String` for an unmapped shape, and any pickler configuration.

#### Scenario: Ambiguities present
- **WHEN** the conversion resolved any property by a default
- **THEN** the report lists each such property with its path, the type chosen, and the alternatives the schema leaves open

#### Scenario: No ambiguities
- **WHEN** every property mapped without a default or fallback
- **THEN** the report says so

#### Scenario: Definitions mirrored from source
- **WHEN** some or all definitions were mirrored from the producer's source
- **THEN** the report names which definitions were mirrored and which were derived from the schema

### Requirement: The report lists the generator defects it followed
The report SHALL list, under a label of its own and apart from the defaults and from the disagreements with the producer's source, every definition or property for which the agent followed what upickle writes over a shape the generator misdescribes, with the schema's shape and the type generated, or SHALL say that there is none.

#### Scenario: Generator defects followed
- **WHEN** the conversion generated `case class Pair(_1: Int, _2: String)` for the positional entry `example.Pair`
- **THEN** the report lists `Pair` under that label, with the schema's two-item `prefixItems` array and the case class generated

#### Scenario: No generator defects
- **WHEN** no definition or property needed what upickle writes in place of the schema
- **THEN** the report says so under that label

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
