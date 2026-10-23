# Scala code style and domain modelling

Apply these rules to all Scala code you write or change in this repository, production code and tests alike, on Scala 3 and on Scala 2.13.

Where a syntax rule below conflicts with a convention the repository's code already follows or its formatter enforces, follow the repository and keep the formatter's output; do not undo its layout. The rules on file structure, modelling, errors and tests apply regardless of the repository's existing habits.

Each rule names the Scala 3 construct; on Scala 2.13 use the equivalent that the last rule under "Syntax and function shape" gives.

Every module these rules govern MUST declare four libraries: mouse (`org.typelevel::mouse`), enumeratum (`com.beachape::enumeratum`), Chimney (`io.scalaland::chimney`) and Monocle (`dev.optics::monocle-core`, with `dev.optics::monocle-macro` on Scala 2.13). When a module lacks one that a rule needs for the code at hand, stop and ask the user to add it, naming its organisation and artifact, the module and the configuration; NEVER add it yourself, and NEVER write the standard-library or hand-written form in its place, which is not an accepted alternative to a required library. On Scala 3, enumeratum's `ValueEnum` also needs the compiler option `-Yretain-trees`, which Scala 2.13 rejects: when a Scala 3 module, or the Scala 3 compilation of a cross-built module, lacks it and the code at hand declares a value enum, ask for it the same way, naming the option and the module, and NEVER change the compiler options yourself.

## Syntax and function shape

- In Scala 3, choose between the indentation-based and the braced syntax yourself: match the file you are in and, in a new file, the repository's prevailing style, and NEVER mix the two styles within one file; on Scala 2.13 use braces, since the language has no other syntax, and follow the repository's layout.
- ALWAYS give every public `def`, `val` and `given` (an `implicit` definition on Scala 2.13) an explicit type, even when the compiler could infer it, so that a refactor that changes the type is reported instead of silently changing the signature; private and local values may stay inferred, except an implicit `val`, `lazy val` or `def`, which always carries its type, since Scala 3 rejects an untyped implicit member and Scala 2.13 warns about one.
- Name every `val`, `var`, `def`, parameter and case class field in lowerCamelCase, constants included, treating acronyms as words: `maxRetries`, `userId`, `httpClient`, never `MAX_RETRIES`, `MaxRetries` or `userID`; name types, objects and enum cases in UpperCamelCase and packages in lowercase; keep another spelling only for a member that overrides one you do not control. Where a wire format uses another name, carry it in the serialisation library's key annotation, such as upickle's `@key("first_name") firstName`, not in the field's name. In a `case` pattern, back-quote a constant, as in ``case `maxRetries` =>``, since a lowercase identifier there binds a new variable instead of comparing against the constant.
- Make each function do one thing: either a single logical operation or a short orchestration of calls to named steps with no inline logic of its own, never both mixed; split responsibilities between entities named after what they do, not where they are used, so a function that parses input and stores the result becomes a parser, a writer and, if needed, an orchestration of the two.
- When a `val` is used exactly once, on the next line, only to feed the next expression, chain instead with mouse's `|>`, `parse(raw) |> validate`, or with `<|` for a side effect such as logging that passes the value on unchanged, both from `import mouse.all.*`; `<|` takes a function returning `Unit`, so discard a side effect's result inside the lambda. Keep the named `val` when its name documents a non-obvious step, such as `normalisedEmail`. Do not use `pipe` or `tap` from `scala.util.chaining`.
- When a non-Unit result is intentionally unused, discard it explicitly, in the form the repository already uses (such as `val _ = Files.deleteIfExists(path)`), rather than leaving a bare non-Unit statement; when the result is needed further down, bind it to a named `val`.
- NEVER load a collection of unbounded size into memory: process it incrementally, as a stream or an iterator, or page through it; a plain in-memory collection is fine only when its size has a known small bound, such as a configuration list.
- Code you write MUST compile with zero warnings under the repository's configured compiler options: fix the cause in code; suppress a warning only in generated code or for a third-party issue you cannot fix, as narrowly as possible with `@nowarn` and a comment naming the generator or the issue; NEVER change compiler options to silence a warning.
- On Scala 2.13, and in modules cross-built for Scala 2.13 and 3, replace the four Scala 3 constructs these rules name with their equivalents and apply every other rule unchanged: a `sealed trait` with case objects and case classes in its companion for an `enum`; a value class or a tagged type for an opaque type; `implicit` definitions and implicit classes for givens and extension methods; `_` for the `*` wildcard in an import, such as `import mouse.all._`.

## Files, packages and visibility

- Before creating or moving a `.scala` file, decide which existing package it belongs to, the visibility of each top-level declaration, and a filename that matches the primary type or concept it holds.
- Give every top-level class, trait, enum and object the narrowest visibility its actual call sites allow, starting narrow and widening one step only when a new caller needs it:
  - public only when it is intentionally part of the API;
  - private to its concept package, such as `private[billing]`, when only that package uses it;
  - private to the root package, such as `private[acme]`, when other modules under the same root call it.
- Decide whether a type gets its own file by where it is constructed, not where it is referenced:
  - a service may share its file with the return types and exceptions that only it constructs;
  - a type constructed in several places gets its own file, named after the type;
  - a sealed hierarchy keeps all its cases in the file that declares it;
  - a trait may share its file with its single canonical implementation;
  - otherwise, one top-level type per file.
- NEVER create a file whose only common theme is the kind of declaration it holds, such as `Types.scala`, `Models.scala`, `Helpers.scala` or any `<Something>Types.scala`, and do not grow an existing one: place each new declaration by the construction-site rule, and leave the existing file as it is unless the user asks for a reorganisation.
- Name packages after concepts, such as `events`, `git` or `billing`, never after mechanisms such as `util`, `io`, `core` or `common`; keep at most one small `util` leaf per root package, and move its files into concept packages once it grows past about five.
- NEVER create a package that holds a single file: put the declaration in the parent package or fold it into a sibling.
- NEVER name a sub-package after a top-level `def` or `val` of the same root package, which Scala 3 rejects; sink the package one level deeper instead, such as `tools.git` beside a top-level `def git`.
- Keep a type's smart constructors, its opaque type's accessors and its type class instances (givens, or implicits on Scala 2.13) in the type's companion object, where implicit resolution finds them without an import; gather instances for types that have no companion in the codebase, such as library types, in one instances object in the concept package that needs them.

## Domain modelling

- Model state as immutable case classes and each transition as a pure function from state to state, testable without the loop or component that drives it; NEVER share mutable state between components: one owns the data and passes immutable snapshots.
- Update a field two or more levels deep through Monocle's `focus`, from `import monocle.syntax.all.*`, never through nested `copy` calls: `user.focus(_.address.streetNumber).modify(_ + 1)`, or `.replace(value)`; declare a lens that more than one place uses once, as a `val` in the state type's companion, and keep `copy` for a one-level update. On Scala 2.13, where `.some` and `.index` are unavailable inside `focus`, compose an `Optional` or a `Traversal` explicitly, with `monocle-macro` declared.
- Declare a `var` only inside a method body, in the smallest scope that needs it, and NEVER as a class or object field; model state that outlives a call as an immutable value the operation returns.
- Use immutable collections in production code; mutable collections belong only in test fakes.
- Pass the clock, the random source and the id generator in as parameters or constructor arguments and NEVER obtain them inside a method, so that results are reproducible and a test can assert on exact values.
- Make invalid data unrepresentable: an enum or sealed hierarchy for a closed set of values instead of a `String`, `Option` for absence and NEVER `null`, and a validated constructor as the only way to obtain a value that has an invariant.
- Declare a closed set whose members carry no data with enumeratum when it is looked up by name or by a wire value, enumerated, or serialised under custom names: entries extending `EnumEntry` with `findValues` in the companion, or a `ValueEnum` for wire codes. Look members up with `withNameOption`, `withNameEither` or their case-insensitive variants (`withValueOpt` or `withValueEither` for a `ValueEnum`), NEVER the throwing `withName`, and wrap the library's `NoSuchMember` in a case of the module's error type. A set that is only matched on stays a plain `enum`, a hierarchy whose cases carry data stays sealed, and a data transfer type decoded by derived codecs stays a plain `enum`, since those codecs do not know enumeratum.
- NEVER pass identifiers, quantities or configuration values around as bare `String`, `Int`, `Long` or `Boolean`: wrap them in opaque types (value classes or tagged types on Scala 2.13) or enums, so that `OrderId` and `Port` replace `String` and `Int` and a product code passed for an order id does not compile.
- Give a value with invariants a smart constructor that returns `Either` with a typed error naming the offending value, NEVER `require` or a throw, and expose no public way to build an invalid value; do not copy an existing `require` pattern into new types.

```scala
enum QuantityError:
  case NotPositive(value: Int)

opaque type Quantity = Int

object Quantity:
  def from(value: Int): Either[QuantityError, Quantity] =
    if value > 0 then Right(value) else Left(QuantityError.NotPositive(value))

  extension (quantity: Quantity) def value: Int = quantity
```

- Replace a Boolean parameter or result whose meaning a call site cannot read with a two-case enum that names both outcomes; a predicate whose name states the question, such as `isEmpty`, keeps its Boolean.

```scala
enum FlushOutcome:
  case Success, Failure

trait FlushMetrics:
  def record(outcome: FlushOutcome): Unit // not record(success: Boolean)
```

- Give identifiers of different entities distinct types, either one opaque identifier type parameterised by the entity it identifies, such as `Id[User]`, or one opaque type per entity, so that a user's id passed where an order's is expected does not compile.
- When a value must have been transformed before use, such as lower-cased, trimmed or hashed, give the result its own type, an opaque subtype on Scala 3, produced only by the function that performs the transformation, so that a lookup by `LowerCasedEmail` rejects a raw `String` and a repository stores only a `HashedPassword`.
- Convert types generated from schemas or wire formats, and types that belong to other systems, into domain types at the boundary where they arrive: their raw strings and numbers stop at the adapter, domain code never imports a generated package, and data transfer objects carry the domain's opaque types instead of primitives.
- Derive a mapping that is mostly field by field, such as a data transfer object to its domain type, with Chimney, from `import io.scalaland.chimney.dsl.*`: `dto.transformInto[User]`, or `dto.into[User]` with `withFieldRenamed`, `withFieldConst` or `withFieldComputed` for the fields that differ, then `.transform`. Make an inbound mapping that validates a `PartialTransformer` built from the smart constructors, and wrap its errors in a case of the module's error type at the adapter; declare instances in the type's companion; write the mapping by hand when most fields need logic of their own.
- Give each module its own configuration case class of typed values, composed into one top-level configuration; wrap every secret in a type whose string form is masked, so that logging the configuration never prints it; validate configuration when it is loaded and fail at startup with a message naming the invalid setting, not at its first use, which is the one place a constructor may throw.

## Error handling

- Return `Either` with a typed error for every failure the caller can act on, such as invalid input, a missing entity, a conflict or a refused permission; NEVER throw for one, directly or through a precondition check, and never hide it behind `try`/`catch`, even where existing code does.
- Let an exception end the current request, message or job for a failure the caller cannot handle, such as a bug, a lost connection, corrupted data or an exhausted resource; NEVER encode one as an `Either` case that no caller can act on.
- Give each module its own sealed error type, an enum or a sealed trait, whose cases carry the failure's data, such as `UserNotFound(id)` or `LoginTaken(login)`, and return `Either` of it from the module's services; NEVER return string errors such as `Left("user not found")`, and NEVER define an open application-wide error class for modules to extend, so that the compiler flags every match that misses a newly added case.
- When a caller combines results from another module, wrap the callee's error in a case of the caller's own error type, such as `CheckoutError.Payment(error: PaymentError)`, and NEVER widen both to a common root type or to `Throwable`; where a transport layer turns errors into responses, it matches each module's error type exhaustively, with no wildcard case.
- Compose an operation's checks as named steps that each return `Either`, sequenced in a for-comprehension so that the first `Left` short-circuits the rest; extract an inner sequence of checks into its own named method instead of nesting it inline.
- Convert an exception that a library throws for a failure the domain expects into `Either` or `Option` at the call boundary: capture the call with `Try`, match the most specific exception type in its `Failure` to a domain error, and rethrow every other failure, so that defects still propagate; NEVER catch `Exception` or `Throwable` wholesale, NEVER turn the `Try` into an `Either` with `toEither`, which leaves a `Throwable` on the left, and NEVER return `Try` from a service.

```scala
import java.nio.file.{Files, NoSuchFileException, Path}
import scala.util.{Failure, Success, Try}

enum ReportError:
  case Missing(path: Path)

def readReport(path: Path): Either[ReportError, String] =
  Try(Files.readString(path)) match
    case Success(report)                 => Right(report)
    case Failure(_: NoSuchFileException) => Left(ReportError.Missing(path))
    case Failure(defect)                 => throw defect
```

- At the outermost boundary of a processing loop, wrap one item's processing in `Try`, log a `Failure` with the item and move on; `Try` captures `NonFatal` failures only, so `InterruptedException` and fatal errors still propagate and end the loop.
- Acquire and release every closeable resource through `scala.util.Using`: `Using.resource(resource)(body)` where a failure should propagate, `Using(resource)(body)` where its `Try` is converted as above, and `Using.Manager` for several resources; NEVER write `try`/`finally`, and write no `try` keyword at all in new code.

## Tests

- Cover exactly one scenario per test: keep the several assertions one scenario needs in one test, split a test that exercises two independent behaviours, and add no test that overlaps with or repeats what another test already covers.
- Make tests deterministic: control time, randomness and identifiers through the injected clock, random source and id generator and assert on exact values; trigger batch or background work yourself by calling the job's body directly; NEVER sleep or poll while waiting for a result.

Adapted from VirtusLab's scala-skill, https://github.com/VirtusLab/scala-skill, licensed under Apache-2.0.
