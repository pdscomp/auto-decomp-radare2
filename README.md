# auto-decomp-radare2

Command-line pseudo-C decompilation for:

- Linux kernel images, including Android boot images, `zImage`, `vmlinuz`, raw kernel blobs, and `vmlinux` ELF files
- Ordinary ELF binaries such as `.ko` modules

The scripts use `radare2` for analysis and `vmlinux-to-elf` to rebuild a usable kernel ELF when the input is not already ELF.

## Scripts

- `kernel_decomp.sh`: extract or normalize a kernel image, rebuild `vmlinux.elf`, prepare symbols, and export pseudo-C
- `decomp.sh`: take an existing ELF and export pseudo-C for the whole binary

## Requirements

`kernel_decomp.sh` needs:

- `r2`
- `binwalk`
- `python3`
- `file`
- `od`
- `dd`
- `cp`
- `gzip`
- `arm-none-eabi-readelf`

If `r2` is missing, the scripts print:

```bash
git clone https://github.com/radareorg/radare2
cd radare2 ; sys/install.sh
```

If `vmlinux-to-elf` is missing, `kernel_decomp.sh` creates a local `.venv/` and installs it automatically.

## Kernel Workflow

```bash
./kernel_decomp.sh [--kallsyms-file FILE] [--subset-file FILE] [--output-dir DIR] [--kallsyms-remap auto|none|0xOFFSET] <kernel-input>
```

Default behavior:

1. Detect the kernel container format.
2. Extract or copy the kernel payload.
3. Rebuild `vmlinux.elf`.
4. If no external `kallsyms` file is provided, generate `*.embedded-kallsyms.txt` from the rebuilt ELF symbol table.
5. Select code symbols, run `radare2`, and write a combined `*.pdc.c` output.

This means the simplest path is now:

```bash
./kernel_decomp.sh kernel-01.03.01.89
```

That produces a reconstructed ELF plus an embedded-symbol pseudo-C export under `out/`.

### External `kallsyms`

If you have runtime symbols from the live device, pass them explicitly:

```bash
./kernel_decomp.sh --kallsyms-file kallsyms-20260704-092316.txt kernel-01.03.01.89
```

That file usually comes from the running kernel:

```bash
cat /proc/kallsyms > kallsyms.txt
```

External `kallsyms` is useful when the running kernel exposes more names than the rebuilt ELF, or when you want runtime naming from the target system.

### Subset File

If you only want a smaller set of functions, pass `--subset-file`:

```bash
./kernel_decomp.sh --subset-file kallsyms-sitronix-20260704-092316.txt kernel-01.03.01.89
```

Or combine both:

```bash
./kernel_decomp.sh \
  --kallsyms-file kallsyms-20260704-092316.txt \
  --subset-file kallsyms-sitronix-20260704-092316.txt \
  kernel-01.03.01.89
```

`--subset-file` can be either:

- one symbol name per line
- full `kallsyms` lines

When a subset file contains full `kallsyms` lines, only the symbol names are used for filtering. In embedded-symbol mode this is the main point: the addresses from the subset file are ignored and names are looked up in the rebuilt kernel ELF.

If the symbol source contains the same name at multiple addresses, one subset name can expand to multiple decompilations.

### Kallsyms Remap

When using an external runtime `kallsyms` file, addresses may need to be shifted back to file addresses. The script supports:

- `--kallsyms-remap auto`
- `--kallsyms-remap none`
- `--kallsyms-remap 0xOFFSET`

Examples:

```bash
./kernel_decomp.sh --kallsyms-file kallsyms.txt --kallsyms-remap auto kernel-01.03.01.89
./kernel_decomp.sh --kallsyms-file kallsyms.txt --kallsyms-remap none kernel-01.03.01.89
./kernel_decomp.sh --kallsyms-file kallsyms.txt --kallsyms-remap 0x4000000 kernel-01.03.01.89
```

`auto` is the default. It tries anchor symbols such as `_stext` first, then falls back to majority matching across shared symbol names. If confidence is weak, it falls back to no remap instead of forcing a bad offset.

In embedded-symbol mode, remapping is disabled because those addresses already come from the rebuilt ELF.

### Kernel Outputs

Common outputs under `out/...`:

- `*.vmlinux.elf`
- `*.vmlinux-to-elf.log`
- `*.embedded-kallsyms.txt` or `*.normalized.txt`
- `*.report.txt`
- `*.selected-symbols.txt`
- `*.skipped-symbols.txt`
- `*.pdc.c`
- `*.open-r2.sh`

`*.report.txt` is the main place to check symbol counts, remap details, and why symbols were skipped.

## ELF Workflow

```bash
./decomp.sh <elf-binary>
```

Example:

```bash
./decomp.sh out/test-default-embedded/kernel-01.03.01.89.vmlinux.elf
```

This script:

1. Verifies the input is ELF.
2. Runs `radare2` analysis once.
3. Enumerates discovered functions.
4. Writes a single combined pseudo-C file next to the ELF.

Outputs:

- `*.functions.json`
- `*.decompile-all.r2`
- `*.pdc.c`

`decomp.sh` does not need a Python virtualenv because it does not rebuild the binary; it only drives `radare2` against an existing ELF.

## Recommended Use

- Use `kernel_decomp.sh <kernel-input>` for the default end-to-end kernel path.
- Add `--subset-file` when you only care about one driver or subsystem.
- Add `--kallsyms-file` when you have `/proc/kallsyms` from the live target and want runtime naming or remap support.
- Use `decomp.sh <elf>` when you already have the final ELF and want a whole-binary pseudo-C dump.

## TODO

- Support more complex kernel address remapping than a single additive offset when a target uses address translation beyond simple KASLR-style shifts.
