# Copilot instructions for TruthDB

## Big picture / repos
- This workspace is a multi-repo org checkout. The cross-repo map and release flow is in [docs/development/specs/TRUTHDB-START-HERE.md](docs/development/specs/TRUTHDB-START-HERE.md).
- Core components: `truthdb/` (Tokio service + systemd unit), `installer/` (initramfs installer), `installer-kernel/` (UEFI kernel config), `installer-iso/` (bootable ISO), `orchestrator/` (release/tagging CLI), `website/` (Vue/Vite), `.github/` (org workflows/docs).
- Architecture baseline: WAL-centric, event-sourced system; WAL is source of truth and snapshots are rebuildable optimizations. See [docs/development/architecture/OVERVIEW.md](docs/development/architecture/OVERVIEW.md).

## Critical flows & integration points
- Installer boot chain: ISO embeds UKI + initramfs; BusyBox `init` launches `truthdb-installer` which partitions, formats, installs Debian payload, and installs `systemd-boot`. Details in [docs/development/specs/TRUTHDB-START-HERE.md](docs/development/specs/TRUTHDB-START-HERE.md).
- Version locking is enforced: `installer-iso` release requires matching `vX.Y.Z` releases from `truthdb`, `installer`, and `installer-kernel`. See [docs/development/specs/TRUTHDB-START-HERE.md](docs/development/specs/TRUTHDB-START-HERE.md).
- `orchestrator release-iso` tags multiple repos and waits for GitHub release assets; requires `GITHUB_TOKEN`/`GH_TOKEN`. See [orchestrator/README.md](orchestrator/README.md).

## Repo-specific conventions
- `truthdb/` is a long-lived Tokio app using `tracing` with `RUST_LOG` for verbosity; shutdown on SIGTERM/SIGINT. See [truthdb/README.md](truthdb/README.md).
- `installer/` runs in initramfs and executes external tools directly (no shell). Disk selection is intentionally strict; destructive steps are gated by a confirmation prompt. Key code: [installer/src/main.rs](installer/src/main.rs), [installer/src/platform/disks.rs](installer/src/platform/disks.rs), [installer/src/platform/partition.rs](installer/src/platform/partition.rs), [installer/src/platform/install.rs](installer/src/platform/install.rs).
- `website/` is Vue 3 + Vite 6 + TypeScript; use npm scripts from [website/README.md](website/README.md).

## Common developer workflows (as documented)
- Build TruthDB: `cd truthdb && cargo build --release`.
- Build installer (musl): `cd installer && rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`.
- Local ISO experiments: `installer-iso/build_and_run.sh` and `installer-iso/run_container.sh`, but the authoritative release steps live in `installer-iso/.github/workflows/release.yml`.
- Website dev: `npm install`, `npm run dev`, `npm run build`, `npm run lint` in `website/`.

## Where to look for source-of-truth docs
- Cross-repo specs/ADRs live under [docs/development](docs/development). Prefer those for architecture, installer, and release mechanics.
- Org-wide GitHub workflows/templates are in [.github/](.github/README.md).
