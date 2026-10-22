# Spec Delta

## Purpose

Defines how an agent represents and propagates failures under the `virtuslab-scala-style` rules file: typed, sealed, per-module errors returned as values for everything a caller can act on, and exceptions only for what ends the current unit of work.

## ADDED Requirements

### Requirement: Recoverable failures are returned as Either
WHEN a failure is one the caller can act on, such as invalid input, a missing entity, a conflict or a refused permission, the method SHALL return `Either` with a typed error, and SHALL NOT throw, `require`, or hide the failure behind `try`/`catch`.

#### Scenario: A missing entity
- **WHEN** a lookup finds nothing
- **THEN** the method returns a `Left` whose case says what was not found, and the caller decides how to respond

#### Scenario: Invalid input
- **WHEN** input fails validation
- **THEN** the method returns a `Left` carrying the offending value, not an exception

#### Scenario: An exception for a recoverable failure
- **WHEN** existing code throws for a missing entity
- **THEN** new code the agent writes returns `Either` instead and does not extend the throwing pattern

### Requirement: Unrecoverable failures throw
WHEN a failure is one the caller cannot handle, such as a bug, a lost connection, corrupted data or an exhausted resource, the agent SHALL let an exception end the current request, message or job, and SHALL NOT encode it as an `Either` case.

#### Scenario: A lost database connection
- **WHEN** the database cannot be reached
- **THEN** the exception propagates and ends the current unit of work

#### Scenario: A defect
- **WHEN** an invariant the code relies on turns out false
- **THEN** the code fails rather than returning a `Left` the caller cannot act on

### Requirement: Errors are typed and sealed per module
Each module SHALL define its own sealed error type, an enum or sealed trait, whose cases carry the failure's data, and its service methods SHALL return `Either` of that type. There SHALL be no stringly-typed errors and no open, application-wide error class that modules extend.

#### Scenario: A module's error type
- **WHEN** the user module can fail with a missing user or a taken login
- **THEN** it declares a sealed type with a case for each, carrying the id or the login

#### Scenario: A new case is added
- **WHEN** a case is added to a module's error type
- **THEN** every match that does not handle it fails to compile

#### Scenario: A string error
- **WHEN** code would return `Left("user not found")`
- **THEN** the agent returns a typed case instead

### Requirement: Crossing a module wraps the callee's error
WHEN a caller combines results from another module, it SHALL wrap the callee's error in a case of its own error type, and SHALL NOT widen both to a common root type or to `Throwable`.

#### Scenario: Checkout calls payment
- **WHEN** the checkout module calls the payment module and may fail with its error
- **THEN** the checkout error type has a case holding the payment error

#### Scenario: Mapping at the edge
- **WHEN** a transport layer turns errors into responses
- **THEN** it matches each module's error type exhaustively, with no wildcard case

### Requirement: Validations compose as named steps
WHEN an operation performs several checks before acting, each check SHALL be a named step returning `Either`, composed in sequence, and an inner composition SHALL be extracted into its own method rather than nested inline.

#### Scenario: Registering a user
- **WHEN** registration validates input, checks uniqueness and then creates the user
- **THEN** each is a named step and the operation composes them in that order, short-circuiting on the first `Left`

#### Scenario: A nested composition
- **WHEN** one step is itself a sequence of checks
- **THEN** that sequence becomes its own method with a name

### Requirement: Expected exceptions are converted at the boundary
WHEN a library throws for a failure that is expected in the domain, the agent SHALL convert the exception to `Either` or `Option` at the call boundary, catching the most specific exception type and mapping it to a domain error, and SHALL NOT catch `Exception` or `Throwable` wholesale.

#### Scenario: A not-found exception from a client library
- **WHEN** a storage client throws its not-found exception for a missing key
- **THEN** the boundary catches exactly that exception type and returns `None` or a `Left` with the domain's not-found case

#### Scenario: A broad catch
- **WHEN** code would catch `Exception` around a library call
- **THEN** the agent narrows it to the expected exception types, so defects still propagate

### Requirement: Bare try/catch only at a loop's outermost boundary
Bare `try`/`catch` SHALL be used only for unexpected defects at the outermost boundary of a processing loop, catching `NonFatal` so that one failing item is logged and the loop continues. `InterruptedException` and fatal errors SHALL propagate.

#### Scenario: A worker loop
- **WHEN** a loop processes items one by one
- **THEN** the catch around one item's processing catches `NonFatal`, logs the failure, and the loop moves on

#### Scenario: Catching Throwable
- **WHEN** code would catch `Throwable` to keep a loop alive
- **THEN** the agent narrows it to `NonFatal`, so interruption and fatal errors still end the loop
