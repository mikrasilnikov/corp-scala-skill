# scala-code-organisation Specification

## Purpose

Defines where an agent puts Scala declarations when it writes under the `virtuslab-scala-style` rules file: which package and file a declaration belongs to, how visible it is, and how packages are named, so that a reader finds a type by its name and nothing is more visible than its callers need.

## Requirements

### Requirement: Package, file and visibility are decided before a file is created
Before creating or moving a `.scala` file, the agent SHALL decide which existing package it belongs to, which visibility each top-level declaration gets, and that the filename matches the primary type or concept it holds.

#### Scenario: A new class
- **WHEN** the agent needs a new service class
- **THEN** it places the file in the concept package the class serves, names the file after the class, and sets each top-level declaration's visibility before writing the body

#### Scenario: A moved file
- **WHEN** the agent moves a file to another package
- **THEN** it makes the same three decisions again for the new location

### Requirement: Top-level declarations get the narrowest visibility their call sites allow
Every top-level class, trait, enum and object SHALL be classified: public only when it is intentionally part of the API, otherwise private to its concept package, or private to the root package when callers in other modules of the same root need it. The agent SHALL decide by reading the actual call sites, starting narrow and widening only when a caller needs access.

#### Scenario: A parser used in one package
- **WHEN** a class is called only from its own package
- **THEN** it is private to that package

#### Scenario: An interface used across modules
- **WHEN** a trait is implemented in one module and called from another under the same root package
- **THEN** it is private to the root package, not public

#### Scenario: Widening on evidence
- **WHEN** a new caller outside the current visibility needs a declaration
- **THEN** the agent widens the visibility one step, to the narrowest that admits the caller

### Requirement: A file is shared only by the types one constructor builds
Whether a type gets its own file SHALL be decided by where it is constructed, not where it is referenced: a service may share its file with return types and exceptions it alone constructs; a type constructed in several places gets its own file named after itself; a sealed hierarchy stays in one file; a trait may share a file with its single canonical implementation; otherwise there is one top-level type per file.

#### Scenario: A result type built by one service
- **WHEN** only one service constructs a result type
- **THEN** the type may live in that service's file

#### Scenario: A type built in several places
- **WHEN** a type is constructed by a service and also by a codec or a second service
- **THEN** it gets its own file named after it

#### Scenario: A sealed hierarchy
- **WHEN** a sealed trait or enum has several cases
- **THEN** all cases stay in the file that declares the hierarchy

#### Scenario: A trait and its canonical implementation
- **WHEN** a trait has exactly one implementation, named as the default one
- **THEN** the two may share a file

### Requirement: No grab-bag files
The agent SHALL NOT create files whose only common theme is the kind of declaration they hold, such as `Types.scala`, `Models.scala` or `Helpers.scala`, and SHALL NOT grow an existing one; it SHALL place each new declaration by the construction-site rule instead.

#### Scenario: A new type for an existing grab bag
- **WHEN** the repository has a `Models.scala` and the agent adds a type
- **THEN** the new type goes in its own file or with its sole constructor, and the existing file is left as it is unless the user asks for a reorganisation

#### Scenario: The "Types" suffix
- **WHEN** the agent is about to name a file `<Something>Types.scala`
- **THEN** it splits the contents by the construction-site rule instead

### Requirement: Packages are named after concepts
Package names SHALL name a concept, such as `events`, `git` or `billing`, not a mechanism such as `util`, `io`, `core` or `common`. At most one small `util` leaf per root package SHALL exist, and it SHALL be split once it grows past about five files.

#### Scenario: A new area of the domain
- **WHEN** the agent adds the first classes for invoicing
- **THEN** they go in a package named `billing` or `invoicing`, not `common`

#### Scenario: The util leaf grows
- **WHEN** the root's `util` package reaches its sixth file
- **THEN** the agent moves the files into concept packages

### Requirement: No single-file packages
The agent SHALL NOT create a package that holds a single file; it SHALL inline the declaration into the parent package or fold it into a sibling package.

#### Scenario: One class in a new package
- **WHEN** a new package would contain only one file
- **THEN** the file goes into the parent or a sibling package instead

### Requirement: A sub-package never shadows a top-level definition
A sub-package name SHALL NOT equal a top-level `def` or `val` in the same root package, which Scala 3 rejects; the agent SHALL sink the package one level deeper instead.

#### Scenario: A name collision
- **WHEN** the root package has a top-level `def git` and a `git` package is wanted
- **THEN** the agent creates `tools.git` or another deeper path

### Requirement: A type's companion holds its constructors and instances
Smart constructors, the accessors of an opaque type, and the type class instances for a type (givens, or implicits on Scala 2.13) SHALL live in that type's companion object. Instances for types that have no companion in the codebase SHALL be gathered in one object in the concept package that needs them.

#### Scenario: A new domain type
- **WHEN** the agent declares an opaque type with an invariant
- **THEN** its companion holds the smart constructor and the value accessor

#### Scenario: An instance for a domain type
- **WHEN** a codec or ordering is needed for a domain type
- **THEN** it is declared in the type's companion, where implicit resolution finds it without an import

#### Scenario: An instance for a third-party type
- **WHEN** an instance is needed for a type from a library
- **THEN** it goes into a single instances object in the concept package, not beside each call site
