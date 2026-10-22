# Design

## Context

See proposal.md for motivation. The facts that shape the approach, taken from a full read of the upstream repository at commit `2b4d8d0` (2026-10-16), of this repository's skills and specs, and of OpenCode's rules documentation as published on 2026-10-20:

- Upstream is one skill, `direct-style-scala`: a 151-line `SKILL.md` that carries the rules in five sections (Scala tooling, Coding style, Performance, Direct-style Scala, Functional programming) and indexes 24 numbered chapters, plus a `CONTRIBUTING.md` with its writing guidelines; about 3,650 lines in all. Every chapter opens with the library coordinates it depends on. The stack is sbt 2, Ox, Tapir, sttp, jsoniter, PureConfig, MacWire, Magnum, Flyway, HikariCP, scalaxb, ox-kafka and OpenTelemetry, and the code is Scala 3 only.
- Three chapters are general in substance: `180-domain-modeling.md` declares no dependencies and is plain Scala 3; `160-code-organization.md` is plain Scala except its closing section on build modules and a lint rule; `200-error-handling.md` states a decision rule and an error-type design, then expresses them through Ox's `either` blocks and `.catching` and Magnum's `transactEither`. The other chapters are library guides in which a general rule appears only occasionally: generated types converted at the boundary (`340`), opaque types kept in DTOs (`350`), a phantom-typed identifier and proof types (`400`), per-module configuration with a masked secret type (`120`), and tests that trigger batch work themselves (`410`, `500`).
- Upstream's tooling section tells an agent to ensure `build.sbt` carries three `-W` flags and to resolve dependency versions from Maven Central. This repository's `sbt-launcher` skill forbids an agent to change compiler options, dependencies (including `dependsOn` edges) or scalafmt configuration, and tells it to keep the repository's style and let scalafmt format the code under it.
- This repository's `upickle` skill serves consumers on Scala 2.13 as well as Scala 3, so repositories on 2.13 exist in the estate a rules file would be installed in. Four constructs the upstream rules name do not exist on 2.13: braceless syntax, enums, opaque types, givens with extension methods.
- OpenCode reads `AGENTS.md` from the project root (or `~/.config/opencode/AGENTS.md` globally) and every file listed under `instructions` in `opencode.json`, local paths or URLs, and concatenates them all into context on every session. It does not follow file references inside them. Skills under `.opencode/skills/` are loaded on demand, when the agent judges a task relevant. `CLAUDE.md` is read as a fallback when no `AGENTS.md` exists.
- The `skill-collection` spec puts every skill at `skills/<name>/`, forbids a `SKILL.md` at the root, and makes the README list every skill. `openspec/specs/` is flat by design D4 of the archived `multi-skill-layout` change.
- The upstream plugin manifest declares `license: Apache-2.0`; the repository has no LICENSE file.
- Nothing in this repository is executable, so verification is by text checks and review, as in every earlier change.

## Goals / Non-Goals

**Goals:**

- A file an agent can hold in context on every session without cost to its other work: rules only, short, and every rule general enough to be right in any corporate Scala repository.
- No sentence that contradicts the `sbt-launcher` skill, and no name of a build tool, formatter, library or framework.
- Rules an agent can follow in a Scala 2.13 module as well as a Scala 3 one.
- Fidelity to upstream wherever upstream is general, with every departure named here and traceable.

**Non-Goals:**

- Concurrency, resource management, dependency injection, HTTP, persistence, streaming and observability guidance. Upstream's rules on those are tied to its libraries and were ruled out of scope.
- A skill edition of the same text, a plugin manifest, or nesting specs per deliverable.
- Keeping the file in step with upstream automatically. A later change can re-derive from a newer commit.
- Any edit to the two existing skills or their specs beyond the one README requirement.

## Decisions

### D1. A rules file under `rules/`, not a skill

The deliverable is `rules/virtuslab-scala-style.md`: a flat Markdown file with no frontmatter, no links and no supporting files, which a team pastes into a project's `AGENTS.md` or lists under `instructions`. A style guide must be in context whenever Scala is written, and OpenCode gives that only to rules files; a skill is loaded when the agent decides the task matches its description, which for a style rule is every task and therefore the wrong mechanism.

Alternatives considered:

- A skill at `skills/scala-style/SKILL.md`, listed under `instructions` to make it always-on. Rejected: the frontmatter would land in context, the file would carry links it must not have, and the repository would describe as a skill something that is not loaded as one.
- An `AGENTS.md` at this repository's root. Rejected: that would govern agents working on this repository, which holds Markdown, not Scala.
- A directory per rules file, `rules/<name>/`. Rejected: a rules file has nothing beside it by requirement, so the directory would always hold one file.

### D2. The file's shape

In order: a title; a preamble of a few lines stating scope and precedence (Scala code, on Scala 3 and 2.13; the repository's existing conventions and its formatter's output win over the syntax rules; the rules on structure, modelling, errors and tests apply regardless); then five `##` sections, `Syntax and function shape`, `Files, packages and visibility`, `Domain modelling`, `Error handling`, `Tests`; then one closing attribution line. Each rule is one bullet in the imperative, on a single line without hard wrapping so that the line count measures the number of rules, with MUST, NEVER and ALWAYS in capitals as upstream writes them, and says why in a clause when the reason is not obvious. The whole file is between 100 and 150 lines. It holds at most three fenced code examples, chosen for the rules most easily misread: an opaque type with its smart constructor and companion, a two-case enum replacing a Boolean, and an expected exception converted to a value at the boundary. Each example is Scala 3; the 2.13 equivalent is named in the rule's text, not duplicated as a second example.

Alternative considered: rules in one file and worked examples in a second, as upstream splits `SKILL.md` from `180-domain-modeling.md`. Rejected: both files would be in context on every session, since OpenCode concatenates every instructions file and follows no references, so the split would save nothing and cost an install step.

### D3. What each section takes from upstream

| Section | Upstream source | Dropped from that source |
|---|---|---|
| Preamble | New: scope, versions, precedence | |
| Syntax and function shape | `SKILL.md` Coding style (braceless, named responsibilities, one concern per function, chaining, explicit return types); Performance (stream or page); tooling (zero warnings, `@nowarn` policy) | sbt commands, Maven lookup, scalafmt, the three `-W` flags, virtual-thread remedies |
| Files, packages and visibility | `160` whole, except its modules section | sbt `dependsOn` edges, the Scalafix rule |
| Domain modelling | `SKILL.md` Functional programming; `180` whole; `400` (`Id[T]`, proof types); `340` and `350` (boundary conversion, opaque types in DTOs); `120` (per-module config, masked secret, fail at load) | the actor exception for class-level `var`, every codec, the configuration library |
| Error handling | `200` decision rule, typed errors, composing validations, converting exceptions at the boundary, `NonFatal` at a loop boundary | `either` blocks and `.ok()`, `.catching` and `.orThrow`, `transactEither`, the nesting rule, HTTP status mapping, the open `Fail` class |
| Tests | `SKILL.md` (targeted tests); `410` and `500` (tests trigger batch work, no waiting) | stub interpreters, typed request builders, assertion-warning suppression |
| Attribution | New | |

Not carried over at all: chapters `100`, `110`, `130`, `140`, `150`, `170`, `210`, `220`, `300`, `310`, `320`, `330`, `420`, `510`, and the Direct-style Scala section of `SKILL.md`. The `CONTRIBUTING.md` writing guidelines (explain the why, no filler, callouts only for pitfalls, no duplication) govern how the file is written and appear in it nowhere.

### D4. Removing the toolset without losing the rule

A rule survives the removal of its library when the language or the standard library expresses it: `pipe` and `tap` come from `scala.util.chaining`, which upstream imports from Ox; streaming or paging is stated without naming a stream type; converting an expected exception at the boundary is stated with "catch the most specific type and return `Either` or `Option`", which `scala.util.Try` or a targeted `catch` satisfies. A rule whose content is the library's API is dropped (`either` blocks, `Flow`, `Actor`, `computeIntensive`). The file names no build tool, formatter, library or framework anywhere, and a text search for the full list of upstream's names is part of verification (D11).

### D5. Errors are sealed per module

Each module owns a sealed error type, an enum or sealed trait, whose cases carry the failure's data; services return `Either` of it; a caller crossing a module wraps the callee's error in a case of its own type; the transport layer maps each module's type exhaustively. This replaces upstream's open `abstract class Fail`, which modules extend and which forces a wildcard case wherever it is matched. The user chose this; the gains are exhaustiveness checking when a case is added and per-case typed data in place of upstream's `NotFound(what: String)`.

Alternatives considered: upstream's open root (rejected by decision: it gives up what sealed hierarchies are for); one sealed application root with per-module sub-hierarchies in one file (rejected: it couples every module through that file, and still needs the wrapping when modules are compiled apart).

### D6. Version-neutral wording with Scala 3 first

Every rule is written so that it reads correctly on Scala 2.13 and Scala 3. Where a rule names one of the four Scala 3 constructs that 2.13 lacks, the Scala 3 construct is the primary form and a short clause names the 2.13 equivalent: braces for braceless syntax; a `sealed trait` with case objects and case classes in its companion for an `enum`; a value class or tagged type for an opaque type; `implicit` definitions and implicit classes for givens and extension methods. One rule, placed with the syntax rules, states this once for the whole file so that the four clauses stay short. The preamble's precedence clause covers the braces case in a braced Scala 3 repository as well.

Alternatives considered: Scala 3 only, stated in the first line (rejected by decision: the estate has 2.13 repositories, and a file installed in one would mislead the agent); two files, one per version (rejected: duplication of every version-neutral rule).

### D7. The warnings rule names no flags

The rule says: code compiles with zero warnings under the repository's configured compiler options; fix the cause in code; suppress only for generated code or an unfixable third-party issue, narrowly and with a comment; never change the compiler options to silence a warning. Upstream's three flag names and its instruction to add them to `build.sbt` are left out, because the `sbt-launcher` skill forbids the agent to change build settings and because the flags are a project decision.

### D8. Exceptions and conventions not carried over

- The one upstream exception to "no class-level `var`", state inside an actor, is dropped rather than generalised. Without the actor the exception would need a vaguer condition that invites misuse; the rule is cleaner absolute.
- Failing fast on invalid configuration at startup is named as the one sanctioned place a constructor throws. It is the unrecoverable branch of the error rule, and without saying so the configuration rule would appear to contradict the smart-constructor rule.
- The reference project's naming habits visible in upstream's examples (`_IN` and `_OUT` suffixes on DTOs; `*Service`, `*Model`, `*Api`; `final class` for services, used inconsistently) are not adopted. Upstream never states them as rules, and corporate repositories have their own.
- Upstream's module-boundary guidance (build modules only for separate artifacts, `dependsOn` edges first, a lint rule last) is dropped. Module edges count as dependencies the agent may not change under the `sbt-launcher` skill; the visibility rules it keeps are the part the agent can act on.

### D9. The README gains one section and one clause

The opening paragraph says the repository holds skills, installed as directories and loaded on demand, and rules files, loaded whole into the agent's context. A new `## Rules files` section follows the three skill sections and holds: a table with one row per file under `rules/` (name in backticks, one-line summary); the install step for OpenCode, both routes in a few lines (paste the file's content into the project's `AGENTS.md`, or add its path to `instructions` in `opencode.json`), with one sentence that the same text serves any agent that reads a Markdown instructions file; and the convention for adding a rules file in plain words (one flat file `rules/<name>.md`, no frontmatter, no links, scope and precedence stated first, a row in this table in the same change). The three skill sections are unchanged.

Alternative considered: a separate `rules/README.md`. Rejected: the root README is the index by the `skill-collection` spec, and a second index would drift from it.

### D10. Specs describe the agent's behaviour; the file is their statement

Five new capabilities, flat under `openspec/specs/` as D4 of `multi-skill-layout` requires, the four content ones prefixed `scala-` so their owning file is clear: `rules-collection` for layout and install, `scala-code-style`, `scala-code-organisation`, `scala-domain-modelling`, `scala-error-handling` for the rules themselves. As the `upickle` specs do, the content specs state what the agent SHALL do, with the rules file as the text that tells it so; the file's shape (length, examples, attribution, no toolset names) is design, verified by D11, not a requirement on agents. One requirement of `skill-collection` is modified so the README's opening covers both kinds of deliverable.

### D11. Verification by text checks

- `test -f rules/virtuslab-scala-style.md` succeeds and `ls rules` prints only `virtuslab-scala-style.md`.
- `head -1 rules/virtuslab-scala-style.md` starts with `# `, so there is no frontmatter; `grep -cE '\]\(' rules/virtuslab-scala-style.md` prints 0, so there is no Markdown link.
- `wc -l rules/virtuslab-scala-style.md` prints a number between 100 and 150; `grep -c '^```' rules/virtuslab-scala-style.md` prints at most 6 (three fenced examples).
- `grep -c '^## ' rules/virtuslab-scala-style.md` prints 5, and `grep '^## '` lists the five sections of D2 in order; the text before the first `## ` names the versions and the precedence.
- `grep -niE 'sbt|scalafmt|\bOx\b|tapir|sttp|jsoniter|pureconfig|macwire|magnum|flyway|hikari|scalaxb|kafka|opentelemetry|netty|bootzooka|direct-style' rules/virtuslab-scala-style.md` prints nothing.
- `grep -c '2\.13' rules/virtuslab-scala-style.md` prints at least 4, one per construct of D6.
- `tail -1 rules/virtuslab-scala-style.md` names VirtusLab, the upstream repository and `Apache-2.0`.
- `grep -c '^## ' README.md` prints 4; `diff <(ls rules | sed 's/\.md$//') <(awk '/^## Rules files/,0' README.md | grep -oE '^\| `[a-z0-9-]+`' | tr -d '|` ')` prints nothing; the README mentions both `AGENTS.md` and `instructions`.
- `git status --porcelain skills/ openspec/specs/` prints nothing: no skill and no main spec changes in this change.
- A read-through maps every bullet of the file to a row of D3 or a decision D5 to D8, and every requirement of the four content specs to a bullet; a bullet with no source, or a requirement with no bullet, is a defect to fix before archive.
- `openspec validate add-virtuslab-scala-style-rules --strict` passes.

## Risks / Trade-offs

- [The rules drift from upstream as it evolves] → The proposal records the upstream commit; a later change re-derives from a newer one. No automatic sync is attempted.
- [A rule stated absolutely collides with a repository's convention] → The preamble's precedence clause yields on syntax; the design rules still apply, and the read-through checks that each is one an agent can follow in any repository.
- [The toolset search has false positives, such as a common word] → The list in D11 is explicit, and the read-through judges meaning; a hit is reworded or justified, never left.
- [Sealed-per-module errors add wrapping at module boundaries] → Accepted for exhaustiveness; the rule asks for a wrapping case, not a re-modelling of the callee's errors.
- [The 2.13 clauses are noise to a Scala 3 reader] → One clause per construct, four in the file, plus one rule that states the policy.
- [An always-on file grows with every good idea] → The 150-line ceiling is a verification check; growing past it needs a change that argues for it.
- [OpenCode changes how it loads rules] → The README names the two routes as plain file mechanisms and dates nothing; both are the documented behaviour on 2026-10-20.
- [Attribution judged insufficient for an Apache-2.0 derivative] → The file names its source and licence in its last line and is a substantial rewrite; whether the repository itself needs a LICENSE or NOTICE file is an open question below.

## Migration Plan

One commit holding the rules file, the README edit and this change's artifacts. Nothing existing moves, so there is nothing to migrate; rollback is reverting the commit. A team adopts the file by pasting it into a project's `AGENTS.md` or listing it under `instructions`, and drops it the same way.

## Open Questions

- Whether this repository should carry a LICENSE or NOTICE file now that it holds text derived from an Apache-2.0 source. Deferrable: the file's own attribution line meets the need at file level, and a repository licence is a separate decision that does not change the file.
- Whether to also publish the same rules as a skill for agents that have no always-on instructions file. Deferrable: D1 explains why not now, and a skill edition would be its own change.
