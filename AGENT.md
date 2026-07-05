# AGENT.md

Working notes for this repo:

## Goal

`r2decomp` is a wrapper around `radare2` that generates pseudo-C quickly for ELF binaries, and `r2decomp-kernel` handles Linux and Android kernel images by reconstructing an ELF first and then decompiling selected symbols.

## Entry points

- `./r2decomp` for existing ELF inputs
- `./r2decomp-kernel` for kernel images, Android boot images, `zImage`, `vmlinuz`, and raw kernel blobs

## Default behavior

- Prefer `pdg` via `r2ghidra` unless the user asks for classic `pdc`
- Default analysis mode is `aa`; `aaa` is slower and should be justified
- `r2decomp-kernel` writes outputs under `out/` by default
- `r2decomp` writes outputs in the current working directory unless `--output-dir` is set

## Editing rules

- Use `apply_patch` for file edits
- Keep shell scripts POSIX-compatible only where they already are bash scripts; do not rewrite them gratuitously
- Preserve ASCII unless a file already uses non-ASCII
- Do not revert or delete user-generated files unless explicitly asked
- Do not use destructive git commands

## Dependency handling

- Fail fast with clear messages when a required tool is missing
- If `pdg` is selected, keep the `r2pm -ci r2ghidra` auto-install path working
- If `vmlinux-to-elf` is missing, `r2decomp-kernel` should bootstrap a local `.venv/`

## Validation

- Run `bash -n r2decomp r2decomp-kernel` after shell edits
- Search for stale names like `decomp.sh`, `kernel_decomp.sh`, or the old repo name after renames
- If behavior changes, prefer a small targeted repro over broad refactors

## Repo hygiene

- Do not commit generated kernel artifacts, test dumps, or the `out/` tree
- The tracked documentation should stay aligned with the renamed scripts and GitHub repo
- If changing the repo metadata, keep the local `origin` pointing at `git@github.com:pdscomp/r2decomp.git`
