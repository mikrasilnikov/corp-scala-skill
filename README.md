# Agent skills

This repository is a collection of agent skills and rules files. A skill is a directory `skills/<name>/` in the `SKILL.md` format that holds every file it needs; it is installed as a directory and loaded on demand, when the agent judges a task relevant to it. A rules file is a single Markdown file `rules/<name>.md` that is loaded whole into the agent's context on every session. The `.claude/` directory holds the OpenSpec tooling for working on this repository; it contains no skills to install.

## Skills

| Skill | Summary |
| --- | --- |
| `sbt-launcher` | Building, testing, formatting, and changing dependencies in corporate Scala projects built with sbt. |
| `upickle` | Converting JSON Schema documents, as produced by upickle-jsonschema, into case classes that upickle reads directly. |

## Installing a skill

Copy the skill's directory, `skills/<name>/`, into your agent's skills directory. The installed directory keeps the skill's name: `skills/sbt-launcher/` installs as `sbt-launcher/`. For Claude Code, for example:

```bash
mkdir -p ~/.claude/skills
cp -R skills/sbt-launcher ~/.claude/skills/
```

This creates `~/.claude/skills/sbt-launcher/SKILL.md`. To install a skill for one project only, copy it into that project's `.claude/skills/` instead.

If you installed this skill under its former name, delete that `corp-scala/` directory from your skills directory: left beside `sbt-launcher/`, it names a launcher that no longer exists.

## Adding a skill

- Create the directory `skills/<name>/` with a `SKILL.md` whose frontmatter `name` is `<name>`. Skills go nowhere else: not at the repository root, and not under `.claude/`, which is tooling.
- Keep every file the skill needs (`references/`, `scripts/`, `assets/`) inside that directory, and make every link relative to it. Do not link into another skill; if two skills need the same text, each keeps its own copy.
- Add a row for the skill to the table above, with a one-line summary of its `description`, in the same change.

## Rules files

| Rules file | Summary |
| --- | --- |
| `virtuslab-scala-style` | Code style, file and package layout, domain modelling, error handling and test style for Scala 3 and Scala 2.13 code, adapted from VirtusLab's scala-skill without its build tools and libraries. |

A rules file is installed per project and applies to every session in it. To install one in OpenCode, take either route:

- Paste the file's content into the project's `AGENTS.md`.
- Copy the file into the project, for example as `rules/virtuslab-scala-style.md`, and add that path to `instructions` in the project's `opencode.json`:

```json
{
  "instructions": ["rules/virtuslab-scala-style.md"]
}
```

The rules are in the agent's context from the project's next session on. The same text works in any agent that reads a Markdown instructions file.

To add a rules file:

- Create one flat file `rules/<name>.md` with a kebab-case `<name>`: no frontmatter, no subdirectory, no supporting files, and no links, since the file is read on its own wherever it is pasted.
- State in its first lines what it applies to and what wins when one of its rules conflicts with the repository it is loaded in.
- Add a row for it to the table above in the same change.
