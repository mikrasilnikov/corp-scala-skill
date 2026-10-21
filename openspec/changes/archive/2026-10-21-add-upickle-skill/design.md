# Design

## Context

See proposal.md for motivation. The facts that shape the approach, verified against the upickle repository at tag 4.4.3 (the `upickle/jsonschema` module's source and snapshot fixtures, the `upickle/implicits` readers and writers, and the core tagged-reader code), and confirmed by running upickle and the generator at 4.4.3 through scala-cli, which corrected the bullets on tuples, named tuples, nested options, and the tagged reader, and added the bullet on producer types whose schema does not describe the wire (verification.md, section 6):

- `upickle.default.schema[T]` emits a Draft 2020-12 document: `$schema`, `$defs` keyed by the fully qualified Scala type name with type arguments in brackets (`upickletest.LinkedList.Node[scala.Int]`), and a root `$ref`. Named tuples of an arity other than 2 get keys of the form `scala.NamedTuple.NamedTuple[...]` and an object entry. A pair, named or not, is emitted inline; other tuple arities get keys `scala.Tuple<N>[...]`, or `scala.*:[...]` when written with `*:`.
- A case class becomes `type: object` with `properties` in declaration order, `required` holding every field that has no default value, and `additionalProperties` set from the `allowUnknownKeys` setting or the class's `@allowUnknownKeys` annotation. No `default` keyword is ever emitted, so a defaulted field's value is not in the schema. `@key` renames the property; `@flatten` hoists the inner object's properties into the outer one.
- `Option[T]` becomes `anyOf: [S, {type: null}, {type: array, minItems: 0, maxItems: 1, items: S}]` while `optionsAsNulls` is true, the default, and the plain array shape when it is false. The generator emits this at every nesting level, so `Option[Option[T]]` is an `anyOf` whose first alternative is itself that shape.
- Scalars: `Int`, `Short`, `Byte` are `integer`; `Long` is `anyOf: [integer, string with pattern ^-?(0|[1-9][0-9]*)$]`; `BigInt` is a string with that pattern; `Double` and `Float` are `number`; `String`, `Char`, `Symbol`, `java.util.UUID`, and `BigDecimal` are all `string`; `Unit` is `null`; `ujson.Value` is `{}`. `Seq`, `List`, `Vector`, and `Array` are `array`; `Set` adds `uniqueItems`; tuples, and case classes whose fields are `_1`, `_2`, ..., are `array` with `prefixItems`, a pair inline and the rest as `$defs` entries. `Map[K, V]` is `object` with `additionalProperties`, plus `propertyNames` for `Int`, `Long`, `Short`, `Byte`, `BigInt` (integral pattern), `Boolean` (pattern), `Char` (length 1), and `UUID` (format).
- A sealed trait or Scala 3 enum becomes `oneOf`: `{const: "Tag"}` for a singleton case, and `allOf: [$ref, {properties: {$type: {const: "Tag"}}, required: [$type]}]` for a case class. The tag key is `$type`, or the `@key` on the family, or the configured `tagName`; the tag value is the short name while `objectTypeKeyWriteFullyQualified` is false, the default.
- The upickle 4 writer, with default settings, writes `Some(x)` as `x` and `None` as `null`, and omits a field whose value equals its default while `serializeDefaults` is false, the default. Only a field declared `Option[Option[T]]` is written as a zero-or-one-element array, and it is read only from such arrays, `[]`, `[null]`, and `[x]`: a bare `null` at the outer level reads as a Scala `null`, not `None`. The reader for a first-level `Option[T]` maps `null` to `None` and hands any other value to T's reader, so a one-element array reaches T's reader and is rejected unless T is itself a collection. A missing key fails with `missing keys in dictionary` for every field type; there is no special rule for `Option` fields. The tagged reader accepts the tag in any position and reads a singleton case from a bare string; a sealed trait accepts the short and the fully qualified tag, a Scala 3 enum only the short one. `Long` reads a number or a string; `BigInt` and `BigDecimal` read strings; `Int` reads strings as well.
- Six producer types get a schema that does not describe what upickle writes. A two-field named tuple is emitted as a plain pair's array but written as an object. A case class whose fields are `_1`, `_2`, ... is emitted as an array but written and read as an object. `Either[A, B]` is emitted as a `$type`-tagged `oneOf` over `scala.util.Left` and `scala.util.Right` entries but written as `[0, a]` or `[1, b]`. A map whose key type upickle does not write as an object key, such as an enum, a case class, or a tuple, is emitted as an object but written as an array of `[key, value]` pairs, and the key type's schema is not emitted at all. A sealed trait with a sealed sub-trait among its cases gets a `oneOf` case tagged with the sub-trait's name, which upickle never writes: it writes the inner case's tag. A generic `Option[T]` field instantiated with an `Option` gets the nested optional schema, but is written unboxed, as `x` or `null`.
- upickle itself ships no readers for `java.time` types, so a `format: date-time` string has no direct counterpart.
- The module is marked experimental ("may change between minor releases"), 4.4.3 is its first release, and it is published for Scala 3 only. upickle 4.4.3 is published for Scala 2.12, 2.13, and 3.
- This repository's conventions, from the `skill-collection` spec and the existing `sbt-launcher` skill: one self-contained directory per skill, no links between skills, a README row per skill, bold-led rule paragraphs, tables for allowlists, and report templates in `text` blocks. Nothing here is executable, so changes are verified by text checks and review. The Agent Skills frontmatter limits `name` to 64 characters and `description` to 1024.
- The `sbt-launcher` skill, when installed beside this one, owns how a corporate repository is built and how dependencies are requested. This skill cannot link to it, so it restates what it needs in its own words and refers to the build as "the way the repository requires".

## Goals / Non-Goals

**Goals:**

- Make `Option[T]` the only answer to the optional shape, stated with enough of the wire-format reasoning that an agent is not tempted to handle the alternatives by hand.
- Give the agent a complete, deterministic mapping for every shape upickle-jsonschema 4.4.3 emits, with the defaults for the shapes the schema leaves ambiguous, so two runs over the same schema produce the same classes.
- Make the source question the first step, because one answer settles names, nesting, type parameters, defaults, narrow scalar types, and custom picklers together.
- Keep the skill self-contained and consistent in voice with `sbt-launcher`.

**Non-Goals:**

- Generating a schema from case classes, or validating documents against a schema. The skill consumes schemas.
- Support for other JSON libraries, or for upickle's binary MessagePack format.
- A code generator or script. The skill is rules for an agent; a `scripts/` directory would need a runtime the corporate repositories do not promise.
- Changing the `sbt-launcher` skill or its specs, or nesting `openspec/specs/` per skill.

## Decisions

### D1. The skill is named `upickle` and its description carries the trigger

The directory is `skills/upickle/` with `name: upickle`, a name broader than this change's scope so that later upickle rules can join it without a rename. The `description` names the moments to read it (a JSON Schema document, a schema from upickle-jsonschema, a request for case classes, models, or readers for a payload, an `anyOf` with `null`) and ends with what it is not for: generating schemas, and projects on another JSON library. It stays under 1024 characters; the name is 7.

Alternatives considered: `json-schema-to-upickle` or `upickle-schema-models`. Rejected by the user in favour of the short name.

### D2. Two files: rules in `SKILL.md`, the full mapping in `references/schema-mapping.md`

`SKILL.md` holds the rules an agent needs on every run, in five sections: "Before generating" (the source question, the upickle precondition, the Scala version, where the files go), "Mapping shapes to types" (the Option rule, the `required` axis, a short table of the scalar, collection, map, and object shapes), "Sealed families", "Never write by hand", and "Verifying and reporting". It stays under 200 lines. The reference holds the complete table with a schema snippet and the Scala for both 2.13 and 3 per row, the ambiguity defaults with their reasons, the boxed-options configuration, the hand-written forms, the `$defs` naming rules with examples, a translation table from Scala 3 producer idioms to Scala 2.13 (enum to sealed trait, named tuple to case class, `derives` to `macroRW`), and one worked example: a complete upickle-jsonschema document covering an optional field, a required optional field, a `Long`, a keyed map, a generic result wrapper, and a recursive sealed family, followed by the generated code in both syntaxes. `SKILL.md` links to the reference as `references/schema-mapping.md`; the reference links back as `../SKILL.md`.

Alternatives considered: everything in `SKILL.md`. Rejected: the worked example alone is longer than the rules, and an agent that has the rules rarely needs the whole table.

### D3. The optional shape is recognised as "S plus null", and the array alternative is part of it

The rule the skill states: a property whose alternatives are one schema S and `{type: null}`, with or without a zero-or-one-element array of S, is `Option[T]`. The array alternative is named explicitly so that an agent does not read it as a collection or as a third case to handle. The reference explains why nothing else is needed: the array is upickle's boxed form, used only for nested options and for a producer configured with `optionsAsNulls = false`; a default producer never writes it at the first level, and a default consumer could not read it there anyway, so code for it is dead. A nested shape is `Option[Option[T]]`, and upickle boxes and unboxes that itself. The plain array shape with no null alternative means a boxed-options producer, and the consumer answers with a pickler configuration object (`override def optionsAsNulls = false`) that every generated `ReadWriter` derives from, named in the report.

Alternatives considered: a custom reader that accepts all three alternatives. Rejected as dead code that diverges from what upickle writes, and as exactly the habit the skill exists to stop. Keying the rule on exactly three alternatives. Rejected: the module is experimental and may drop or reorder the array branch; "S plus null" survives that and also covers hand-written nullable forms.

### D4. `required` decides defaults, and the agent never invents one

The generator derives `required` from whether a field has a default value, and the producer omits a defaulted field whose value equals the default. So: required fields have no default; a non-required field takes the producer's default when the source is at hand, `Some(30)` for an Option included; otherwise a non-required Option field defaults to `None` and any other non-required field becomes `Option[T] = None`, with the property named in the report. Under an `optionsAsNulls = false` pickler that last answer fails, because an `Option` there reads only arrays while the producer writes the bare value, so the agent asks the user for the default instead. Required Option fields are not given `= None` because the direct mapping keeps the consumer's own writes valid against the schema, and because a missing required field should fail loudly rather than read as `None`.

Alternatives considered: `= None` on every Option field. Rejected: it changes the consumer's write shape and hides a producer that stopped sending a required field. Guessing a scalar default such as `0` or `""`. Rejected: silent data corruption with no error.

### D5. The source question comes first, and the schema wins on disagreement

The agent asks once, before generating, whether the producer's Scala source is available, and uses only what the user points it at. With source, it mirrors names, nesting, type parameters, defaults, `@key` and `@allowUnknownKeys`, narrow scalar types, and any custom pickler for a reached type, translating syntax to the consumer's Scala version. The schema still decides which definitions are needed and what each shape is, except where the producer declares one of D13's types: it is the published contract, and the source the user has may be a different version. Any disagreement is reported, not resolved silently, and a D13 type is listed among the generator defects instead.

Alternatives considered: always deriving from the schema. Rejected: the schema loses packages, default values, the narrow scalar types, and custom picklers, and the user asked for the question. Searching the machine for the producer's repository. Rejected: the agent should not roam beyond what the user names.

### D6. Without source, names come from the `$defs` keys by fixed rules

The keys are `TypeRepr.show` output, so the rules are mechanical: the last dotted segment before any `[` is the name; capitalised segments before it are enclosing objects, and the definition nests in their companions; lowercase-first segments are the producer's package, replaced by the consumer's. A type argument of a key outside the `scala` package becomes a type parameter when the argument's schema is a property's schema in that entry, or the S of a property's optional, array, or map shape, and several instantiations share one generic definition; otherwise the definition is concrete, and a second instantiation gets the argument's name appended. A named-tuple key becomes a case class named after the property that uses it, or, for a type argument of a generic definition, the property typed with that instantiation, the first in the order of the referring entries and their properties when several do, or by its field names joined when only the root does. A tuple key, `scala.Tuple<N>[...]` or `scala.*:[...]`, generates no definition: the property takes the tuple type. No key in the `scala` package is named by the dotted rules. A `definitions` section is read as `$defs`. Property names that are legal identifiers are field names verbatim, with no `@key`; others get a camel-case field name and `@key`. The verbatim rule gives an exact wire match with zero annotations; if corporate style wants camel case everywhere, every field becomes one `@key` away from it, and that is a review decision recorded as an assumption in the proposal.

Alternatives considered: concrete classes for every instantiation. Rejected: the usual case is a result wrapper used once per payload type, where a generic class is what the producer wrote. Camel-casing every field with `@key`. Rejected as default: it doubles the annotations for no behavioural gain.

### D7. Sealed families: `enum` on Scala 3, `sealed trait` on 2, cases in the companion, tags pinned by name or `@key`

upickle derives a case's tag from the suffix of its full name after the prefix common to all cases, so cases placed together in the family's companion get their simple names as tags, which is what the schema's `const` values are. A case whose tag is not a legal name carries `@key("<tag>")`, and so does any case the agent chooses to name differently. A tag property other than `$type` is `@key("<property>")` on the family. Singleton cases are case objects or parameterless enum cases, which upickle writes and reads as bare strings, matching the bare `const` entries. Nothing reads the tag by hand. A sub-family is the exception (D13): upickle takes the prefix across every sealed ancestor on Scala 3 and across the whole hierarchy on Scala 2, so both families are `sealed trait`s on every version and every case of both carries `@key`.

Alternatives considered: `@key` on every case regardless. Rejected: redundant where the name already matches; kept available for the cases that need it.

### D8. Ambiguities resolve to the widest safe reading type, and every resolution is reported

| Shape | Default | Why |
|---|---|---|
| `{type: string}` | `String` | `Char`, `Symbol`, `UUID`, and `BigDecimal` look the same; `String` reads any of them |
| `{type: integer}` | `Int` | `Long` has its own shape, so a bare `integer` is `Int`, `Short`, or `Byte` |
| `{type: number}` | `Double` | reads every `Float` |
| integral `propertyNames` | `Map[Long, V]` | the pattern covers `Long`; an `Int` key would truncate silently |
| `array` with `items` | `Seq[T]` | `List`, `Vector`, and `Array` read the same documents |
| `string` with an unknown `format` | `String` | lossless; upickle has no `java.time` readers |
| anything else | `ujson.Value` | lossless; the user sees it in the report |
| `object` with `additionalProperties` and no `propertyNames` | `Map[String, V]` | reads string keys in either wire form; a map keyed by an enum or a case class has the same schema (D13) |
| inline two-item `prefixItems` | `(A, B)` | the schema's own reading; a two-field named tuple has the same schema (D13) |

The last two rows are not the widest reading type but the schema's own reading, because D13's shapes cannot be told apart from the schema alone. With source, the narrow type is mirrored instead. The report names every property resolved by this table, with its path.

### D9. The skill lists what is never written by hand

For every mapped shape: no custom `Reader` or `ReadWriter`, no `readwriter[ujson.Value].bimap`, no reading into `ujson.Value` followed by dispatch, no `visitNull` or `visitArray` overrides, no manual tag reading. The only configuration code the skill permits is a pickler object mirroring a producer's non-default settings (`optionsAsNulls`, `tagName`), and the only custom pickler it permits is one reproduced from the producer's source for a type the schema reaches. Both are named in the report.

### D10. upickle is a precondition, and the build is "the way the repository requires"

The agent checks the module's declared dependencies for upickle before writing anything, by reading the build definition or a read-only build query, and stops with a structured request (coordinates, module, configuration, reason) when it is missing. It never adds upickle, a JSON Schema validator, or a test framework. This restates, in the skill's own words, the rule the `sbt-launcher` skill owns, because skills cannot link to each other. For the same reason the skill says "run the build and tests the way the repository requires" rather than naming the launcher; where `sbt-launcher` is installed, it supplies the how.

### D11. Verification: a unit test on the consumer side, text checks and a live probe on this change

The generated definitions are verified by a test in the module's declared framework that reads documents into the root type: for each optional field a value, `null`, and, where not required, the absent key, except that a nested optional field reads the boxed `[x]`, `[null]`, and `[]` in place of the value and `null`, because upickle rejects a bare value there and reads a bare `null` as a Scala `null` (a generic `Option[T]` instantiated with an `Option` is tested as declared, unboxed), and that a producer with `optionsAsNulls = false` gets the boxed `[x]` and `[]`, or `[[x]]`, `[[]]`, and `[]` for a nested field; for each sealed family one document per case; plus any payload the user supplies. The documents are derived from the schema's own alternatives, so the test exercises what the schema permits rather than what the agent assumed, except for a nested optional field and D13's shapes, whose documents are what upickle writes. The change itself is verified as earlier changes were: the frontmatter and length limits, link resolution within the skill directory, the README row, scenario-by-scenario coverage of both deltas by sentences of the skill, `openspec validate add-upickle-skill --strict`, and a read-through. A scratch script outside the repository, run with scala-cli, confirmed the claims the Context makes about the writer, the reader, and the generator at 4.4.3, and corrected four of them; verification.md, section 6, records the run.

### D12. Two capabilities, both prefixed `upickle-`, and no spec change elsewhere

`upickle-schema-mapping` owns shape-to-type behaviour; `upickle-schema-workflow` owns the steps around it. The prefix keeps the owning skill clear in the flat `openspec/specs/` layout, as the multi-skill change's design asked. `skill-collection` is exercised, not modified: its README-row scenario is satisfied by the implementation.

### D13. Where the generator misdescribes the wire, the generated type follows the wire

The Context lists six producer types whose 4.4.3 schema does not describe what upickle writes. Following the schema there gives a consumer that cannot read the producer's payloads, so the generated type follows what upickle writes. Four of the shapes are recognisable from the schema alone. Once tuple keys are set apart, a `$defs` entry with the `prefixItems` shape in upickle-jsonschema output can only come from a case class with positional fields, which becomes `case class X(_1: A, _2: B)`; an `Either` has its own `scala.util` keys, which become `Either[A, B]`; and a sub-family is an `allOf` whose `$ref` is itself a `oneOf`, which becomes a `sealed trait` extending its parent, with `@key` on every case of both families; and a nested optional shape inside a type argument stays `Option[T]` through the type-argument rule, as the generic definition upickle writes unboxed. The other two are not. An inline pair looks the same as a two-field named tuple, and a map's key type is not in the schema. Both keep the schema's reading, a pair and `Map[String, V]`, named in the report as ambiguities. A pair becomes a case class when the producer's source, a sample payload, or an enclosing `$defs` key that prints the named tuple shows one, and there a payload wins over the source, because it is what upickle wrote. A map takes the producer's key type from the source alone; without source, a payload whose pairs have non-string keys leads to a question for the key type. These rules apply only to upickle-jsonschema output, which the user names or the `$defs` keys show: in a hand-written document a `prefixItems` entry is a tuple and a pair or a string-keyed map is not reported. Every correction is listed under its own report label, `Generator defects:`, apart from the defaults and from the disagreements with the source.

Alternatives considered: following the schema everywhere. Rejected: the consumer cannot read the producer's payloads. `ujson.Value` for these shapes. Rejected: it discards the types the schema does carry. Asking the user each time. Rejected for the positional case class and `Either`, which are decidable; kept for a payload that shows a map with non-string keys and no source to name the key type.

## Risks / Trade-offs

- [A later upickle-jsonschema release changes the optional shape] → D3 keys the rule on "S plus null"; the reference names 4.4.3 as the version described, and a later change re-checks the fixtures.
- [The agent asks the source question when the conversation already answered it] → the rule is "once"; the skill says to take the answer from the conversation when it is already there.
- [A producer with `optionsAsNulls = false` or a custom `tagName`] → the plain array shape and the renamed tag property are both in the specs, with the pickler configuration and `@key` answers.
- [Scala 3 producer idioms in a 2.13 consumer: enums, named tuples, `derives`] → the reference's translation table; named tuples become case classes.
- [`@flatten` in the producer] → the schema shows hoisted properties, so a consumer without source gets a flat case class that reads the same documents; with source, the `@flatten` is mirrored.
- [Verbatim field names clash with corporate style] → recorded as an assumption; each field is one `@key` from camel case, and the wire format does not change.
- [Hand-built sample documents mis-state the wire format] → documents are derived from the schema's alternatives, except for nested optional fields and D13's shapes, whose documents are what upickle writes; user payloads are preferred, and the test asserts `Some` and `None`, not raw JSON.
- [A later release fixes or changes the misdescribed shapes] → D13 is keyed on the 4.4.3 shapes, and the reference's version section lists them for re-check.
- [The description exceeds 1024 characters or `SKILL.md` grows past its cap] → tasks check both with `wc`.

## Migration Plan

One commit adding `skills/upickle/SKILL.md` and `skills/upickle/references/schema-mapping.md` and the README row. Nothing installed changes: `sbt-launcher` copies are untouched, and the new skill is installed by copying `skills/upickle/`. Rollback is reverting the commit.

## Open Questions

- Whether a `java.time` mapping should be added if a later upickle release ships those readers. Deferrable: today the only lossless answer is `String`, which the specs state.
- Whether later changes add general upickle usage rules (configuration objects, custom picklers for existing types) under the same skill. Deferrable: D1 chose the name so that they can.
