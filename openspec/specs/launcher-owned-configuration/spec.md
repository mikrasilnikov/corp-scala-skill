# launcher-owned-configuration Specification

## Purpose

Defines the build configuration an agent must treat as read-only because the launcher owns it: authentication, repositories, and memory, the files and environment variables that carry them, and the launcher script itself as a black box.

## Requirements

### Requirement: Authentication, repositories, and memory belong to the launcher
The agent SHALL treat the build's authentication, repository, and memory configuration as owned by `./sbt-sber.sh` and SHALL NOT create, modify, delete, select, or override any of it by any means. When a failure traces back to one of these areas, the agent SHALL report it to the user rather than change it.

#### Scenario: Artifact cannot be resolved
- **WHEN** a build fails because an artifact is not available from the configured repositories
- **THEN** the agent reports the coordinates and the error and does not add resolvers or credentials, edit any repository configuration, or point sbt at another repository

#### Scenario: Build runs out of memory
- **WHEN** a build fails with an out-of-memory error
- **THEN** the agent reports it and does not raise heap or change any JVM memory setting

#### Scenario: Java version mismatch
- **WHEN** the launcher or sbt reports an unsuitable or missing Java installation
- **THEN** the agent reports it and does not set `JAVA_HOME`, pass `-java-home`, or install a JDK

### Requirement: Launcher-owned files are read-only
The agent SHALL NOT modify `sbt-sber.sh`, any repository configuration file at the repository root, `.sbtopts`, `.sbtxopts`, `.jvmopts`, anything under the user's `~/.sbt` directory, or resolver and credential settings in any build definition file. A file at the root that contains a `[repositories]` section is a repository configuration file; when unsure whether a root file is launcher-owned, the agent leaves it unchanged.

#### Scenario: Several repository configuration files for different environments
- **WHEN** the repository root contains more than one repository configuration file, each for a different build environment
- **THEN** the agent edits none of them, creates no new one, and does not make the launcher use a different one than it chooses itself

#### Scenario: Launcher permissions or content
- **WHEN** the launcher fails to run or behaves unexpectedly
- **THEN** the agent does not edit the script, change its permissions, or replace it, and reports the problem instead

#### Scenario: Global sbt configuration
- **WHEN** a change under `~/.sbt`, such as `repositories`, `.credentials`, a global `*.sbt` file, or a global plugin, would make a build succeed
- **THEN** the agent does not make it and reports what is missing to the user

#### Scenario: Option files in the project
- **WHEN** a setting could be added to `.sbtopts`, `.sbtxopts`, or `.jvmopts`
- **THEN** the agent leaves those files unchanged

### Requirement: No overrides through the environment or runner flags
The agent SHALL NOT set, change, or unset `SBT_OPTS`, `SBTX_OPTS`, `JVM_OPTS`, `JAVA_OPTS`, `JAVA_HOME`, or `JDK_HOME` for a launcher invocation, and SHALL NOT pass `-D`, `-J`, `-S`, `-mem`, `-java-home`, `-jvm-opts`, `-sbt-opts`, `-sbtx-opts`, `-ivy`, `-sbt-dir`, `-sbt-boot`, `-no-share`, `-offline`, `-sbt-launch-repo`, `-sbt-jar`, `-sbt-version`, or `-sbt-create` to the launcher.

#### Scenario: Environment variable prefix
- **WHEN** the agent would run `SBT_OPTS=... ./sbt-sber.sh ...` or export such a variable first
- **THEN** it does not, because sbt-extras parses these variables as if they were command-line flags

#### Scenario: Selecting a repository configuration
- **WHEN** the agent would pass `-Dsbt.repository.config=<file>` or `-Dsbt.override.build.repos=false`
- **THEN** it does not, and leaves repository selection to the launcher

#### Scenario: Working around a launcher failure
- **WHEN** a launcher-owned flag such as `-sbt-launch-repo`, `-sbt-jar`, or `-offline` would get past a failure
- **THEN** the agent does not pass it and reports the failure instead

### Requirement: The launcher is a black box
The agent SHALL NOT read `sbt-sber.sh` or any launcher-owned file to obtain credentials, tokens, or repository URLs, SHALL NOT print their contents, and SHALL NOT reconstruct an equivalent sbt or sbt-extras invocation from what the launcher does.

#### Scenario: A credential or URL seems necessary
- **WHEN** the agent believes it needs a token, password, or repository URL that the launcher holds
- **THEN** it does not open the launcher or its files to find it and asks the user instead

#### Scenario: Reproducing the launcher
- **WHEN** the launcher fails and the agent could run sbt directly with the same repository and memory settings
- **THEN** it does not, and reports the launcher failure
