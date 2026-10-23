# Design

## Context

See proposal.md for motivation. The facts that shape the approach, taken from the rules file, the archived `add-virtuslab-scala-style-rules` design, the `upickle` skill and its specs, the `project-dependencies` spec, and the four libraries' documentation read on 2026-10-23:

- `rules/virtuslab-scala-style.md` is 104 lines in five sections with three fenced examples. The archived design caps it at 150 lines and three examples (D2, D11), verifies that it names no library (D4, D11), and lists four Scala 3 constructs that 2.13 lacks, braces among them (D6). The second is reversed here (D9) and the third is revised (D2, D12); every other one stands.
- The `project-dependencies` spec, which belongs to the `sbt-launcher` skill, forbids the agent to add a dependency, has it ask with coordinates, module, configuration and reason, and pauses the task there. One of its scenarios says a declared library that covers the need wins over a more familiar one.
- The `upickle` skill keeps a legal-identifier property name verbatim, lets a family case keep its tag's spelling, and mirrors a producer's field and case names when it has the source. It forbids hand-written readers: derived `ReadWriter`s decode every generated type.
- mouse exports `|>` and `thrush` (pipe) and `<|` and `unsafeTap` (tap, taking a function that returns `Unit`) through `import mouse.all.*`, is published for Scala 2.12, 2.13 and 3, and depends on cats-core. Monocle 3 is published for 2.13 and 3; its install lists `monocle-core` and `monocle-macro`; the `focus` syntax comes from `import monocle.syntax.all.*`; `.some` and `.index` inside `focus` are Scala 3 only. enumeratum (`com.beachape::enumeratum`) and Chimney (`io.scalaland::chimney`) publish for 2.13 and 3. `scala.util.Using` exists since Scala 2.13.0, and `Try` captures `NonFatal` failures only. A wildcard import written `import x.*` compiles on Scala 2.13 only under `-Xsource:3`, while `import x._` compiles with no warning on Scala 2.13, with or without `-Xsource:3`, and on Scala 3 at its default source level (checked with fatal warnings on 2.13.16, 3.3.7 and 3.7.4); Scala 3 rejects it under `-source:future` and warns under `-source:future-migration`.
- enumeratum's README lists among its Scala 3 limitations that `-Yretain-trees` must be set when using ValueEnums, and its Scala 3 `ValueEnum` macro aborts compilation without the option (`macros/src/main/scala-3/enumeratum/ValueEnumMacros.scala`). Scala 2.13.16 rejects `-Yretain-trees` as a bad option, while 3.3.7 and 3.7.4 accept it with no warning. An `implicit val` or `implicit def` member without a type is an error on Scala 3.3.7 and 3.7.4 and a default warning on Scala 2.13.16, an error under `-Xsource:3`; a local `implicit val` without a type compiles on 2.13 but is an error on Scala 3 when its right-hand side searches for implicits.
- Nothing in this repository is executable; verification is by text checks and review, as in every earlier change.

## Goals / Non-Goals

**Goals:**

- Each rule that names a library states, in one bullet, when it applies, the form to write and the import, so that the file stays under the line ceiling.
- One mechanism, stated once in the preamble, for a module that lacks a required library, consistent with the dependency rules of the `sbt-launcher` skill.
- The `upickle` skill and the rules file agree on naming, with no window in which they contradict each other.
- Everything else the archived design fixed stays: five sections, three examples, no upstream toolset name, the attribution line.

**Non-Goals:**

- Rules for the rest of mouse, cats, Chimney, Monocle or enumeratum beyond the four jobs named.
- Pinning or recommending library versions anywhere in this repository.
- Rewriting existing code in consumer projects, including upickle consumers generated under the verbatim rule.
- Any edit to the `sbt-launcher` skill or its specs.

## Decisions

### D1. Required libraries: ask, never substitute

The preamble gains a paragraph naming the four libraries by organisation and artifact (`org.typelevel::mouse`, `com.beachape::enumeratum`, `io.scalaland::chimney`, `dev.optics::monocle-core` with `dev.optics::monocle-macro` on Scala 2.13), without versions, and says: when a module lacks one that a rule needs, stop and ask the user to add it, naming organisation, artifact, module and configuration; never add it; never write the standard-library or hand-written form in its place. Each library rule then needs no clause of its own about absence. D14 extends the same mechanism to the one compiler option a required library needs.

Ask-and-stop over ask-and-fall-back: the user chose it. A fallback would leave a codebase mixing `pipe` and `|>`, or nested `copy` and `focus`, depending on which module the agent happened to be in, and the fallback code would need rewriting once the library arrived. The accepted cost: in a project that lacks a library the agent pauses at the first use its rule covers.

Coordinates in the preamble: the dependency rules want the ask to name organisation and artifact, and an always-on file is the cheapest place to keep them. Versions are left out because they drift and the project chooses them.

Precedence against the dependency rules: the `project-dependencies` scenario "A declared library covers the need" has the agent use a declared library over a more familiar one. The standard library is always declared, so that scenario read alone would have the agent write `pipe`. The rules file therefore says that the standard-library form is not an accepted alternative to a required library: the reason is a house rule, not familiarity, which is what that scenario guards against. The dependency spec and the `sbt-launcher` skill are not edited; this reading is recorded here so that a later reader does not take the two texts for a contradiction.

Alternatives considered: fall back to the standard-library form while asking (rejected by the user); an install-time check in the README with no rule text (rejected: the agent must know what to do in a module that lacks a library, and the README cannot reach it); a clarifying scenario in `project-dependencies` (rejected for now: it would edit another deliverable's spec for a case the rules file already states; it stays available as a later change if an agent is ever seen to substitute).

### D2. Braces are the agent's choice

The braceless bullet is replaced by: match the file you are in, match the repository's prevailing style in a new file, never mix the two styles within one file; on Scala 2.13 use braces. The 2.13 rule loses its braces clause; D12 adds the import wildcard in its place, so it still names four constructs. The precedence scenario about a braced Scala 3 codebase stays and now does the work the mandate did. A consistency clause rather than silence: mixing styles in one file is legal Scala 3 and the one outcome everyone agrees is wrong; where a repository configures a formatter rewrite, the formatter decides the rest, as the preamble already says.

### D3. Naming

One bullet: lowerCamelCase for every `val`, `var`, `def`, parameter and case class field, constants included and acronyms treated as words (`userId`, `httpClient`, `maxRetries`); UpperCamelCase for types, objects and enum cases; lowercase packages; the override exemption; wire names carried by the serialisation library's key annotation, with upickle's `@key("first_name") firstName` as the example; and back-quoting a constant in a pattern, with the reason, that a lowercase identifier in a `case` binds a new variable instead of comparing. The user confirmed the literal reading for constants against the Scala style guide's UpperCamelCase constants.

Upickle alignment lands in the same change, so that no revision exists in which the skill says verbatim and the rules say camelCase: three paragraphs of `skills/upickle/SKILL.md` ("With source, mirror it", "Objects with properties become case classes", "Cases are named by their tags"), one table row and one sentence of `references/schema-mapping.md`, and the two upickle specs. Family cases follow the same principle, UpperCamelCase with `@key`, as the consistent reading of the rule; the user saw this extension during exploration and did not object.

Consequence for Chimney: it matches fields by name, so a DTO field `@key("first_name") firstName` maps to a domain `firstName` without a rename. That is one more reason to carry the wire spelling in the annotation rather than in the field.

### D4. Chaining with mouse

`parse(raw) |> validate`, `<|` for a side effect that passes the value on, `import mouse.all.*`, and an explicit "do not use `pipe` or `tap` from `scala.util.chaining`" so that the no-substitution rule is visible at the one place it will bite most often. `<|` takes a function returning `Unit`, so the rule tells the agent to discard a side effect's result inside the lambda; otherwise the zero-warnings rule and a discarded-value warning collide. `thrush` and `unsafeTap` are aliases and go unmentioned, to keep one form.

### D5. Enumeratum

Condition: a closed set of members that carry no data and that is looked up by name or by a wire value, enumerated, or serialised under custom names. Form: entries extending `EnumEntry` with `findValues` in the companion, or a `ValueEnum` for wire codes. Lookups through `withNameOption`, `withNameEither` or their case-insensitive variants, never the throwing `withName`, with the library's `NoSuchMember` wrapped into the module's error type, so that the error-handling rules hold unchanged. Exclusions, stated in the rule: a set that is only matched on stays a plain `enum`; a hierarchy whose cases carry data stays sealed; a data transfer type decoded by derived codecs stays a plain enum, because upickle's derived readers do not know enumeratum and the upickle skill forbids hand-written readers. On Scala 2.13 enumeratum also removes the hand-maintained `values` list, which the 2.13 equivalence rule does not need to mention. On Scala 3 a `ValueEnum` needs the compiler option `-Yretain-trees` (D14).

### D6. Chimney

Condition: a mapping that is mostly field by field, such as a DTO to its domain type, a generated type to the domain's, or one version of a type to another. Form: `transformInto` or `into(...).transform` with `withFieldRenamed`, `withFieldConst` or `withFieldComputed` for the fields that differ, `import io.scalaland.chimney.dsl.*`. Inbound validation: a `PartialTransformer` whose field transformers come from the smart constructors, and an adapter that wraps the partial result's errors into a case of the module's error type, so that no Chimney type reaches a module's error. Instances in the type's companion, as the organisation spec already requires; Chimney does not see through an opaque type, which is why the companion instance is needed at all. Hand-written when most fields need logic of their own.

### D7. Monocle

Threshold: a field two or more levels deep. Form: `focus(_.a.b).modify(f)` or `.replace(v)` with `import monocle.syntax.all.*`; a lens that more than one place uses becomes a `val` in the state type's companion; a one-level update stays `copy`. The 2.13 clause: `.some` and `.index` are unavailable inside `focus` there, so an `Optional` or a `Traversal` is composed explicitly, and `monocle-macro` is required. Two levels rather than one: a single nested `copy` is readable; two is where the pyramid begins.

### D8. Try and Using

"Try/Using instead of try/finally keywords" is read as the whole keyword family: `Try` replaces `try`/`catch`, `Using` replaces `try`/`finally`, and the `try` keyword does not appear in new code. `Try` is a conversion step, never a result type: the boundary rule says capture the call with `Try`, match the most specific exception type in `Failure` to a domain error and rethrow the rest; never `toEither`, which puts `Throwable` on the left, which the error rules forbid; never return `Try` from a service. The loop rule becomes `Try(process(item))` with the `Failure` logged; `Try` captures `NonFatal` only, so the `InterruptedException` clause holds by construction. `Using.resource` where a failure should propagate, `Using` where its `Try` is converted, `Using.Manager` for several resources. The boundary example is rewritten:

```scala
def readReport(path: Path): Either[ReportError, String] =
  Try(Files.readString(path)) match
    case Success(report)                 => Right(report)
    case Failure(_: NoSuchFileException) => Left(ReportError.Missing(path))
    case Failure(defect)                 => throw defect
```

The explicit `throw defect` is kept over a `recover(...).get` chain because it shows the propagation the rule asks for.

### D9. What the archived design loses, and what it keeps

- D4 and D11 of `add-virtuslab-scala-style-rules`: the file now names four libraries, and the verification grep expects them to be present. The grep for upstream's toolset names (sbt, scalafmt, Ox, Tapir and the rest) still runs and still expects nothing.
- D6: the 2.13 rule still names four constructs, but braces leave it and the import wildcard joins it (D12).
- D2's shape is kept: five sections, one example per rule most easily misread, three examples, one closing attribution line.
- The file name keeps `virtuslab`: the file is still adapted from VirtusLab's rules; the README row says the four libraries are this repository's additions.

### D10. Specs

Each library rule sits in its topical spec: chaining in `scala-code-style`; enumeratum, Chimney and Monocle in `scala-domain-modelling`; `Try` and `Using` in `scala-error-handling`. The missing-library mechanism sits in `scala-code-style` beside the precedence rule. The loop-boundary requirement is removed and re-added under a new name rather than modified, because its name named the `try` keyword. "Public members declare their return types" is MODIFIED, repeating its whole block with the implicit clause and scenario added (D13); the `-Yretain-trees` ask is its own ADDED requirement, "A compiler option a required library needs is asked for", beside "Required libraries are declared or asked for", and both `rules-collection` requirements name the compiler option beside the libraries (D14). The braces-on-2.13 scenario moves from the 2.13 equivalence requirement, whose text no longer lists braces, to the brace-choice requirement, whose text states it, matching the rules file, where the clause sits in the brace-choice bullet. The `scala-code-style` Purpose sentence that says the rules name no library is edited directly in the main spec, which the OpenSpec instructions allow for a Purpose, since deltas carry requirements only. In the upickle mapping spec the two MODIFIED requirements keep every existing scenario name with updated content, as the validator requires; the workflow spec gets an ADDED requirement on spelling instead of a longer mirroring requirement.

### D11. Verification by text checks

- `wc -l rules/virtuslab-scala-style.md` prints a number between 100 and 150; `grep -c '^## ' rules/virtuslab-scala-style.md` prints 5; `grep -c '^```' rules/virtuslab-scala-style.md` prints 6.
- `grep -ciE 'mouse|enumeratum|chimney|monocle' rules/virtuslab-scala-style.md` prints at least 4, and `awk '/^## /{exit} {print}' rules/virtuslab-scala-style.md | grep -cE 'org\.typelevel|com\.beachape|io\.scalaland|dev\.optics'` prints at least 1, so the coordinates sit in the preamble.
- `grep -niE 'sbt|scalafmt|\bOx\b|tapir|sttp|jsoniter|pureconfig|macwire|magnum|flyway|hikari|scalaxb|kafka|opentelemetry|netty|bootzooka|direct-style' rules/virtuslab-scala-style.md` prints nothing.
- `grep -nE '^\s+(try|catch|finally)\b' rules/virtuslab-scala-style.md` prints nothing, so no fenced example uses the keywords; `grep -c 'indentation-based' rules/virtuslab-scala-style.md` prints 1, in the choice rule.
- `grep -c 'four Scala 3 constructs' rules/virtuslab-scala-style.md` prints 1 and `grep -c 'wildcard in an import' rules/virtuslab-scala-style.md` prints 1, both in the 2.13 rule; `grep -nE 'three Scala 3 constructs|braceless' rules/virtuslab-scala-style.md` prints nothing.
- `tail -1 rules/virtuslab-scala-style.md` still names VirtusLab, the upstream repository and `Apache-2.0`.
- `grep -c 'always carries its type' rules/virtuslab-scala-style.md` prints 1, in the explicit-types rule; `awk '/^## /{exit} {print}' rules/virtuslab-scala-style.md | grep -c 'Yretain-trees'` prints 1, so the option sits in the preamble; `awk '/^## Rules files/,0' README.md | grep -c 'Yretain-trees'` prints 1, in the install step.
- `grep -n 'pipe' rules/virtuslab-scala-style.md` prints only the chaining line, which forbids it; `grep -c '2\.13' rules/virtuslab-scala-style.md` prints at least 4.
- `grep -n 'first_name' skills/upickle/SKILL.md skills/upickle/references/schema-mapping.md` prints only lines in the `@key("first_name") firstName` form; `grep -n 'verbatim' skills/upickle/SKILL.md` prints only the lowerCamelCase case.
- `awk '/^## Rules files/,0' README.md | grep -ciE 'mouse|enumeratum|chimney|monocle'` prints at least 2, one for the row and one for the install step.
- `git status --porcelain skills/sbt-launcher openspec/specs/` lists only `openspec/specs/scala-code-style/spec.md`, and `git diff openspec/specs/scala-code-style/spec.md` touches only its Purpose paragraph.
- A read-through maps every bullet of the file to a requirement of the four `scala-` specs, including this change's deltas, and every requirement to a bullet; a bullet with no requirement, or a requirement with no bullet, is a defect to fix before archive.
- `openspec validate revise-scala-style-rules --strict` passes.

### D12. Wildcard imports on Scala 2.13

The library rules name their imports in the Scala 3 form, `import mouse.all.*`, `import monocle.syntax.all.*` and `import io.scalaland.chimney.dsl.*`, as every rule names the Scala 3 construct. Scala 2.13 rejects `*` there unless the build enables `-Xsource:3`, and the 2.13 rule says every other rule applies unchanged, so read literally the file would have the agent write imports that do not compile. The 2.13 rule therefore gains a fourth equivalent, `_` for the `*` wildcard in an import, with `import mouse.all._` as the example. `_` compiles with no warning on Scala 2.13, with or without `-Xsource:3`, and on Scala 3 at its default source level, so the rule that a cross-built module uses the 2.13 forms still holds. A repository that enables `-Xsource:3` and writes `*` throughout, or compiles Scala 3 under `-source:future`, which rejects `_`, is covered by the precedence of repository conventions on syntax, so the rule mentions neither flag.

Alternatives considered: the 2.13 form beside each import (rejected: it repeats one translation in three rules, while the file's convention is that each rule names the Scala 3 construct and the 2.13 rule holds every translation); the `_` form everywhere (rejected: `*` is the Scala 3 form, and the file names Scala 3 constructs first).

### D13. Implicit definitions always carry their type

The explicit-types rule exempts private and local values. Once the 2.13 rule has turned a given into an `implicit` definition, that exemption lets the agent write `private implicit val ordering = Ordering.by[Score, Int](_.value)`. Scala 3 rejects it, and Scala 2.13 warns by default, which the zero-warnings rule forbids, and rejects it under `-Xsource:3`. A local implicit without a type compiles on Scala 2.13 but fails on Scala 3 when its right-hand side searches for implicits. The exemption therefore stops at implicit definitions: every `implicit val`, `lazy val` or `def`, private and local included, carries its type. One unconditional clause rather than a split by visibility and right-hand side: the split is a fine distinction an agent gets wrong, and the type costs one annotation. Implicit classes and implicit objects declare no result type and are unaffected, and a Scala 3 `given` always states its type. The requirement keeps its name, "Public members declare their return types", and is MODIFIED rather than renamed.

### D14. `-Yretain-trees` for enumeratum value enumerations on Scala 3

enumeratum's Scala 3 `ValueEnum` macro aborts compilation unless the compiler option `-Yretain-trees` is set, and Scala 2.13 rejects the option. The check runs only where `findValues` expands, and `findValues` is `protected inline`, so it expands only in the object that declares the enumeration: a module that only looks a value enumeration up, or enumerates one declared elsewhere, needs no option (`enumeratum-core/src/main/scala-3/enumeratum/values/ValueEnumCompat.scala`). The rules forbid the agent to change compiler options, and D1's ask names only coordinates, so without a rule the agent would write a value enumeration it can neither compile nor ask for. The preamble therefore names the option beside enumeratum's coordinates, and D1's mechanism extends to it: when the code at hand declares a value enumeration in a Scala 3 module, or the Scala 3 compilation of a cross-built module, that lacks the option, the agent stops and asks the user to add it, naming the option and the module; it does not change the compiler options itself and does not substitute a hand-written enumeration. The README's install step names the option too.

Alternatives considered: plain `EnumEntry` entries that carry their code as a field (rejected: a hand-written form in place of the library's, which D1 rules out, with a hand-written lookup by value); the option in the enumeratum bullet only (rejected: the preamble is where both the installer and the agent learn what a module must declare, and the ask is stated there once).

## Risks / Trade-offs

- [Agents pause often in projects that lack a library] → Accepted by decision; the README tells installers to declare the four libraries first, and the ask names coordinates so that it is quick to grant.
- [A Scala 3 module without `-Yretain-trees` pauses at its first value enumeration] → Accepted on the same grounds (D14); the README names the option, and the agent asks for it for the Scala 3 compilation only, since Scala 2.13 rejects it.
- [cats-core arrives transitively with mouse and Monocle] → Named in the proposal and the README; a project that cannot take cats cannot install this file.
- [Chimney and Monocle macros lengthen compile times] → Accepted; both are used at adapters and state updates, not throughout.
- [A `<|` lambda that returns a value trips the zero-warnings rule] → The rule says to discard inside the lambda.
- [A constant forgotten in back-quotes binds instead of comparing] → The rule states the reason; the compiler reports the following cases as unreachable, which the zero-warnings rule catches.
- [Regenerating an existing upickle consumer renames its fields] → Marked BREAKING in the proposal; existing generated code is not rewritten by this change.
- [The Purpose edit to `scala-code-style` bypasses spec sync] → It is a Purpose edit, which the OpenSpec instructions say to make directly; the tasks list it and D11 checks that nothing else in the main spec changes.
- [A library's API drifts from the forms the rules name] → The forms named are each library's stable entry points (`|>`, `EnumEntry`, `transformInto`, `focus`); a later change re-reads the documentation.

## Migration Plan

One commit holding the rules file, the README, the upickle skill edits, the Purpose edit and this change's artifacts. Rollback is reverting the commit. A project adopting the new file declares the four libraries first; a project that cannot keeps the previous revision of the file.

## Open Questions

- Whether existing upickle consumers generated under the verbatim rule should be regenerated in a change of their own. Deferrable: it changes neither the rules nor the skill text.
