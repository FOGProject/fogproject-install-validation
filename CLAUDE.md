# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A GitHub Actions-only repo (no application code) that validates the FOG Project (`FOGProject/fogproject`, `dev-branch`) installs cleanly on a matrix of Linux distros before a stable release. There is nothing to build, lint, or unit-test locally — the only way to "run" this project is to trigger the workflows on GitHub.

## Architecture

- **`.github/workflows/reusable_distro_workflow.yml`** — the actual test logic, invoked via `workflow_call` by every per-distro workflow. It:
  1. Ensures `crun` is >= 1.14.3 (apt on the `ubuntu-24.04` runner is often too old), upgrading via apt or, failing that, downloading a release from the `containers/crun` GitHub API (with a pinned fallback URL if the API call is rate-limited/fails).
  2. Derives a distrobox container name from the `distro` input (last path segment, `:` replaced with `_`).
  3. Installs Distrobox and creates a rootless-but-`--root` container from the given image, adding `git systemd <package_man> sudo` plus `libpam-systemd` when `package_man` is `apt`, plus any `additional_packages` input.
  4. Clones `FOGProject/fogproject` (`dev-branch`) and runs `bin/installfog.sh -y` inside the container as root.
- **`.github/workflows/distro_*.yml`** — one thin wrapper per distro/version (e.g. `distro_fedora_44.yml`, `distro_ubuntu_24_04.yml`). Each just sets `name`, triggers (`workflow_call` + `workflow_dispatch`), and calls the reusable workflow with `distro` (an OCI image ref), `package_man` (`apt` or `dnf`), and optionally `additional_packages` (e.g. Fedora needs `gawk`).
- **`.github/workflows/run_all_distros.yml`** — orchestrator, `workflow_dispatch`-only. Builds a matrix by listing every `distro_*.yml` file in `.github/workflows/`, dispatches each one via `gh workflow run`, then polls `gh run list` / `gh run view` for each to reach a terminal `conclusion` and fails the job if any distro run fails.

## Adding a new distro

Copy an existing `distro_*.yml` that uses the same package manager, update `name`, the job id, and the `with:` block (`distro` image ref, `package_man`, optional `additional_packages`). No other file needs to change — `run_all_distros.yml` discovers new `distro_*.yml` files automatically by globbing the directory.

## Conventions

- Every distro workflow supports both `workflow_call` (so `run_all_distros.yml` / other workflows can invoke it) and `workflow_dispatch` (so it can be run standalone from the Actions tab).
- `package_man` is only ever `"apt"` or `"dnf"`; the reusable workflow branches on this string directly rather than using a lookup table.
- Runners are pinned to `ubuntu-24.04`.
