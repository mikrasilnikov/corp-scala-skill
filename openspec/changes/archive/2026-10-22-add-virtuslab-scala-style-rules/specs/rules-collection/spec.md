# Spec Delta

## Purpose

Defines how rules files are organised in this repository and installed into an agent: one self-contained Markdown file per rules file, directly under `rules/`, meant to be loaded whole into the agent's context rather than on demand, and a root README that lists them with the OpenCode install step.

## ADDED Requirements

### Requirement: Every rules file is one Markdown file directly under `rules/`
Each rules file in this repository SHALL be a single Markdown file `rules/<name>.md` with a kebab-case `<name>`. No rules file SHALL exist anywhere else, and `rules/` SHALL contain nothing but rules files: no subdirectories and no supporting files.

#### Scenario: The first rules file
- **WHEN** the repository is checked out after this change
- **THEN** `rules/virtuslab-scala-style.md` exists, and `rules/` contains no other entry

#### Scenario: A second rules file is added
- **WHEN** a new rules file named `<name>` is added
- **THEN** it is created as `rules/<name>.md`, and no existing rules file or skill changes because of the addition

#### Scenario: A rules file is not a skill
- **WHEN** an agent or a person inspects `rules/`
- **THEN** they find no `SKILL.md` and no directory there, so nothing under `rules/` is loadable as a skill, and `skills/` is unchanged by the presence of rules files

### Requirement: A rules file is self-contained and has no frontmatter
A rules file SHALL begin with a Markdown heading rather than YAML frontmatter, SHALL NOT link to or depend on any other file in this repository, and SHALL read correctly when its text alone is pasted into an agent's instructions file or when that file is listed as one.

#### Scenario: Pasted into an instructions file
- **WHEN** a person pastes the whole text of a rules file into a project's `AGENTS.md`
- **THEN** every sentence still makes sense in place, no link in it dangles, and no frontmatter block appears in the agent's context

#### Scenario: Listed as an instructions file
- **WHEN** a project lists the file's path under `instructions` in `opencode.json`
- **THEN** the file loads whole and needs nothing else from this repository

#### Scenario: Text shared with a skill
- **WHEN** a rules file and a skill need the same supporting text
- **THEN** each keeps its own copy rather than linking to the other

### Requirement: A rules file states its scope and precedence first
Before its first rule, a rules file SHALL state what it applies to (the language, the language versions, and the kind of code) and what wins when one of its rules conflicts with the repository it is loaded in.

#### Scenario: Opening a rules file
- **WHEN** an agent reads the file from the top
- **THEN** it learns the scope and the precedence before it meets any rule

#### Scenario: A rule conflicts with the repository
- **WHEN** a rule contradicts a convention the repository's code already follows
- **THEN** the agent knows from the opening lines which one to follow, without reading further

### Requirement: The root README lists every rules file
The repository root `README.md` SHALL contain a section that lists every file under `rules/` by name with a one-line summary, tells the reader how to install a rules file in OpenCode (paste its content into the project's `AGENTS.md`, or list its path under `instructions` in `opencode.json`), and states the convention a new rules file must follow.

#### Scenario: Finding the rules files
- **WHEN** a reader opens `README.md`
- **THEN** they see every file under `rules/` listed by name with a summary, and no file present in `rules/` is missing from the list

#### Scenario: A rules file is added
- **WHEN** a new file is added under `rules/`
- **THEN** `README.md` gains an entry for it in the same change

#### Scenario: Installing from the README
- **WHEN** a reader follows the README's install step for `virtuslab-scala-style`
- **THEN** the file's rules are in the agent's context on the project's next OpenCode session, by either of the two routes the README names
