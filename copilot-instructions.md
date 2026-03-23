# Copilot instructions for TruthDB

## Big picture / repos
- This workspace is a multi-repo org checkout.
- Core components: `truthdb/` (Tokio service + systemd unit), `installer/` (initramfs installer), `installer-kernel/` (UEFI kernel config), `installer-iso/` (bootable ISO), `orchestrator/` (release/tagging CLI), `website/` (Vue/Vite), `.github/` (org workflows/docs).
- Current storage design work lives in `docs/development/specs/STORAGE.md`.
- Cross-repo architecture and historical/product context now live under `docs/development/` and `docs/product/`.

## Critical flows & integration points
- Installer boot chain: ISO boots through `GRUB`, loads the installer kernel plus initramfs, and the custom initramfs `/init` launches `truthdb-installer`. The installer partitions, formats, installs the Debian payload, and installs `systemd-boot`.
- `installer-iso` release pulls the latest published `truthdb`, `installer`, and `installer-kernel` artifacts available at build time.
- `orchestrator release-iso` tags multiple repos and waits for GitHub release assets; local token support is documented in `orchestrator/README.md`.

## Repo-specific conventions
- `truthdb/` is a long-lived Tokio app using `tracing` with `RUST_LOG` for verbosity; shutdown on SIGTERM/SIGINT.
- `installer/` runs in initramfs and executes external tools directly (no shell). Disk selection is intentionally strict; destructive steps are gated by a confirmation prompt.
- `website/` is Vue 3 + Vite 6 + TypeScript.

## Common developer workflows (as documented)
- Build TruthDB: `cd truthdb && cargo build --release`.
- Build installer (musl): `cd installer && rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`.
- Local ISO experiments: use `installer-iso/build_in_container.sh` for normal local work and `INPUT_MODE=release installer-iso/build_in_container.sh` for release-like local inputs. For a local kernel + ISO chain, use `installer-kernel/build_iso_with_local_kernel.sh`.
- Website dev: `npm install`, `npm run dev`, `npm run build`, `npm run lint` in `website/`.

## Where to look for source-of-truth docs
- Prefer repo READMEs and GitHub workflow files for current operational behavior.
- Use `docs/development/specs/STORAGE.md` for the active storage design draft.
- Use `docs/development/` and `docs/product/` for current cross-repo docs.
