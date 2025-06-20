# Cline Configuration Repository

This repository contains the configuration files for Cline, an AI software engineer. These files define Cline's behavior, rules, and workflows, ensuring consistent and efficient operation.

## Structure

The repository is organized into three main directories:

- `Rules/`: Contains Markdown files that define the core principles and constraints that Cline must follow.
- `Workflows/`: Contains Markdown files that outline specific, detailed processes for common tasks.
- `memory-bank/`: Stores files that help Cline maintain context and knowledge across sessions.

## Memory Bank

The `memory-bank/` directory is crucial for Cline's operation, as Cline's memory resets between sessions. It contains the following core files, which are based on the `Rules/memory-bank.md` and `Rules/cline-continuous-improvement-protocol.md` guidelines:

- `projectbrief.md`: Defines the project's core requirements, goals, and scope.
- `productContext.md`: Explains why the project exists, the problems it solves, and user experience goals.
- `activeContext.md`: Tracks the current work focus, recent changes, next steps, and active decisions.
- `systemPatterns.md`: Describes the system architecture, key technical decisions, and design patterns.
- `techContext.md`: Details the technologies used, development setup, and technical constraints.
- `progress.md`: Logs completed work, work in progress, known issues, and the evolution of project decisions.
- `consolidated_learnings.md`: Stores curated, actionable insights derived from `raw_reflection_log.md`.
- `raw_reflection_log.md`: Contains detailed, timestamped raw entries from task reviews and analyses.

## Rules

The `Rules/` directory includes the following files:

- `cline-continuous-improvement-protocol.md`: Defines a mandatory protocol for self-reflection, knowledge capture, and continuous improvement. It requires Cline to log learnings and difficulties to a `raw_reflection_log.md` and consolidate them into a `consolidated_learnings.md` file.
- `code-style.md`: Specifies the coding style guidelines that Cline must adhere to, including SOLID principles, commenting practices, and API documentation standards.
- `context-watch.md`: Provides a mandatory strategy for managing the context window, including when to initiate a task handoff using the `new_task` tool.
- `gh.md`: Confirms that Cline has access to the GitHub CLI (`gh`) and is authenticated.
- `git.md`: Outlines specific rules for using Git, such as including a trailing empty line in source code files and using the `--no-pager` argument.
- `mcp-development.md`: Describes how to restart MCP servers.
- `markdown-style.md`: Specifies that lists in markdown should use a single space after the hyphen.
- `memory-bank.md`: Explains the structure and purpose of Cline's Memory Bank, which is essential for maintaining context between sessions.
- `pull-request.md`: Defines the rules for creating pull requests, including following templates and creating them as drafts.

## Workflows

The `Workflows/` directory includes the following file:

- `pr-review-cline.md`: A detailed workflow for reviewing pull requests within the Cline project, utilizing the `gh` CLI and providing a step-by-step process for analysis, user confirmation, and decision-making.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details.
