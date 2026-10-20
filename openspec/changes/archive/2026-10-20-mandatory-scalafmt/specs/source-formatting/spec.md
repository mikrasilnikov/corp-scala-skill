# Spec Delta

## Purpose

Defines how an agent keeps a corporate Scala project formatted with scalafmt: `scalafmtAll` through the launcher at the start of every build, the scope of formatting left to the repository's own scalafmt settings, that configuration read-only to the agent, and what the agent does when formatting is unavailable or fails.

## ADDED Requirements

### Requirement: Every build starts with scalafmtAll
The agent SHALL make `scalafmtAll` the first sbt command of every launcher invocation that compiles or tests, in the same invocation as those tasks, and SHALL run `./sbt-launcher.sh -batch scalafmtAll` on its own when a change to the repository ends without a build. This applies to compiling and testing invocations that other capabilities specify as well.

#### Scenario: Compiling and testing after a change
- **WHEN** the agent has changed source files and runs the build
- **THEN** the invocation is `./sbt-launcher.sh -batch scalafmtAll compile test`, or a module-scoped equivalent such as `./sbt-launcher.sh -batch scalafmtAll core/compile core/test`, with `scalafmtAll` first

#### Scenario: Re-running after a fix
- **WHEN** a compile error or a failing test has been fixed and the agent runs the build again
- **THEN** the invocation is the same one, still beginning with `scalafmtAll`

#### Scenario: A change with no build
- **WHEN** the agent's change ends without compiling or testing, for example because the user asked for an edit and no build
- **THEN** the agent still runs `./sbt-launcher.sh -batch scalafmtAll` before reporting the change as done

#### Scenario: Read-only query
- **WHEN** the agent runs only a read-only sbt command, such as `show core/libraryDependencies` or `projects`
- **THEN** it does not prepend `scalafmtAll`, because nothing is built

#### Scenario: Invocations specified by other capabilities
- **WHEN** a scenario of another capability, such as `sbt-invocation`, gives a compiling or testing invocation without `scalafmtAll`, for example `./sbt-launcher.sh -batch compile` or `./sbt-launcher.sh -batch clean compile test`
- **THEN** the agent runs it with `scalafmtAll` first, such as `./sbt-launcher.sh -batch scalafmtAll compile` or `./sbt-launcher.sh -batch scalafmtAll clean compile test`

### Requirement: Formatting scope belongs to the repository
The agent SHALL run `scalafmtAll` unscoped from the repository root and SHALL NOT narrow or replace it. Which files it formats is decided by the repository's scalafmt settings, and files it reformats outside the agent's own edits SHALL be left formatted.

#### Scenario: Working in one module
- **WHEN** the agent scopes compile and test to a module
- **THEN** it still runs `scalafmtAll` without a module prefix, `./sbt-launcher.sh -batch scalafmtAll core/compile core/test`, and not `core/scalafmtAll` in its place

#### Scenario: Tempted to format only the changed files
- **WHEN** the agent could run `scalafmtOnly <files>`, `scalafmt`, or `Test/scalafmt` to touch fewer files
- **THEN** it runs `scalafmtAll` anyway; narrowing the scope is the repository owner's decision, made through `scalafmtFilter` or `.scalafmt.conf`

#### Scenario: Files outside the change are reformatted
- **WHEN** `scalafmtAll` reformats files the agent did not edit
- **THEN** the agent leaves them formatted, does not revert them, and names them when it reports its work

#### Scenario: Build definition files
- **WHEN** the agent has changed a `*.sbt` or `project/*.scala` file in a way the skill permits
- **THEN** it does not run `scalafmtSbt`; formatting build definition files is the repository's to enable, for example through `scalafmtSbtOnLoad` (sbt-scalafmt 2.6.2 and later)

### Requirement: Modules the root does not aggregate are formatted too
WHEN the root project does not aggregate a module whose sources the agent changed, the agent SHALL also run `<module>/scalafmtAll` immediately after the unscoped `scalafmtAll`, in addition to it and never in its place, unless the repository has turned the sbt-scalafmt plugin off for that module.

#### Scenario: Changed module not aggregated by the root
- **WHEN** the agent changed sources in a module the root project does not aggregate, for example a root defined as `project in file(".")` with `.dependsOn(core)` and no `.aggregate`, or one whose `.aggregate(...)` does not reach the module directly or through a module it aggregates
- **THEN** the invocation is `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll core/compile core/test`, or `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll` for a change with no build, and the agent does not finish with that module unformatted

#### Scenario: Module with the plugin turned off
- **WHEN** `<module>/scalafmtAll` fails with `No such setting/task` after the unscoped `scalafmtAll` succeeded in the same invocation, because the build turns the plugin off for that module (`disablePlugins`)
- **THEN** the agent runs the same invocation again without the module task, `scalafmtAll` still first, names the module as excluded from formatting by the repository in its report, does not stop, and does not turn the plugin on

### Requirement: Scalafmt configuration is read-only
The agent SHALL NOT create, modify, or delete `.scalafmt.conf`, and SHALL NOT change any scalafmt setting in the build definition, such as `scalafmtFilter`, `scalafmtOnCompile`, `scalafmtConfig`, `scalafmtFailOnErrors`, or `scalafmtSbtOnLoad`, whether in a file or with `set` on the launcher command line. It SHALL NOT turn the sbt-scalafmt plugin on or off for any project (`enablePlugins`, `disablePlugins`).

#### Scenario: The pinned version seems old or cannot be fetched
- **WHEN** the `version` pinned in `.scalafmt.conf` is older than the agent would choose, or the build cannot download it
- **THEN** the agent does not change it; a download failure is reported as an unresolved artifact, with area repositories

#### Scenario: A filter would shrink the diff
- **WHEN** setting `scalafmtFilter` would stop `scalafmtAll` from touching files outside the agent's change
- **THEN** the agent does not set it, neither in the build definition nor with `set`, and leaves the scope to the repository owner

#### Scenario: Style disagreement
- **WHEN** the agent would prefer a different style rule than `.scalafmt.conf` sets
- **THEN** it keeps the repository's configuration and lets scalafmt format the code under it

### Requirement: Formatting runs only through the launcher
The agent SHALL format only by running `scalafmtAll` through `./sbt-launcher.sh`, and SHALL NOT run the scalafmt command-line tool, `cs launch scalafmt` or any coursier or scala-cli equivalent, a Metals or IDE formatter, or an editor plugin.

#### Scenario: A scalafmt binary is on the PATH
- **WHEN** `scalafmt` is installed on the machine
- **THEN** the agent does not run it and formats through the launcher

#### Scenario: Formatting without the plugin
- **WHEN** the plugin is missing and a command-line tool or IDE could format the files instead
- **THEN** the agent does not use them; the missing plugin stops the task

### Requirement: Missing formatting setup stops the task
WHEN sbt rejects a correctly spelled `scalafmtAll` with `Not a valid key` or `Not a valid command`, or the plugin reports that `.scalafmt.conf` is missing, the agent SHALL stop and report with area `formatting`, the exact command, and the output verbatim. It SHALL NOT add the plugin, create the file, format by other means, or continue without formatting.

#### Scenario: Plugin missing
- **WHEN** the build loads and then prints `[error] Not a valid key: scalafmtAll` for a task spelled exactly `scalafmtAll`
- **THEN** the agent reports it as formatting not available and stops; it does not add `sbt-scalafmt` to `project/plugins.sbt` and does not run the task again

#### Scenario: Configuration missing
- **WHEN** `scalafmtAll` fails because the plugin cannot find `.scalafmt.conf`
- **THEN** the agent reports it and stops; it does not create the file

#### Scenario: Stop-and-report, not a dependency request
- **WHEN** the agent reports a missing scalafmt plugin
- **THEN** it uses the stop-and-report format with area `formatting`, not the dependency-request format, because the plugin is required of every repository rather than needed by the agent's task

#### Scenario: Continuing unformatted
- **WHEN** the agent could finish the code change and leave formatting to the user
- **THEN** it does not; the task stops at the report

### Requirement: Scalafmt source errors are the agent's work
WHEN `scalafmtAll` fails after the load marker because scalafmt cannot parse a source file, the agent SHALL treat it as a build failure, SHALL NOT drop `scalafmtAll` from the invocation to get past it, and for a file it changed SHALL fix the file and run the same invocation again.

#### Scenario: Parse error in a file the agent changed
- **WHEN** scalafmt reports a parse error at a line of a file the agent edited or created
- **THEN** the agent fixes the syntax and runs `./sbt-launcher.sh -batch scalafmtAll compile test` again

#### Scenario: Parse error in a file the agent did not change
- **WHEN** scalafmt reports a parse error in a file the agent never touched
- **THEN** the agent tells the user which file and which error, does not run the build without `scalafmtAll` to get past it, and leaves the decision to fix or exclude that file to the user
