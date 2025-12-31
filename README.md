# TruthDB `.github`

This repository contains shared GitHub configuration for the TruthDB organization.

## What belongs here

- Shared GitHub Actions workflows (reusable workflows, composite actions)
- Organization-wide issue/PR templates
- Organization-wide community health files (for example: CONTRIBUTING, SECURITY, Code of Conduct)

## What does not belong here

- Product documentation, architecture docs, or “what is TruthDB” overviews (keep those in the main product repositories and/or the website)

## Project repositories

- `truthdb/` (core)
- `installer/`
- `installer-iso/`
- `installer-kernel/`
- `installer-kernel-builder-image/`
- `orchestrator/`
- `website/`

## Using shared workflows

Workflows in this repo can be referenced from other repos as reusable workflows.

Example:

```yaml
jobs:
	ci:
		uses: Truthdb/.github/.github/workflows/<workflow>.yml@main
```
