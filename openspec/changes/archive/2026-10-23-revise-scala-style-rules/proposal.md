# Proposal

## Why

The `virtuslab-scala-style` rules file was written to name no library, so it sends an agent to the standard library for chaining and exception handling, to hand-written field-by-field mappings and nested `copy` calls for everything else, mandates braceless Scala 3, and says nothing about naming. The corporate house style has since settled on four libraries for exactly those jobs, on lowerCamelCase for every member, on `Try` and `Using` in place of the `try` keyword, and on leaving the brace question to the agent. Until the rules file and the `upickle` skill say so, agents keep producing code that reviewers rewrite.

## What Changes

- Scala 3 brace style: the braceless mandate is removed. The agent chooses, matching the file it is in and the repository's prevailing style, and never mixes the two styles in one file. The Scala 2.13 equivalence rule drops its braces clause.
- Naming: every `val`, `var`, `def`, parameter and case class field is lowerCamelCase, constants included and back-quoted when they appear in a pattern; types, objects and enum cases are UpperCamelCase; packages are lowercase. Another spelling is kept only for a member that overrides one the agent does not control. A wire name that differs from the member's name is mapped with the serialisation library's key annotation.
- Explicit types: the exemption that lets private and local values stay inferred no longer covers an implicit `val`, `lazy val` or `def`, which always carries its type; implicit classes and objects declare no result type and are unaffected. Scala 3 rejects an implicit member without a type, and a local one whose right-hand side searches for implicits; Scala 2.13 warns about the member, which breaks the zero-warnings rule once the 2.13 rule has turned givens into implicits.
- Four required libraries, named in the preamble with organisation and artifact: mouse for chaining, where `|>` and `<|` replace `pipe` and `tap`; enumeratum for enumerations looked up by name or value; Chimney for mappings that are mostly field by field; Monocle for updates nested two or more levels deep. When a module lacks a library that a rule needs, the agent stops and asks for it, as the dependency rules already require, and never substitutes the standard-library or hand-written form. The library rules name their imports in the Scala 3 form, such as `import mouse.all.*`, which Scala 2.13 rejects unless the build enables `-Xsource:3`, so the Scala 2.13 equivalence rule gains `_` for the `*` wildcard and still names four constructs. On Scala 3, enumeratum's `ValueEnum` also needs the compiler option `-Yretain-trees`, which Scala 2.13 rejects; the preamble names it, and the agent asks for it, for the Scala 3 compilation only, as for a missing library, and never changes compiler options itself. **BREAKING** for a project that installed the file without those libraries: the agent now pauses and asks instead of working around them.
- Exceptions and resources: `Try` replaces `try`/`catch`, both at the conversion boundary and at a processing loop's outermost boundary; `Using` replaces `try`/`finally`; the `try` keyword does not appear in new code. The boundary example is rewritten in the `Try` form.
- The `upickle` skill aligns with the naming rule: a property name that is not lowerCamelCase gets a lowerCamelCase field with `@key("<property>")`; a tag that is not UpperCamelCase gets an UpperCamelCase case with `@key("<tag>")`; mirroring a producer's source mirrors names in meaning, camel-cased, with `@key` carrying the producer's spelling. **BREAKING** for consumers regenerated from a schema with snake_case properties: their fields change name.
- The README's rules-file row names the four libraries, and the install step says they must be declared before the file is installed, and that a Scala 3 module declaring an enumeratum value enumeration sets `-Yretain-trees` on its Scala 3 compilation.
- The Purpose paragraph of the `scala-code-style` spec no longer claims that the rules name no library.
- Unchanged: the `sbt-launcher` skill and its specs (`project-dependencies`, `sbt-invocation`, `launcher-owned-configuration`, `source-formatting`), `skill-collection`, `scala-code-organisation`, the archived changes and `.claude/`.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `scala-code-style`: the braceless requirement is removed and a consistency requirement replaces it; the 2.13 equivalence requirement drops braces and gains the import wildcard; the explicit-types requirement no longer exempts private or local implicit definitions; chaining moves to mouse; a naming requirement is added; a requirement that the libraries the rules file names are declared or asked for, never substituted, is added, and another that the compiler option enumeratum's value enumerations need on Scala 3 is asked for, never set by the agent itself.
- `scala-domain-modelling`: three requirements are added: enumerations looked up by name or value use enumeratum, mappings that are mostly field by field are derived with Chimney, updates nested two or more levels deep go through Monocle.
- `scala-error-handling`: the boundary-conversion requirement moves to `Try`; the bare try/catch requirement is removed and a `Try` at the loop boundary requirement replaces it; a requirement that resources are released through `Using` is added.
- `rules-collection`: a rules file also states the libraries it requires, with any compiler option one of them needs, and what the agent does when a module lacks one; the README's install step says they must be declared first.
- `upickle-schema-mapping`: object properties and family cases get camel-cased names, with `@key` carrying the wire spelling whenever it differs.
- `upickle-schema-workflow`: mirroring the producer's definitions mirrors names in meaning, camel-cased, with `@key` for the producer's spelling.

## Impact

- Files: `rules/virtuslab-scala-style.md` (preamble, one rule removed, six added, five rewritten, one example rewritten; about 114 lines, under the archived design's 150-line ceiling); `README.md` (one table row, three install sentences, and one clause in the convention for adding a rules file); `skills/upickle/SKILL.md` (three paragraphs) and `skills/upickle/references/schema-mapping.md` (one table row, one sentence each in sections 5 and 6, and the worked example's fields, test and report in section 9); `openspec/specs/scala-code-style/spec.md` (Purpose paragraph only, which delta specs cannot reach).
- Downstream: every project that installs the rules file must declare mouse, enumeratum, Chimney and Monocle in each module the file governs; mouse and Monocle bring cats-core with them, Scala 2.13 modules need monocle-macro for the `focus` syntax, and a Scala 3 module that declares an enumeratum value enumeration needs `-Yretain-trees` in its compiler options. In a project missing one, the agent pauses at the first chain, nested update, structural mapping or named lookup and asks, as the `project-dependencies` spec already describes.
- The `project-dependencies` spec is untouched. Its scenario that a declared library wins over a more familiar one concerns familiarity; the rules file states that the standard-library form is not an accepted alternative to a required library, and the design records that reading.
- Consumers generated by the `upickle` skill from schemas with snake_case or kebab-case properties get camel-cased fields and cases; existing generated code is not rewritten by this change.
- Attribution: the file stays adapted from VirtusLab's scala-skill, the four libraries being this repository's additions; the closing attribution line is unchanged.
- Verification stays text-based, as in every earlier change; the design lists the checks, including the grep that now expects the four library names to be present.
