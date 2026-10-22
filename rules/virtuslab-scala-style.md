# Scala code style and domain modelling

Apply these rules to all Scala code you write or change in this repository, production code and tests alike, on Scala 3 and on Scala 2.13.

Where a syntax rule below conflicts with a convention the repository's code already follows or its formatter enforces, follow the repository and keep the formatter's output; do not undo its layout. The rules on file structure, modelling, errors and tests apply regardless of the repository's existing habits.

Each rule names the Scala 3 construct; on Scala 2.13 use the equivalent that the last rule under "Syntax and function shape" gives.

## Syntax and function shape

- In Scala 3, write every definition and control structure in the indentation-based syntax and NEVER introduce braces; on Scala 2.13 use braces, since the language has no other syntax, and follow the repository's layout.
- ALWAYS give every public `def`, `val` and `given` (an `implicit` definition on Scala 2.13) an explicit type, even when the compiler could infer it, so that a refactor that changes the type is reported instead of silently changing the signature; private and local values may stay inferred.
- Make each function do one thing: either a single logical operation or a short orchestration of calls to named steps with no inline logic of its own, never both mixed; split responsibilities between entities named after what they do, not where they are used, so a function that parses input and stores the result becomes a parser, a writer and, if needed, an orchestration of the two.
- When a `val` is used exactly once, on the next line, only to feed the next expression, chain instead with `pipe`, or with `tap` for a side effect such as logging that passes the value on unchanged, both from `scala.util.chaining`: `parse(raw).pipe(validate)`; keep the named `val` when its name documents a non-obvious step, such as `normalisedEmail`.
- When a non-Unit result is intentionally unused, discard it explicitly, in the form the repository already uses (such as `val _ = Files.deleteIfExists(path)`), rather than leaving a bare non-Unit statement; when the result is needed further down, bind it to a named `val`.
- NEVER load a collection of unbounded size into memory: process it incrementally, as a stream or an iterator, or page through it; a plain in-memory collection is fine only when its size has a known small bound, such as a configuration list.
- Code you write MUST compile with zero warnings under the repository's configured compiler options: fix the cause in code; suppress a warning only in generated code or for a third-party issue you cannot fix, as narrowly as possible with `@nowarn` and a comment naming the generator or the issue; NEVER change compiler options to silence a warning.
- On Scala 2.13, and in modules cross-built for Scala 2.13 and 3, replace the four Scala 3 constructs these rules name with their equivalents and apply every other rule unchanged: braces for braceless syntax; a `sealed trait` with case objects and case classes in its companion for an `enum`; a value class or a tagged type for an opaque type; `implicit` definitions and implicit classes for givens and extension methods.

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
- Declare a `var` only inside a method body, in the smallest scope that needs it, and NEVER as a class or object field; model state that outlives a call as an immutable value the operation returns.
- Use immutable collections in production code; mutable collections belong only in test fakes.
- Pass the clock, the random source and the id generator in as parameters or constructor arguments and NEVER obtain them inside a method, so that results are reproducible and a test can assert on exact values.
- Make invalid data unrepresentable: an enum or sealed hierarchy for a closed set of values instead of a `String`, `Option` for absence and NEVER `null`, and a validated constructor as the only way to obtain a value that has an invariant.
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
- Give each module its own configuration case class of typed values, composed into one top-level configuration; wrap every secret in a type whose string form is masked, so that logging the configuration never prints it; validate configuration when it is loaded and fail at startup with a message naming the invalid setting, not at its first use, which is the one place a constructor may throw.

## Error handling

- Return `Either` with a typed error for every failure the caller can act on, such as invalid input, a missing entity, a conflict or a refused permission; NEVER throw for one, directly or through a precondition check, and never hide it behind `try`/`catch`, even where existing code does.
- Let an exception end the current request, message or job for a failure the caller cannot handle, such as a bug, a lost connection, corrupted data or an exhausted resource; NEVER encode one as an `Either` case that no caller can act on.
- Give each module its own sealed error type, an enum or a sealed trait, whose cases carry the failure's data, such as `UserNotFound(id)` or `LoginTaken(login)`, and return `Either` of it from the module's services; NEVER return string errors such as `Left("user not found")`, and NEVER define an open application-wide error class for modules to extend, so that the compiler flags every match that misses a newly added case.
- When a caller combines results from another module, wrap the callee's error in a case of the caller's own error type, such as `CheckoutError.Payment(error: PaymentError)`, and NEVER widen both to a common root type or to `Throwable`; where a transport layer turns errors into responses, it matches each module's error type exhaustively, with no wildcard case.
- Compose an operation's checks as named steps that each return `Either`, sequenced in a for-comprehension so that the first `Left` short-circuits the rest; extract an inner sequence of checks into its own named method instead of nesting it inline.
- Convert an exception that a library throws for a failure the domain expects into `Either` or `Option` at the call boundary: catch the most specific exception type, map it to a domain error, and NEVER catch `Exception` or `Throwable` wholesale, so that defects still propagate.

```scala
import java.nio.file.{Files, NoSuchFileException, Path}

enum ReportError:
  case Missing(path: Path)

def readReport(path: Path): Either[ReportError, String] =
  try Right(Files.readString(path))
  catch case _: NoSuchFileException => Left(ReportError.Missing(path))
```

- Apart from that conversion, use `try`/`catch` only for unexpected defects at the outermost boundary of a processing loop: catch `NonFatal` around one item's processing, log the failure and move on, so that `InterruptedException` and fatal errors still propagate and end the loop.

## Tests

- Cover exactly one scenario per test: keep the several assertions one scenario needs in one test, split a test that exercises two independent behaviours, and add no test that overlaps with or repeats what another test already covers.
- Make tests deterministic: control time, randomness and identifiers through the injected clock, random source and id generator and assert on exact values; trigger batch or background work yourself by calling the job's body directly; NEVER sleep or poll while waiting for a result.

Adapted from VirtusLab's scala-skill, https://github.com/VirtusLab/scala-skill, licensed under Apache-2.0.
