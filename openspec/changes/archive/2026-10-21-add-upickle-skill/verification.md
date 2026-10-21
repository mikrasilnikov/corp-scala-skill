# Verification

Verification for the `add-upickle-skill` change, per design D11 and tasks 7.1 to 7.5. Files checked: `skills/upickle/SKILL.md` (S) and `skills/upickle/references/schema-mapping.md` (R). Line numbers refer to the versions in this change.

## 1. Scenario trace

Every `#### Scenario` across the two spec deltas, as revised in task 8 (102 in total: 66 mapping, 36 workflow), mapped to the lines of `SKILL.md` that cover it. A script checked that each cited line contains the phrase that covers its scenario, and that the scenario list equals the specs' list: 102 of 102 traced, 0 missing, 0 stale citations. The trace was regenerated against the final text of tasks 8.1 to 8.3, so every line number refers to it.

### upickle-schema-mapping

| # | Requirement | Scenario | Covered by |
|---|---|---|---|
| 1 | Optional shapes map to Option | Optional field as upickle-jsonschema emits it | S45, S48, S51 |
| 2 | Optional shapes map to Option | Optional reference | S45, S57 |
| 3 | Optional shapes map to Option | Nested optional | S51 |
| 4 | Optional shapes map to Option | Nested optional through a type argument | S51 |
| 5 | Optional shapes map to Option | Hand-written nullable forms | S51 |
| 6 | Optional shapes map to Option | Boxed optional from a producer configured without nulls | S51 |
| 7 | Mapped shapes get no hand-written deserialization | Tempted to dispatch the optional shape by hand | S135, S139 |
| 8 | Mapped shapes get no hand-written deserialization | Long as integer or string | S70, S139 |
| 9 | Mapped shapes get no hand-written deserialization | Discriminated union | S115, S127, S137 |
| 10 | Membership in required decides defaults | Required optional field | S57 |
| 11 | Membership in required decides defaults | Optional field not required | S57 |
| 12 | Membership in required decides defaults | Optional field with a producer default | S57 |
| 13 | Membership in required decides defaults | Required scalar | S58 |
| 14 | Membership in required decides defaults | Scalar not required, producer known | S58 |
| 15 | Membership in required decides defaults | Scalar not required, producer unknown | S58, S60, S176 |
| 16 | Scalar shapes map to Scala scalars | BigInt | S71 |
| 17 | Scalar shapes map to Scala scalars | Plain string with the producer unknown | S69, S79 |
| 18 | Scalar shapes map to Scala scalars | Plain string with the producer known | S79, S16 |
| 19 | Scalar shapes map to Scala scalars | Empty schema | S73 |
| 20 | Collections, tuples, and maps | Sequence and set | S74 |
| 21 | Collections, tuples, and maps | Tuple | S75 |
| 22 | Collections, tuples, and maps | String-keyed map | S76 |
| 23 | Collections, tuples, and maps | Integral-keyed map with the producer unknown | S76, S79 |
| 24 | Collections, tuples, and maps | Integral-keyed map with the producer known | S79 |
| 25 | Objects with properties become case classes | Property names kept verbatim | S81 |
| 26 | Objects with properties become case classes | Property name that is not an identifier | S81 |
| 27 | Objects with properties become case classes | Unknown keys rejected by the producer | S81 |
| 28 | Objects with properties become case classes | Referenced object | S77 |
| 29 | Objects with properties become case classes | Empty object | S81 |
| 30 | Discriminated families become sealed families | Family tagged with $type | S115, S121, S125, S127 |
| 31 | Discriminated families become sealed families | Hand-written enum of strings | S125 |
| 32 | Discriminated families become sealed families | Recursive family | S125 |
| 33 | Family cases are named by their tags | Family with a renamed tag property | S127 |
| 34 | Family cases are named by their tags | Tag that is not a legal name | S127 |
| 35 | Family cases are named by their tags | Case name differs from the tag | S127 |
| 36 | Names derive from $defs keys when the producer is unknown | Nested type | S90 |
| 37 | Names derive from $defs keys when the producer is unknown | Top-level type | S89, S91 |
| 38 | Names derive from $defs keys when the producer is unknown | Older definitions keyword | S87 |
| 39 | Type arguments in $defs keys become type parameters | Generic type used by a property | S92 |
| 40 | Type arguments in $defs keys become type parameters | Two instantiations of one constructor | S92 |
| 41 | Type arguments in $defs keys become type parameters | Argument not used by any property | S93 |
| 42 | Unmapped shapes fall back to ujson.Value | Union without a discriminator | S83 |
| 43 | Unmapped shapes fall back to ujson.Value | Pattern properties | S83 |
| 44 | Unmapped shapes fall back to ujson.Value | Date-time string | S83 |
| 45 | Tuple entries in $defs map to tuples | Tuple entry in $defs | S94 |
| 46 | Tuple entries in $defs map to tuples | Tuple written with *: | S94 |
| 47 | Named-tuple entries become case classes named by their use | Named-tuple entry | S95 |
| 48 | Named-tuple entries become case classes named by their use | Named tuple shared by two properties | S95 |
| 49 | Named-tuple entries become case classes named by their use | Named tuple as a type argument | S95 |
| 50 | Inline pairs stay pairs unless a named tuple is shown | Pair with the producer unknown | S101 |
| 51 | Inline pairs stay pairs unless a named tuple is shown | Pair confirmed by the source | S101 |
| 52 | Inline pairs stay pairs unless a named tuple is shown | Pair shown by an enclosing key | S101 |
| 53 | Named pairs become case classes | Named pair in the producer's source | S101, S99 |
| 54 | Named pairs become case classes | Sample payload shows an object | S101 |
| 55 | Named pairs become case classes | Named pair shown by an enclosing key | S101 |
| 56 | Named pairs become case classes | Source and payload disagree on a pair | S101 |
| 57 | Positional case classes follow what upickle writes | Case class with positional field names | S103, S99 |
| 58 | Positional case classes follow what upickle writes | Hand-written prefixItems entry | S103, S41 |
| 59 | Either entries map to Either | Either entry | S105 |
| 60 | Maps keyed by other types take the producer's key type | Map keyed by an enum in the producer's source | S109 |
| 61 | Maps without key information stay string-keyed | Map with no key information | S109 |
| 62 | Maps without key information stay string-keyed | Payload shows pairs with object keys | S109 |
| 63 | Generator rules apply only to upickle-jsonschema output | Hand-written document | S99 |
| 64 | Generator rules apply only to upickle-jsonschema output | Producer in the default package | S99 |
| 65 | Sub-families are read by their inner tags | Sealed family with a sealed sub-trait | S107, S36 |
| 66 | Unknown defaults under boxed options ask the user | Unknown default under a boxed pickler | S60 |

### upickle-schema-workflow

| # | Requirement | Scenario | Covered by |
|---|---|---|---|
| 1 | The agent asks once whether the producer's source is available | Source available | S14, S16 |
| 2 | The agent asks once whether the producer's source is available | Source not available | S18 |
| 3 | The agent asks once whether the producer's source is available | Source available for some types only | S16, S165 |
| 4 | Mirroring the producer's definitions | Scala 3 enum into a Scala 2.13 consumer | S16 |
| 5 | Mirroring the producer's definitions | Defaults carried over | S16, S58 |
| 6 | Mirroring the producer's definitions | Source and schema disagree | S16, S172 |
| 7 | Mirroring the producer's definitions | Producer uses a custom pickler | S16, S141, S171 |
| 8 | The producer's definition decides the shapes the generator misdescribes | Source shows a shape the generator misdescribes | S16 |
| 9 | The producer's definition decides the shapes the generator misdescribes | Another type at a misdescribed shape | S16 |
| 10 | The producer's definition decides the shapes the generator misdescribes | Value types differ at a misdescribed type | S16 |
| 11 | The producer's definition decides the shapes the generator misdescribes | Payload contradicts a named pair in the source | S16 |
| 12 | upickle must already be declared | upickle declared | S20 |
| 13 | upickle must already be declared | upickle absent | S20, S24, S25, S26, S27 |
| 14 | upickle must already be declared | A validator would make checking easier | S30 |
| 15 | Derivation syntax follows the module's Scala version | Scala 3 module | S36 |
| 16 | Derivation syntax follows the module's Scala version | Scala 2.13 module | S37 |
| 17 | Derivation syntax follows the module's Scala version | Cross-built module | S37, S39 |
| 18 | Derivation syntax follows the module's Scala version | Recursive type on Scala 2 | S39 |
| 19 | Derivation syntax follows the module's Scala version | Module with its own pickler configuration | S39 |
| 20 | Generated definitions follow the module's layout | Existing models package | S41 |
| 21 | Generated definitions follow the module's layout | No obvious place | S41 |
| 22 | Generated definitions follow the module's layout | Entry with no definition of its own | S41, S105 |
| 23 | Generated definitions follow the module's layout | Unreachable definition | S41 |
| 24 | Generated definitions are verified by a unit test through the repository's build | Optional field states | S148, S147 |
| 25 | Generated definitions are verified by a unit test through the repository's build | Sealed family cases | S152 |
| 26 | Generated definitions are verified by a unit test through the repository's build | Sample payload from the user | S154 |
| 27 | Generated definitions are verified by a unit test through the repository's build | No test framework declared | S156 |
| 28 | Generated definitions are verified by a unit test through the repository's build | Build not run | S158 |
| 29 | Test documents follow what upickle writes | Nested optional field states | S149 |
| 30 | Test documents follow what upickle writes | Generator defect documents | S153 |
| 31 | Boxed optional fields are tested with arrays | Boxed optional field states | S150 |
| 32 | The report names every default and ambiguity | Ambiguities present | S169, S170, S176 |
| 33 | The report names every default and ambiguity | No ambiguities | S176 |
| 34 | The report names every default and ambiguity | Definitions mirrored from source | S165 |
| 35 | The report lists the generator defects it followed | Generator defects followed | S173 |
| 36 | The report lists the generator defects it followed | No generator defects | S173, S176 |

## 2. Size and structure

| Check | Result |
|---|---|
| Frontmatter | Pass. Lines 1 and 4 are `---`, line 2 is `name: upickle`, line 3 is the only `description:` line. Ruby `YAML.load` reads `name: upickle` and a 981-character description. Line 3 is 995 bytes (limit 1040). |
| `SKILL.md` length | Pass: 176 lines (limit 200). The reference is 684 lines. |
| `SKILL.md` headings | Pass: the five `##` headings of design D2, in order, plus `### Names from $defs keys` and `### Shapes the generator misdescribes` inside "Mapping shapes to types". |
| Reference headings | Pass: `grep -c "^## "` prints 11 (Contents and sections 1 to 10). |
| Links | Pass: 5 of 5 resolve inside `skills/upickle/`. `SKILL.md` links to `references/schema-mapping.md` once plainly and three times with the anchors of sections 6 and 7. The reference links to `../SKILL.md` and its ten section anchors, all of which match the headings. |
| Self-containment | Pass: `grep -rn "sbt-launcher" skills/upickle` and `grep -rn "\.\./\.\." skills/upickle` print nothing; `ls skills/upickle` prints `SKILL.md` and `references`. |
| README | Pass: `grep -c "^| .$d. |" README.md` prints 1 for `sbt-launcher` and for `upickle`; `git diff --numstat README.md` prints `1 0`. |
| OpenSpec | Pass: `openspec validate add-upickle-skill --strict` reports the change valid, with every requirement text at most 500 characters. |
| Scope | Pass: `git status --porcelain` shows only `README.md`, `skills/upickle/`, and `openspec/changes/add-upickle-skill/`. |
| Section checks of tasks 2.1, 3.1 to 3.4, 4.1, 5.1 to 5.4 | Pass: every required string is in its section; "Verifying and reporting" has exactly one `text` block; the section 9 `json` block passes `python3 -I -m json.tool` and equals the generator's output; section 6's table has 12 rows. |
| Task 8 checks | Pass: `grep -c stringKeyRW skills/upickle/SKILL.md` prints 1; `grep -c "schema alone"` prints 0; no sentence says every reachable entry becomes a definition; the defaults sentences name `(A, B)` and `Map[String, T]`; the Verifying section's one `text` block contains `Generator defects:` and `followed <the schema \| the payload>`; line 3 is unchanged; in the reference, `grep -c "^## "` prints 11, `grep -c "disagreement between source and schema"` and `grep -c "or the root, that uses it"` print 0, section 9's Scala blocks and test are byte-identical to the compiled files, and every `json` block parses. |

## 3. Read-through

Followed the skill over the worked example's document as an agent would: the source question (answered "not available"), the dependency and Scala version checks, reachability from the root `$ref`, the mapping of each property, the family, the test, and the report. The five sections agree with each other and with the reference's tables in sections 2, 3, and 9. Three sentences were reworded, keeping their meaning:

- "Names from $defs keys": the type-argument rule said "used as a property's schema", while the worked example (and the spec scenario for `ApiResult`) relies on the argument being the S of a property's optional shape. It now says "a property's schema in that entry, or the S of a property's optional, array, or map shape", as the reference's rule 4 already did.
- "Sealed families": the companion sentence described only the sealed-trait tag derivation. It now says an enum case is tagged by its own name and a sealed trait's case by its full name minus the shared prefix, matching reference section 5.
- "Before generating": the enum translation now says it applies "for a Scala 2 consumer".

After the rewording, the checks of 2.1, 3.1 to 3.4, 4.1, and 7.1 were run again and still pass, and the scenario trace above was regenerated against the final text.

## 4. Trigger dry-runs

Method: five independent subagents (Claude Sonnet), no tools. Each was shown a workspace listing, a user request, and five skills by name and description: `upickle` (the real frontmatter description), `sbt-launcher` (its real description, because both skills apply in the same repositories), and decoys `circe-json`, `openapi-clients`, and `git-hygiene`. Each was asked which skills it would load before acting.

| # | Workspace | Request | Expected | Loaded | Verdict |
|---|---|---|---|---|---|
| T1 | sbt repository; `core` declares upickle 4.4.3, `orders` dependsOn `core` | A pasted upickle-jsonschema document (`$defs` key `com.acme.billing.Invoice`, an optional `paid_at`) with "create the models for it in the orders module" | `upickle` | `upickle`, `sbt-launcher` | Pass |
| T2 | sbt repository; `core` declares upickle 4.4.3; `schemas/payment-status.schema.json` | "I need upickle readers for that payload in core; the schema is schemas/payment-status.schema.json." | `upickle` | `upickle`, `sbt-launcher` | Pass |
| T3 | sbt repository on upickle; `schemas/order.json` | "Every optional property is an anyOf of the type, null, and some one-element array thing. Why is it like that, and how should I model those fields?" | `upickle` | `upickle` | Pass |
| N1 | sbt repository; `api` declares circe-core and circe-generic only | "Generate case classes for the customer payload described in schemas/customer.schema.json ... so we can decode it." | not `upickle` | `circe-json`, `sbt-launcher` | Pass |
| N2 | sbt repository on upickle; `Shipment` derives `ReadWriter` | "Our partners want a JSON Schema for the Shipment case class ... produce one we can publish." | not `upickle` | `sbt-launcher` | Pass |

Result: 5 of 5 as expected; the description needed no adjustment. N1 named the circe exclusion ("its description excludes projects on another JSON library such as circe"), and N2 named the direction ("its description says it is not for generating a JSON Schema from existing case classes"). T3 deferred `sbt-launcher` because a question needs no build.

## 5. Behaviour dry-runs

Method: seven independent subagents (Claude Sonnet), each told to read `SKILL.md` (and the reference only if `SKILL.md` sent it there), treat it as binding, run nothing, and say exactly what it would do in a hypothetical repository. Each situation stated that the producer's source is not available and offered a tempting wrong move. Responses are condensed, with the decisive parts quoted.

| # | Situation (and the temptation offered) | Response | Spec expectation | Verdict |
|---|---|---|---|---|
| P1 | Scala 3 module; `reading` is the three-way optional shape over `number`, in `required`. Temptation: a colleague wants a custom Reader accepting a number, null, and the one-element array. | `case class Sensor(sensor_id: String, reading: Option[Double]) derives ReadWriter`, no default, no reader. Tells the user the array is "upickle's boxed form of an option ... a default producer never sends `[x]` for this field ... so code for it would be dead", and that a real `[21.5]` on the wire would mean a pickler configuration object, "not a custom Reader". Test reads a value and `null`; no absent-key document because the field is required. | mapping "Optional field as upickle-jsonschema emits it", "Tempted to dispatch the optional shape by hand", "Required optional field"; workflow "Optional field states" | Pass |
| P2 | Scala 2.13 module; `coupon` is the optional shape in `required`, `discount` the optional shape not in `required`. Temptation: the user asks for `= None` on every Option field. | `case class Order(order_id: String, coupon: Option[String], discount: Option[String] = None)` with `object Order { implicit val rw: ReadWriter[Order] = macroRW }`. Declines the default on `coupon`: "A default would make a payload missing the key read silently as `None` instead of failing." Test reads both fields with a value and `null`, and `discount` absent; no absent-`coupon` document. | mapping "Required optional field", "Optional field not required"; workflow "Scala 2.13 module", "Optional field states" | Pass |
| P3 | Scala 3 module; `nickname` is a plain `string` not in `required`, `age` a required `integer`. Temptation: the user suggests `""` as nickname's default. | `case class Profile(age: Int, nickname: Option[String] = None) derives ReadWriter`. Declines `""`: "A guess ('probably') is not the producer's default." Test reads nickname present and absent. Report lists `Profile.age: Int` and `Profile.nickname: Option[String] = None` under `Defaults and fallbacks:`. | mapping "Scalar not required, producer unknown", "Required scalar"; workflow "Ambiguities present" | Pass |
| P4 | Scala 3 module; `geo.Shape` is a `oneOf` of `{"const": "Empty"}` and two `allOf` cases tagged by `kind`. Temptation: a Reader that dispatches on `kind`. | `@key("kind") enum Shape derives ReadWriter` with `case Empty`, `case Circle(radius: Double)`, `case Square(side: Double)`; no per-case annotation and no reader ("upickle's derived ReadWriter already dispatches on `kind`"). Test reads `"Empty"` and one tagged object per case class. | mapping "Family with a renamed tag property", "Discriminated union", "Nested type"; workflow "Sealed family cases" | Pass |
| P5 | Scala 2.13 module, no configuration object; `middle_name` is a plain zero-or-one-item array with no null alternative, in `required`. Temptation: the user suggests `Seq[String]`. | `middle_name: Option[String]`, and a new `object ProducerPickler extends upickle.AttributeTagged { override def optionsAsNulls = false }` from which `Person`'s `implicit val rw: ProducerPickler.ReadWriter[Person] = ProducerPickler.macroRW` derives; test reads `["Grace"]` and `[]` through `ProducerPickler.read`. Report: `Pickler configuration: optionsAsNulls = false, object ProducerPickler ...`. | mapping "Boxed optional from a producer configured without nulls"; workflow "Scala 2.13 module" | Pass |
| P6 | Scala 3 module; root `api.Page[api.User]` with `items` an array of `$ref` to `api.User` and `next_cursor` optional, not required; an unreachable `api.LegacyUser`. | `case class Page[T](items: Seq[T], next_cursor: Option[String] = None) derives ReadWriter` and `case class User(id: Long, email: String) derives ReadWriter` in `Page.scala`; root read as `read[Page[User]](json)`; nothing generated for `LegacyUser`, which is mentioned to the user. Reasoned that `api.User` "is the S of a property's array shape, so the definition is generic". | mapping "Generic type used by a property", "Long as integer or string", "Optional field not required"; workflow "Unreachable definition" | Pass |
| P7 | `reporting` declares only munit and has no `dependsOn`; `core` declares upickle. Temptation: "just add it to reporting's libraryDependencies, it's a one-liner." | Writes nothing and edits no build file. Sends a `Dependency request` with coordinates `com.lihaoyi %% upickle % 4.4.3`, module `reporting`, configuration `Compile`, and a reason; "the task pauses there". Does not ask about the producer's source again. | workflow "upickle absent", "Source not available" | Pass |

Small deviations that are not rule violations: P3 and P1 named alternatives in the report slightly differently from the skill's list (P3 added `Long` to `Int`'s alternatives, P1 listed `Sensor.reading`'s `Double`); P6's `Root type:` line gave `Page` rather than `Page[User]`. None changes the generated code, and the skill's report template and reference example already show the intended form, so the skill text was not changed for them.

### Second round (task 8.5)

Method: a workflow ran seventeen independent dry-run agents, one per new or revised rule, each told to read `SKILL.md` (and the reference only if `SKILL.md` sent it there), treat it as binding, run nothing, and give every file and the report for a hypothetical module. Each response then went to an independent judge agent with the spec requirement it exercises and an expectation statement, graded strictly on those points. All situations say the producer's source is not available unless they give it.

| # | Situation | Response, condensed | Spec | Judge |
|---|---|---|---|---|
| D1 | inline pair, no source, no payload | `point: (Int, Int)`; `Defaults and fallbacks:` names it as a pair that could be a two-field named tuple; `Generator defects: none` | mapping "Inline pairs stay pairs unless a named tuple is shown" | Pass |
| D2 | named pair in the producer source | `case class Point(x: Int, y: Int)`, `point: Point`; listed under `Generator defects:`, `Disagreements with source: none`; test reads `{"x": 1, "y": 2}` | mapping "Named pairs become case classes"; workflow "The producer's definition decides the shapes the generator misdescribes" | Pass |
| D3 | payload shows an object for a pair | `Point(x: Int, y: Int)` from the payload's keys and the items' types; listed under `Generator defects:`; test reads the payload | mapping "Named pairs become case classes", "Sample payload shows an object" | Pass |
| D4 | named pair through an enclosing `Box` key | `Box[T](value: T)`, `Point(x: Int, y: Int)`, `point: Box[Point]`; a generator defect, not a possible named tuple | mapping "Named pair shown by an enclosing key" | Pass |
| D5 | positional entry, no source, Scala 2.13 | `case class Pair(_1: Int, _2: String)` with its `macroRW` companion; under `Generator defects:`; test reads `{"_1": 1, "_2": "a"}` | mapping "Positional case classes follow what upickle writes" | Pass |
| D6 | hand-written `prefixItems` entry | `at: (Double, Double)`, no `Coordinates` definition, `Generator defects: none`, no possible-named-tuple line | mapping "Hand-written prefixItems entry", "Hand-written document" | Pass |
| D7 | `Either` entry | `e: Either[Int, String]`, no definition for the three entries, a generator defect, test `[0, 1]` and `[1, "s"]` | mapping "Either entries map to Either"; workflow "Entry with no definition of its own" | Pass |
| D8 | map keyed by an enum in the source | `enum Color` mirrored, `counts: Map[Color, Int]`, a generator defect, test `[["Red", 1]]` | mapping "Maps keyed by other types take the producer's key type" | Pass |
| D9 | string-keyed map, no source, no payload | `counts: Map[String, Int]`, named under `Defaults and fallbacks:` as possibly keyed by another type; `Generator defects: none` | mapping "Map with no key information" | Pass |
| D10 | payload with object keys for that map | stops before writing and asks "What is the key type of `Stats.counts`?" | mapping "Payload shows pairs with object keys" | Pass |
| D11 | sealed sub-family, Scala 2.13 | two `sealed trait`s, `@key` on all four cases, no `UserEvent` case, a generator defect, no `UserEvent` test document | mapping "Sub-families are read by their inner tags" | Pass |
| D12 | nested optional field | `flag: Option[Option[Boolean]] = None`; tests `[true]`, `[null]`, `[]`, absent key; "deliberately no `\"flag\": null` document" | workflow "Nested optional field states" | Pass |
| D13 | `optionsAsNulls = false` fields | `ProducerPickler` with `optionsAsNulls = false`, everything derived and read through it; `[2]`, `[]` and `[[3]]`, `[[]]`, `[]`; no bare value or `null` | workflow "Boxed optional fields are tested with arrays" | Pass |
| D14 | unknown default under a boxed pickler | asks for the producer's default of `name` instead of `Option[String] = None` | mapping "Unknown defaults under boxed options ask the user" | Pass |
| D15 | generic `Option[T]` with an `Option` | `R[T](data: Option[T])`, `r: R[Option[Int]]`, a generator defect, tests `{"data": 3}` and `{"data": null}` | mapping "Nested optional through a type argument" | Pass |
| D16 | `Option` field with a `Some(30)` default in the source | `timeout: Option[Int] = Some(30)` mirrored; its absent-key test expects `Some(30)`, which the test rule now states (see below) | mapping "Optional field with a producer default" | Pass |
| D17 | `scala.Tuple3` entry | `t: (Int, String, Boolean)`, no `Tuple3` definition, `Generator defects: none` | mapping "Tuple entry in $defs" | Pass |

Result: 17 of 17 pass. Deviations the judges noted are harmless extras (additional test documents, extra `Defaults and fallbacks:` lines, D4 labelling the defect `Box.value` rather than `point`, D11 nesting `UserEvent` in `Event`'s companion) or one gap in the test rule, below, since fixed.

**Gap found by D16, fixed.** For an `Option` field whose producer default the source gives, such as `timeout: Option[Int] = Some(30)`, the document without the key reads `Some(30)`, not `None`, which matches the probe's `Timeout` row in section 6. The workflow scenario "Optional field states" and `SKILL.md`'s test bullet expected `None`. Both now say the absent key reads the field's default, which is `None` unless the producer's source gives another.

## 6. Live probe

`scala-cli` 1.18.0 is installed, so the claims of design's Context were run, in a scratch directory outside the repository, against `com.lihaoyi::upickle:4.4.3`, `com.lihaoyi::upickle-jsonschema:4.4.3`, and `com.lihaoyi::upickle-implicits-named-tuples:4.4.3` on Scala 3.9.0 (generator and producer side), and against upickle 4.4.3 on Scala 3.3.7 and 2.13.16 (consumer side). Maven Central lists 4.4.3 as the only upickle-jsonschema version and has no `upickle-jsonschema_2.13` artifact. The upickle 4.4.3 documentation calls the module experimental: "This API is experimental and may change between minor releases."

### Generator output

`upickle.default.schema[Envelope[Payload]]` for the producer classes `case class Envelope[T](data: Option[T] = None, total_count: Int, trace_id: Long)`, `case class Payload(id: String, name: String = "unnamed", tags: Seq[String], labels: Map[Long, String], kind: Kind)`, and `enum Kind { case Standard; case Custom(label: String) }` printed the document in reference section 9 exactly (compared as parsed JSON, key order included). A catalogue class covering every shape of section 1 printed the snippets of section 1 and the rows of section 2, including `required` lists that hold every field without a default and no other, and the optional shape at both nesting levels.

A second round, run for the spec revisions of task 8, printed these keys and entries on Scala 3.9.0:

- `Int *: String *: Boolean *: EmptyTuple` is keyed `scala.*:[scala.Int, scala.*:[scala.Predef.String, scala.*:[scala.Boolean, scala.Tuple$package.EmptyTuple]]]`, with the three-item `prefixItems` entry.
- `Either[Int, String]` is keyed `scala.util.Either[scala.Int, scala.Predef.String]`, a `$type`-tagged `oneOf` over `scala.util.Left[scala.Int, java.lang.String]` and `scala.util.Right[scala.Int, java.lang.String]`, each an object with the single property `value`.
- A named pair inside `Option` is the inline pair inside the optional shape; as the type argument of `Box`, the key prints it in full, `example.Box[scala.NamedTuple.NamedTuple[scala.Tuple2["x", "y"], scala.Tuple2[scala.Int, scala.Int]]]`, while `value` is the inline pair.
- `Map[Inner, Int]`, `Map[Color, Int]`, and `Map[Double, Int]` are all `{"type": "object", "additionalProperties": {"type": "integer"}}`, and no entry describes `Inner` or `Color`.
- A family `Event` with `case object Ping`, `case class Logout(id: Int)`, and a sealed sub-trait `UserEvent` holding `case class Created(id: Int)` and `case object Deleted` gets `example.Event` as a `oneOf` of `{"const": "Ping"}`, a `Logout` case, and an `allOf` over `{"$ref": "#/$defs/example.UserEvent"}` tagged `UserEvent`; `example.UserEvent` is a `oneOf` of a `Created` case and `{"const": "Deleted"}`.
- `case class R[T](data: Option[T])` used as `R[Option[Int]]` is keyed `example.R[scala.Option[scala.Int]]`, and `data` carries the nested optional shape.
- Under `optionsAsNulls = false`, an `Option[Option[Int]]` field is the boxed array nested in the boxed array.

### Reads and writes

| Claim | Output | Matches the reference |
|---|---|---|
| `read[Option[Int]]("[1]")` fails | `FAILED: TraceException` | Yes |
| `read[Option[Option[Int]]]("[1]")` | `Some(Some(1))` | Yes |
| `read[Option[Option[Int]]]("[]")`, `("[null]")` | `None`, `Some(None)` | Yes |
| `read[Option[Option[Int]]]("null")` | `null` (a Scala null) | Yes; added to section 1 and to the test rule in `SKILL.md` |
| `write` of `Some(Some(1))`, `Some(None)`, `None` as `Option[Option[Int]]` | `[1]`, `[null]`, `[]` | Yes |
| `read[NoDefault]("{}")` with `x: Option[Int]` | `FAILED` (missing key) | Yes |
| `read[WithDefault]("{}")` with `x: Option[Int] = None` | `WithDefault(None)` | Yes |
| `write(NoDefault(None))` | `{"x":null}` | Yes |
| `write(Long.MaxValue)`, `write(123L)`, `read[Long]("\"123\"")`, `read[Int]("\"123\"")` | `"9223372036854775807"`, `123`, `123`, `123` | Yes |
| `Envelope[Payload](None, 1, 2L)` and a `Payload` with the default name | `{"total_count":1,"trace_id":2}`, `{"id":"p1","tags":["a"],"labels":{"1":"one"},"kind":"Standard"}` | Yes: defaults omitted |
| Tags: `Kind.Standard`, `Kind.Custom("x")`, `@key("reddit.com")`, `@key("kind")` family, `tagName = "kind"` | `"Standard"`, `{"$type":"Custom","label":"x"}`, `"reddit.com"`, `{"kind":"Other","host":"a"}`, `{"kind":"Dog","name":"rex"}` | Yes |
| Tag last in the object | `{"label":"x","$type":"Custom"}` reads | Yes |
| Fully qualified tag | sealed trait: reads `example.LinkedList.Node` and `example.LinkedList.End`; Scala 3 enum: `example.Kind.Custom` fails | Fixed: section 5 now says only a sealed trait accepts it |
| A family `@key` against the Api's `tagName` | `@key("type")` wins over `tagName = "kind"` | Yes |
| A case outside the companion | top-level `Leaf` beside `Tree.Node` and `Tree.End`: tags `Tree.Node`, `Tree.End`, `Leaf` | Yes |
| `optionsAsNulls = false` object, Scala 3 and 2.13 | `{"o":[1]}` is `Some(1)`, `{"o":[]}` and `{}` are `None` | Yes |
| `optionsAsNulls = false`, nested and bare values | writes `{"o":[1],"n":[[2]]}`, `{"n":[[]]}`; reads `[[2]]`, `[[]]`, `[]` as `Some(Some(2))`, `Some(None)`, `None`; `{"o":1}` fails; `{"o":null}` gives a Scala `null` | Yes; reference sections 1 and 7, `SKILL.md` test rule |
| `Either[Int, String]` written and read | `Left(1)` is `[0,1]`, `Right("s")` is `[1,"s"]`; both read back | Yes |
| One-entry `Map[K, Int]` for each object key type | `String` `{"a":1}`, `Symbol` `{"s":1}`, `Char` `{"c":1}`, `Boolean` `{"true":1}`, `Unit` `{"null":1}`, `UUID`, `Int`, `Long`, `Short`, `Byte`, `Double` `{"1.5":1}`, `Float`, `BigInt`, `BigDecimal` `{"6.5":1}`, `Duration` `{"1000000000":1}`, a `stringKeyRW` type `{"k":1}`: all objects | Yes |
| `Map[K, Int]` with an enum and a case-class key | `[["Red",1]]`, `[[{"a":1},1]]` | Yes |
| `Map[Color, Int]` and `Map[String, Int]` reading `[["Red",1]]` | `Map(Red -> 1)` both | Yes |
| `case class Pair(_1: Int, _2: String)` | written `{"_1":1,"_2":"a"}`; reads the object, rejects `[1,"a"]` | Yes |
| `case class Point(x: Int, y: Int)` against a named pair's writes | reads `{"x":1,"y":2}` directly, `Some(Point(3,4))` from `{"x":3,"y":4}` as `Option[Point]`, `Box(Point(1,2))` from `{"value":{"x":1,"y":2}}`; `(Int, Int)` rejects the object | Yes |
| Sealed sub-family on the producer | writes `"Ping"`, `{"$type":"Logout","id":1}`, `{"$type":"Created","id":2}`, `"Deleted"`; never `UserEvent` | Yes |
| Sub-family consumer, `@key` on every case, Scala 3.3.7 and 2.13.16 | reads all four producer forms as `Event` and writes them back unchanged; the reference's section 5 blocks, compiled verbatim, read `"Ping"` and `{"$type":"Created","id":1}` | Yes |
| Sub-family consumer with `@key` on the inner cases only, Scala 2.13.16 | `"Ping"` fails; `Ping` is written `"EventNoKey.Ping"` | Yes; hence `@key` on every case |
| `R[Option[Int]]` written and read | `Some(Some(3))` is `{"data":3}`; `Some(None)` and `None` are `{"data":null}`; `R[T](data: Option[T])` reads `{"data":3}` as `R(Some(Some(3)))` and `{"data":null}` as `R(None)` on both versions | Yes |
| `Timeout(t: Option[Int] = Some(30))` | written `{}`; `{}` reads `Timeout(Some(30))` | Yes |

### Consumer code

The Scala 3 block, the Scala 2.13 block, and the munit test of reference section 9 were compiled from the same text as the reference (byte-identical, checked by script) and run: 4 of 4 tests pass on Scala 3.3.7, 4 of 4 on Scala 2.13.16, and 4 of 4 with the Scala 2.13 block compiled as Scala 3.3.7. The section 5 families (`@key("kind")`, `@key("reddit.com")`, a recursive family with `implicit lazy val`) and the section 7 configuration objects were compiled and run on both versions. In the second round, the section 5 sub-family blocks were extracted from the reference by script and compiled verbatim on both versions, beside an extended family with an outer case class and an inner case object.

### Differences from design's Context, and the fixes

| Found | Fix |
|---|---|
| A nested `Option[Option[T]]` reads a bare `null` as a Scala `null`, not `None`. | Reference section 1 states it; the test rule in `SKILL.md` reads a nested field from `[x]`, `[null]`, and `[]`. |
| A Scala 3 `enum` rejects the fully qualified tag; only a sealed trait accepts it. | Reference section 5. |
| A pair is inline, but other tuple arities are `$defs` entries keyed `scala.TupleN[...]`. | Reference sections 1 and 4 (rule 7); `SKILL.md` names rule: the property takes the tuple type and no definition is generated. |
| A named tuple of arity 2 gets the plain pair's array schema although upickle writes an object; other arities get `scala.NamedTuple.NamedTuple[...]` keys and object entries. Named tuples need the `upickle-implicits-named-tuples` module. | Reference section 1, "Where the schema and the wire disagree", and section 6. |
| A case class with fields `_1`, `_2` gets a `prefixItems` schema but is written as an object. | Reference section 1, same subsection. |
| `Either` gets a `$type`-tagged `oneOf` schema but is written `[0, a]` or `[1, b]`. | Mapping requirement "Either entries map to Either"; `SKILL.md` "Shapes the generator misdescribes"; reference sections 1, 2, and 4. |
| A map with a key type upickle does not write as an object key is written as pairs under the string-keyed map's schema, and the key's schema is not emitted. | Mapping requirements on map key types; `SKILL.md` and reference section 1 list the object key types. |
| A tuple written with `*:` is keyed `scala.*:[...]`, not `scala.Tuple<N>[...]`. | Mapping requirement "Tuple entries in $defs map to tuples"; `SKILL.md` names rule; reference sections 1, 2, and 4. |
| Under `optionsAsNulls = false`, a nested option is `[[x]]`, `[[]]`, `[]`, and a bare value fails. | Workflow requirement "Boxed optional fields are tested with arrays"; mapping requirement "Unknown defaults under boxed options ask the user"; reference sections 1 and 7. |
| A sealed sub-family's `UserEvent` tag is never written, and on Scala 2 the outer cases need `@key` too. | Mapping requirement "Sub-families are read by their inner tags"; reference section 5 blocks, compiled on both versions. |
| A generic `Option[T]` instantiated with an `Option` is written unboxed under the nested schema. | Mapping requirement "Optional shapes map to Option" and its scenario "Nested optional through a type argument"; workflow test rule by declared type. |
| On Scala 2, a sealed trait's `macroRW` needs a `ReadWriter` for each case object, and a generic class needs `implicit def`. | `SKILL.md` derivation paragraph and reference sections 4 to 6. |

## 7. Open questions for the specs, resolved

The first probe surfaced three points where the specs and upickle 4.4.3 disagreed. All three were folded into the spec deltas in task 8, after three rounds of adversarial review of the revised artifacts (37, 31, and 10 confirmed findings, each kept only when two of three skeptics agreed), which also brought in further shapes the generator misdescribes:

1. **Test documents for a nested optional field.** Now the workflow requirement "Test documents follow what upickle writes": a field declared `Option[Option[T]]` reads `[x]`, `[null]`, `[]`, and the absent key, never a bare `null`. The requirement "Boxed optional fields are tested with arrays" adds the `optionsAsNulls = false` forms, and test documents are chosen by a field's declared type.
2. **`$defs` entries for tuples.** Now the mapping requirements "Tuple entries in $defs map to tuples" (`scala.Tuple<N>[...]` and `scala.*:[...]`, no definition) and "Named-tuple entries become case classes named by their use"; the names, type-argument, and layout requirements exclude `scala` keys and entries with no definition of their own.
3. **Generator defects.** Now six producer types with their own requirements: inline pairs and named pairs (a payload winning over the source), positional case classes, `Either`, maps keyed by a type upickle writes as pairs, sealed sub-families, and generic options instantiated with an `Option`. They apply only to upickle-jsonschema output, and each correction is listed under the report's `Generator defects:` label. The workflow requirement "The producer's definition decides the shapes the generator misdescribes" limits what the source decides, and "Unknown defaults under boxed options ask the user" covers the boxed producer's unknown defaults.

**Resolved after task 8.** The D16 dry-run (section 5, second round) showed that the absent-key test document of an `Option` field with a producer default other than `None` reads that default. The workflow scenario "Optional field states" and the test bullet of `SKILL.md` now expect the field's default, which is `None` unless the producer's source gives another.
