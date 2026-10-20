# Spec Delta

## MODIFIED Requirements

### Requirement: No side channels for artifacts
The agent SHALL NOT bring artifacts onto the classpath by means outside the build definition: `set libraryDependencies ...` or similar on the launcher command line, `coursier` or `cs fetch`, `scala-cli --dep`, Ammonite `import $ivy`, downloading jars manually, or copying jars from local caches into the project.

#### Scenario: Setting a dependency from the command line
- **WHEN** the agent could run `./sbt-sber.sh -batch "set libraryDependencies += ..." compile`
- **THEN** it does not

#### Scenario: Fetching an artifact with another tool
- **WHEN** coursier, scala-cli, or Ammonite could fetch an artifact the build does not declare
- **THEN** the agent does not use them to obtain it

#### Scenario: Copying from a local cache
- **WHEN** a needed jar exists under `~/.ivy2`, `~/.cache/coursier`, or another local cache
- **THEN** the agent does not copy or reference it from the project

### Requirement: Work with what the module already declares
The agent SHALL implement with the dependencies already declared for the module it is working in, including those it inherits through `dependsOn`, and SHALL check the module's declared dependencies before concluding that one is missing.

#### Scenario: Writing tests
- **WHEN** the agent writes tests
- **THEN** it uses the test framework and matchers the module already declares rather than asking for a different one

#### Scenario: A declared library covers the need
- **WHEN** a declared library can do what the agent needs, even if a different library would be more familiar
- **THEN** the agent uses the declared library

#### Scenario: Discovering declared dependencies
- **WHEN** the agent needs to know what a module declares
- **THEN** it reads the build definition files or runs a read-only sbt command such as `./sbt-sber.sh -batch "show core/libraryDependencies"`
