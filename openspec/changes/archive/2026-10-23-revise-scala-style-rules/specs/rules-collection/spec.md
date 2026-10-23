# Spec Delta

## ADDED Requirements

### Requirement: The README names a rules file's required libraries as an install precondition
For a rules file whose rules rely on libraries, the root `README.md` SHALL say, in its install step, that every module the file will govern must declare those libraries, and set any compiler option one of them needs, before the file is installed, and the file's summary row SHALL name the libraries.

#### Scenario: Installing a file that requires libraries
- **WHEN** a reader installs `virtuslab-scala-style` from the README
- **THEN** the README has told them which libraries each governed module must declare first, and which compiler option enumeratum's value enumerations need on Scala 3, and the summary row names the libraries

#### Scenario: A rules file without library rules
- **WHEN** a rules file relies on no library
- **THEN** its row and the install step say nothing about libraries

## MODIFIED Requirements

### Requirement: A rules file states its scope and precedence first
Before its first rule, a rules file SHALL state what it applies to (the language, the language versions, and the kind of code), what wins when one of its rules conflicts with the repository it is loaded in, and, when its rules rely on libraries, which libraries every governed module must declare, by organisation and artifact, together with any compiler option one of them needs, and what the agent does when a module lacks one.

#### Scenario: Opening a rules file
- **WHEN** an agent reads the file from the top
- **THEN** it learns the scope and the precedence before it meets any rule

#### Scenario: A rule conflicts with the repository
- **WHEN** a rule contradicts a convention the repository's code already follows
- **THEN** the agent knows from the opening lines which one to follow, without reading further

#### Scenario: A rule relies on a library the module lacks
- **WHEN** the agent reaches a rule that needs a library the current module does not declare
- **THEN** it already knows from the opening lines that it asks for the library, by organisation and artifact, and does not substitute another form
