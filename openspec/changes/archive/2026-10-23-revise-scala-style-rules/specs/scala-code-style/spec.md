# Spec Delta

## ADDED Requirements

### Requirement: Scala 3 brace style is the agent's choice, consistent within a file
In Scala 3 code the agent SHALL choose between the indentation-based and the braced syntax itself, matching the file it is in and, in a new file, the repository's prevailing style, and SHALL NOT mix the two styles within one file. WHEN the module compiles with Scala 2.13, the agent SHALL use braces.

#### Scenario: Editing an existing file
- **WHEN** the agent adds a definition to a Scala 3 file written in one style
- **THEN** the new definition is written in that file's style

#### Scenario: A new file
- **WHEN** the agent creates a Scala 3 file in a repository whose files share one style
- **THEN** the new file uses that style

#### Scenario: A repository with no prevailing style
- **WHEN** the repository's Scala 3 files use both styles and the agent creates a new file
- **THEN** the agent picks either style and uses it throughout the file

#### Scenario: Braces on Scala 2.13
- **WHEN** the module is on Scala 2.13, or cross-built for Scala 2.13 and Scala 3
- **THEN** the agent uses braces, since Scala 2.13 has no other syntax, and follows the repository's layout

### Requirement: Members are named in lowerCamelCase
Every `val`, `var`, `def`, parameter and case class field the agent declares SHALL be named in lowerCamelCase, constants included, with acronyms treated as words. Types, objects and enum cases SHALL be UpperCamelCase and packages lowercase. Another spelling SHALL be kept only for a member that overrides one the agent does not control.

#### Scenario: A constant
- **WHEN** the agent declares a constant such as the maximum number of retries
- **THEN** it is named `maxRetries`, not `MaxRetries` or `MAX_RETRIES`

#### Scenario: A constant in a pattern
- **WHEN** a lowerCamelCase constant is used in a `case` pattern
- **THEN** the agent back-quotes it, so the pattern compares against the constant instead of binding a new variable

#### Scenario: An acronym in a name
- **WHEN** a name contains an acronym such as ID or HTTP
- **THEN** it is written as `userId` or `httpClient`, not `userID` or `HTTPClient`

#### Scenario: A field that mirrors a wire name
- **WHEN** a case class field is read from or written to a wire format under a name that is not lowerCamelCase, such as `first_name`
- **THEN** the field is `firstName` and the wire name is carried by the serialisation library's key annotation, such as upickle's `@key("first_name")`

#### Scenario: An overridden member
- **WHEN** the agent overrides a member of a library or Java type whose name is not lowerCamelCase
- **THEN** the override keeps that name, since the agent cannot change it

### Requirement: Required libraries are declared or asked for
The libraries the rules file names as required SHALL be declared in every module the rules govern. WHEN a module lacks one that a rule needs for the code at hand, the agent SHALL stop and ask the user to add it, naming organisation, artifact, module and configuration, SHALL NOT add it itself, and SHALL NOT write the standard-library or hand-written form in its place.

#### Scenario: A chain in a module without mouse
- **WHEN** the agent would chain a single-use value in a module that does not declare mouse
- **THEN** it asks for mouse with its coordinates and does not write `pipe` from `scala.util.chaining` instead

#### Scenario: A nested update in a module without Monocle
- **WHEN** the agent would update a field nested two levels deep in a module that does not declare Monocle
- **THEN** it asks for Monocle and does not write nested `copy` calls instead

#### Scenario: The library's rule does not apply
- **WHEN** the code at hand needs none of a library's rules, such as a one-level `copy` or a plain enum that is only matched on
- **THEN** the agent writes it and asks for nothing

#### Scenario: The user has added the library
- **WHEN** the user confirms the dependency has been added
- **THEN** the agent continues with the library's form

### Requirement: A compiler option a required library needs is asked for
WHEN the code at hand declares an enumeratum value enumeration and a Scala 3 module, or the Scala 3 compilation of a cross-built module, lacks the compiler option `-Yretain-trees`, the agent SHALL stop and ask the user to add it, naming the option and the module, SHALL NOT change the module's compiler options itself, and SHALL NOT write a hand-written enumeration in its place.

#### Scenario: A value enumeration in a Scala 3 module without -Yretain-trees
- **WHEN** the agent would declare an enumeratum value enumeration in a Scala 3 module whose compiler options lack `-Yretain-trees`
- **THEN** it asks the user to add the option for that module, does not change the compiler options itself, and does not write a hand-written enumeration instead

#### Scenario: A value enumeration in a cross-built module
- **WHEN** the agent would declare an enumeratum value enumeration in a module cross-built for Scala 2.13 and Scala 3 that lacks `-Yretain-trees` on its Scala 3 compilation
- **THEN** the agent asks for the option on the Scala 3 compilation only, since Scala 2.13 rejects it

#### Scenario: A value enumeration on Scala 2.13 only
- **WHEN** the agent would declare an enumeratum value enumeration in a module that compiles with Scala 2.13 only
- **THEN** the agent asks for no compiler option, since only Scala 3 needs it

#### Scenario: A module that only looks a value enumeration up
- **WHEN** a Scala 3 module looks up or enumerates a value enumeration declared in another module, and declares none itself
- **THEN** the agent asks for no compiler option, since only the module that declares the enumeration needs it

## MODIFIED Requirements

### Requirement: Public members declare their return types
Every public `def`, `val` and `given` (or `implicit` definition on Scala 2.13) SHALL carry an explicit return type. Every `implicit val`, `lazy val` and `def`, private and local included, SHALL carry an explicit type; an implicit class or object declares no result type and is unaffected.

#### Scenario: A new public method
- **WHEN** the agent adds a public method
- **THEN** its result type is written out, even when the compiler could infer it

#### Scenario: A refactor changes an implementation
- **WHEN** a refactor would change the inferred type of a public member
- **THEN** the explicit type makes the compiler report the change instead of silently altering the signature

#### Scenario: A local value
- **WHEN** the agent declares a private or local `val` that is not `implicit`
- **THEN** it may leave the type inferred

#### Scenario: A private or local implicit definition
- **WHEN** the agent declares a private or local `implicit val`, `lazy val` or `def`, such as an ordering in a companion
- **THEN** it writes the type out, `private implicit val ordering: Ordering[Score] = ...`, since Scala 3 rejects an implicit member without one, and a local one whose right-hand side searches for implicits, and Scala 2.13 warns about the member

### Requirement: Single-use values are chained
WHEN a `val` is used exactly once, on the next line, only to feed the next expression, the agent SHALL chain the expressions with `|>` from mouse instead of naming the value, and SHALL use `<|` for a side effect that passes the value on unchanged. WHEN the name documents intent, the agent SHALL keep the named `val`. The agent SHALL NOT use `pipe` or `tap` from `scala.util.chaining`.

#### Scenario: A pass-through value
- **WHEN** `val parsed = parse(raw)` is followed only by `validate(parsed)`
- **THEN** the agent writes `parse(raw) |> validate`

#### Scenario: A side effect in the middle of a chain
- **WHEN** a value must be logged and then passed on unchanged
- **THEN** the agent uses `<|` for the logging step, with a function that returns `Unit`

#### Scenario: A side effect that returns a value
- **WHEN** the side effect inside `<|` would return a value
- **THEN** the agent discards that value explicitly inside the lambda, so no discarded-value warning is produced

#### Scenario: A name that documents intent
- **WHEN** the value's name explains a non-obvious step, such as `val normalisedEmail`
- **THEN** the agent keeps the named `val`

### Requirement: Scala 2.13 equivalents
WHEN the module compiles with Scala 2.13, the agent SHALL use the equivalents of the four Scala 3 constructs these rules name: a `sealed trait` with case objects and case classes in its companion for an `enum`, a value class or tagged type for an opaque type, `implicit` definitions and implicit classes for givens and extension methods, and `_` for the `*` wildcard in an import. Every other rule SHALL apply unchanged.

#### Scenario: A closed set on Scala 2.13
- **WHEN** the rules call for an enum and the module is on Scala 2.13
- **THEN** the agent declares a `sealed trait` with case objects in its companion

#### Scenario: A domain value on Scala 2.13
- **WHEN** the rules call for an opaque type and the module is on Scala 2.13
- **THEN** the agent declares a value class or a tagged type with the same smart constructor

#### Scenario: A wildcard import on Scala 2.13
- **WHEN** a rule names an import such as `import mouse.all.*` and the module is on Scala 2.13
- **THEN** the agent writes `import mouse.all._`, which Scala 2.13 accepts without `-Xsource:3`

#### Scenario: A cross-built module
- **WHEN** a module compiles for both Scala 2.13 and Scala 3
- **THEN** the agent uses the Scala 2.13 forms, which both versions accept

## REMOVED Requirements

### Requirement: Scala 3 code is braceless
**Reason**: The house style leaves the choice between braceless and braced Scala 3 syntax to the agent; a mandate for one style contradicted repositories written in the other and decided nothing the formatter does not already decide.
**Migration**: Follow the added requirement "Scala 3 brace style is the agent's choice, consistent within a file" together with the existing precedence of repository conventions on syntax; its Scala 2.13 scenario now lives under that added requirement.
