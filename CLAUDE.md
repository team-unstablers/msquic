# CLAUDE.md

This file provides guidance to AI coding agents working with code in this repository.

## Project Overview

MsQuic is Microsoft's cross-platform C implementation of the IETF QUIC protocol (RFC 9000). It targets Windows (user & kernel mode), Linux, macOS, FreeBSD, Android, and iOS.

This is a fork maintained at `team-unstablers/msquic`.

## Fork Policy

This fork follows MsQuic upstream conventions by default to minimize merge conflicts and ease upstream tracking. However, when upstream conventions conflict with our requirements, **our style and needs take priority without hesitation**. Divergences from upstream are intentional and should not be second-guessed.

## Build Commands

All build/test scripts require PowerShell (`pwsh`). TLS defaults to `schannel` on Windows, `quictls` on Linux/macOS.

```bash
# Install build dependencies
pwsh ./scripts/prepare-machine.ps1 -ForBuild -Tls quictls

# Debug build (default)
pwsh ./scripts/build.ps1

# Release build
pwsh ./scripts/build.ps1 -Config Release

# Build with specific options
pwsh ./scripts/build.ps1 -Config Debug -Arch arm64 -Tls quictls

# Clean build
pwsh ./scripts/build.ps1 -Clean

# Static library build
pwsh ./scripts/build.ps1 -Static

# Build with AddressSanitizer
pwsh ./scripts/build.ps1 -SanitizeAddress

# Static code analysis
pwsh ./scripts/build.ps1 -CodeCheck
```

Build artifacts go to `artifacts/bin/<platform>/<arch>_<Config>_<tls>/`.

## Testing

Three test binaries, all using Google Test:

| Binary | Source | Scope |
|---|---|---|
| `msquicplatformtest` | `src/platform/unittest/` | Platform layer (crypto, TLS, datapath) |
| `msquiccoretest` | `src/core/unittest/` | Core protocol (frames, packet numbers, ranges) |
| `msquictest` | `src/test/bin/` | Full end-to-end functional tests |

```bash
# Run all tests
pwsh ./scripts/test.ps1

# Filter specific tests (gtest filter syntax)
pwsh ./scripts/test.ps1 -Filter "TlsTest*"

# Run a single test binary directly
./artifacts/bin/macos/arm64_Debug_quictls/msquicplatformtest --gtest_filter="TlsTest.PemFile"

# List available tests
./artifacts/bin/macos/arm64_Debug_quictls/msquicplatformtest --gtest_list_tests

# Repeat tests for flakiness detection
pwsh ./scripts/test.ps1 -Filter "SomeTest*" -NumIterations 10
```

## Architecture

Two-layer design:

```
Application (HQUIC handles, QUIC_API_TABLE)
    │
    ▼
QUIC Core Layer (src/core/)          ← Platform-independent protocol logic
    │  calls CxPlat* APIs
    ▼
Platform Abstraction Layer (src/platform/)  ← OS/TLS/crypto/datapath
```

### Key directories

- `src/core/` — QUIC protocol logic (connection, stream, congestion control, packet handling)
- `src/platform/` — PAL with per-OS implementations (epoll/kqueue/IOCP, schannel/quictls, bcrypt/openssl)
- `src/inc/` — Public API header (`msquic.h`) and internal headers
- `src/generated/` — Auto-generated CLOG (structured logging) headers
- `src/test/` — Functional/integration tests
- `src/tools/` — Utilities (sample app, spin fuzzer, interop, load balancer)
- `src/bin/` — Shared library entry points

### Core object hierarchy

```
QUIC_LIBRARY (singleton)
  └── QUIC_PARTITION[] (per-processor)
        └── QUIC_WORKER (event loop + timer wheel)
  └── QUIC_REGISTRATION (per-app)
        ├── QUIC_CONFIGURATION (TLS config + settings)
        ├── QUIC_LISTENER (server)
        └── QUIC_CONNECTION
              ├── QUIC_STREAM[]
              ├── QUIC_CRYPTO (TLS handshake)
              ├── QUIC_CONGESTION_CONTROL (Cubic/BBR, vtable-based)
              ├── QUIC_PACKET_SPACE[3] (Initial/Handshake/1-RTT)
              └── QUIC_PATH[] (multipath)
```

### Platform file mapping

| Subsystem | Windows | Linux | macOS |
|---|---|---|---|
| Datapath | `datapath_winuser.c` | `datapath_epoll.c` | `datapath_kqueue.c` |
| TLS | `tls_schannel.c` or `tls_quictls.c` | `tls_quictls.c` | `tls_quictls.c` |
| Crypto | `crypt_bcrypt.c` | `crypt_openssl.c` | `crypt_openssl.c` |

## Code Conventions

### Naming

- **All identifiers use PascalCase**, including local variables (`int OutSize`, `BOOLEAN IsServer`)
- **Prefix rules**:
  - `QUIC_` — public types/constants (`QUIC_CONNECTION`, `QUIC_STATUS`)
  - `CXPLAT_` — platform abstraction types/macros (`CXPLAT_TLS`, `CXPLAT_ALLOC_NONPAGED`)
  - `CxPlat` — PAL function names (`CxPlatTlsInitialize`)
  - `QuicConn`/`QuicStream`/etc. — core internal functions
  - `MsQuic` — public API functions
- Pointer: `Type* VarName` (star attached to type)

### Formatting

- 4-space indentation, Allman-variant brace style
- Function parameters with SAL annotations go on separate lines
- Closing `)` on its own indented line
- Section comments use `//\n// Description\n//` style

### SAL annotations

SAL is used **extensively and strictly** on all function declarations, including test helpers. Non-Windows platforms use stubs from `quic_sal_stub.h`.

### Error handling

Single-exit pattern via `goto Exit` / `goto Error` (Windows kernel style).

### Memory allocation

Always use `CXPLAT_ALLOC_NONPAGED(size, QUIC_POOL_TAG)` / `CXPLAT_FREE(ptr, QUIC_POOL_TAG)` with a pool tag.

### Conditional compilation

- Platform: `_WIN32`, `_KERNEL_MODE`, `__linux__`, `__APPLE__`
- Features: `QUIC_API_ENABLE_PREVIEW_FEATURES`, `QUIC_API_ENABLE_INSECURE_FEATURES`
- Test guards: `QUIC_DISABLE_PFX_TESTS`, `QUIC_ENABLE_CA_CERTIFICATE_FILE_TESTS`, etc.
- `#endif` must have the original condition as a comment: `#endif // QUIC_DISABLE_PFX_TESTS`

### CLOG (structured logging)

Every `.c`/`.cpp` file ends with a conditional CLOG include:
```c
#ifdef QUIC_CLOG
#include "filename.c.clog.h"
#endif
```
After modifying logging statements, run `pwsh ./scripts/update-sidecar.ps1` and verify no diff.

## Test Conventions

- Test fixtures use **`struct`** (not `class`), inheriting `::testing::Test` or `::testing::TestWithParam<T>`
- File naming: `<Component>Test.cpp`
- Test naming: `TEST_F(FixtureName, PascalCaseScenarioName)`
- RAII via nested helper structs with constructors/destructors for resource management
- SAL annotations are applied even in test helper methods
- Use `VERIFY_QUIC_SUCCESS(result)` for QUIC status checks
- Conditional tests: compile-time via `#ifndef QUIC_DISABLE_*` guards, runtime via `GTEST_SKIP()`
- Parameterized tests: `INSTANTIATE_TEST_SUITE_P` at file end

## Commit Message Convention

Format: `<scope>: <description in lowercase>`

- Scope is the path-like component area: `platform/tls_quictls`, `platform/unittest/TlsTest`, `test`, etc.
- Description starts lowercase, imperative mood
- Examples:
  - `platform/tls_quictls: fix off-by-one in PEM password buffer overflow check`
  - `platform/unittest/TlsTest: gate PEM and chain tests by TLS provider flags`

## Key Files for Reference

- `src/inc/msquic.h` — Public C API (all handle types, callbacks, settings)
- `src/inc/msquic.hpp` — C++ RAII wrapper
- `src/inc/quic_tls.h` — TLS abstraction interface
- `src/inc/quic_datapath.h` — Datapath abstraction interface
- `src/inc/quic_platform.h` — Platform macros and type definitions
- `src/core/connection.c` — Core connection state machine
- `src/core/api.c` — Public API implementation (API call → QUIC_OPERATION)
