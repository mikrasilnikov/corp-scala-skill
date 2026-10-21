# Proposal

## Why

Corporate services publish the JSON Schema of their payloads with upickle-jsonschema, and the consumers of those payloads, also on upickle, need matching case classes. An agent handed such a schema today reads the `anyOf: [T, null, array]` that the generator emits for every optional field as a union it must dispatch by hand, and writes custom readers for shapes that a derived upickle `ReadWriter` already covers: optional fields, longs, tagged sealed families, keyed maps. The result is more code than the producer ever wrote, and code that diverges from the wire format upickle actually produces.

## What Changes

- A second skill, `upickle`, is added as `skills/upickle/SKILL.md` with `skills/upickle/references/schema-mapping.md`, following the `skill-collection` convention, and a row for it is added to `README.md` in the same change.
- The skill applies when an agent is asked to turn a JSON Schema document into Scala case classes that upickle reads. It targets the output of upickle-jsonschema 4.4.3 first: Draft 2020-12 documents whose `$defs` are keyed by fully qualified Scala type names. Common hand-written forms (`type: ["string", "null"]`, a two-way `anyOf` with null, `enum` of strings, `definitions`) are covered in a secondary section; any other shape maps to `ujson.Value` and is named in the agent's report.
- The core rule: a property whose alternatives are a schema S plus `null`, with or without a zero-or-one-element array of S, is `Option[T]` where T is the mapping of S. The agent writes no custom deserialization for that shape, nor for any other shape the mapping covers (`Long` as integer-or-string, sealed families with a `$type` discriminator, singleton cases as bare strings, maps with typed keys). A plain `Option[T]` field reads everything a default-configured upickle producer writes.
- Where upickle-jsonschema 4.4.3 emits a schema that does not describe what upickle writes, found by running both during implementation, the generated type follows what upickle writes: a `$defs` entry with the `prefixItems` shape under a non-tuple key is a case class with fields `_1`, `_2`, ...; the `scala.util.Either` family is `Either[A, B]`; a pair that the producer's source, a sample payload, or an enclosing `$defs` key shows to be a two-field named tuple is a case class; a map takes the producer's key type when upickle writes that key type as an array of pairs; a sealed sub-family and its parent are `sealed trait`s with `@key` on every case; and a generic `Option[T]` instantiated with an `Option` stays generic, as upickle writes it unboxed. These rules apply to upickle-jsonschema output only, and the report lists each under its own label.
- Membership in `required` decides defaults, independently of Option-ness: a field not in `required` takes the producer's default when the producer's source is available; otherwise an Option field gets `= None` and any other field becomes `Option[T] = None` with a note, except under an `optionsAsNulls = false` producer, where the agent asks for the default. The agent never invents a default value.
- The skill's first step asks the user once whether the producer's Scala source is available. With it, names, nesting, generics, defaults, and `@key` annotations are mirrored and translated to the consumer's Scala version. Without it, they are derived from the `$defs` keys by stated rules.
- Preconditions and verification: upickle must already be declared in the target module; if it is not, the agent stops and asks for the dependency instead of adding it. The generated classes are verified by a unit test in the module's declared test framework that reads sample documents covering each Option state, run through the repository's own build. No JSON Schema validator dependency is introduced.
- Unchanged: the `sbt-launcher` skill and its five specs, the flat `openspec/specs/` layout, and the OpenSpec tooling under `.claude/`.

## Capabilities

### New Capabilities

- `upickle-schema-mapping`: how a JSON Schema document, as produced by upickle-jsonschema, maps to Scala types that upickle reads: the Option rule, the `required` axis, scalars, collections, tuples, maps, sealed families and their discriminators, names from `$defs` keys, the shapes the generator misdescribes, and which shapes never get hand-written readers.
- `upickle-schema-workflow`: how the agent carries out the conversion: asking once for the producer's source and what it mirrors from it, the upickle dependency precondition, the target Scala version and derivation syntax, verification by unit test, and the report of the defaults and ambiguities it resolved and of the generator defects it followed.

### Modified Capabilities

None. `skill-collection` already requires the directory layout and the README row for a new skill, and the four build-related capabilities specify behaviour the new skill does not change.

## Impact

- New files: `skills/upickle/SKILL.md` and `skills/upickle/references/schema-mapping.md`. One edited file: `README.md`, one table row. Nothing else in the repository changes; the `sbt-launcher` skill is untouched.
- Downstream: agent sessions that convert a schema into consumer models in corporate Scala projects on upickle. The skill constrains what the agent generates and how it verifies it; it does not change upickle, the producers, or any project build.
- External dependency: the mapping describes upickle-jsonschema 4.4.3, its first release, marked experimental and published for Scala 3 only, and the upickle 4.4.3 reader and writer, published for Scala 2.12, 2.13, and 3. The shapes were verified against the module's source and snapshot fixtures at that tag, and by running the generator and upickle at 4.4.3, which showed six producer types whose schema does not describe what upickle writes. A later release may change the Option shape, so the rule is keyed on "S plus null" rather than on exactly three alternatives.
- Relation to the `sbt-launcher` skill: both apply in the same repositories, but skills are self-contained, so the `upickle` skill restates the dependency rule in its own words and says "build and test the way the repository requires" instead of naming the launcher.
- Assumptions recorded for review: consumers may be on Scala 2.13 or 3, so both derivation syntaxes are shown; required Option fields mirror the schema with no default; without source, a `$defs` key with a type argument becomes a generic case class when the argument is a property's schema or the S of its optional, array, or map shape, and a concrete class otherwise; verification is by unit test and text checks, with a live probe through scala-cli outside the repository; where the generator's schema and upickle's wire format disagree, the consumer follows the wire (design D13).
