# Spec Delta

## Purpose

Defines the rule that an agent never changes a corporate Scala project's dependency set: what counts as a dependency, the side channels that are closed as well, how the agent works with dependencies already declared, and how it asks the user when new functionality needs one.

## ADDED Requirements

### Requirement: The dependency set is immutable to the agent
The agent SHALL NOT add, remove, upgrade, downgrade, re-scope, exclude, or override any project dependency, in any file and by any mechanism. Dependency changes are made by people after the agent asks for them.

#### Scenario: Code needs a library the module does not declare
- **WHEN** the code the agent is writing would require a library that is not declared for the module
- **THEN** the agent does not declare it and instead asks the user as described below

#### Scenario: A newer version would help
- **WHEN** a bug or missing feature would be resolved by a different version of an existing dependency
- **THEN** the agent does not change the version and asks the user

#### Scenario: A dependency exists in another configuration
- **WHEN** a library declared with `% Test` is needed in main code, or the reverse
- **THEN** the agent does not change the configuration and asks the user

### Requirement: Everything that shapes the classpath counts as a dependency
The agent SHALL treat every one of the following as a dependency under this rule: `libraryDependencies`, `dependencyOverrides`, `excludeDependencies`, `addSbtPlugin` and `addCompilerPlugin`, `scalaVersion` and `crossScalaVersions`, `sbt.version` in `project/build.properties`, unmanaged jars under `lib/` or `unmanagedJars`, and inter-module `dependsOn`, wherever declared: `*.sbt`, `project/*.sbt`, or `project/*.scala`.

#### Scenario: Task needs an sbt plugin
- **WHEN** a task the agent wants to run requires a plugin not present in `project/plugins.sbt`
- **THEN** the agent does not add the plugin and asks the user

#### Scenario: Versions are centralised in a Scala build file
- **WHEN** dependencies are declared in a file such as `project/Dependencies.scala`
- **THEN** the agent treats those declarations exactly as it treats `libraryDependencies` in `build.sbt` and does not edit them

#### Scenario: Scala or sbt version
- **WHEN** a different `scalaVersion` or `sbt.version` would make code compile or a feature available
- **THEN** the agent does not change either and asks the user

#### Scenario: Jar dropped into the project
- **WHEN** a jar could be placed under `lib/` to make a class available
- **THEN** the agent does not add it

#### Scenario: Wiring modules together
- **WHEN** code in one module needs code from another module that it does not depend on
- **THEN** the agent does not add a `dependsOn` and asks the user

### Requirement: No side channels for artifacts
The agent SHALL NOT bring artifacts onto the classpath by means outside the build definition: `set libraryDependencies ...` or similar on the launcher command line, `coursier` or `cs fetch`, `scala-cli --dep`, Ammonite `import $ivy`, downloading jars manually, or copying jars from local caches into the project.

#### Scenario: Setting a dependency from the command line
- **WHEN** the agent could run `./sbt-launcher.sh -batch "set libraryDependencies += ..." compile`
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
- **THEN** it reads the build definition files or runs a read-only sbt command such as `./sbt-launcher.sh -batch "show core/libraryDependencies"`

### Requirement: Asking for a dependency
WHEN the agent determines that its task cannot be completed without a dependency change, it SHALL stop and ask the user, stating the proposed coordinates (organisation, artifact, version if known), the module, the configuration such as `Compile` or `Test`, why it is needed, and which declared alternatives it ruled out. It SHALL NOT add the dependency provisionally while waiting.

#### Scenario: Structured request
- **WHEN** the agent asks for a dependency
- **THEN** its message names the coordinates, module, configuration, reason, and the declared alternatives it considered, and the task pauses there

#### Scenario: Wanting to try before asking
- **WHEN** the agent wants to confirm that a library would solve the problem
- **THEN** it does not add the library temporarily to test it

#### Scenario: User has added the dependency
- **WHEN** the user confirms the dependency has been added
- **THEN** the agent continues by running the build through the launcher, which picks up the changed build definition on its next start
