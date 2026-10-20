# skill-collection Specification

## Purpose

Defines how skills are organised in this repository so that several can coexist and be installed independently: one directory per skill under `skills/`, each self-contained, and a root README that lists them.

## Requirements

### Requirement: Every skill lives in its own directory under `skills/`
Each skill in this repository SHALL be a directory `skills/<name>/` that contains a `SKILL.md` with YAML frontmatter, where `<name>` is equal to the frontmatter `name`. The repository root SHALL NOT itself contain a `SKILL.md`, and no skill directory SHALL exist anywhere other than directly under `skills/`.

#### Scenario: The existing skill
- **WHEN** the repository is checked out after this change
- **THEN** `skills/corp-scala/SKILL.md` exists with `name: corp-scala` in its frontmatter, and no `corp-scala/` directory exists at the repository root

#### Scenario: A second skill is added
- **WHEN** a new skill with frontmatter `name: <name>` is added to the repository
- **THEN** it is created as `skills/<name>/SKILL.md`, and no file of any existing skill changes because of the addition

#### Scenario: The root is not a skill
- **WHEN** an agent or a person inspects the repository root
- **THEN** they find no `SKILL.md` there, so the repository as a whole is not loadable as a single skill

### Requirement: A skill is self-contained within its directory
Every file a skill needs (its `SKILL.md`, and any `references/`, `scripts/`, or `assets/`) SHALL live inside that skill's directory, and every link or path inside the skill SHALL be relative to a location inside that directory. A skill SHALL NOT link to or depend on files outside its directory, including files of other skills, so that copying `skills/<name>/` alone installs a working skill.

#### Scenario: Installing one skill
- **WHEN** `skills/<name>/` is copied, on its own, into an agent's skills directory
- **THEN** every link in its `SKILL.md` and supporting files resolves within the copy, and the skill loads without anything else from this repository

#### Scenario: Links inside the existing skill
- **WHEN** `skills/corp-scala/SKILL.md` refers to its reference document and the reference refers back
- **THEN** the links are `references/sbt-launcher.md` and `../SKILL.md`, both resolving inside `skills/corp-scala/`

#### Scenario: Shared material between skills
- **WHEN** two skills need the same supporting text
- **THEN** each keeps its own copy inside its directory rather than linking across skill directories

### Requirement: The root README lists every skill
The repository root SHALL contain a `README.md` that states that the repository is a collection of skills, lists every directory under `skills/` by name with a one-line summary, tells the reader how to install a skill (copy `skills/<name>/` into the agent's skills directory), and states the convention a new skill must follow.

#### Scenario: Finding the skills
- **WHEN** a reader opens `README.md`
- **THEN** they see every skill under `skills/` listed by name with a summary, and no skill present in `skills/` is missing from the list

#### Scenario: A skill is added
- **WHEN** a new directory is added under `skills/`
- **THEN** `README.md` gains an entry for it in the same change

#### Scenario: Installing from the README
- **WHEN** a reader follows the README's install instruction for `corp-scala`
- **THEN** they copy `skills/corp-scala/` into their agent's skills directory and the installed directory is named `corp-scala`
