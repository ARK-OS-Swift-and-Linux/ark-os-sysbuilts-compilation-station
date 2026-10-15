# ARK-OS Sysbuilts Compilation Station

## Overview

The **Sysbuilts Compilation Station** is an automated CI/CD build subsystem dedicated to compiling, packaging, and distributing the base C standard library (`musl libc`) and high-performance memory allocator (`mimalloc`) for the ARK-OS target execution environment.

It produces a standalone, reproducible target sysroot for `x86_64-unknown-linux-musl`, ensuring that ARK-OS components (such as `arkrt` and `libark`) compile and link against the project-owned target environment rather than host runtime libraries.

---

## Architecture and Automation

### 1. Upstream Source Tracking
- **musl libc:** Tracks upstream `v1.*` releases (default: `v1.2.5`).
- **mimalloc:** Tracks upstream `v2.*` releases (default: `v2.1.9`).
- **Triggering:** Runs on a weekly schedule, on push to `main`/`master`, or manually via `workflow_dispatch`.

### 2. Build Pipeline
The workflow in `.github/workflows/compile-sysbuilts.yml`:
1. Clones `musl libc` at the designated release tag.
2. Configures `musl` with `--prefix=/ --syslibdir=/lib --enable-wrapper=all` for `x86_64`.
3. Compiles `libc.a`, `libc.so`, the dynamic linker `ld-musl-x86_64.so.1`, and ELF startup objects (`crt1.o`, `crti.o`, `crtn.o`, `rcrt1.o`, `Scrt1.o`).
4. Stages the compiled musl libraries and C headers into `dist/sysroot`.
5. Clones `mimalloc` and builds it against the newly staged musl sysroot (`musl-gcc`).
6. Installs static library `libmimalloc.a`, shared library `libmimalloc.so`, override object `mimalloc.o`, and headers (`mimalloc.h`, `mimalloc-override.h`) into `dist/sysroot`.
7. Validates sysroot integrity and generates `sysbuilts-metadata.json`.
8. Compresses the sysroot into `sysbuilts-x86_64.tar.xz`.
9. Calculates `SHA256SUMS.txt` cryptographic hashes and publishes a GitHub Release.

---

## Sysroot Structure

When the published `sysbuilts-x86_64.tar.xz` is extracted, it provides the following canonical sysroot layout:

```text
sysroot/
├── bin/
│   └── musl-gcc                 # GCC spec wrapper preconfigured for this sysroot
├── include/
│   ├── stdio.h, unistd.h, ...   # Complete musl standard C headers
│   ├── sys/                     # POSIX and Linux system headers
│   └── mimalloc.h               # mimalloc allocator C/C++ API headers
├── lib/
│   ├── libc.a                   # Static musl standard library
│   ├── libc.so                  # Dynamic musl standard library
│   ├── ld-musl-x86_64.so.1      # Dynamic linker symlink to libc.so
│   ├── crt1.o, crti.o, crtn.o   # Standard ELF executable startup objects
│   ├── rcrt1.o, Scrt1.o         # Position-independent (PIE) startup objects
│   ├── libmimalloc.a            # Static mimalloc library
│   ├── libmimalloc.so           # Dynamic mimalloc library
│   └── mimalloc.o               # Linkable override object
├── sysbuilts-metadata.json      # Machine-readable provenance and version manifest
└── version.txt                  # Release tag string
```

---

## Reassembling and Consuming Artifacts

ARK-OS provides an automated synchronization script (`components/boot/scripts/sync.sh`) that discovers, downloads, verifies, and installs sysbuilt releases directly into the local build environment:

```bash
# In the ARK-OS repository:
./components/boot/scripts/sync.sh sysbuilts
```

To manually download and extract the sysroot:

```bash
# 1. Download release assets
curl -LO https://github.com/ARK-OS-Swift-and-Linux/ark-os-sysbuilts-compilation-station/releases/latest/download/sysbuilts-x86_64.tar.xz
curl -LO https://github.com/ARK-OS-Swift-and-Linux/ark-os-sysbuilts-compilation-station/releases/latest/download/SHA256SUMS.txt

# 2. Verify archive integrity
sha256sum -c SHA256SUMS.txt

# 3. Extract into target directory
mkdir -p build/sysroot
tar -xf sysbuilts-x86_64.tar.xz -C build/sysroot
```

---

## License

This repository and build station assets are licensed under the Apache License, Version 2.0. See [LICENSE](file:///home/aarav/repo/Stations/Sysbuilts/LICENSE) for details. The compiled `musl libc` is licensed under the standard MIT License. The compiled `mimalloc` library is licensed under the MIT License.
