# sbt launcher

In this environment the build works only through `./sbt-launcher.sh`, which starts the sbt-extras runner with the corporate credentials, repositories, and memory settings. Dependencies are approved and added by people, not agents. The rules below apply to every sbt build in such a repository.

Apply these rules throughout this repository if it builds its Scala code with sbt, on any Scala and sbt version, to every sbt command you run, every Scala source change you make, and every dependency you consider for its Scala code or its sbt build. They are not conventions: they do not yield to the repository's habits or to any other tool that is installed, and where a rule hands a decision to the repository, such as which files scalafmt formats or which modules carry the plugin, the rule says so. This repository must have `./sbt-launcher.sh` at its root, the sbt-scalafmt plugin in its build, and a `.scalafmt.conf`; if any of them is missing, these rules still apply: when a rule below finds it missing, stop and report it as that rule describes, and never supply it yourself.

## Running the build

**The launcher is the only way to run sbt.** Run every sbt task as `./sbt-launcher.sh` from the repository root. Never start sbt any other way, even when the alternative is installed or already running: no global `sbt`, no `sbtn` or `sbt --client`, no coursier (`cs launch sbt`), no bloop, no Metals or other BSP build server, no `java -jar sbt-launch.jar`.

**Run from the repository root.** The working directory for every invocation is the root, the directory that holds `sbt-launcher.sh`. If you are in a module or source directory, `cd` to the root first. Scope work to a module with sbt's `module/task` syntax, such as `core/compile`, never by running from the module's directory.

**No launcher, no build.** If `./sbt-launcher.sh` does not exist at the root, stop and tell the user the repository is not set up for agent builds. Do not search other directories for it, fetch or write one, or fall back to any other sbt, and do not offer the user another way to build. The next step is theirs: putting the launcher at the root. A shell error `no such file or directory: ./sbt-launcher.sh` means one of these two cases: you are not at the root (change to it and run there), or the launcher is missing (stop as above).

**Every invocation is `./sbt-launcher.sh -batch [output flags] <sbt commands>`.**

- `-batch` on every call. It closes standard input so sbt can never wait for a terminal, including under `run` and `console`.
- At least one sbt command on every call. With no command the runner opens the interactive sbt shell, so if you have nothing to run, do not invoke the launcher.
- Only these may appear on the command line:

  | Item | Use |
  |---|---|
  | `-batch` | Always. |
  | `-no-colors` | Plain output without ANSI codes. Add it whenever you save or parse the output. |
  | `--debug` | Verbose log, for one diagnostic run only. |
  | `-trace <n>` | Stack traces up to `n` frames (`n` of 0 or more), for one diagnostic run only. |
  | sbt tasks and commands | Work to run or settings to print: `scalafmtAll`, `compile`, `test`, `clean`, `testOnly ...`, `show ...`, `projects`. |

- Everything else is forbidden, because it changes how the build runs rather than what it prints. That covers every other runner flag (`-D...`, `-J...`, `-S...`, `-mem`, `-java-home`, `-sbt-version`, and the rest), and the `set ...` and `++ ...` commands (Scala version, compiler options, resolvers, dependencies). If a task seems to need anything outside the table, ask the user. Widening this command-line list is their decision.
- A command with arguments is one quoted shell argument: `./sbt-launcher.sh -batch "testOnly com.acme.FooSpec"`. Unquoted, sbt receives `testOnly` and `com.acme.FooSpec` as two separate commands.
- Several tasks go in one invocation, in order: `./sbt-launcher.sh -batch clean compile test`. sbt stops at the first command that fails, so `compile test` already means "test only if it compiles". Do not start sbt once per task, or a second time just for the next step of a sequence you already know. Starting again after you change code is expected.

```bash
./sbt-launcher.sh -batch scalafmtAll compile test
./sbt-launcher.sh -batch compile
./sbt-launcher.sh -batch test
./sbt-launcher.sh -batch core/test
./sbt-launcher.sh -batch "core/testOnly com.acme.core.FooSpec"
```

### Reading output

- Never use `--error` (or its runner alias `-q`): it hides `[warn]` lines. If you change the log level at all, only raise verbosity with `--debug`. Skip `--warn` too, because it hides the test summary.
- Judge a build from its complete output. Do not pipe the only copy through `tail`, `head`, or `grep`.
- When the output is too long to read inline, capture all of it to a file outside the repository, then inspect every `[warn]` and `[error]` line in it before deciding what to do:

  ```bash
  ./sbt-launcher.sh -batch -no-colors test > "${TMPDIR:-/tmp}/sbt-build.log" 2>&1; echo "exit=$?"
  grep -nE '^\[(warn|error)\]' "${TMPDIR:-/tmp}/sbt-build.log"
  ```

### Slow runs are not failures

The first run in a session may download the sbt launcher jar, sbt itself, and the project's dependencies. Several minutes is normal. Run the launcher with a generous timeout (allow at least 10 minutes on a cold start), or in the background, and wait for the process to exit before judging the result. Download progress or a quiet pause is not a hang: do not kill and retry.

### When a build fails

Decide which side of sbt's load marker the failure is on. The marker is sbt printing `welcome to sbt` or `loading settings for project`. Match it case-insensitively: sbt before 1.4 prints `Loading settings for project` and no welcome line.

**First, rule out your own invocation mistake.** If the failure points at the command you typed rather than at the build or the infrastructure, correct the command and run it again. That covers three cases:

- You were not at the repository root: the shell cannot find `./sbt-launcher.sh`, or the runner says the directory `doesn't appear to be an sbt project`.
- A permitted option was malformed, for example `Aborting: -trace requires <integer> argument`.
- sbt rejected a command you mistyped, left unquoted, or scoped to the wrong module: `Not a valid command`, `Not a valid key`, or `Not a valid project ID`. A `Not a valid key` or `Not a valid command` for a task spelled exactly `scalafmtAll` is not a typo: do not respell or retry it, and handle it as formatting not available (below). A `No such setting/task` for a `<module>/scalafmtAll` you added, after the unscoped `scalafmtAll` succeeded, means the build turns the plugin off for that module: drop that module task, keep `scalafmtAll` first, run again, and name the module as excluded from formatting in your report.

A correction may only bring the command into line with the rules above. It never adds a forbidden flag, a `set`, an environment variable, a configuration change, or another tool, and never `-sbt-create`. If the corrected command follows every rule and still fails before the marker, it was not your mistake: stop and report.

**Otherwise, before the marker, the launcher or the infrastructure failed: stop and report.** This includes the launcher's own error (for example an unmet network or credential precondition), the launcher not being executable, sbt-extras failing to download or verify `sbt-launch.jar`, sbt failing to boot, and a missing or unsuitable Java. Do not work around it: no other tool, no added or different flags (`-sbt-launch-repo`, `-sbt-jar`, `-sbt-version`, `-offline`), no configuration change, no edit to the launcher, no `chmod`, and no running it as `sh sbt-launcher.sh` or `bash sbt-launcher.sh`.

**After the marker, a compile error or failing test is your work.** `[error]` lines that point at source files, or failed test cases: read the diagnostics, fix the code or the test, and re-run the same command through the launcher, such as `./sbt-launcher.sh -batch scalafmtAll compile` or the failing suite with `testOnly`. Add any further steps you already plan to that same invocation instead of starting sbt again: `./sbt-launcher.sh -batch scalafmtAll billing/compile billing/test` (`test` also compiles the test sources). A scalafmt parse error pointing at a source file is your work too: fix the syntax in the file you changed and run the same invocation again, `scalafmtAll` kept. If the file is one you never touched, tell the user which file and which error, leave fixing or excluding it to them, and never drop `scalafmtAll` to get past it.

**Three failures after the marker are still stop-and-report:**

1. **Unresolved artifact.** `Error downloading`, `Error fetching artifacts`, `download failed`, `download error`, `scalafmt: [v<version>] failed to download` (the scalafmt version pinned in `.scalafmt.conf`; report that version), `unresolved dependency`, `ResolveException`, `not found: https://...` or `not found: .../ivy.xml`, or a repository refusing access (`unauthorized: https://...`, `forbidden: https://...`, HTTP 401, 403, or 404). Report the coordinates. Do not add resolvers or credentials, change a version, edit repository configuration, or point sbt at another repository. A compiler message such as `not found: value foo` or `not found: type Bar` at a source line is a compile error, not this.
2. **Out of memory.** `java.lang.OutOfMemoryError` (heap or Metaspace) or `GC overhead limit exceeded`. Report it. Do not pass `-mem` or `-J-Xmx...`, set `SBT_OPTS` or `JVM_OPTS`, or edit `.jvmopts`.
3. **Formatting not available.** sbt rejects a correctly spelled `scalafmtAll` with `Not a valid key` or `Not a valid command`, which means the build does not declare the sbt-scalafmt plugin or declares one older than 2.0.0-RC5, or the plugin reports that `.scalafmt.conf` is missing (`scalafmt: File does not exist: .../.scalafmt.conf`). Report it with area formatting. This is a stop-and-report, not a dependency request: the plugin and the file are required of every repository, not needed by your task. Do not add the plugin to `project/plugins.sbt`, create `.scalafmt.conf`, run the task again, format with another tool, or finish the change unformatted.

Report in this shape, then stop the task:

```text
Build stopped: <launcher precondition | launcher jar download | repositories | memory | Java | formatting>
Command:   ./sbt-launcher.sh -batch <exact arguments>
Directory: <absolute path of the working directory>
Output (verbatim):
<the error lines exactly as printed, including any artifact coordinates>
Changed before the stop: <files you edited and files scalafmtAll reformatted, or none>
I have not changed anything to work around this failure.
```

## Formatting

Every Scala source change is formatted with scalafmt, through the sbt-scalafmt plugin and the `.scalafmt.conf` that every corporate repository has.

**Every build starts with `scalafmtAll`.** Make `scalafmtAll` the first sbt command of every invocation that compiles or tests, in that same invocation: `./sbt-launcher.sh -batch scalafmtAll compile test`, or for one module `./sbt-launcher.sh -batch scalafmtAll core/compile core/test`. This holds for every compiling or testing command shown elsewhere in these rules: put `scalafmtAll` in front. With the plugin's default settings a parse error fails `scalafmtAll`, and sbt stops at the first command that fails, so nothing compiles that scalafmt could not parse. Re-running after a fix is the same invocation, `scalafmtAll` still first. Read-only queries such as `show core/libraryDependencies` or `projects` build nothing and need no `scalafmtAll`.

**A change without a build is formatted too.** If your change ends without compiling or testing, for example because the user asked for an edit and no build, run `./sbt-launcher.sh -batch scalafmtAll` on its own before you report the change as done.

**The repository decides what is formatted.** Run `scalafmtAll` unscoped from the root, even when compile and test are scoped to a module. Never narrow or replace it to touch fewer files: no `scalafmtOnly <files>`, no `scalafmt` or `Test/scalafmt`, no `module/scalafmtAll` such as `core/scalafmtAll` in place of `scalafmtAll`, and no skipping it. Unscoped `scalafmtAll` reaches a module only through the root project's aggregation: with no root project defined, sbt's default root aggregates every module; an explicit root (`project in file(".")`) aggregates the modules its `.aggregate(...)` lists and, in turn, whatever those aggregate; and `scalafmtAll / aggregate := false` turns aggregation off. If the root does not aggregate a module you changed, append `<module>/scalafmtAll` right after `scalafmtAll`, never instead of it: `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll core/compile core/test`, or `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll` for a change with no build. If you cannot tell, add it; where the root does aggregate the module, nothing changes. If the module task fails with `No such setting/task`, the build turns the plugin off for that module, as "When a build fails" describes. The scope is the repository owner's, set through `scalafmtFilter`, `scalafmtOnCompile`, the `project.*` keys of `.scalafmt.conf`, and which modules have the plugin. Do not run `scalafmtSbt`, even after you change a `*.sbt` or `project/*.scala` file: formatting the build definition is the repository's to enable, for example through `scalafmtSbtOnLoad` (sbt-scalafmt 2.6.2 and later).

**Reformatted files stay formatted.** When `scalafmtAll` reformats files you did not edit, leave them formatted, do not revert them, and name them when you report your work.

**Formatting runs only through the launcher.** Format only by running `scalafmtAll` through `./sbt-launcher.sh`, even when a `scalafmt` binary is on the `PATH` or the plugin is missing: no scalafmt command-line tool, no `cs launch scalafmt` or other coursier or scala-cli route, no Metals or IDE formatter, no editor plugin.

**Scalafmt configuration is read-only.** Never create, modify, or delete `.scalafmt.conf`, and never change a scalafmt setting in the build definition (`scalafmtFilter`, `scalafmtOnCompile`, `scalafmtConfig`, `scalafmtFailOnErrors`, `scalafmtSbtOnLoad`, or any other), neither in a file nor with `set` on the command line, and never turn the plugin on or off for a project (`enablePlugins`, `disablePlugins`). Keep the repository's style where you would choose another, and let scalafmt format the code under it. Keep the `version` that `.scalafmt.conf` pins even when it looks old. If the build cannot download it (`scalafmt: [v<version>] failed to download`), that is an unresolved artifact, not formatting not available: report it with area repositories.

A rejected `scalafmtAll`, a missing `.scalafmt.conf`, and a scalafmt parse error are classified under "When a build fails" above.

## Launcher-owned configuration

Authentication, repositories, and memory belong to `./sbt-launcher.sh`. Do not create, modify, delete, select, or override any of them by any means. When a failure traces back to one of them, report it using the format above instead of changing anything.

**Read-only files.** Never modify:

- `sbt-launcher.sh`: no edits, no permission changes, no replacement.
- Every repository configuration file at the repository root. There may be several, one per build environment. Any root file containing a `[repositories]` section is one. Edit none of them, create no new one, and do not make the launcher use a different one from the one it picks.
- `.sbtopts`, `.sbtxopts`, `.jvmopts`, even when a setting could simply be added there.
- Anything under `~/.sbt/`: `repositories`, `.credentials`, global `*.sbt` files, global plugins. If a change there would make the build succeed, report what is missing instead.
- Resolver and credential settings in any build definition file (`resolvers`, `externalResolvers`, `credentials`).

If you cannot tell whether a root file is source, build definition, or documentation, treat it as launcher-owned and leave it unchanged.

**No overrides through the environment.** Never set, change, or unset `SBT_OPTS`, `SBTX_OPTS` (`SBT_X_OPTS` in older sbt-extras), `JVM_OPTS`, `JAVA_OPTS`, `JAVA_HOME`, or `JDK_HOME` for a launcher run, neither as a prefix (`SBT_OPTS=... ./sbt-launcher.sh ...`) nor with `export` first. sbt-extras parses `SBT_OPTS` and `SBTX_OPTS` exactly like command-line flags and hands `JVM_OPTS` straight to the JVM, so they are the forbidden flags by another route.

**No overrides through runner flags.** The command-line allowlist above already excludes them. In particular, never select repositories with `-Dsbt.repository.config=<file>` or `-Dsbt.override.build.repos=...`; repository selection is the launcher's. Never pass `-sbt-launch-repo`, `-sbt-jar`, or `-offline` to get past a failure.

**The launcher is a black box.** Do not open `sbt-launcher.sh` or any launcher-owned file to find tokens, passwords, or repository URLs, or to learn which files it uses, and never print their contents. If you believe you need a credential or URL the launcher holds, ask the user. Never reconstruct the launcher by running sbt, sbt-extras, or `java` directly with the same settings, even when the launcher fails.

**Java.** If the launcher or sbt reports a missing or unsuitable Java, report it with area Java. Do not set `JAVA_HOME`, pass `-java-home`, or install or switch a JDK. A Java version-mismatch warning on its own is not a failure: an ordinary compile error or failing test after the load marker is still your work when that warning was printed.

## Dependencies

**The dependency set is immutable to you.** Never add, remove, upgrade, downgrade, re-scope (for example `% Test` to main code or the reverse), exclude, or override a dependency, in any file, by any mechanism. People make dependency changes after you ask for them.

**What counts as a dependency**, wherever it is declared (`*.sbt`, `project/*.sbt`, or `project/*.scala` such as `project/Dependencies.scala`):

- `libraryDependencies`, `dependencyOverrides`, `excludeDependencies`
- `addSbtPlugin` (usually in `project/plugins.sbt`) and `addCompilerPlugin`
- `scalaVersion` and `crossScalaVersions`, and `sbt.version` in `project/build.properties`
- unmanaged jars under `lib/` or in `unmanagedJars`
- inter-module `dependsOn`

Versions and module IDs centralised in a Scala build file are declarations exactly like `libraryDependencies` in `build.sbt`. Do not edit them.

**No side channels.** Do not bring artifacts onto the classpath from outside the build definition: no `set libraryDependencies ...` or similar on the launcher command line, no `coursier` or `cs fetch`, no `scala-cli --dep`, no Ammonite `import $ivy`, no manual jar downloads, and no copying or referencing jars from `~/.ivy2`, `~/.cache/coursier`, `~/.m2`, or any other local cache.

**Work with what the module declares.** Implement with the dependencies already declared for the module you are working in, including those it inherits through `dependsOn`. Before concluding that something is missing, check: read the build definition files, including `project/*.scala`, or run a read-only query such as `./sbt-launcher.sh -batch -no-colors "show core/libraryDependencies"`. Write tests with the test framework and matchers the module already declares. If a declared library can do the job, use it, even if another would be more familiar.

**Asking for a dependency.** When the task cannot be done without a dependency change (a new library, another version, another configuration, a plugin, a Scala or sbt version, a new `dependsOn`), stop and ask the user in this shape:

```text
Dependency request
Coordinates:   <organisation> % <artifact> % <version, if known>
Module:        <module, such as core>
Configuration: <Compile, Test, ...>
Reason:        <what the task needs it for>
Ruled out:     <declared alternatives considered, and why each does not fit>
```

The task pauses there. Do not add the dependency provisionally, and do not add it temporarily to check that it would work. When the user confirms it has been added, continue by running the build through the launcher, which picks up the changed build definition on its next start.
