# Python Static Build for Android

Cross-compile CPython for Android (ARM64/aarch64) using GitHub Actions.

## Build Modes

| Mode | Description | Binary Type |
|------|-------------|-------------|
| **static** | Manually patched build with `-static` linker flags. Produces a standalone binary with no `.so` dependencies. | `ELF 64-bit, statically linked` |
| **shared** | Uses the official `android.py` script. Produces `python3` + `libpython3.x.so`. | `ELF 64-bit, dynamically linked` |

## Quick Start

### Trigger a Build (GitHub Actions)

1. Go to **Actions** → **Build Python for Android**
2. Click **Run workflow**
3. Choose parameters:
   - **CPython branch**: `3.14` (default), `3.13`, `main`, etc.
   - **Host triplet**: `aarch64-linux-android` (default)
   - **Build mode**: `static` or `shared`
4. Download the artifact from the completed run

### Supported Architectures

| Triplet | Android ABI | Notes |
|---------|-------------|-------|
| `aarch64-linux-android` | `arm64-v8a` | Modern 64-bit ARM (recommended) |
| `arm-linux-androideabi` | `armeabi-v7a` | Legacy 32-bit ARM |
| `x86_64-linux-android` | `x86_64` | Emulator (64-bit) |
| `i686-linux-android` | `x86` | Emulator (32-bit) |

## Deploy to Android

```bash
# Download artifact from GitHub Actions, then:
adb push python3-aarch64 /data/local/tmp/python3
adb shell chmod +x /data/local/tmp/python3
adb shell /data/local/tmp/python3 --version
```

### Verify binary type

```bash
adb shell file /data/local/tmp/python3
# Expected (static): ELF 64-bit LSB executable, ARM aarch64, statically linked
# Expected (shared): ELF 64-bit LSB executable, ARM aarch64, dynamically linked
```

### Deploy stdlib (optional)

```bash
# Extract the stdlib tarball
adb push python3-stdlib-aarch64-android.tar.gz /data/local/tmp/
adb shell "cd /data/local/tmp && tar xzf python3-stdlib-aarch64-android.tar.gz"

# Set PYTHONHOME so Python can find its stdlib
adb shell "PYTHONHOME=/data/local/tmp /data/local/tmp/python3 -c 'import sys; print(sys.version)'"
```

## How It Works

### Static Build Pipeline

```
┌─────────────────────────────────────────────────────────┐
│  1. Build "build" Python (native x86_64 Linux)          │
│     └── Used as --with-build-python for cross-compile   │
├─────────────────────────────────────────────────────────┤
│  2. Download pre-built dependencies (BeeWare)           │
│     └── openssl, libffi, sqlite, bzip2, xz, zstd       │
├─────────────────────────────────────────────────────────┤
│  3. Configure cross-compile with STATIC flags           │
│     └── --disable-shared, LDFLAGS="-static"             │
│     └── Patches out --enable-shared from android.py     │
├─────────────────────────────────────────────────────────┤
│  4. Patch Modules/Setup.local                           │
│     └── Force key modules to compile statically:        │
│         _ssl, _sqlite3, _ctypes, zlib, math, etc.       │
├─────────────────────────────────────────────────────────┤
│  5. make -j$(nproc)                                     │
│     └── Produces: cross-build/<triplet>/python          │
└─────────────────────────────────────────────────────────┘
```

### Why Not Use `android.py build` Directly?

The official `android.py` **hardcodes** these flags in `configure_host_python()`:

```python
"--enable-shared",
"--without-static-libpython",
```

This means `android.py` always produces a shared library build. For a static binary, we must bypass these hardcoded flags and run `configure` manually with `--disable-shared` and `-static` linker flags.

## Creating a Release

Push a tag to trigger an automatic GitHub Release:

```bash
git tag v3.14.0-static-aarch64
git push origin v3.14.0-static-aarch64
```

The release will contain:
- `python3-aarch64` — the Python binary
- `python3-stdlib-aarch64-android.tar.gz` — standard library

## Troubleshooting

### Build fails at configure

The Android NDK version must match what CPython expects. The `android-env.sh` script specifies the required NDK version and downloads it automatically.

### Static build linker errors

Static linking on Android is complex. Common issues:
- **Missing `-fPIC`**: All object files must be compiled with Position Independent Code
- **Undefined references**: Some modules may depend on libraries not available as `.a` files
- **`dlopen` unavailable**: Static binaries cannot use `dlopen()`, so all extension modules must be compiled in

### Python can't find stdlib

Set `PYTHONHOME` to the directory containing `lib/python3.x/`:

```bash
export PYTHONHOME=/data/local/tmp
```

## License

CPython is licensed under the [PSF License](https://docs.python.org/3/license.html).
