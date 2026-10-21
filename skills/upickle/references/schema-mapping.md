# upickle schema mapping reference

Background for the rules in [../SKILL.md](../SKILL.md). The skill file is what you obey. This file shows what upickle-jsonschema emits for each Scala type, the full mapping back to Scala for both 2.13 and 3, and one conversion worked end to end.

## Contents

1. [The shapes upickle-jsonschema emits](#1-the-shapes-upickle-jsonschema-emits)
2. [Shape-to-type table](#2-shape-to-type-table)
3. [Ambiguity defaults](#3-ambiguity-defaults)
4. [Names and nesting from `$defs` keys](#4-names-and-nesting-from-defs-keys)
5. [Sealed families and tags](#5-sealed-families-and-tags)
6. [Translating Scala 3 producer idioms to Scala 2.13](#6-translating-scala-3-producer-idioms-to-scala-213)
7. [Producer configurations: boxed options and renamed tags](#7-producer-configurations-boxed-options-and-renamed-tags)
8. [Hand-written schema forms](#8-hand-written-schema-forms)
9. [Worked example](#9-worked-example)
10. [Version checked](#10-version-checked)

## 1. The shapes upickle-jsonschema emits

A producer calls `upickle.default.schema[T]` (an extension on `upickle.Api`, imported from `upickle.jsonschema`) and gets a Draft 2020-12 document. Every case class, sealed family, and tuple of an arity other than 2 that `T` reaches is one `$defs` entry, keyed by the type's full name as the Scala 3 compiler prints it, and the root is a `$ref` to the entry for `T`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$defs": {
    "example.Box[scala.Int]": {
      "type": "object",
      "properties": {"value": {"type": "integer"}},
      "required": ["value"],
      "additionalProperties": true
    }
  },
  "$ref": "#/$defs/example.Box[scala.Int]"
}
```

When `T` itself has no entry, the root schema sits in `"allOf": [<root>]` next to `$defs`, or, with no `$defs` at all, carries `$schema` itself. Every snippet below is the generator's output for the Scala type named with it.

### Case classes

`case class Inner(a: Int, b: String = "x")`:

```json
{
  "type": "object",
  "properties": {"a": {"type": "integer"}, "b": {"type": "string"}},
  "required": ["a"],
  "additionalProperties": true
}
```

- `properties` follow declaration order.
- `required` lists every field without a default value, whatever its type: `x: Option[Int]` is required, and `b: String = "x"` is not.
- No `default` keyword is ever written, so the value `"x"` is not in the schema. The producer also omits a field whose value equals its default when it writes (`serializeDefaults` is false by default).
- `additionalProperties` is the Api's `allowUnknownKeys`, `true` by default, or the class's annotation: `@allowUnknownKeys(false) case class Strict(x: Int)` gives `"additionalProperties": false`.
- `case class Empty()` gives `{"type": "object", "properties": {}, "required": [], "additionalProperties": true}`.
- `@key("content-type") contentType: String` gives the property `content-type`, listed under that name in `required`.
- `@flatten inner: Hoisted`, with `case class Hoisted(h1: Int, h2: String = "x")`, hoists `h1` and `h2` into the outer object's `properties`, with `h1` in its `required`. No property `inner` appears.

### Optional values

`Option[Int]`, while `optionsAsNulls` is true, the default:

```json
{
  "anyOf": [
    {"type": "integer"},
    {"type": "null"},
    {"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "integer"}}
  ]
}
```

`Option[Option[Boolean]]` is the same shape with the inner optional shape as S, in the first alternative and in `items`:

```json
{
  "anyOf": [
    {"anyOf": [{"type": "boolean"}, {"type": "null"}, {"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "boolean"}}]},
    {"type": "null"},
    {
      "type": "array",
      "minItems": 0,
      "maxItems": 1,
      "items": {"anyOf": [{"type": "boolean"}, {"type": "null"}, {"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "boolean"}}]}
    }
  ]
}
```

`Option[Int]` from a producer whose Api overrides `optionsAsNulls = false`:

```json
{"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "integer"}}
```

Its `Option[Option[Int]]` nests the boxed array, `{"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "array", "minItems": 0, "maxItems": 1, "items": {"type": "integer"}}}`, and such a producer writes `[[2]]` for `Some(Some(2))`, `[[]]` for `Some(None)`, and `[]` for `None`. Read through a pickler with that setting, a bare value fails and a bare `null` reads as a Scala `null`.

What upickle 4.4.3 writes and reads with default settings, as run:

| Value or field | Written | Read |
|---|---|---|
| `Some(1): Option[Int]` | `1` | `1` is `Some(1)`, `null` is `None`, `[1]` fails |
| `Some(Some(1)): Option[Option[Int]]` | `[1]` | `[1]` is `Some(Some(1))`, `1` fails |
| `Some(None): Option[Option[Int]]` | `[null]` | `[null]` is `Some(None)` |
| `None: Option[Option[Int]]` | `[]` | `[]` is `None`; a bare `null` reads as a Scala `null`, not `None` |
| field `x: Option[Int]`, no default | `"x": null` for `None` | a missing key fails with `missing keys in dictionary` |
| field `x: Option[Int] = None` | nothing for `None` | a missing key is `None` |

So a first-level `Option[T]` never meets the array alternative, and a nested one only ever meets arrays. This is why the skill writes no code for the array, and why its test reads a nested field from `[x]`, `[null]`, and `[]`.

Boxing follows the type a field is declared with, not its type argument. `case class R[T](data: Option[T])` used as `R[Option[Int]]` gets the nested schema of `Option[Option[Int]]`, yet upickle writes `Some(Some(3))` as `3` and both `Some(None)` and `None` as `null`, because the argument's `ReadWriter` is not recognised as an option. A field with a default other than `None`, such as `timeout: Option[Int] = Some(30)`, is omitted when it equals that default, so `{}` reads `Some(30)` only in a class that declares the same default.

### Scalars

| Scala | Schema |
|---|---|
| `Int`, `Short`, `Byte` | `{"type": "integer"}` |
| `Long` | `{"anyOf": [{"type": "integer"}, {"type": "string", "pattern": "^-?(0\|[1-9][0-9]*)$"}]}` |
| `BigInt` | `{"type": "string", "pattern": "^-?(0\|[1-9][0-9]*)$"}` |
| `Double`, `Float` | `{"type": "number"}` |
| `String`, `Char`, `Symbol`, `java.util.UUID`, `BigDecimal` | `{"type": "string"}` |
| `Boolean` | `{"type": "boolean"}` |
| `Unit` | `{"type": "null"}` |
| `ujson.Value` | `{}` |

upickle writes a `Long` as a number up to 2^53 in magnitude and as a string beyond: `123L` is `123`, and `Long.MaxValue` is `"9223372036854775807"`. The `Long` reader takes both, which is what the two-way `anyOf` says. `Int` reads a numeric string too.

### Collections and tuples

| Scala | Schema |
|---|---|
| `Seq[Double]`, `List[Double]`, `Vector[Double]`, `Array[Double]` | `{"type": "array", "items": {"type": "number"}}` |
| `Set[Double]` | `{"type": "array", "items": {"type": "number"}, "uniqueItems": true}` |
| `(Double, Double)` | `{"type": "array", "prefixItems": [{"type": "number"}, {"type": "number"}], "minItems": 2, "maxItems": 2}`, inline |
| `(Int, String, Boolean)` | `{"$ref": "#/$defs/scala.Tuple3[scala.Int, scala.Predef.String, scala.Boolean]"}`, whose entry is the `prefixItems` array with `"minItems": 3, "maxItems": 3` |
| `Int *: String *: Boolean *: EmptyTuple` | `{"$ref": "#/$defs/scala.*:[scala.Int, scala.*:[scala.Predef.String, scala.*:[scala.Boolean, scala.Tuple$package.EmptyTuple]]]"}`, the same `prefixItems` entry, written `[1, "a", true]` |

### Maps

`Map[K, V]` is `{"type": "object", "additionalProperties": <V>}`, plus `propertyNames` by key type. upickle writes the map as an object, with the keys as strings, only when the key type is one it writes as an object key: strings, `Symbol`, `Char`, `Boolean`, `Unit`, `java.util.UUID`, the numeric types with `BigInt` and `BigDecimal`, `Duration`, and types wrapped in `stringKeyRW`. `Map(1L -> "one")` is `{"1": "one"}`. Any other key type is written as an array of `[key, value]` pairs under the same schema, and the key type's own schema is not emitted at all: `Map(Color.Red -> 1)` with an enum key is `[["Red", 1]]`, and `Map(Inner(1) -> 2)` with a case-class key is `[[{"a": 1}, 2]]`. A `Map[String, T]` reads both forms when the keys are strings, so it reads `[["Red", 1]]` but not `[[{"a": 1}, 2]]`.

| K | `propertyNames` |
|---|---|
| `String` | none |
| `Int`, `Long`, `Short`, `Byte`, `BigInt` | `{"type": "string", "pattern": "^-?(0\|[1-9][0-9]*)$"}` |
| `Boolean` | `{"type": "string", "pattern": "^(true\|false)$"}` |
| `Char` | `{"type": "string", "minLength": 1, "maxLength": 1}` |
| `java.util.UUID` | `{"type": "string", "format": "uuid"}` |
| any key type upickle does not write as an object key | none, and the map is written as pairs |

### Sealed families

`sealed trait LinkedList` with `case object End` and `case class Node(value: Int, next: LinkedList)` in its companion, as two `$defs` entries:

```json
{
  "example.LinkedList.Node": {
    "type": "object",
    "properties": {"value": {"type": "integer"}, "next": {"$ref": "#/$defs/example.LinkedList"}},
    "required": ["value", "next"],
    "additionalProperties": true
  },
  "example.LinkedList": {
    "oneOf": [
      {"const": "End"},
      {
        "allOf": [
          {"$ref": "#/$defs/example.LinkedList.Node"},
          {"type": "object", "properties": {"$type": {"const": "Node"}}, "required": ["$type"]}
        ]
      }
    ]
  }
}
```

A Scala 3 `enum` gives the same shape. The `oneOf` entries follow the cases' declaration order. The tag property is `$type`, or the family's `@key`, or the Api's `tagName`. The tag value is the case's short name while `objectTypeKeyWriteFullyQualified` is false, the default, or the case's `@key`. Section 5 says how the short name is derived.

A family with a sealed sub-trait among its cases, `Event` with `case object Ping` and `sealed trait UserEvent` holding `case class Created(id: Int)`, gets for the sub-trait a `oneOf` case tagged with its name, `{"allOf": [{"$ref": "#/$defs/example.UserEvent"}, {"type": "object", "properties": {"$type": {"const": "UserEvent"}}, "required": ["$type"]}]}`, and `example.UserEvent` is itself a `oneOf` over `Created`. upickle never writes `UserEvent`: it writes `"Ping"` and `{"$type": "Created", "id": 1}`.

`Either[Int, String]` gets the family `scala.util.Either[scala.Int, scala.Predef.String]`, a `oneOf` over `scala.util.Left[scala.Int, java.lang.String]` and `scala.util.Right[scala.Int, java.lang.String]`, each an object with the single property `value`, tagged `Left` and `Right`. upickle writes `Left(1)` as `[0, 1]` and `Right("s")` as `[1, "s"]`.

### `$defs` keys

Keys are the compiler's printed types: `example.Payload`; `example.Kind.Custom` for a case inside `Kind`; `example.Box[scala.Int]` and `example.Box[scala.Predef.String]` for two instantiations; `example.Marker[example.Inner]`; `scala.Tuple3[scala.Int, scala.Predef.String, scala.Boolean]`, and `scala.*:[scala.Int, scala.*:[scala.Predef.String, scala.*:[scala.Boolean, scala.Tuple$package.EmptyTuple]]]` for the same tuple written with `*:`; `scala.util.Either[scala.Int, scala.Predef.String]` with its `scala.util.Left[...]` and `scala.util.Right[...]` entries. A named tuple of an arity other than 2, such as `(x: Int, y: Int, z: Int)`, gets the key `scala.NamedTuple.NamedTuple[scala.Tuple3["x", "y", "z"], scala.Tuple3[scala.Int, scala.Int, scala.Int]]` (with the inner quotes escaped in JSON), and its entry is an object with `x`, `y`, and `z` as required properties. A pair, named or not, has no entry of its own, but a key that takes it as a type argument prints it in full: `Box[(x: Int, y: Int)]` is keyed `example.Box[scala.NamedTuple.NamedTuple[scala.Tuple2["x", "y"], scala.Tuple2[scala.Int, scala.Int]]]`, and `Box[(Int, Int)]` is keyed `example.Box[scala.Tuple2[scala.Int, scala.Int]]`.

### Where the schema and the wire disagree in 4.4.3

Six producer types get a schema that does not describe what upickle writes for them:

- A named tuple of arity 2, `(x: Int, y: Int)`, gets the inline two-item `prefixItems` array of a plain pair, but upickle, with the `upickle-implicits-named-tuples` module, writes the object `{"x": 1, "y": 2}`. It is inlined the same way inside `Option`, `Seq`, or a map value, and as a type argument, where only the enclosing key shows its names.
- A case class whose fields are named `_1`, `_2`, ..., such as `case class Pair(_1: Int, _2: String)`, gets a `prefixItems` array entry under its own key, but upickle writes and reads it as the object `{"_1": 1, "_2": "a"}`.
- `Either[A, B]` gets a `$type`-tagged family over `scala.util.Left` and `scala.util.Right` objects, but is written as `[0, a]` or `[1, b]`.
- A map whose key type upickle does not write as an object key gets the object schema of a string-keyed map, but is written as `[key, value]` pairs, and its key type's schema is not emitted.
- A sealed trait with a sealed sub-trait among its cases gets a `oneOf` case tagged with the sub-trait's name, which upickle never writes.
- A generic `Option[T]` field instantiated with an `Option` gets the nested optional schema, but is written unboxed, as `x` or `null`.

The skill's rule for these ("Shapes the generator misdescribes" in `SKILL.md`) is to generate the type that reads what upickle writes, list it under `Generator defects:`, and test it with a document in that form. Four of the shapes are recognisable from the schema alone: a positional entry becomes a case class with the fields `_1` to `_N`; an `Either` entry becomes `Either[A, B]`, with no definition for its three entries; a sub-family becomes two `sealed trait`s, the inner extending the outer, with `@key` on every case of both; and a nested optional shape inside a type argument stays `Option[T]` in the generic definition. The other two are not. An inline pair stays a pair, reported as possibly a named tuple, until the producer's source, a sample payload, or an enclosing key shows the names, with a payload winning over the source. A map without `propertyNames` stays `Map[String, T]`, reported, until the source gives the key type; a payload with keys that are not strings is a question for the user. These rules apply only to upickle-jsonschema output, which the user names or the `$defs` keys show: in a hand-written document a `prefixItems` entry is a tuple, and a pair or a string-keyed map is not reported.

## 2. Shape-to-type table

Field declarations unless the row says otherwise. On Scala 2.13 the imports are `import upickle.default.{macroRW, ReadWriter}` and, for annotations, `import upickle.implicits.{allowUnknownKeys, key}`; on Scala 3, `import upickle.default.ReadWriter` and the same annotations.

| Schema shape | Scala 2.13 | Scala 3 |
|---|---|---|
| optional shape over `{"type": "integer"}`, in `required` | `count: Option[Int]` | `count: Option[Int]` |
| optional shape over a `$ref` | `next: Option[Node]` | `next: Option[Node]` |
| optional shape, not in `required` | `simple2: Option[SimpleEnum] = None` | `simple2: Option[SimpleEnum] = None` |
| optional shape nested in an optional shape | `flag: Option[Option[Boolean]]` | `flag: Option[Option[Boolean]]` |
| optional shape nested inside a type argument | `case class R[T](data: Option[T])` with `implicit def rw[T: ReadWriter]`, read as `R[Option[Int]]` | `case class R[T](data: Option[T]) derives ReadWriter`, read as `R[Option[Int]]` |
| optional shape not in `required`, producer default `Some(30)` | `timeout: Option[Int] = Some(30)` | `timeout: Option[Int] = Some(30)` |
| boxed array with no null alternative (`optionsAsNulls = false` producer) | `count: Option[Int]`, derived from the section 7 object | `count: Option[Int]`, derived from the section 7 object |
| `{"type": ["string", "null"]}`, or a two-way `anyOf` with `null` | `nick: Option[String]` | `nick: Option[String]` |
| `{"type": "integer"}` | `i: Int` | `i: Int` |
| `{"type": "number"}` | `d: Double` | `d: Double` |
| `{"type": "boolean"}` | `b: Boolean` | `b: Boolean` |
| `{"type": "string"}` | `s: String` | `s: String` |
| `anyOf` of `integer` and an integral-pattern `string` | `id: Long` | `id: Long` |
| `string` with the integral pattern | `big: BigInt` | `big: BigInt` |
| `{"type": "null"}` | `u: Unit` | `u: Unit` |
| `{}` | `raw: ujson.Value` | `raw: ujson.Value` |
| `array` with `items` | `xs: Seq[Double]` | `xs: Seq[Double]` |
| `array` with `items` and `"uniqueItems": true` | `xs: Set[Double]` | `xs: Set[Double]` |
| `array` with two `prefixItems`, `minItems` and `maxItems` 2 | `point: (Double, Double)` | `point: (Double, Double)` |
| `$ref` to a `scala.Tuple3[...]` entry | `t: (Int, String, Boolean)` | `t: (Int, String, Boolean)` |
| `$ref` to a `scala.*:[...]` entry | `t: (Int, String, Boolean)` | `t: (Int, String, Boolean)` |
| inline pair that the source, a payload, or an enclosing key shows to be a named tuple | `point: Point` and `case class Point(x: Int, y: Int)` with its companion | `point: Point` and `case class Point(x: Int, y: Int) derives ReadWriter` |
| positional entry: `prefixItems` under a key outside the `scala` package | `case class Pair(_1: Int, _2: String)` and `object Pair { implicit val rw: ReadWriter[Pair] = macroRW }` | `case class Pair(_1: Int, _2: String) derives ReadWriter` |
| `$ref` to a `scala.util.Either[...]` entry | `e: Either[Int, String]` | `e: Either[Int, String]` |
| map with no `propertyNames` | `byName: Map[String, Value]` | `byName: Map[String, Value]` |
| map with the integral pattern | `labels: Map[Long, String]` | `labels: Map[Long, String]` |
| map with the boolean pattern | `flags: Map[Boolean, Int]` | `flags: Map[Boolean, Int]` |
| map with `minLength` and `maxLength` 1 | `byChar: Map[Char, Int]` | `byChar: Map[Char, Int]` |
| map with `"format": "uuid"` | `byId: Map[java.util.UUID, Int]` | `byId: Map[java.util.UUID, Int]` |
| map whose producer key type is not an object key (source only) | `counts: Map[Color, Int]`, with `Color` mirrored | `counts: Map[Color, Int]`, with `Color` mirrored |
| `$ref` to an object entry | `address: Address` | `address: Address` |
| property name that is not an identifier | `@key("content-type") contentType: String` | `@key("content-type") contentType: String` |
| object entry with `properties` | `case class Address(street: String)` and `object Address { implicit val rw: ReadWriter[Address] = macroRW }` | `case class Address(street: String) derives ReadWriter` |
| object entry with `"additionalProperties": false` | `@allowUnknownKeys(false) case class Strict(x: Int)` and its companion as above | `@allowUnknownKeys(false) case class Strict(x: Int) derives ReadWriter` |
| object entry with no properties | `case class Ping()` and its companion as above | `case class Ping() derives ReadWriter` |
| `oneOf` of bare `const` and tagged `allOf` entries | `sealed trait` with case objects and case classes in its companion (section 5) | `enum` with parameterless and parametrised cases (section 5) |
| sub-family: a `oneOf` case that is an `allOf` over another `oneOf` | `sealed trait Event`, `sealed trait UserEvent extends Event`, `@key` on every case (section 5) | the same `sealed trait`s with `derives ReadWriter`, `@key` on every case (section 5) |
| `string` with a `format` | `at: String` | `at: String` |
| anything else | `extra: ujson.Value` | `extra: ujson.Value` |

## 3. Ambiguity defaults

Several Scala types produce the same shape. Without the producer's source, the skill picks the widest type that reads every one of them, and names the property in the report. The last two rows are exceptions, for upickle-jsonschema output only: they are the schema's own reading, not the widest type, because a two-field named tuple and a map keyed by an enum or a case class share their schema, and neither default reads them.

| Shape | Default | Also produced by | Why |
|---|---|---|---|
| `{"type": "string"}` | `String` | `Char`, `Symbol`, `java.util.UUID`, `BigDecimal` | `String` reads any of them |
| `{"type": "integer"}` | `Int` | `Short`, `Byte` | `Long` has its own shape, so a bare `integer` is at most an `Int` |
| `{"type": "number"}` | `Double` | `Float` | reads every `Float` |
| integral `propertyNames` | `Map[Long, V]` | `Int`, `Short`, `Byte`, `BigInt` keys | the pattern admits every `Long`; an `Int` key could not hold them all |
| `array` with `items` | `Seq[T]` | `List`, `Vector`, `Array` | all read the same documents |
| field not in `required`, not the optional shape | `Option[T] = None` | any default value | the schema never carries the default, and none is invented; under an `optionsAsNulls = false` pickler the agent asks for it instead |
| `string` with a `format` other than a map key's `uuid` | `String` | anything rendered as a string | lossless; upickle has no `java.time` readers |
| anything not in section 2 | `ujson.Value` | anything | lossless; the user sees it in the report |
| inline two-item `prefixItems` | `(A, B)` | a two-field named tuple, written as an object | the schema's own reading |
| `object` with `additionalProperties` and no `propertyNames` | `Map[String, V]` | maps keyed by a type upickle writes as pairs | the schema's own reading; reads string keys in either form |

When the producer's source is at hand, the narrow type and the real default are mirrored instead, and the property is not a default to report.

## 4. Names and nesting from `$defs` keys

These rules apply when the producer's source is not available. They read the key as the compiler printed it. Rules 1 to 5 apply to keys outside the `scala` package; a key in the `scala` package follows rule 6 or 7, or is an `Either` (section 1):

1. The definition's name is the last dotted segment before any `[`.
2. Capitalised segments before the name are enclosing objects, and the definition goes in their companions. When the enclosing name is itself a family in `$defs`, the nested entry is one of its cases (section 5); otherwise the enclosing name is an `object` that holds the nested definitions.
3. Segments starting with a lowercase letter are the producer's package. The consumer's package replaces them.
4. A type argument becomes a type parameter when the argument's schema is the schema, or the S of an optional, array, or map shape, of a property of the entry. Instantiations of one constructor share one generic definition. One argument is `T`; several are `A`, `B`, and on.
5. When no property uses the argument, the definition is concrete. The first instantiation takes the plain name, and each further one has the argument's name appended.
6. A `scala.NamedTuple.NamedTuple[...]` key becomes a case class with the tuple's names as fields, named, capitalised and camel-cased, after the property that refers to it or, for a type argument of a generic definition, after the property typed with that instantiation. When several properties qualify, the first in the order of the referring entries and of their properties names it; when only the root refers to it, its field names joined do.
7. A `scala.Tuple<N>[...]` key, or `scala.*:[...]` for a tuple written with `*:`, is a tuple: the property that refers to it takes the tuple type, and no definition is generated.
8. A `definitions` section and `#/definitions/<key>` references are treated exactly as `$defs`.

| Key | Definition |
|---|---|
| `upickletest.ColorEnum.Mix` | `Mix`, nested in the companion of `ColorEnum`, in the consumer's package |
| `com.acme.billing.Invoice` | `Invoice`, top level in the consumer's package; no `com.acme.billing` in the code |
| `example.ApiResult[example.Person]`, `data` the optional shape over `example.Person` | `ApiResult[T]` with `data: Option[T]`; the root is read as `ApiResult[Person]` |
| `example.Box[scala.Int]` and `example.Box[scala.Predef.String]`, each with `value` | one `Box[T]` with `value: T` |
| `example.Marker[example.Foo]` and `example.Marker[example.Bar]`, no property using either | `Marker` and `MarkerBar` |
| `scala.NamedTuple.NamedTuple[scala.Tuple3["x", "y", "z"], scala.Tuple3[scala.Int, scala.Int, scala.Int]]`, used by `position` | `case class Position(x: Int, y: Int, z: Int)` |
| `example.Box[scala.NamedTuple.NamedTuple[scala.Tuple3["x", "y", "z"], scala.Tuple3[scala.Int, scala.Int, scala.Int]]]`, used by `point` | `Box[T]` and `case class Point(x: Int, y: Int, z: Int)`; the field is `point: Box[Point]` |
| `scala.Tuple3[scala.Int, scala.Predef.String, scala.Boolean]`, or the same tuple keyed `scala.*:[...]` | none; the property is `(Int, String, Boolean)` |
| `scala.util.Either[scala.Int, scala.Predef.String]` with its `Left` and `Right` entries | none; the property is `Either[Int, String]` |

The generic case on both versions, for `example.ApiResult[example.Person]` with properties `data` and `error`, both optional and required:

```scala
// Scala 3
case class ApiResult[T](data: Option[T], error: Option[String]) derives ReadWriter

// Scala 2.13
case class ApiResult[T](data: Option[T], error: Option[String])
object ApiResult {
  implicit def rw[T: ReadWriter]: ReadWriter[ApiResult[T]] = macroRW
}
```

## 5. Sealed families and tags

**How upickle derives a case's tag.** For a Scala 3 `enum`, the tag is the case's own name. For a `sealed trait`, on Scala 2 and 3, it is the case's full name with the leading segments shared by all cases of the family dropped: `example.LinkedList.End` and `example.LinkedList.Node` share `example.LinkedList`, so the tags are `End` and `Node`. Were a third case, `case class Leaf`, declared at the top level of package `example` instead of in the companion, the shared prefix would shrink to `example`, and the tags would become `LinkedList.End`, `LinkedList.Node`, and `Leaf`. Placing every case in the family's companion keeps each tag equal to the case's simple name, which is what the schema's `const` values are.

**`@key` on a case** replaces its tag, for writing and reading: `@key("reddit.com") case object RedditCom` is written as `"reddit.com"`.

**`@key` on the family** replaces the tag property for that family: `@key("kind") sealed trait Site` writes `{"kind": "Other", "host": "a"}`. The Api's `tagName` (section 7) replaces it for every family at once.

**Singleton cases** (case objects, parameterless enum cases) are written as the bare tag string and read from it: `"End"`.

**Reading is lenient about position and, for a sealed trait, about qualification.** The tag may appear anywhere in the object: `{"value": 1, "$type": "Node", "next": "End"}` reads. A `sealed trait` also accepts the fully qualified tag, `"example.LinkedList.Node"` or `"example.LinkedList.End"`; a Scala 3 `enum` accepts only the short one. A producer configured with `objectTypeKeyWriteFullyQualified = true` writes full names as the `const` values of a sealed trait's cases; those are not legal names, so each consumer case carries `@key` with the full name.

Both versions, checked against upickle 4.4.3:

```scala
// Scala 3
@key("kind")
enum Site derives ReadWriter:
  @key("reddit.com") case RedditCom
  case Other(host: String)

enum LinkedList derives ReadWriter:
  case End
  case Node(value: Int, next: LinkedList)
```

```scala
// Scala 2.13
@key("kind")
sealed trait Site
object Site {
  @key("reddit.com") case object RedditCom extends Site {
    implicit val rw: ReadWriter[RedditCom.type] = macroRW
  }
  case class Other(host: String) extends Site
  object Other {
    implicit val rw: ReadWriter[Other] = macroRW
  }
  implicit val rw: ReadWriter[Site] = macroRW
}

sealed trait LinkedList
object LinkedList {
  case object End extends LinkedList {
    implicit val rw: ReadWriter[End.type] = macroRW
  }
  case class Node(value: Int, next: LinkedList) extends LinkedList
  object Node {
    implicit lazy val rw: ReadWriter[Node] = macroRW
  }
  implicit lazy val rw: ReadWriter[LinkedList] = macroRW
}
```

**Sub-families carry `@key` on every case.** When a case of a family is itself a sealed family, the schema's tag for it is never written: upickle writes the tag of the inner case. upickle takes the tag prefix across every sealed ancestor on Scala 3, and across the whole hierarchy on Scala 2, so with the cases placed in their own families' companions the tags grow, `"Event.Ping"` on Scala 2 where the producer writes `"Ping"`. Declare both families as `sealed trait`s on every version, the inner extending the outer, put `@key("<tag>")` on every case of both, and write no case for the outer `const`. Both versions read `"Ping"` and `{"$type": "Created", "id": 1}` as an `Event`:

```scala
// Scala 3
sealed trait Event derives ReadWriter
object Event:
  @key("Ping") case object Ping extends Event

sealed trait UserEvent extends Event derives ReadWriter
object UserEvent:
  @key("Created") case class Created(id: Int) extends UserEvent derives ReadWriter
```

```scala
// Scala 2.13
sealed trait Event
object Event {
  @key("Ping") case object Ping extends Event {
    implicit val rw: ReadWriter[Ping.type] = macroRW
  }
  implicit val rw: ReadWriter[Event] = macroRW
}

sealed trait UserEvent extends Event
object UserEvent {
  @key("Created") case class Created(id: Int) extends UserEvent
  object Created {
    implicit val rw: ReadWriter[Created] = macroRW
  }
  implicit val rw: ReadWriter[UserEvent] = macroRW
}
```

## 6. Translating Scala 3 producer idioms to Scala 2.13

When the producer's source is Scala 3 and the consumer module is on Scala 2.13 or 2.12, or cross-built with Scala 2, mirror each idiom in its Scala 2 form. The names, and so the tags and property names, stay the same.

| Scala 3 producer | Scala 2.13 consumer |
|---|---|
| `enum Kind derives ReadWriter` with `case Standard` and `case Custom(label: String)` | `sealed trait Kind` and `object Kind` holding `case object Standard extends Kind` and `case class Custom(label: String) extends Kind` |
| `case class X(...) derives ReadWriter` | `object X { implicit val rw: ReadWriter[X] = macroRW }` |
| a parameterless enum case, `case Standard` | `case object Standard extends Kind { implicit val rw: ReadWriter[Standard.type] = macroRW }` |
| `case class Box[T](value: T) derives ReadWriter` | `object Box { implicit def rw[T: ReadWriter]: ReadWriter[Box[T]] = macroRW }` |
| `given ReadWriter[X] = ...` | `implicit val rw: ReadWriter[X] = ...` |
| `given [T: ReadWriter]: ReadWriter[Box[T]] = ...` | `implicit def rw[T: ReadWriter]: ReadWriter[Box[T]] = ...` |
| a recursive type with `derives ReadWriter` | `implicit lazy val rw` in every companion on the cycle |
| a named tuple, `position: (x: Int, y: Int, z: Int)` | `case class Position(x: Int, y: Int, z: Int)` with its companion, and `position: Position` |
| `derives MyApi.ReadWriter` | `implicit val rw: MyApi.ReadWriter[X] = MyApi.macroRW` |
| `@key("reddit.com") case RedditCom` in an enum | `@key("reddit.com") case object RedditCom extends Site { ... }` |
| `import upickle.default.*` | `import upickle.default.{macroRW, ReadWriter}` |
| significant indentation | braces |

A named tuple becomes a case class on Scala 3 too: the consumer then needs no `upickle-implicits-named-tuples` module, and the case class reads the same object.

## 7. Producer configurations: boxed options and renamed tags

A producer whose Api overrides a setting writes a different wire format, and the schema shows it: the boxed array with no null alternative for `optionsAsNulls = false`, and a tag property other than `$type` in every family for `tagName`. Mirror the setting in one configuration object, and derive every generated `ReadWriter` from it. Deriving any of them from `upickle.default` instead reads with the default setting, and the module's code and the test read with this object, `ProducerPickler.read[Root](json)`, not `upickle.default.read`.

Boxed options, from a producer with `override def optionsAsNulls = false`:

```scala
// Scala 3
object ProducerPickler extends upickle.AttributeTagged {
  override def optionsAsNulls = false
}

case class Boxed(o: Option[Int] = None) derives ProducerPickler.ReadWriter
```

```scala
// Scala 2.13
object ProducerPickler extends upickle.AttributeTagged {
  override def optionsAsNulls = false
}

case class Boxed(o: Option[Int] = None)
object Boxed {
  implicit val rw: ProducerPickler.ReadWriter[Boxed] = ProducerPickler.macroRW
}
```

Through this object `{"o": [1]}` reads as `Boxed(Some(1))`, and `{"o": []}` and `{}` as `Boxed(None)`; an `Option[Option[Int]]` field reads `[[2]]`, `[[]]`, and `[]`. A bare value fails there, and a bare `null` reads as a Scala `null`. A plain field whose default the producer's source does not show cannot become `Option[T] = None` under this object, because the producer writes its bare value: ask the user for the default.

A renamed tag, from a producer with `override def tagName = "kind"`:

```scala
object ProducerPickler extends upickle.AttributeTagged {
  override def tagName = "kind"
}
```

Every family derived from it reads and writes `{"kind": "Dog", ...}`, and a family's own `@key` still wins for that family. A producer with both settings gets one object with both overrides. When the module already has its own configuration object with other settings, ask the user before changing it: every type that derives from it would change too.

## 8. Hand-written schema forms

Schemas written by people, or by other generators, use forms upickle-jsonschema never emits. These map as follows; everything else is `ujson.Value`, named in the report.

| Form | Mapping |
|---|---|
| `{"type": ["string", "null"]}`, or any `[T, "null"]` type list | `Option[String]`, `Option[T]` |
| `{"anyOf": [S, {"type": "null"}]}`, or the same with `oneOf` | `Option[T]` |
| OpenAPI 3.0 `{"type": "string", "nullable": true}` | `Option[String]` |
| `{"enum": ["A", "B"]}` | a family of case objects `A` and `B` (section 5) |
| `definitions` and `#/definitions/<key>` | exactly as `$defs` |
| `{"type": "string", "format": "date-time"}`, and every other `format`: `date`, `time`, `email`, `uri`, `uuid` | `String`, with the property and its format in the report |
| an object with `properties` and no `required` | every property is not required: `Option[T] = None` for each field, by the `required` rule |
| a `$defs` or `definitions` key with no dots, such as `Person` or `person_record` | the key itself is the name, capitalised as a type name: `Person`, `PersonRecord` |
| a `definitions` entry that is a `prefixItems` array | the tuple of its items, with no definition |
| an inline two-item `prefixItems` array, or an `object` with `additionalProperties` and no `propertyNames` | a pair, or `Map[String, T]`, with no report entry: section 1's generator rules apply only to upickle-jsonschema output |

Mapped to `ujson.Value`: a `oneOf` or `anyOf` of several non-null schemas without the tag shape, `patternProperties`, `if` and `then`, `not`, an `allOf` that merges several objects, and an OpenAPI `discriminator` whose entries do not each require a tag property with a `const`.

## 9. Worked example

The producer's source is not available. The document, as upickle-jsonschema 4.4.3 generates it for the producer's `Envelope[Payload]`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$defs": {
    "example.Kind.Custom": {
      "type": "object",
      "properties": {"label": {"type": "string"}},
      "required": ["label"],
      "additionalProperties": true
    },
    "example.Kind": {
      "oneOf": [
        {"const": "Standard"},
        {
          "allOf": [
            {"$ref": "#/$defs/example.Kind.Custom"},
            {"type": "object", "properties": {"$type": {"const": "Custom"}}, "required": ["$type"]}
          ]
        }
      ]
    },
    "example.Payload": {
      "type": "object",
      "properties": {
        "id": {"type": "string"},
        "name": {"type": "string"},
        "tags": {"type": "array", "items": {"type": "string"}},
        "labels": {
          "type": "object",
          "additionalProperties": {"type": "string"},
          "propertyNames": {"type": "string", "pattern": "^-?(0|[1-9][0-9]*)$"}
        },
        "kind": {"$ref": "#/$defs/example.Kind"}
      },
      "required": ["id", "tags", "labels", "kind"],
      "additionalProperties": true
    },
    "example.Envelope[example.Payload]": {
      "type": "object",
      "properties": {
        "data": {
          "anyOf": [
            {"$ref": "#/$defs/example.Payload"},
            {"type": "null"},
            {
              "type": "array",
              "minItems": 0,
              "maxItems": 1,
              "items": {"$ref": "#/$defs/example.Payload"}
            }
          ]
        },
        "total_count": {"type": "integer"},
        "trace_id": {
          "anyOf": [{"type": "integer"}, {"type": "string", "pattern": "^-?(0|[1-9][0-9]*)$"}]
        }
      },
      "required": ["total_count", "trace_id"],
      "additionalProperties": true
    }
  },
  "$ref": "#/$defs/example.Envelope[example.Payload]"
}
```

Reading it by the rules:

- The root is `example.Envelope[example.Payload]`. It reaches `example.Payload`, then `example.Kind`, then `example.Kind.Custom`; all four entries are generated.
- `Envelope[T]` is generic: its argument `example.Payload` is the S of `data`'s optional shape. `data` is not in `required`, so it is `Option[T] = None`. `total_count` is a bare `integer`, so `Int`, reported. `trace_id` is the `Long` shape.
- In `Payload`, `id` is a required `String`, reported. `name` is not in `required` and not the optional shape, and the producer's default is unknown, so it is `Option[String] = None`, reported. `tags` is `Seq[String]`, reported. `labels` has the integral `propertyNames`, so `Map[Long, String]`, reported. `kind` refers to the family.
- `example.Kind` is a tagged `oneOf`: the bare `const` `Standard` is a singleton case, and `example.Kind.Custom` nests in `Kind` as its case class, tagged `Custom` by `$type`. `Custom.label` is a `String`, reported.
- The consumer module keeps its models in `com.acme.orders.model`, so the file is `Envelope.scala` there.

Scala 3:

```scala
package com.acme.orders.model

import upickle.default.ReadWriter

case class Envelope[T](
    data: Option[T] = None,
    total_count: Int,
    trace_id: Long
) derives ReadWriter

case class Payload(
    id: String,
    name: Option[String] = None,
    tags: Seq[String],
    labels: Map[Long, String],
    kind: Kind
) derives ReadWriter

enum Kind derives ReadWriter:
  case Standard
  case Custom(label: String)
```

Scala 2.13, which also compiles on Scala 3 for a cross-built module:

```scala
package com.acme.orders.model

import upickle.default.{macroRW, ReadWriter}

case class Envelope[T](
    data: Option[T] = None,
    total_count: Int,
    trace_id: Long
)
object Envelope {
  implicit def rw[T: ReadWriter]: ReadWriter[Envelope[T]] = macroRW
}

case class Payload(
    id: String,
    name: Option[String] = None,
    tags: Seq[String],
    labels: Map[Long, String],
    kind: Kind
)
object Payload {
  implicit val rw: ReadWriter[Payload] = macroRW
}

sealed trait Kind
object Kind {
  case object Standard extends Kind {
    implicit val rw: ReadWriter[Standard.type] = macroRW
  }
  case class Custom(label: String) extends Kind
  object Custom {
    implicit val rw: ReadWriter[Custom] = macroRW
  }
  implicit val rw: ReadWriter[Kind] = macroRW
}
```

The test, in munit here because that is what this module declares, compiles unchanged against either version. Its four documents cover `data` with a value, with `null`, and absent; `name` present and absent; both cases of `Kind`; and `trace_id` both as a number and as a string:

```scala
package com.acme.orders.model

import upickle.default.read

class EnvelopeSpec extends munit.FunSuite {
  private def envelope(json: String): Envelope[Payload] = read[Envelope[Payload]](json)

  test("data with a value, name present, singleton case, trace_id as a number") {
    val e = envelope(
      """{"data": {"id": "p-1", "name": "first", "tags": ["a"], "labels": {"1": "one"}, "kind": "Standard"},
        | "total_count": 1, "trace_id": 42}""".stripMargin
    )
    assertEquals(e.data, Some(Payload("p-1", Some("first"), Seq("a"), Map(1L -> "one"), Kind.Standard)))
    assertEquals(e.trace_id, 42L)
  }

  test("data null") {
    assertEquals(envelope("""{"data": null, "total_count": 0, "trace_id": 1}""").data, None)
  }

  test("data absent") {
    assertEquals(envelope("""{"total_count": 0, "trace_id": 1}""").data, None)
  }

  test("name absent, tagged case, trace_id as a string") {
    val e = envelope(
      """{"data": {"id": "p-2", "tags": [], "labels": {"-7": "neg"}, "kind": {"$type": "Custom", "label": "vip"}},
        | "total_count": 1, "trace_id": "9223372036854775807"}""".stripMargin
    )
    assertEquals(e.data.map(_.name), Some(None))
    assertEquals(e.data.map(_.kind), Some(Kind.Custom("vip")))
    assertEquals(e.data.map(_.labels), Some(Map(-7L -> "neg")))
    assertEquals(e.trace_id, Long.MaxValue)
  }
}
```

The report for this run:

```text
Schema conversion done
Root type:     Envelope[Payload] in com.acme.orders.model, orders/src/main/scala/com/acme/orders/model/Envelope.scala
Source:        derived from the schema
Files written: orders/src/main/scala/com/acme/orders/model/Envelope.scala, orders/src/test/scala/com/acme/orders/model/EnvelopeSpec.scala
Test:          com.acme.orders.model.EnvelopeSpec, 4 documents
Build:         orders/compile and orders/testOnly com.acme.orders.model.EnvelopeSpec, run the way the repository requires: passed, 4 tests
Defaults and fallbacks:
  Envelope.total_count: Int; schema allows Int, Short, Byte
  Payload.id: String; schema allows String, Char, Symbol, java.util.UUID, BigDecimal
  Payload.name: Option[String] = None; schema allows a String with a default it does not carry
  Payload.tags: Seq[String]; schema allows Seq, List, Vector, Array
  Payload.labels: Map[Long, String]; schema allows keys of Int, Long, Short, Byte, BigInt
  Kind.Custom.label: String; schema allows String, Char, Symbol, java.util.UUID, BigDecimal
Pickler configuration: none
Disagreements with source: none
Generator defects: none
```

## 10. Version checked

This reference describes upickle-jsonschema 4.4.3, the module's first release, marked experimental ("may change between minor releases") and published for Scala 3 only, and the upickle 4.4.3 reader and writer, published for Scala 2.12, 2.13, and 3. It was checked on 2026-10-21 against the source of upickle-jsonschema, upickle-implicits, upickle-core, and ujson at 4.4.3, and by running the generator on Scala 3 and the examples of sections 5, 7, and 9 on Scala 3 and 2.13, including the six producer types of section 1 whose schema does not describe the wire.

On an upgrade, re-check these facts first, because the skill's rules rest on them:

1. **The optional shape**: `Option[T]` as an `anyOf` of S, `{"type": "null"}`, and a zero-or-one-item array of S, emitted at every nesting level; the rule is keyed on "S plus null" so that it survives a change in the array branch.
2. **`required` derived from default values**: every field without a default is required, and no `default` keyword is written.
3. **The `oneOf` tag shape**: a bare `const` per singleton case and an `allOf` of a `$ref` and an object requiring the tag property per case class, with short tags by default.

Also re-check the 4.4.3 specifics: a pair inline but other tuple arities in `$defs`, under `scala.Tuple<N>[...]` or `scala.*:[...]` keys; the six producer types of section 1's last subsection (a two-field named tuple, a case class with fields `_1`, `_2`, ..., `Either`, a map keyed by a type upickle writes as pairs, a sealed sub-family, and a generic `Option[T]` instantiated with an `Option`); and the fully qualified tag being rejected by an `enum`'s reader.
