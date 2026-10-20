# Spec Delta

## MODIFIED Requirements

### Requirement: The agent corrects its own invocation mistakes
WHEN a launcher run fails because the agent's own command did not follow this capability's rules (it ran from a directory other than the repository root, passed a permitted option in a malformed form, or mistyped, misquoted, or misscoped an sbt command), the agent SHALL correct the command so that it follows every rule in this capability and run it again.

#### Scenario: Launcher run outside the repository root
- **WHEN** the shell cannot find `./sbt-launcher.sh` because the agent was not at the repository root, or sbt-extras reports that the current directory does not appear to be an sbt project
- **THEN** the agent changes to the repository root and runs the same sbt commands there, and does not pass `-sbt-create`

#### Scenario: Malformed permitted option
- **WHEN** sbt-extras aborts because a permitted option was malformed, such as `-trace` without a non-negative integer
- **THEN** the agent corrects the option to a permitted form and runs the command again

#### Scenario: Mistyped sbt command
- **WHEN** sbt reports that a command the agent passed is not valid, for example because a `testOnly` argument was not quoted and arrived as a separate command
- **THEN** the agent corrects the spelling, quoting, or module scope and runs the command again

#### Scenario: A correctly spelled scalafmt task is rejected
- **WHEN** sbt reports `Not a valid key` or `Not a valid command` for `scalafmtAll`, and the agent confirms the task is spelled exactly `scalafmtAll`
- **THEN** this is not an invocation mistake; the agent does not respell or retry it and handles the failure as formatting not available, as the `source-formatting` capability requires
