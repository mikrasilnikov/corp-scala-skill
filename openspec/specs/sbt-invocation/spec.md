# sbt-invocation Specification

## Purpose

Defines how an agent runs the sbt build in a corporate Scala project: the launcher as the only entry point, the working directory, non-interactive invocation, which command-line options are permitted, and how infrastructure failures are told apart from build failures and handled.

## Requirements

### Requirement: The launcher is the only way to run sbt
The agent SHALL run sbt exclusively through `./sbt-sber.sh` in the repository root. It SHALL NOT start sbt by any other means, including a globally installed `sbt`, `sbtn` or `sbt --client`, coursier, bloop, a Metals or BSP connection, or a direct `java -jar sbt-launch.jar` invocation.

#### Scenario: Compiling the project
- **WHEN** the agent needs to compile the project
- **THEN** it runs `./sbt-sber.sh -batch compile` from the repository root

#### Scenario: Another sbt is available on the machine
- **WHEN** an `sbt` binary is on the `PATH` or an IDE build server is already running for the project
- **THEN** the agent still runs every build task through `./sbt-sber.sh` and does not use the other tool as a substitute

#### Scenario: Several tasks are needed
- **WHEN** the agent needs more than one sbt task, such as `clean`, `compile`, and `test`
- **THEN** it passes them in order to a single launcher invocation, `./sbt-sber.sh -batch clean compile test`, rather than starting sbt once per task

### Requirement: The launcher runs from the repository root
The agent SHALL invoke the launcher with the repository root as the working directory and SHALL scope work to a module through sbt's `module/task` syntax rather than by changing directory.

#### Scenario: Working inside a module directory
- **WHEN** the agent's current directory is a module or source directory below the root
- **THEN** it changes to the repository root first and runs, for example, `./sbt-sber.sh -batch core/compile`

#### Scenario: Launcher absent from the root
- **WHEN** `./sbt-sber.sh` does not exist at the repository root
- **THEN** the agent stops, reports to the user that the repository is not set up for agent builds, and does not search other directories, fetch a launcher, or fall back to any other sbt

### Requirement: Every invocation is non-interactive
The agent SHALL pass `-batch` on every launcher invocation and SHALL always supply at least one sbt command, so that sbt never waits for terminal input.

#### Scenario: Running the test suite
- **WHEN** the agent runs tests
- **THEN** the invocation is `./sbt-sber.sh -batch test` or a scoped equivalent, with `-batch` present

#### Scenario: No command to run
- **WHEN** the agent has no sbt command to execute
- **THEN** it does not invoke the launcher, because a bare invocation opens the interactive sbt shell

#### Scenario: Task that reads standard input
- **WHEN** the agent runs an application with `run` or opens a REPL with `console`
- **THEN** `-batch` is present so standard input is closed and the process cannot block waiting for a terminal

### Requirement: Only sbt commands and output-only settings go on the command line
The agent SHALL place only sbt tasks and commands, and settings whose sole effect is on sbt's output (colour, log level, stack-trace depth), on the launcher command line. It SHALL NOT pass runner flags or `set` commands that change the JVM, the sbt or Scala version, repositories, directories, or compiler options.

#### Scenario: Plain output for parsing
- **WHEN** the agent wants output without ANSI escape codes
- **THEN** it adds `-no-colors` to the invocation

#### Scenario: Diagnosing a failing test
- **WHEN** the agent needs a full stack trace or verbose logging for one run
- **THEN** it adds `-trace <n>` or `--debug` to that invocation only

#### Scenario: Running a single test suite
- **WHEN** the agent runs one suite
- **THEN** it passes the sbt command as a single quoted argument, `./sbt-sber.sh -batch "testOnly com.acme.FooSpec"`, so that sbt receives one command rather than two

#### Scenario: Tempted to change compilation from the command line
- **WHEN** the agent would pass `-S<scalac option>`, `set scalacOptions ...`, `-D<key>=<value>`, `-J<jvm option>`, or `-sbt-version`
- **THEN** it does not, because these change how the build runs rather than what it prints

### Requirement: Warnings remain visible
The agent SHALL NOT use `--error` or any other setting that suppresses `[warn]` output, and SHALL inspect both warnings and errors from the complete output of a build rather than from a filtered or truncated view.

#### Scenario: Reducing or increasing log output
- **WHEN** the agent changes sbt's log level
- **THEN** it only increases it with `--debug`, never raises the threshold to `--error`

#### Scenario: Output too long to read inline
- **WHEN** a build prints more output than the agent can view directly
- **THEN** the agent captures the complete output to a file and inspects every `[warn]` and `[error]` line before deciding what to do

### Requirement: Launcher and infrastructure failures stop the task
WHEN the launcher fails before sbt has loaded the build, and the failure is not one of the agent's own invocation mistakes, the agent SHALL stop the current task and report to the user the exact command it ran and the error output verbatim. It SHALL NOT retry with another tool, different runner flags, changed configuration, or a modified launcher.

#### Scenario: Launcher precondition fails
- **WHEN** the launcher exits with its own error before sbt prints its loading messages, for example because a network or credential precondition is unmet
- **THEN** the agent reports the command and the launcher's message to the user and does not attempt any workaround

#### Scenario: sbt launcher jar cannot be fetched
- **WHEN** sbt-extras fails to download or verify `sbt-launch.jar`
- **THEN** the agent reports the failure and does not pass `-sbt-launch-repo`, `-sbt-jar`, `-sbt-version`, or `-offline` to get past it

#### Scenario: Launcher is not executable
- **WHEN** running `./sbt-sber.sh` fails with a permission error
- **THEN** the agent reports it to the user and does not change the file's permissions or run it through `sh` or `bash` instead

### Requirement: The agent corrects its own invocation mistakes
WHEN a launcher run fails because the agent's own command did not follow this capability's rules (it ran from a directory other than the repository root, passed a permitted option in a malformed form, or mistyped, misquoted, or misscoped an sbt command), the agent SHALL correct the command so that it follows every rule in this capability and run it again.

#### Scenario: Launcher run outside the repository root
- **WHEN** the shell cannot find `./sbt-sber.sh` because the agent was not at the repository root, or sbt-extras reports that the current directory does not appear to be an sbt project
- **THEN** the agent changes to the repository root and runs the same sbt commands there, and does not pass `-sbt-create`

#### Scenario: Malformed permitted option
- **WHEN** sbt-extras aborts because a permitted option was malformed, such as `-trace` without a non-negative integer
- **THEN** the agent corrects the option to a permitted form and runs the command again

#### Scenario: Mistyped sbt command
- **WHEN** sbt reports that a command the agent passed is not valid, for example because a `testOnly` argument was not quoted and arrived as a separate command
- **THEN** the agent corrects the spelling, quoting, or module scope and runs the command again

### Requirement: Corrections stay within the invocation rules
A correction SHALL only bring the command into line with this capability's rules: the agent SHALL NOT add a forbidden flag, `set` command, or environment variable, change configuration, or use another tool. If the corrected command still fails before sbt has loaded the build, the agent SHALL stop and report as for any launcher or infrastructure failure.

#### Scenario: The corrected command still fails
- **WHEN** a corrected command that follows every rule in this capability fails before sbt has loaded the build
- **THEN** the agent stops and reports the failure with the corrected command and its output

#### Scenario: A workaround is not a correction
- **WHEN** getting past a failure would need a forbidden flag, a configuration change, another tool, or a changed launcher
- **THEN** the failure is not an invocation mistake, and the agent stops and reports it

### Requirement: Build failures are the agent's work
WHEN sbt has loaded the build, which is visible from its `welcome to sbt` or `loading settings for project` messages, and a task then fails because of compilation errors or failing tests, the agent SHALL treat the failure as ordinary development work: read the diagnostics, change the code, and re-run through the launcher.

#### Scenario: Compilation error
- **WHEN** `compile` fails with `[error]` lines pointing at source files
- **THEN** the agent fixes the code and re-runs `./sbt-sber.sh -batch compile`

#### Scenario: Failing test
- **WHEN** `test` reports failed test cases
- **THEN** the agent corrects the code or the test and re-runs the relevant suite through the launcher

### Requirement: Resolution and memory failures are reported, not worked around
WHEN a build fails because an artifact cannot be resolved or the JVM runs out of memory, the agent SHALL stop and report the failure with the artifact coordinates or the memory error, because repositories and memory belong to the launcher and are not the agent's to change.

#### Scenario: Dependency not found
- **WHEN** sbt reports that an artifact could not be resolved or downloaded
- **THEN** the agent reports the coordinates to the user and does not add resolvers, change dependency versions, or alter repository configuration

#### Scenario: Out of memory
- **WHEN** the build fails with `java.lang.OutOfMemoryError` or a GC overhead error
- **THEN** the agent reports it and does not pass `-mem`, `-J-Xmx`, set `SBT_OPTS` or `JVM_OPTS`, or edit `.jvmopts`

### Requirement: Slow invocations are not failures
The agent SHALL give launcher invocations enough time to complete, including the first run in a session when sbt-extras fetches the launcher jar and sbt resolves dependencies, and SHALL NOT treat a slow invocation as a hang or a failure.

#### Scenario: Cold start
- **WHEN** the agent runs the launcher for the first time in a session
- **THEN** it runs the command with a generous timeout or in the background and waits for the process to exit before judging the result
