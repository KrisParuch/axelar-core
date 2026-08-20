# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`axelar-core` is the main Cosmos SDK application for the Axelar network — it builds the `axelard` binary and docker image that runs an Axelar core node. This working copy is a fork:

- `origin` → `git@github.com:krisparuch/axelar-core.git`
- `upstream` → `https://github.com/axelarnetwork/axelar-core.git`

The commit history is overwhelmingly upstream Axelar history (standard `fix:`/`feat:`/`chore:` commits with PR numbers in the `#23xx` range, authored by the Axelar team). **Treat this repo as tracking upstream, not a place for casual custom feature work**, unless you're specifically continuing the fork-local changes described below.

## Fork-local divergence

A small cluster of commits with low PR numbers (`#21`, `#22`, `#23`) sit on top of upstream and are fork-specific, not from `axelarnetwork/axelar-core`:

- `a18979d0 chore(cometbft): replace public release with patched cometbft-sec-tachyon (#21)`
- `3249bf6f fix(docker): use build secrets for cometbft-tachyon PAT instead of global git config (#22)`
- `7de3621e fix(docker): pass COMETBFT_TACHYON_PAT token in debug docker image (#23)`
- `12ff4e79 fix(workflow): pass COMETBFT_TACHYON_PAT to the debug docker image's env`
- `054d2ef2 fix(ci): pass COMETBFT_TACHYON_PAT when building docker image for release`

These swap the public CometBFT dependency for a private, patched `cometbft-sec-tachyon` release in `go.mod`/`go.sum`, and thread a `COMETBFT_TACHYON_PAT` (personal access token) secret through CI workflows and Docker builds so they can pull it. If you touch `go.mod`, CI workflows, or the Dockerfiles, be aware this dependency swap exists and don't accidentally revert it back to the public CometBFT release during an upstream sync/rebase without checking whether the patched dependency is still required.

Before merging upstream changes, diff against `upstream/main` (add/fetch the remote if not already fetched locally) to see what's fork-specific vs. inherited, rather than assuming `origin/main` == upstream.

## Commands

```powershell
make build-static     # recommended — Docker-based, statically linked, portable axelard binary in ./bin (no glibc dependency)
make build              # dynamic binary via `go build`, depends on system glibc
make docker-image       # builds axelar/core:latest image
make docker-image-debug  # debug image with delve
make lint                 # golangci-lint run && go mod verify
go test ./... -race -coverprofile=coverage.txt -covermode=atomic   # as run in CI (ci-test.yaml); no `make test` target exists
```

`make build-static` requires Docker (it builds via `docker-image` then extracts the binary — needed because static linking with WASM support requires musl libc + `libwasmvm_muslc.a`, which the Makefile gets from an Alpine-based Docker build rather than the host toolchain).

Smart-contract bytecode dependency: building/running locally also requires importing gateway contract bytecode — see `contract-version.json` for the required `axelar-cgp-solidity` release, download it, unzip under `contract-artifacts/`, then `make generate`. Proto/mock generation: `make generate` (runs `prereqs`, `docs`, `generate-mocks`).

## Release process

Release tags are checked out directly (`git checkout vX.Y.Z`) before building for a release build — see `RELEASE.md` for the full process and `CHANGELOG.md` for release history.
