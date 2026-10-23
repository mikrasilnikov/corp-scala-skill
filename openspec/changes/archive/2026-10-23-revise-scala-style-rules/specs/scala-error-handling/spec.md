# Spec Delta

## ADDED Requirements

### Requirement: Defects at a loop boundary are captured with Try
At the outermost boundary of a processing loop, the agent SHALL wrap one item's processing in `Try`, log a failure and move on, so that one failing item does not end the loop. Since `Try` captures `NonFatal` failures only, `InterruptedException` and fatal errors SHALL propagate and end the loop.

#### Scenario: A worker loop
- **WHEN** a loop processes items one by one
- **THEN** each item's processing is wrapped in `Try`, a `Failure` is logged with the item, and the loop moves on

#### Scenario: Interruption
- **WHEN** the thread is interrupted while an item is processed
- **THEN** the `InterruptedException` is not captured and ends the loop

#### Scenario: A catch that would keep the loop alive
- **WHEN** code would catch `Throwable` around an item to keep a loop running
- **THEN** the agent uses `Try` instead, which leaves fatal errors to propagate

### Requirement: Resources are released through Using
Every closeable resource SHALL be acquired and released through `scala.util.Using`: `Using.resource` where a failure should propagate, `Using` where its `Try` is converted to a typed result, and `Using.Manager` for several resources. The agent SHALL NOT write `try`/`finally`, and SHALL NOT write the `try` keyword at all in new code.

#### Scenario: Reading a file
- **WHEN** code opens a source, reads it and must close it whatever happens
- **THEN** the agent writes `Using.resource(source)(body)` and no `try`/`finally`

#### Scenario: A resource whose failure is expected
- **WHEN** the body's failure is one the domain expects and converts to a typed error
- **THEN** the agent writes `Using(resource)(body)` and matches the resulting `Try` as the boundary-conversion rule describes

#### Scenario: Several resources
- **WHEN** an operation needs several resources at once
- **THEN** the agent acquires them through `Using.Manager`, which releases them in reverse order

#### Scenario: Existing try/finally
- **WHEN** existing code uses `try`/`finally` to close a resource
- **THEN** new code the agent writes uses `Using` and does not extend the pattern

## MODIFIED Requirements

### Requirement: Expected exceptions are converted at the boundary
WHEN a library throws for a failure that is expected in the domain, the agent SHALL convert the exception to `Either` or `Option` at the call boundary by capturing the call with `Try`, matching the most specific exception type to a domain error, and rethrowing every other failure. The agent SHALL NOT catch `Exception` or `Throwable` wholesale, SHALL NOT convert a `Try` to an `Either` with `Throwable` on the left, and SHALL NOT return `Try` from a service.

#### Scenario: A not-found exception from a client library
- **WHEN** a storage client throws its not-found exception for a missing key
- **THEN** the boundary matches exactly that exception type in the `Failure`, returns `None` or a `Left` with the domain's not-found case, and rethrows any other failure

#### Scenario: A broad catch
- **WHEN** code would catch `Exception` around a library call
- **THEN** the agent narrows it to the expected exception types, so defects still propagate

#### Scenario: Try as a result type
- **WHEN** a service method would return `Try`
- **THEN** the agent converts the `Try` at the boundary and returns `Either` with the module's error type

#### Scenario: toEither with Throwable
- **WHEN** code would write `Try(call).toEither`
- **THEN** the agent matches the specific exception type instead, so the left side is a typed domain error

## REMOVED Requirements

### Requirement: Bare try/catch only at a loop's outermost boundary
**Reason**: The `try` keyword is no longer written in new code; `Try` captures `NonFatal` failures and leaves fatal errors to propagate, which is what this requirement asked a `catch NonFatal` to do.
**Migration**: Follow the added requirement "Defects at a loop boundary are captured with Try"; existing `try`/`catch` at loop boundaries is left as it is until that code is touched.
