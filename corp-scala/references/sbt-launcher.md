# sbt launcher reference

Background for the rules in [../SKILL.md](../SKILL.md). The skill file is what you obey. This file explains what each layer does with your arguments and why the allowlist is drawn where it is.

## Contents

1. [Layers](#1-layers)
2. [Runner flags](#2-runner-flags)
3. [Option files and environment variables](#3-option-files-and-environment-variables)
4. [Failure markers](#4-failure-markers)
5. [Worked examples](#5-worked-examples)
6. [sbt-extras revision](#6-sbt-extras-revision)

## 1. Layers

```text
agent
  │   ./sbt-launcher.sh -batch -no-colors "core/testOnly com.acme.core.FooSpec"
  ▼
sbt-launcher.sh            corporate wrapper, a black box to the agent
  │   adds authentication, repository, and memory configuration
  │   forwards every other argument unchanged
  ▼
sbt-extras `sbt` script    the runner (bash)
  │   consumes the runner flags it recognises (section 2)
  │   reads .sbtopts, .sbtxopts, .jvmopts and SBT_OPTS, SBTX_OPTS, JVM_OPTS (section 3)
  │   checks the working directory holds build.sbt or project/
  │   downloads and verifies sbt-launch.jar into ~/.sbt/launchers/<version>/
  │   execs:  java <jvm options> -jar sbt-launch.jar <runner-generated commands> <your commands>
  ▼
sbt                        sbt-launch.jar boots the sbt version in project/build.properties
      prints "welcome to sbt", loads project/ and build.sbt, resolves dependencies,
      runs the commands in order, exits
```

What this means for the rules:

- The launcher holds the corporate settings. Everything you add on the command line reaches sbt-extras alongside the launcher's own arguments, so a runner flag from you can override or conflict with what the launcher set. That is why the command line is an allowlist.
- sbt-extras turns some flags into sbt commands of its own (`-S` into `set scalacOptions ...`, `-offline` into `set offline ...`, `-trace` into `set traceLevel ...`). These run before your commands, so a flag can change the build even when it looks like an option.
- Everything sbt-extras does not recognise becomes a residual argument and is handed to sbt as a command. Each separate shell argument is one sbt command, which is why `testOnly com.acme.FooSpec` must be quoted.
- With no residual argument at all, sbt-extras supplies `shell`, the interactive prompt. `-batch` closes standard input, but a bare invocation still has nothing useful to do, so the skill requires at least one command.

## 2. Runner flags

Every flag recognised by sbt-extras at the revision in section 6, plus `-mem`, which the specs name. **Allowed** means it is on the skill's allowlist. Everything else is **Forbidden**, and the third column says why.

### Allowed

| Flag | Class | What sbt-extras does with it |
|---|---|---|
| `-batch` | Allowed, required | Runs java with standard input from `/dev/null`, so sbt cannot wait for a terminal. Not passed on to sbt. |
| `-no-colors` | Allowed | Adds the JVM properties `-Dsbt.log.noformat=true -Dsbt.color=false`. Changes output only. |
| `-trace <n>` | Allowed, one diagnostic run | Becomes the sbt command `set traceLevel in ThisBuild := <n>`: stack traces up to `n` frames. `n` must be 0 or more, because the runner aborts on any value starting with `-` (so `-trace -1` fails). Leave `-trace` off to keep sbt's default. Output only. Revisions before August 2014 generated `set every traceLevel := <n>` instead. |
| `--debug` | Allowed, one diagnostic run | Not a runner flag. Passed through to sbt as its log-level command. |

### Forbidden: memory, JVM, Java

| Flag | Class | What sbt-extras does with it |
|---|---|---|
| `-D<key>=<value>` | Forbidden | Passes a JVM system property. This is how repository selection is overridden, for example `-Dsbt.repository.config=<file>` or `-Dsbt.override.build.repos=false`. |
| `-J<option>` | Forbidden | Passes `<option>` to the JVM with `-J` stripped, for example `-J-Xmx4g`. Changes memory and JVM behaviour. |
| `-mem <n>` | Forbidden | Not recognised at the checked revision (removed in March 2013). Older sbt-extras and the official sbt runner script use it to set heap size. Here it would be forwarded to sbt as an argument. Forbidden either way because memory belongs to the launcher. |
| `-jvm-opts <path>` | Forbidden | Reads JVM options from `<path>` instead of `.jvmopts`. |
| `-java-home <path>` | Forbidden | Runs `<path>/bin/java`, adds `set javaHome in ThisBuild := ...`, and exports `JAVA_HOME`, `JDK_HOME`, and a new `PATH`. |
| `-jvm-debug <port>` | Forbidden | Adds `-Xdebug -Xrunjdwp:...` and opens a debugger port. |
| `-debug-inc` | Forbidden | Adds the JVM property `-Dxsbt.inc.debug=true`. Use `--debug` for verbose output. |

### Forbidden: repositories, caches, directories

| Flag | Class | What sbt-extras does with it |
|---|---|---|
| `-offline` | Forbidden | Adds `set offline in Global := true`. Changes how dependencies resolve. |
| `-ivy <path>` | Forbidden | Adds `-Dsbt.ivy.home=<path>`, a different local Ivy repository. |
| `-sbt-dir <path>` | Forbidden | Adds `-Dsbt.global.base=<path>`, a different global settings and plugins directory (normally under `~/.sbt`). |
| `-sbt-boot <path>` | Forbidden | Adds `-Dsbt.boot.directory=<path>`, a different boot directory. |
| `-no-share` | Forbidden | Moves every cache into the project: `-Dsbt.global.base=project/.sbtboot -Dsbt.boot.directory=project/.boot -Dsbt.ivy.home=project/.ivy -Dsbt.coursier.home=project/.coursier`. |
| `-sbt-launch-repo <url>` | Forbidden | Downloads `sbt-launch.jar` from `<url>` instead of the default repository. |
| `-sbt-launch-dir <path>` | Forbidden | Keeps downloaded launchers in `<path>` instead of `~/.sbt/launchers`. |
| `-sbt-jar <path>` | Forbidden | Uses `<path>` as the sbt launcher jar. |
| `-sbt-opts <path>` | Forbidden | Reads runner flags from `<path>` instead of `.sbtopts`. |
| `-sbtx-opts <path>` | Forbidden | Reads runner flags from `<path>` instead of `.sbtxopts`. Spelled `-sbt-x-opts` before April 2020. |

### Forbidden: sbt and Scala versions, compilation

| Flag | Class | What sbt-extras does with it |
|---|---|---|
| `-S<option>` | Forbidden | Collects the option. All `-S` options become one sbt command, `set scalacOptions in ThisBuild += "<options joined by spaces>"`. Changes compilation. |
| `-sbt-version <v>` | Forbidden | Adds `-Dsbt.version=<v>`, overriding `project/build.properties`, and fetches a launcher for `<v>`. |
| `-sbt-force-latest` | Forbidden | Forces the runner's built-in sbt release (1.13.0 at the checked revision). |
| `-sbt-dev` | Forbidden | Forces the runner's pre-release sbt version. |
| `-28` `-29` `-210` `-211` `-212` `-213` | Forbidden | Adds `++ <latest 2.x>` (for example `-213` is `++ 2.13.18`), switching the Scala version. |
| `-scala-version <v>` | Forbidden | Adds `++ <v>`. For a `-SNAPSHOT` version it also adds `set resolvers += Resolver.sonatypeRepo("snapshots")`. |
| `-binary-version <v>` | Forbidden | Adds `set scalaBinaryVersion in ThisBuild := "<v>"`, which changes which artifacts resolve. |
| `-scala-home <path>` | Forbidden | Adds `set scalaHome in ThisBuild := Some(file("<path>"))`, a local Scala build. |

### Forbidden: everything else

| Flag | Class | What sbt-extras does with it |
|---|---|---|
| `-sbt-create` | Forbidden | Starts sbt even when the working directory is not an sbt project, which hides a wrong working directory. |
| `-v` | Forbidden | Verbose runner. Prints every argument of the final java command line, including whatever the launcher added, which breaks the black-box rule. |
| `-d` | Forbidden | Alias that queues `--debug`. Pass `--debug` itself so the command line stays on the allowlist. |
| `-w` | Forbidden | Alias for `--warn`. Hides `[info]`, including the test summary. |
| `-q` | Forbidden | Alias for `--error`. Hides `[warn]`. |
| `-x` | Forbidden | Accepted and ignored at the checked revision. |
| `-h`, `-help` | Forbidden | Prints the runner's usage and exits. This file documents the flags instead. |
| `-prompt <expr>` | Forbidden | Adds `set shellPrompt in ThisBuild := ...`. Only meaningful in the interactive shell. |
| `-script <file>` | Forbidden | Runs `<file>` as a Scala script through `sbt.ScriptMain`, outside the project build. |
| `new` (as an argument) | Forbidden | Special-cased: forces the runner's sbt version, skips the project check, and runs `sbt new` to generate a project from a template. |

### sbt log-level commands

These are sbt commands, not runner flags. sbt-extras passes them through.

| Command | Class | Note |
|---|---|---|
| `--debug` | Allowed | Raises verbosity for one run. |
| `--info` | Allowed, unnecessary | The default level. |
| `--warn` | Not recommended | Changes output only, but hides `[info]`, including the test summary. |
| `--error` | Forbidden | Hides `[warn]`. |

`set` and `++` typed as commands are forbidden by the skill: they change settings, versions, resolvers, or dependencies. The only `set` your own allowed arguments can generate is the one `-trace` produces. The launcher's arguments and any `.sbtopts` or `.sbtxopts` in the repository may add others, and those are not yours to change.

## 3. Option files and environment variables

sbt-extras reads these on every start. The skill makes all of them read-only and forbids setting the variables for a launcher run, because each one is another way to pass the forbidden flags in section 2.

| Source | How sbt-extras reads it | Effect |
|---|---|---|
| `.sbtopts` in the working directory | Each non-blank, non-`#` line is one argument, fed through the same flag parser as the command line. | Any runner flag or sbt command, including `-D`, `-J`, `-S`, `-sbt-version`, and `set ...`. |
| `SBT_OPTS` | Used only when no `.sbtopts` (or `-sbt-opts` file) is readable. Split on spaces and fed through the flag parser. A value `@<path>` names a file to use instead. | Same as `.sbtopts`. |
| `.sbtxopts`, `SBTX_OPTS` | Same mechanism as `.sbtopts` and `SBT_OPTS`. Intended for runner flags. Revisions before April 2020 named the variable `SBT_X_OPTS` (and the flag `-sbt-x-opts`); the old names are forbidden the same way. | Same as `.sbtopts`. |
| `.jvmopts` in the working directory | Each non-blank, non-`#` line is passed verbatim to the JVM before `-jar`. Replaces sbt-extras's default JVM options, which depend on the Java version: `-Xms512m -Xss2m -XX:MaxInlineLevel=18` on Java 8, 9, and 17 or later; those plus `-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler` on Java 10 to 16 (except on Apple Silicon); `-XX:MaxPermSize=384m` added before Java 8. | Heap, stack, GC, and JVM system properties. |
| `JVM_OPTS` | Used only when no `.jvmopts` (or `-jvm-opts` file) is readable. Split on spaces and passed verbatim to the JVM. `@<path>` names a file. | Same as `.jvmopts`. |
| `JAVA_OPTS` | Not read by sbt-extras at the checked revision; its usage text says so explicitly. | Still forbidden: the launcher or other JVM tooling may read it, and it carries JVM and memory settings. |
| `JAVA_HOME`, `JDK_HOME` | sbt-extras runs `java` from `PATH`, but compares it with these and warns on a mismatch. `-java-home` sets both. | Java selection belongs to the launcher and the user. |
| `~/.sbt/launchers/` | Cache of downloaded `sbt-launch.jar` files. | Launcher-owned. |
| `~/.sbt/` (rest) | Read by sbt, not by sbt-extras: `~/.sbt/repositories`, `~/.sbt/.credentials` or similar credential files, global `~/.sbt/1.0/*.sbt`, global plugins in `~/.sbt/1.0/plugins/`, `~/.sbt/boot/`. | Launcher-owned. |

Precedence: sbt-extras parses the command line first and the option file or variable second, so a flag in `.sbtopts` or `SBT_OPTS` can override the same flag given on the command line.

Repository configuration files. sbt reads the file named by `-Dsbt.repository.config=<file>`. With `-Dsbt.override.build.repos=true` it uses only the repositories listed there and ignores the build's own resolvers. The file must contain a `[repositories]` section, for example:

```text
[repositories]
  local
  corp-maven: https://repo.example.corp/maven/
```

The launcher picks one such file per build environment from the repository root. Their names vary, so the skill recognises them by the `[repositories]` section rather than by name, and leaves any unidentified root file alone.

## 4. Failure markers

The load marker is the first line sbt prints after it boots:

```text
[info] welcome to sbt 1.9.7 (Eclipse Adoptium Java 17.0.9)
[info] loading global plugins from /home/user/.sbt/1.0/plugins
[info] loading settings for project acme-build from plugins.sbt ...
[info] loading project definition from /repo/project
[info] loading settings for project root from build.sbt ...
[info] set current project to acme (in build file:/repo/)
```

This is sbt 1.4 and later. Older sbt prints no `welcome to sbt` line and capitalises the rest (`[info] Loading settings for project ...`), so match the marker case-insensitively.

Anything that fails before `welcome to sbt` or `loading settings for project` belongs to the launcher, sbt-extras, the sbt boot, or Java: stop and report. The one exception is a mistake in your own command (see [Your own invocation mistakes](#your-own-invocation-mistakes)), which you correct and run again. The examples below are typical and abbreviated. Exact wording varies by version.

### Before the launcher runs

```text
zsh: no such file or directory: ./sbt-launcher.sh
bash: ./sbt-launcher.sh: No such file or directory
```

The shell did not find the launcher, so nothing ran. Either you are not at the repository root (change to it and run the command from there, as the skill requires), or the launcher is missing at the root (stop and tell the user the repository is not set up for agent builds; see "No launcher, no build" in the skill).

### Launcher (`sbt-launcher.sh`)

Its own messages are corporate and not known to this skill. Recognise them by position: they come before any sbt-extras or sbt line, and the command exits non-zero. A launcher that is present but cannot be executed also belongs here:

```text
zsh: permission denied: ./sbt-launcher.sh
bash: ./sbt-launcher.sh: Permission denied
```

Report area: launcher precondition. Do not `chmod` it, run it through `sh` or `bash`, or read it to see what it checks.

### sbt-extras

Verbatim strings at the checked revision. Progress first, which is normal on a cold start:

```text
Downloading sbt launcher for 1.9.7:
  From  https://repo1.maven.org/maven2/org/scala-sbt/sbt-launch/1.9.7/sbt-launch-1.9.7.jar
    To  /home/user/.sbt/launchers/1.9.7/sbt-launch.jar
```

Failures:

```text
Checksum does not match
Could not find an MD5 command
Could not download and verify the launcher. Obtain the jar manually and place it at /home/user/.sbt/launchers/1.9.7/sbt-launch.jar
```

Report area: launcher jar download. Do not follow the "obtain the jar manually" advice, and do not pass `-sbt-launch-repo`, `-sbt-jar`, `-sbt-version`, or `-offline`.

```text
/repo/core doesn't appear to be an sbt project.
If you want to start sbt anyway, run:
  <runner path> -sbt-create
```

The runner was started outside the repository root. This is your own invocation mistake: change to the root and run the same sbt commands there. Never pass `-sbt-create`, which only hides the wrong directory. If you see it when running from the root, it is not your mistake: stop and report (area: launcher precondition).

```text
Aborting: -trace requires <integer> argument
```

The runner rejected a malformed argument. This is your own invocation mistake. `-trace` needs an integer of 0 or more, and anything starting with `-` (including `-1`) is rejected. Correct the argument to an allowlisted form, such as `-trace 100`, or drop it, and run again. The same applies to any `Aborting: <flag> requires <...> argument` for a flag you are allowed to pass. If the message names a flag outside the allowlist, remove the flag; never supply its argument.

```text
Warning: Java version mismatch between PATH and JAVA_HOME/JDK_HOME, sbt will use the one in PATH
Can't parse java version from: ...
```

A warning by itself is not a failure, and on many machines it appears on every run. If the run then fails before the load marker, report with area Java and include these lines. A compile error or failing test after the marker is still your work, whatever warnings came first.

### sbt boot (before the marker)

sbt-launch.jar resolving sbt itself. Progress:

```text
[info] [launcher] getting org.scala-sbt sbt 1.9.7  (this may take some time)...
```

Failure:

```text
Error during sbt execution: Error retrieving required libraries
  (see /home/user/.sbt/boot/update.log for complete log)
Error: Could not retrieve sbt 1.9.7
```

Report area: repositories. Java problems at this stage:

```text
java: command not found
Unable to locate a Java Runtime.
java.lang.UnsupportedClassVersionError: ... class file version 61.0, this version of the Java Runtime only recognizes class file versions up to 52.0
```

Report area: Java. Do not set `JAVA_HOME`, pass `-java-home`, or install a JDK.

### After the marker: stop-and-report exceptions

Unresolved artifact (coursier, sbt 1.3 and later). Plugins from `project/plugins.sbt` resolve after the marker too, so a missing plugin looks like this as well:

```text
[error] (core / update) sbt.librarymanagement.ResolveException: Error downloading com.acme:widgets_2.13:1.4.0
[error]   Not found
[error]   not found: /home/user/.ivy2/local/com.acme/widgets_2.13/1.4.0/ivys/ivy.xml
[error]   not found: https://repo.example.corp/maven/com/acme/widgets_2.13/1.4.0/widgets_2.13-1.4.0.pom
```

When the metadata resolves but the jar cannot be fetched, coursier reports it differently. It writes HTTP 401, 403, and 404 as `unauthorized`, `forbidden`, and `not found`, not as numbers:

```text
[error] (core / update) ...FetchError$DownloadingArtifacts: Error fetching artifacts:
[error] https://repo.example.corp/maven/com/acme/widgets_2.13/1.4.0/widgets_2.13-1.4.0.jar: unauthorized: https://repo.example.corp/maven/com/acme/widgets_2.13/1.4.0/widgets_2.13-1.4.0.jar
```

Unresolved artifact (Ivy, older builds):

```text
[error] unresolved dependency: com.acme#widgets_2.13;1.4.0: not found
```

Report area: repositories, with the coordinates (`com.acme:widgets_2.13:1.4.0`).

Out of memory:

```text
[error] java.lang.OutOfMemoryError: Java heap space
java.lang.OutOfMemoryError: Metaspace
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

Report area: memory.

### After the marker: your work

Compile error, Scala 2 and Scala 3. The compiler's `not found` names a value or type at a source position, not an artifact:

```text
[error] /repo/core/src/main/scala/com/acme/Foo.scala:42:7: not found: value parseWidget
[error] (core / Compile / compileIncremental) Compilation failed

[error] -- [E006] Not Found Error: /repo/core/src/main/scala/com/acme/Foo.scala:42:7
[error] 42 |      parseWidget(input)
[error]    |      ^^^^^^^^^^^
[error]    |      Not found: parseWidget
```

Failing test:

```text
[info] - parses an empty widget *** FAILED ***
[info]   "" was not equal to "widget" (FooSpec.scala:18)
[error] Failed tests:
[error] 	com.acme.core.FooSpec
[error] (core / Test / test) sbt.TestsFailedException: Tests unsuccessful
```

An error in `build.sbt` or `project/*.scala` after the marker is a build failure too. Fix it only if the fix leaves dependency, resolver, and credential settings untouched. Otherwise report it.

### Your own invocation mistakes

These failures come from the command you typed, not from the build or the infrastructure, and can appear on either side of the marker. Correct the command so it follows every rule in the skill and run it again. A correction never adds a forbidden flag, a `set`, an environment variable, a configuration change, or another tool. If the corrected command follows every rule and still fails before the marker, it was not your mistake: stop and report.

| Symptom | Printed by | Correction |
|---|---|---|
| `no such file or directory: ./sbt-launcher.sh` | the shell, before the launcher runs | Change to the repository root. If the launcher is missing there, follow "No launcher, no build". |
| `... doesn't appear to be an sbt project.` | sbt-extras | Change to the repository root. Never `-sbt-create`. |
| `Aborting: -trace requires <integer> argument` | sbt-extras | `-trace <n>` with `n` of 0 or more, or drop `-trace`. |
| `Not a valid command`, `Not a valid key`, `Not a valid project ID`, `Expected ...` | sbt, after the marker | Fix the spelling, quote a command that has arguments, or fix the module name. |

An unquoted `testOnly` argument, for example (typical, abbreviated). sbt first runs `testOnly` with no argument, which runs every test, and then rejects the class name as a command:

```text
$ ./sbt-launcher.sh -batch testOnly com.acme.FooSpec
...
[error] Not a valid command: com.acme.FooSpec
[error] Not a valid project ID: com.acme.FooSpec
[error] Not a valid key: com.acme.FooSpec
```

Correction: `./sbt-launcher.sh -batch "testOnly com.acme.FooSpec"`.

## 5. Worked examples

All from the repository root.

| Goal | Command |
|---|---|
| Compile everything | `./sbt-launcher.sh -batch compile` |
| Compile main and test sources | `./sbt-launcher.sh -batch Test/compile` |
| Run all tests | `./sbt-launcher.sh -batch test` |
| Clean, compile, and test in one start | `./sbt-launcher.sh -batch clean compile test` |
| Run one suite | `./sbt-launcher.sh -batch "testOnly com.acme.FooSpec"` |
| Run suites by pattern | `./sbt-launcher.sh -batch "testOnly *FooSpec"` |
| Run one suite and pass arguments to the test framework | `./sbt-launcher.sh -batch "testOnly com.acme.FooSpec -- -z empty"` |
| Compile one module | `./sbt-launcher.sh -batch core/compile` |
| Test one module | `./sbt-launcher.sh -batch core/test` |
| One suite in one module | `./sbt-launcher.sh -batch "core/testOnly com.acme.core.FooSpec"` |
| List modules | `./sbt-launcher.sh -batch -no-colors projects` |
| A module's declared dependencies | `./sbt-launcher.sh -batch -no-colors "show core/libraryDependencies"` |
| Modules `core` depends on (`dependsOn`) | `./sbt-launcher.sh -batch -no-colors "show core/projectDependencies"` |
| Scala version in use | `./sbt-launcher.sh -batch -no-colors "show core/scalaVersion"` |
| Run the application | `./sbt-launcher.sh -batch "core/run --input sample.json"` |
| Full stack trace for one failing suite | `./sbt-launcher.sh -batch -trace 100 "core/testOnly com.acme.core.FooSpec"` |
| Verbose log for one run | `./sbt-launcher.sh -batch --debug core/compile` |
| Capture long output | `./sbt-launcher.sh -batch -no-colors test > "${TMPDIR:-/tmp}/sbt-build.log" 2>&1; echo "exit=$?"` |

Common mistakes:

| Wrong | Why | Right |
|---|---|---|
| `./sbt-launcher.sh -batch testOnly com.acme.FooSpec` | sbt receives two commands: `testOnly`, which runs every test, then `com.acme.FooSpec`, which is not a command. | `./sbt-launcher.sh -batch "testOnly com.acme.FooSpec"` |
| `cd core && ../sbt-launcher.sh -batch compile` | Not run from the root. | `./sbt-launcher.sh -batch core/compile` from the root |
| `./sbt-launcher.sh compile` | No `-batch`. | `./sbt-launcher.sh -batch compile` |
| `./sbt-launcher.sh -batch` | No command, so the runner opens the interactive shell. | Do not invoke the launcher without a command. |
| `./sbt-launcher.sh -batch compile && ./sbt-launcher.sh -batch test` | Two sbt starts. | `./sbt-launcher.sh -batch compile test` |
| `./sbt-launcher.sh -batch test \| tail -50` | Truncated output, and the exit status is lost. | Capture to a file, then read every `[warn]` and `[error]` line. |
| `./sbt-launcher.sh -batch --error test` | Hides warnings. | `./sbt-launcher.sh -batch test` |
| `SBT_OPTS=-J-Xmx4g ./sbt-launcher.sh -batch test` | Environment override of memory. | Report the out-of-memory failure. |
| `./sbt-launcher.sh -batch "set scalacOptions += \"-Xfatal-warnings\"" compile` | Changes compilation. | `./sbt-launcher.sh -batch compile` |

## 6. sbt-extras revision

- Checked against: `sbt` from [dwijnand/sbt-extras](https://github.com/dwijnand/sbt-extras) (formerly paulp/sbt-extras), last changed in commit `90e2b1ac8884f032005c2728e9ff283750e9a244`, "sbt 1.13.0", committed 2026-08-29.
- Date checked: 2026-10-19.
- The launcher pins its own sbt-extras revision, which is not known to this skill. Notable changes in older revisions (not an exhaustive list):
  - `-mem <n>` set heap size until March 2013 (commit `34c7370`) and is not recognised since.
  - `-d`, `-w`, and `-q` became aliases for `--debug`, `--warn`, and `--error` in May 2014 (commit `90170b2`).
  - `-trace <n>` generated `set every traceLevel := <n>` until August 2014 (commit `0ba09e0`), and `set traceLevel in ThisBuild := <n>` since.
  - `-offline` generated `set offline := true` until February 2017 and `set offline in Global := true` since (commit `bec7a05`).
  - `SBT_X_OPTS` and `-sbt-x-opts` were renamed `SBTX_OPTS` and `-sbtx-opts` in April 2020 (commit `78a179b`).
  - `-no-colors` added only `-Dsbt.log.noformat=true` until November 2020, and `-Dsbt.color=false` as well since (commit `0afd2ba`).
- The classification in section 2 does not depend on the revision. Only the "what it does" column would need re-checking if the launcher moved to a different sbt-extras version.
