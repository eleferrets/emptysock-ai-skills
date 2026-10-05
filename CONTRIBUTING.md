# Contributing

> **Archived.** EmptySock development has stopped and this repository is no longer maintained. The notes below are kept for reference; issues and pull requests are not being reviewed.

This is the public documentation repository for EmptySock.

## What you can contribute

- **Typo / accuracy fixes** — open a PR directly
- **Missing documentation** — open an issue describing what's missing
- **New skill file content** — open an issue first to discuss scope
- **Translation** — documentation in languages other than English is welcome

## What belongs here vs. the engine repo

This repo contains only:
- Public API documentation (what the engine exposes)
- Usage patterns and examples
- AI agent reference files

It does not contain:
- Engine source code
- Internal implementation details
- Build tooling

## Reporting a documentation error

Open an issue with:
1. The file and section that is wrong
2. What it currently says
3. What it should say
4. Your engine version (`emptysock.project.json > version`)

## Branch and versioning

- The pack is frozen at the engine's final state (see `CHANGELOG.md`)
- No further engine releases or version tags are planned
