# Proposal

## Why

The skill tells agents to run the build only through a launcher it names by its exact file name, and that name must match the script agents find at the repository root. The corporate launcher is called `sbt-sber.sh`; the skill still says `sbt-launcher.sh` in every rule, command example, error-message example, and trigger phrase. An agent following the skill as written would either type commands that fail with `no such file or directory`, or apply the "No launcher, no build" rule, decide the repository is not set up for agent builds, and stop.

## What Changes

- Every mention of the launcher's file name in `corp-scala/SKILL.md` and `corp-scala/references/sbt-launcher.md` changes from `sbt-launcher.sh` to `sbt-sber.sh`: the frontmatter description that triggers the skill, the rules, the command examples, the worked-example and common-mistakes tables, the layer diagram, the shell error examples (`no such file or directory: ./sbt-launcher.sh`, `permission denied: ./sbt-launcher.sh`), and the reference's section heading "Launcher (`sbt-launcher.sh`)".
- The three capability specs change the same way in every requirement and scenario that quotes the file name. Requirement names, structure, and SHALL statements are otherwise untouched; only the literal file name moves.
- **BREAKING** for agents: the skill stops recognising `sbt-launcher.sh`. Under the renamed rules a repository whose launcher still carries the old name is "not set up for agent builds", and the skill's description no longer triggers on that file. No alias, fallback, or transition note for the old name is added.
- Unchanged: the generic word "launcher" throughout, the reference document's own file name `references/sbt-launcher.md` and its title (it describes the launcher stack, not the script), and the archived change `2026-10-19-launcher-and-dependency-rules`, which stays as the historical record under the old name.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `sbt-invocation`: seven requirements quote `./sbt-launcher.sh` in their text or scenarios and are modified to quote `./sbt-sber.sh`: "The launcher is the only way to run sbt", "The launcher runs from the repository root", "Every invocation is non-interactive", "Only sbt commands and output-only settings go on the command line", "Launcher and infrastructure failures stop the task", "The agent corrects its own invocation mistakes", and "Build failures are the agent's work". The other four requirements of this capability do not name the file and are not touched.
- `launcher-owned-configuration`: all four requirements name the file and are modified: "Authentication, repositories, and memory belong to the launcher", "Launcher-owned files are read-only", "No overrides through the environment or runner flags" (its "Environment variable prefix" scenario), and "The launcher is a black box".
- `project-dependencies`: two requirements quote the file in one scenario each and are modified: "No side channels for artifacts" ("Setting a dependency from the command line") and "Work with what the module already declares" ("Discovering declared dependencies"). The other three requirements are not touched.

## Impact

- Skill files: `corp-scala/SKILL.md` (26 occurrences of `sbt-launcher.sh`, including two in the frontmatter description) and `corp-scala/references/sbt-launcher.md` (44 occurrences, including one section heading and the layer diagram). After the change `grep -rn "sbt-launcher.sh" corp-scala/` returns nothing, while the three links to `references/sbt-launcher.md` in `SKILL.md` keep resolving.
- Specs: 17 occurrences across 13 requirements in the three main specs, carried by this change's delta specs and applied to `openspec/specs/` when the change is archived or synced. No requirement is added, removed, or renamed.
- Downstream: every agent session that loads the skill. The skill now triggers on `./sbt-sber.sh` at the root and treats only that file as the launcher. Any repository that still ships the launcher as `sbt-launcher.sh` has to rename it for agent builds to work; that rename is outside this repository.
- Assumptions, recorded here so review can overturn them: the launcher keeps its location at the repository root and its contract (a wrapper over the sbt-extras runner that adds corporate settings and forwards every other argument), so nothing but the name changes; the reference document keeps its file name; the old name gets no compatibility mention in the skill.
- Verification stays as in the first change: by review and text checks, since no launcher or sbt project is available in this repository.
