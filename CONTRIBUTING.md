# Contributing

Thanks for your interest in contributing to TruthDB.

## Where to contribute

- Bugs, tasks, and design discussions: GitHub Issues
- General questions and longer-form discussion: GitHub Discussions (if enabled)

## Before you open an issue

- Search existing issues to avoid duplicates.
- Include what you expected to happen and what happened instead.
- Include logs, screenshots, or minimal repro steps when possible.

## Pull requests

- Keep PRs focused (one logical change per PR).
- Include a clear description of the change and why it’s needed.
- Prefer adding/adjusting tests when behavior changes.
- Follow the project’s existing style and conventions.

### Rust changes

- Format: `cargo fmt`
- Lint: `cargo clippy --all-targets --all-features`
- Test: `cargo test`

### Website changes

- Lint: `npm run lint` (or `pnpm run lint` depending on the repo)
- Typecheck: `npm run typecheck` (if present)
- Build: `npm run build`

## Commit messages

Write commit messages that are clear and describe the intent of the change.

## License

By contributing, you agree that your contributions will be licensed under the license in the target repository.
