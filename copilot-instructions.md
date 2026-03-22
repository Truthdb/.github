# Copilot instructions for TruthDB

## Big picture / repos
- This workspace is a multi-repo org checkout.
- Core components: `truthdb/` (Tokio service + systemd unit), `installer/` (initramfs installer), `installer-kernel/` (UEFI kernel config), `installer-iso/` (bootable ISO), `orchestrator/` (release/tagging CLI), `website/` (Vue/Vite), `.github/` (org workflows/docs).
- Current storage design work lives in `docs/STORAGE.md`.
- Historical cross-repo context remains under `docs/obsolete/`.

## Critical flows & integration points
- Installer boot chain: ISO embeds a UKI + initramfs; BusyBox `init` launches `truthdb-installer`, which partitions, formats, installs the Debian payload, and installs `systemd-boot`.
- `installer-iso` release pulls the latest published `truthdb`, `installer`, and `installer-kernel` artifacts available at build time.
- `orchestrator release-iso` tags multiple repos and waits for GitHub release assets; local token support is documented in `orchestrator/README.md`.

## Repo-specific conventions
- `truthdb/` is a long-lived Tokio app using `tracing` with `RUST_LOG` for verbosity; shutdown on SIGTERM/SIGINT.
- `installer/` runs in initramfs and executes external tools directly (no shell). Disk selection is intentionally strict; destructive steps are gated by a confirmation prompt.
- `website/` is Vue 3 + Vite 6 + TypeScript.

## Common developer workflows (as documented)
- Build TruthDB: `cd truthdb && cargo build --release`.
- Build installer (musl): `cd installer && rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`.
- Local ISO experiments: `installer-iso/build_and_run.sh` and `installer-iso/run_container.sh`, but the authoritative release steps live in `installer-iso/.github/workflows/release.yml`.
- Website dev: `npm install`, `npm run dev`, `npm run build`, `npm run lint` in `website/`.

## Where to look for source-of-truth docs
- Prefer repo READMEs and GitHub workflow files for current operational behavior.
- Use `docs/STORAGE.md` for the active storage design draft.
- Treat `docs/obsolete/` as historical context rather than a guaranteed current source of truth.
