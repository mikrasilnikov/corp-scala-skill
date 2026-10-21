# Spec Delta

## Purpose

Defines how a JSON Schema document, as produced by upickle-jsonschema, maps to Scala types that upickle reads directly: optional shapes as `Option[T]`, the `required` list as the source of defaults, scalars, collections, tuples, maps, sealed families with their discriminators, names derived from `$defs` keys, the shapes whose generated schema does not describe what upickle writes, and the shapes for which an agent never writes deserialization by hand.

## ADDED Requirements

### Requirement: Optional shapes map to Option
A property schema whose alternatives are one schema S together with `{"type": "null"}`, with or without a further alternative that is an array of zero or one items of S, SHALL be modelled as `Option[T]`, where T is the mapping of S. A nested occurrence of that shape SHALL be `Option[Option[T]]`, unless its inner shape is a type argument's, which the type-argument rule keeps `Option[T]`. The agent SHALL treat the array alternative as part of the optional shape, never as a collection.

#### Scenario: Optional field as upickle-jsonschema emits it
- **WHEN** a property reads `{"anyOf": [S, {"type": "null"}, {"type": "array", "minItems": 0, "maxItems": 1, "items": S}]}`, for example with S equal to `{"type": "integer"}`
- **THEN** the field is `Option[Int]`, with no further code for the null or array alternatives

#### Scenario: Optional reference
- **WHEN** S is `{"$ref": "#/$defs/example.Node"}`
- **THEN** the field is `Option[Node]`

#### Scenario: Nested optional
- **WHEN** the first alternative of an optional shape is itself an optional shape over `{"type": "boolean"}`
- **THEN** the field is `Option[Option[Boolean]]`

#### Scenario: Nested optional through a type argument
- **WHEN** the `$defs` key `example.R[scala.Option[scala.Int]]` has a property `data` whose optional shape has the optional shape over `{"type": "integer"}` as its S, and the producer's definition is not known
- **THEN** the definition is `case class R[T](data: Option[T])`, read as `R[Option[Int]]`, which reads the unboxed `{"data": 3}` and `{"data": null}` that upickle writes there, and the report lists `data` among the generator defects

#### Scenario: Hand-written nullable forms
- **WHEN** a property reads `{"type": ["string", "null"]}` or `{"anyOf": [{"type": "string"}, {"type": "null"}]}`
- **THEN** the field is `Option[String]`

#### Scenario: Boxed optional from a producer configured without nulls
- **WHEN** a property reads `{"type": "array", "minItems": 0, "maxItems": 1, "items": S}` with no null alternative
- **THEN** the field is still `Option[T]`, the generated types are read through a pickler configured with `optionsAsNulls = false`, and the report names that configuration

### Requirement: Mapped shapes get no hand-written deserialization
For every shape this capability maps, the agent SHALL rely on upickle's derived readers and SHALL NOT write a custom `Reader` or `ReadWriter`, a `bimap` over `ujson.Value`, a manual null or array check, or any dispatch over `anyOf` or `oneOf` alternatives.

#### Scenario: Tempted to dispatch the optional shape by hand
- **WHEN** the agent considers reading a property into `ujson.Value` and choosing among its alternatives in code
- **THEN** it declares the field as `Option[T]` and writes nothing else for it

#### Scenario: Long as integer or string
- **WHEN** a property reads `{"anyOf": [{"type": "integer"}, {"type": "string", "pattern": "^-?(0|[1-9][0-9]*)$"}]}`
- **THEN** the field is `Long`, with no custom reader for the string alternative

#### Scenario: Discriminated union
- **WHEN** a `$defs` entry outside the `scala` package is a `oneOf` whose entries carry a tag property with a `const`
- **THEN** the agent declares a sealed family and writes no code that reads the tag

### Requirement: Membership in required decides defaults
Whether a field has a default value SHALL be decided by the property's membership in the object's `required` list, independently of whether the field is an Option. A required field SHALL have no default. A field that is not required SHALL take the producer's default when the producer's definition is known; otherwise an Option field SHALL default to `None`, and any other field SHALL become `Option[T] = None`. The agent SHALL NOT invent a default value.

#### Scenario: Required optional field
- **WHEN** `next` has the optional shape and is listed in `required`
- **THEN** the field is `next: Option[Node]` with no default

#### Scenario: Optional field not required
- **WHEN** `simple2` has the optional shape and is not listed in `required`
- **THEN** the field is `simple2: Option[SimpleEnum] = None`

#### Scenario: Optional field with a producer default
- **WHEN** `timeout` has the optional shape, is not listed in `required`, and the producer declares `timeout: Option[Int] = Some(30)`
- **THEN** the field is `timeout: Option[Int] = Some(30)`, because upickle omits the field when it equals that default

#### Scenario: Required scalar
- **WHEN** `i` reads `{"type": "integer"}` and is listed in `required`
- **THEN** the field is `i: Int`

#### Scenario: Scalar not required, producer known
- **WHEN** `s` reads `{"type": "string"}`, is not listed in `required`, and the producer's definition gives it the default `"x"`
- **THEN** the field is `s: String = "x"`

#### Scenario: Scalar not required, producer unknown
- **WHEN** `s` reads `{"type": "string"}`, is not listed in `required`, and the producer's definition is not known
- **THEN** the field is `s: Option[String] = None`, and the report names `s` as a field whose default the schema does not carry

### Requirement: Scalar shapes map to Scala scalars
`{"type": "integer"}` SHALL map to `Int`, `{"type": "number"}` to `Double`, `{"type": "boolean"}` to `Boolean`, `{"type": "string"}` to `String`, the two-way `anyOf` of `integer` and an integral-pattern `string` to `Long`, a `string` with the integral pattern to `BigInt`, `{"type": "null"}` to `Unit`, and the empty schema `{}` to `ujson.Value`. When the producer's definition is known, its narrower type, such as `Short`, `Float`, `Char`, `java.util.UUID`, or `BigDecimal`, SHALL be used instead.

#### Scenario: BigInt
- **WHEN** a property reads `{"type": "string", "pattern": "^-?(0|[1-9][0-9]*)$"}`
- **THEN** the field is `BigInt`

#### Scenario: Plain string with the producer unknown
- **WHEN** a property reads `{"type": "string"}` and the producer's definition is not known
- **THEN** the field is `String`, and the report notes that the schema cannot distinguish `String` from `Char`, `java.util.UUID`, or `BigDecimal`

#### Scenario: Plain string with the producer known
- **WHEN** a property reads `{"type": "string"}` and the producer's definition declares it as `BigDecimal`
- **THEN** the field is `BigDecimal`

#### Scenario: Empty schema
- **WHEN** a property reads `{}`
- **THEN** the field is `ujson.Value`

### Requirement: Collections, tuples, and maps
An `array` with `items` S SHALL map to `Seq[T]`, or to `Set[T]` when `uniqueItems` is true. An inline `array` with `prefixItems` and equal `minItems` and `maxItems` SHALL map to a tuple of the mapped items. An `object` with `additionalProperties` S and no `properties` SHALL map to `Map[K, T]`, where K is `String`, or `Long` for an integral `propertyNames` pattern, `Boolean` for the boolean pattern, `Char` for the single-character constraint, and `java.util.UUID` for `format: uuid`.

#### Scenario: Sequence and set
- **WHEN** a property reads `{"type": "array", "items": {"type": "number"}}`, with or without `"uniqueItems": true`
- **THEN** the field is `Seq[Double]` without it and `Set[Double]` with it

#### Scenario: Tuple
- **WHEN** a property reads `{"type": "array", "prefixItems": [{"type": "number"}, {"type": "number"}], "minItems": 2, "maxItems": 2}`
- **THEN** the field is `(Double, Double)`

#### Scenario: String-keyed map
- **WHEN** a property reads `{"type": "object", "additionalProperties": {"$ref": "#/$defs/example.Value"}}` with no `properties`
- **THEN** the field is `Map[String, Value]`

#### Scenario: Integral-keyed map with the producer unknown
- **WHEN** the map schema also carries `"propertyNames": {"type": "string", "pattern": "^-?(0|[1-9][0-9]*)$"}` and the producer's definition is not known
- **THEN** the field is `Map[Long, T]`, and the report notes that the key could be any integral type

#### Scenario: Integral-keyed map with the producer known
- **WHEN** the same map schema appears and the producer's definition declares `Map[Int, String]`
- **THEN** the field is `Map[Int, String]`

### Requirement: Objects with properties become case classes
An `object` schema with `properties` SHALL map to a case class with one constructor field per property, in the schema's property order. A property name that is a legal Scala identifier SHALL be the field name verbatim; any other SHALL get a camel-case field name annotated with `@key("<property>")`. `additionalProperties: false` SHALL add `@allowUnknownKeys(false)` to the class, and `true` SHALL add nothing. An object with no properties SHALL be an empty case class.

#### Scenario: Property names kept verbatim
- **WHEN** an object has the properties `first_name` and `total_count`
- **THEN** the fields are `first_name` and `total_count`, with no `@key` annotation

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

### Requirement: Discriminated families become sealed families
A `oneOf` whose entries are each either `{"const": "<tag>"}` or an `allOf` of a `$ref` and an object requiring a tag property with a `const` SHALL map to a sealed family, unless it is a `scala.util.Either[...]` entry, which maps to `Either`. The family SHALL have a case object, or parameterless enum case, per bare `const`, and the referenced case class per `allOf` entry, with every case placed in the family's companion.

#### Scenario: Family tagged with $type
- **WHEN** a `$defs` entry reads `{"oneOf": [{"const": "End"}, {"allOf": [{"$ref": "#/$defs/example.LinkedList.Node"}, {"type": "object", "properties": {"$type": {"const": "Node"}}, "required": ["$type"]}]}]}`
- **THEN** the agent declares a sealed family with a case object `End` and a case class `Node`, both in the family's companion, and no annotation for the tag

#### Scenario: Hand-written enum of strings
- **WHEN** a property reads `{"enum": ["A", "B"]}`
- **THEN** it is treated as a `oneOf` of bare `const` entries: a sealed family of case objects `A` and `B`

#### Scenario: Recursive family
- **WHEN** a case class of the family refers back to the family through `$ref`
- **THEN** the generated definitions refer to each other directly, with no indirection added for the recursion

### Requirement: Family cases are named by their tags
Each case of a sealed family SHALL be named exactly as its tag, or SHALL carry `@key("<tag>")` when the tag is not a legal name. A tag property other than `$type` SHALL be declared with `@key("<property>")` on the family. No other handling of the tag SHALL be written.

#### Scenario: Family with a renamed tag property
- **WHEN** the tag property in every `allOf` entry is `kind` instead of `$type`
- **THEN** the family carries `@key("kind")`, and its cases are otherwise unchanged

#### Scenario: Tag that is not a legal name
- **WHEN** a `oneOf` entry reads `{"const": "reddit.com"}`
- **THEN** the case is `@key("reddit.com") case object RedditCom`, or the equivalent enum case

#### Scenario: Case name differs from the tag
- **WHEN** the agent would prefer a case name other than the tag, for example a different capitalisation
- **THEN** it keeps the tag as the name, or names the case as it prefers and annotates it with `@key("<tag>")`

### Requirement: Names derive from $defs keys when the producer is unknown
When the producer's definition is not known, a `$defs` entry outside the `scala` package that gets a definition, not one mapped to a tuple, SHALL be named by the last dotted segment of its key before any `[`. Capitalised segments before it SHALL become enclosing objects that nest it, and lowercase-first segments are the producer's package, which SHALL be replaced by the consumer's. Entries keyed in the `scala` package SHALL follow the tuple, named-tuple, and `Either` rules.

#### Scenario: Nested type
- **WHEN** a `$defs` key reads `upickletest.ColorEnum.Mix`
- **THEN** the definition `Mix` is nested in the companion of `ColorEnum`, in the consumer's package

#### Scenario: Top-level type
- **WHEN** a `$defs` key reads `com.acme.billing.Invoice`
- **THEN** the definition is `Invoice` in the consumer's package, with no `com.acme.billing` in the generated code

#### Scenario: Older definitions keyword
- **WHEN** a document uses `definitions` and `#/definitions/...` references instead of `$defs`
- **THEN** the entries are treated exactly as `$defs` entries

### Requirement: Type arguments in $defs keys become type parameters
When the producer's definition is not known, a `$defs` key outside the `scala` package with a type argument SHALL become a generic definition when the argument's schema is a property's schema, or the S of a property's optional, array, or map shape, and a concrete definition otherwise. Several instantiations of one constructor SHALL share one generic definition.

#### Scenario: Generic type used by a property
- **WHEN** a `$defs` key reads `example.ApiResult[example.Person]` and its property `data` is an optional shape over `{"$ref": "#/$defs/example.Person"}`
- **THEN** the definition is `ApiResult[T]` with `data: Option[T]`, and the root is read as `ApiResult[Person]`

#### Scenario: Two instantiations of one constructor
- **WHEN** `$defs` holds both `example.Box[scala.Int]` and `example.Box[scala.Predef.String]`
- **THEN** one generic definition `Box[T]` is generated

#### Scenario: Argument not used by any property
- **WHEN** a `$defs` key reads `example.Marker[example.Foo]` and no property of the entry refers to `example.Foo`
- **THEN** the definition is the concrete `Marker`, and a second such instantiation gets the argument's name appended, as in `MarkerBar`

### Requirement: Unmapped shapes fall back to ujson.Value
A property whose schema matches none of the mapped shapes SHALL be modelled as `ujson.Value`, and a `string` carrying a `format` this capability does not name SHALL stay `String`. The agent SHALL name every such property in its report and SHALL NOT guess a narrower type.

#### Scenario: Union without a discriminator
- **WHEN** a property reads `{"oneOf": [{"type": "string"}, {"type": "integer"}]}`
- **THEN** the field is `ujson.Value`, and the report names the property

#### Scenario: Pattern properties
- **WHEN** a property schema uses `patternProperties`
- **THEN** the field is `ujson.Value`, and the report names the property

#### Scenario: Date-time string
- **WHEN** a property reads `{"type": "string", "format": "date-time"}`
- **THEN** the field is `String`, and the report names the property and its format

### Requirement: Tuple entries in $defs map to tuples
A `$defs` entry keyed `scala.Tuple<N>[...]`, or `scala.*:[...]` for a tuple written with `*:`, SHALL map to the tuple of its `prefixItems`, mapped in order, wherever it is referred to, and SHALL get no definition of its own.

#### Scenario: Tuple entry in $defs
- **WHEN** a property reads `{"$ref": "#/$defs/scala.Tuple3[scala.Int, scala.Predef.String, scala.Boolean]"}`, and that entry is an `array` with three `prefixItems` and `minItems` and `maxItems` 3
- **THEN** the field is `(Int, String, Boolean)`, and no definition is generated for the entry

#### Scenario: Tuple written with *:
- **WHEN** a property refers to the `$defs` key `scala.*:[scala.Int, scala.*:[scala.Predef.String, scala.*:[scala.Boolean, scala.Tuple$package.EmptyTuple]]]`, whose entry is an `array` with three `prefixItems`
- **THEN** the field is `(Int, String, Boolean)`, and no definition is generated for the entry

### Requirement: Named-tuple entries become case classes named by their use
A `$defs` entry keyed `scala.NamedTuple.NamedTuple[...]` SHALL become a case class with the named tuple's names as fields, typed by its properties, and named, capitalised and camel-cased, after the property that refers to it or, for a type argument of a generic definition, the property typed with that instantiation. When several properties qualify, the first, in the order of the referring entries and of their properties, SHALL name it; when only the root does, its joined field names SHALL.

#### Scenario: Named-tuple entry
- **WHEN** a property `position` refers to the `$defs` key `scala.NamedTuple.NamedTuple[scala.Tuple3["x", "y", "z"], scala.Tuple3[scala.Int, scala.Int, scala.Int]]`, whose entry is an object with the required properties `x`, `y`, and `z`, each an `integer`
- **THEN** the definition is `case class Position(x: Int, y: Int, z: Int)`, and the field is `position: Position`

#### Scenario: Named tuple shared by two properties
- **WHEN** the properties `start` and `end` of one entry, in that order, both refer to the same named-tuple key
- **THEN** one definition, `Start`, is generated, and both fields have the type `Start`

#### Scenario: Named tuple as a type argument
- **WHEN** a property `point` refers to `example.Box[scala.NamedTuple.NamedTuple[scala.Tuple3["x", "y", "z"], scala.Tuple3[scala.Int, scala.Int, scala.Int]]]`, whose property `value` refers to the named-tuple entry
- **THEN** the definitions are `case class Box[T](value: T)` and `case class Point(x: Int, y: Int, z: Int)`, and the field is `point: Box[Point]`

### Requirement: Inline pairs stay pairs unless a named tuple is shown
An inline two-item `prefixItems` array SHALL map to a pair unless a named tuple is shown there. When none of the producer's definition, a sample payload, or an enclosing `$defs` key that prints the pair's type settles which it is, the agent SHALL name the property in its report as a pair that could be a two-field named tuple, which upickle writes as an object.

#### Scenario: Pair with the producer unknown
- **WHEN** a property `point` reads `{"type": "array", "prefixItems": [{"type": "integer"}, {"type": "integer"}], "minItems": 2, "maxItems": 2}`, and neither the producer's definition nor a sample payload is at hand
- **THEN** the field is `point: (Int, Int)`, and the report names `point` as a pair that could be a two-field named tuple

#### Scenario: Pair confirmed by the source
- **WHEN** the producer's definition declares that property as `point: (Int, Int)`
- **THEN** the field is `point: (Int, Int)`, and the report does not name it

#### Scenario: Pair shown by an enclosing key
- **WHEN** a property `pp` refers to `example.Box[scala.Tuple2[scala.Int, scala.Int]]`, whose property `value` is an inline two-item `prefixItems` array of `integer`s
- **THEN** the field is `pp: Box[(Int, Int)]`, and the report names `pp` neither as a possible named tuple nor among the generator defects

### Requirement: Named pairs become case classes
Where the producer's definition, a sample payload, or an enclosing `$defs` key that prints `scala.NamedTuple.NamedTuple[scala.Tuple2[...], ...]` for it shows that an inline pair is a two-field named tuple, the pair SHALL instead be a case class with the named tuple's names as fields, typed by the items in order and named as a named-tuple entry is, and the agent SHALL list it among the generator defects. Where the definition and a payload disagree, the payload SHALL decide.

#### Scenario: Named pair in the producer's source
- **WHEN** the producer's definition declares the inline pair `point` as `point: (x: Int, y: Int)`
- **THEN** the field is `point: Point` with `case class Point(x: Int, y: Int)`, and the report lists `point` among the generator defects

#### Scenario: Sample payload shows an object
- **WHEN** the producer's definition is not known and a sample payload the user supplies has `"point": {"x": 1, "y": 2}` for the inline pair `point`
- **THEN** the field is `point: Point` with `case class Point(x: Int, y: Int)`, its fields named by the payload's keys in order and typed by the items in order, and the report lists `point` among the generator defects

#### Scenario: Named pair shown by an enclosing key
- **WHEN** a property `point` refers to the `$defs` key `example.Box[scala.NamedTuple.NamedTuple[scala.Tuple2["x", "y"], scala.Tuple2[scala.Int, scala.Int]]]`, whose property `value` is an inline two-item `prefixItems` array of `integer`s
- **THEN** the definitions are `case class Box[T](value: T)` and `case class Point(x: Int, y: Int)`, the field is `point: Box[Point]`, and the report lists `point` among the generator defects and does not name it as a possible named tuple

#### Scenario: Source and payload disagree on a pair
- **WHEN** the producer's definition declares `point: (Int, Int)` and a sample payload has `"point": {"x": 1, "y": 2}`
- **THEN** the field is `point: Point` with `case class Point(x: Int, y: Int)`, and the report names the disagreement with the source

### Requirement: Positional case classes follow what upickle writes
In upickle-jsonschema output, an entry that is an `array` with `prefixItems` and equal `minItems` and `maxItems`, keyed outside the `scala` package, SHALL become a case class with the fields `_1` to `_N`, typed by the items in order, because upickle writes and reads such a class as an object with those keys, and the agent SHALL list it among the generator defects. In any other document such an entry SHALL map to a tuple, with no definition.

#### Scenario: Case class with positional field names
- **WHEN** the `$defs` entry `example.Pair` reads `{"type": "array", "prefixItems": [{"type": "integer"}, {"type": "string"}], "minItems": 2, "maxItems": 2}`
- **THEN** the definition is `case class Pair(_1: Int, _2: String)`, which reads `{"_1": 1, "_2": "a"}`, and the report lists `Pair` among the generator defects

#### Scenario: Hand-written prefixItems entry
- **WHEN** a hand-written document has the `definitions` entry `Coordinates`, an `array` with two `prefixItems` of `{"type": "number"}` and `minItems` and `maxItems` 2
- **THEN** a property referring to it is `(Double, Double)`, no definition is generated, and nothing is listed among the generator defects

### Requirement: Either entries map to Either
A `$defs` entry keyed `scala.util.Either[...]`, a `oneOf` over entries keyed `scala.util.Left[...]` and `scala.util.Right[...]` whose only property is `value`, SHALL map to `Either[A, B]` of the two `value` types instead of a sealed family, with no definition for the three entries, because upickle writes an `Either` as `[0, a]` or `[1, b]`. The agent SHALL list it among the generator defects in its report.

#### Scenario: Either entry
- **WHEN** a property `e` refers to the `$defs` key `scala.util.Either[scala.Int, scala.Predef.String]`, a `oneOf` over `scala.util.Left[scala.Int, java.lang.String]` and `scala.util.Right[scala.Int, java.lang.String]` tagged by `$type`
- **THEN** the field is `e: Either[Int, String]`, which reads `[0, 1]` and `[1, "s"]`, no definition is generated for the three entries, and the report lists `e` among the generator defects

### Requirement: Maps keyed by other types take the producer's key type
When the producer's definition declares a map key type that upickle does not write as an object key, the field SHALL take that key type, mirrored from the source, as the schema omits it, because upickle writes such a map as `[key, value]` pairs, and the agent SHALL list it among the generator defects. Object keys are strings, `Symbol`, `Char`, `Boolean`, `Unit`, `java.util.UUID`, the numeric types with `BigInt` and `BigDecimal`, `Duration`, and types wrapped in `stringKeyRW`.

#### Scenario: Map keyed by an enum in the producer's source
- **WHEN** the producer's definition declares `counts: Map[Color, Int]`, where `Color` is an enum, and the schema shows `{"type": "object", "additionalProperties": {"type": "integer"}}`
- **THEN** the field is `counts: Map[Color, Int]`, `Color` is mirrored from the producer's source although no `$defs` entry describes it, the field reads `[["Red", 1]]`, and the report lists `counts` among the generator defects

### Requirement: Maps without key information stay string-keyed
When the producer's definition is not known, an `object` with `additionalProperties` and neither `properties` nor `propertyNames` SHALL stay `Map[String, T]`, and the agent SHALL name it in its report as a map whose producer may use another key type. When a sample payload shows an array of pairs there whose keys are not strings, the agent SHALL ask the user for the key type.

#### Scenario: Map with no key information
- **WHEN** a property `counts` reads `{"type": "object", "additionalProperties": {"type": "integer"}}` and the producer's definition is not known
- **THEN** the field is `counts: Map[String, Int]`, and the report names `counts` as a map whose producer may use another key type

#### Scenario: Payload shows pairs with object keys
- **WHEN** a sample payload has `"counts": [[{"a": 1}, 2]]` for that property
- **THEN** the agent asks the user for the key type before generating the field

### Requirement: Generator rules apply only to upickle-jsonschema output
A document SHALL count as upickle-jsonschema output when the user says it is, or when its `$defs` keys are Scala type names with dotted package paths or `scala.` type arguments. The requirements on positional case classes, on inline pairs that could be named tuples, on `Either` entries, on sub-families, and on maps without key information SHALL apply only to such output; in any other document a pair is a pair and a string-keyed map is a map, with no report entry.

#### Scenario: Hand-written document
- **WHEN** a hand-written document, with `definitions` keys such as `Address` and no `$schema` from upickle-jsonschema, has a property `point` that is an inline two-item `prefixItems` array of `integer`s and a property `tags` that is an `object` with `additionalProperties`
- **THEN** the fields are `point: (Int, Int)` and `tags: Map[String, T]`, and the report names neither as a possible named tuple or another key type

#### Scenario: Producer in the default package
- **WHEN** the user says the schema came from upickle-jsonschema and its `$defs` keys are bare names such as `Pair`
- **THEN** the document counts as upickle-jsonschema output, and a `Pair` entry with the `prefixItems` shape becomes a case class with positional fields

### Requirement: Sub-families are read by their inner tags
A `oneOf` case that is an `allOf` of a `$ref` to another `oneOf` entry and a tag `const` is a sub-family, whose `const` upickle never writes: it writes the tag of the inner case. The agent SHALL declare both families as `sealed trait`s, the inner extending the outer, on every Scala version, SHALL annotate every case of both families with `@key("<tag>")` from its `oneOf`, SHALL generate no case for the outer `const`, and SHALL list the sub-family among the generator defects.

#### Scenario: Sealed family with a sealed sub-trait
- **WHEN** `example.Event` is a `oneOf` of `{"const": "Ping"}` and an `allOf` of `{"$ref": "#/$defs/example.UserEvent"}` tagged `UserEvent`, and `example.UserEvent` is a `oneOf` over `example.Created` tagged `Created`
- **THEN** the definitions are `sealed trait Event` with `@key("Ping") case object Ping`, `sealed trait UserEvent extends Event`, and `@key("Created") case class Created(id: Int) extends UserEvent`, which read `"Ping"` and `{"$type": "Created", "id": 1}` as an `Event` on Scala 3 and on 2.13, and the report lists `UserEvent` among the generator defects

### Requirement: Unknown defaults under boxed options ask the user
When the generated types are read through a pickler with `optionsAsNulls = false` and the producer's definition is not known, a non-Option field that is not required SHALL NOT become `Option[T] = None`, because that pickler reads an `Option` only from an array while the producer writes the field's bare value. The agent SHALL ask the user for the producer's default of each such field.

#### Scenario: Unknown default under a boxed pickler
- **WHEN** the document's optional fields have the boxed shape with no null alternative, and `name` reads `{"type": "string"}`, is not listed in `required`, and the producer's definition is not known
- **THEN** the agent asks the user for the default of `name` instead of generating `name: Option[String] = None`, which could not read the producer's `{"name": "y"}`
