# Module A Specification: `cambrian_cobol` Runtime & Transpilation Integration Engine (`cambrian_cobol_spec.md`)

* **Version:** 1.2.1-RELEASE (GnuCOBOL 3.2 ABI-Locked, External Header Resolution)
* **Target Architecture:** C11 / POSIX Compliant Runtimes, Linked via Static/Dynamic Object Harnesses (`libcob` drop-in replacement).
* **Preferred implementation language: Rust (with C-language only where 100% necessary)
* **Component Owner:** Cambrian Radiation Core Engineering

---

## 1. Architectural Scope & Translation Pipeline

Writing a custom COBOL lexer, syntax parser, or static source code analyzer is explicitly out of scope. `cambrian_cobol` avoids parser bloat entirely by decoupling syntax translation from runtime execution via the upstream GnuCOBOL compiler frontend (`cobc -C`).

```text
[Legacy COBOL Source (.cbl)] 
          │
          ▼
 [Upstream Translator: cobc -C] (Emits standard GnuCOBOL 3.2 C source)
          │
          ▼
 [cambrian_cobol ABI Compatibility Layer] (libcambrian_cobol drop-in for libcob)
          │
          ▼
 [High-Performance Storage & Sort Engine] (mbunta-isam / mb_sort)
          │
          ▼
 [Native Compiler: GCC / Clang -O3] 
          │
          ▼
 [High-Performance Standalone Binary / Mesh Shard]
```

* **Phase 1 (Source Translation):** Enterprise COBOL source files pass through standard GnuCOBOL 3.2 (`cobc -C`) to emit clean, standards-compliant intermediate C source code.
* **Phase 2 (ABI Interception):** Rather than linking against standard `libcob`, the binary links against `libcambrian_cobol`, which implements the exact public function entry points and data structures required by GnuCOBOL 3.2.
* **Phase 3 (Engine Routing):** Standard file operations and sorts called by the generated C code are intercepted at the ABI boundary and redirected to our custom high-performance `mbunta-isam` (Cambrian's/Marabunta's own ISAM, specified elsewhere) and memory sort engines.

---

## 2. GnuCOBOL 3.2 ABI Compatibility & External Header Sourcing

To maintain strict compliance and eliminate licensing, distribution, or static-link entanglements associated with LGPL runtime artifacts, `libcambrian_cobol` neither distributes nor recreates GnuCOBOL header files.

### 2.1 Host-Filesystem Header Resolution
* **Zero Bundling Policy:** No upstream GnuCOBOL headers (`libcob.h`, `common.h`, etc.) exist within the repository source tree.
* **Build-Time System Inclusion:** Compilation directives explicitly reference the target host development environment's filesystem paths for GnuCOBOL headers. The build harness requires the developer or deployment target to have `libcob-dev` (or local system equivalent) installed, resolving headers dynamically via compiler include paths:

```bash
gcc -O3 -shared -fPIC src/cambrian_cobol.c -I/usr/include/cobol -lcob -o libcambrian_cobol.so
```

This guarantees that struct layouts, memory alignments, and symbol definitions (`struct cob_file`, `cob_field`) strictly match the exact local compiler version without repository pollution or compliance overhead.

---

## 3. Packed Decimal (COMP-3) FFI & Memory Layout Subsystem

Packed decimal data types must maintain strict alignment and bitwise integrity across memory boundaries, mirroring GnuCOBOL 3.2 internal field arithmetic layouts.

### 3.1 BCD Layout Specifications & Sign Rules
* Each byte holds two decimal digits (high nibble and low nibble).
* The final low-order nibble of the field is reserved for the algebraic sign:
  * **Positive values:** `0xC` (Default generated sign), `0xF`, `0xA`, `0xE`
  * **Negative values:** `0xD` (Default generated sign), `0xB`

### 3.2 Type-Safe Implementation (`C11 static inline`)
Guarded against `INT64_MIN` undefined behavior, uninitialized high nibbles, and silent numeric overflow.

```c
#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>
#include <limits.h>
#include <string.h>

/**
 * Unpacks a GnuCOBOL 3.2 / ANSI COMP-3 buffer into a native 64-bit signed integer.
 */
static inline int mb_unpack_comp3(const uint8_t *src, size_t len, int64_t *dest) {
    if (len == 0 || src == NULL || dest == NULL) {
        return -1;
    }

    int64_t accum = 0;
    bool negative = false;

    for (size_t i = 0; i < len; i++) {
        uint8_t b = src[i];
        if (i == len - 1) {
            uint8_t high = (b >> 4) & 0x0F;
            uint8_t low = b & 0x0F;

            if (high > 9) return -1;

            if (accum > (INT64_MAX - high) / 10) return -1;
            accum = (accum * 10) + high;

            if (low == 0x0D || low == 0x0B) {
                negative = true;
            } else if (low != 0x0C && low != 0x0F && low != 0x0A && low != 0x0E) {
                return -1;
            }
        } else {
            uint8_t high = (b >> 4) & 0x0F;
            uint8_t low = b & 0x0F;

            if (high > 9 || low > 9) return -1;

            if (accum > (INT64_MAX - high) / 10) return -1;
            accum = (accum * 10) + high;

            if (accum > (INT64_MAX - low) / 10) return -1;
            accum = (accum * 10) + low;
        }
    }

    if (negative) {
        if (accum > (uint64_t)INT64_MAX + 1) return -1;
        *dest = (accum == (uint64_t)INT64_MAX + 1) ? INT64_MIN : -((int64_t)accum);
    } else {
        if (accum > INT64_MAX) return -1;
        *dest = (int64_t)accum;
    }
    return 0;
}

/**
 * Packs a native 64-bit signed integer into a COMP-3 buffer.
 */
static inline int mb_pack_comp3(int64_t val, uint8_t *dest, size_t len) {
    if (len == 0 || dest == NULL) {
        return -1;
    }

    memset(dest, 0, len);

    uint8_t sign = (val < 0) ? 0x0D : 0x0C;

    uint64_t uval;
    if (val == INT64_MIN) {
        uval = (uint64_t)-(INT64_MAX) + 1; 
    } else {
        uval = (val < 0) ? (uint64_t)-val : (uint64_t)val;
    }

    dest[len - 1] = ((uval % 10) << 4) | sign;
    uval /= 10;

    for (size_t i = len - 1; i > 0; i--) {
        size_t idx = i - 1;
        uint8_t low = uval % 10; 
        uval /= 10;
        uint8_t high = uval % 10; 
        uval /= 10;
        dest[idx] = (high << 4) | low;
    }

    if (uval > 0) {
        return -1;
    }

    return 0;
}
```

---

## 4. Embedded External Merge Sort Engine (`cambrian_sort`)

`cambrian_sort` integrates directly with GnuCOBOL's table and file sort hooks to service native COBOL `SORT` and `MERGE` verbs without spawning external cluster nodes.

### 4.1 Execution Lifecycle
* **Allocation & Buffering:** The runtime allocates a dedicated memory workspace (default 512MB). Input records stream directly from GnuCOBOL `USING` procedures.
* **Key Comparison Dispatch:** Sorting routines use GnuCOBOL-compatible comparator function pointers evaluating field positions, byte lengths, and explicit `COLLATING SEQUENCE` overrides.
* **Spill and Merge Pipeline:**
  * When memory saturates, records quicksort in-memory and flush to temporary unlinked files (`/tmp/mb_sort_[PID]_[RUN].tmp`).
  * A multi-way binary min-heap priority queue merges temporary runs concurrently, streaming output to the `GIVING` target file or an `OUTPUT PROCEDURE` coroutine.

---

## 5. Operational Boundaries & Structural Scope

To maintain bare-metal execution speed, the following legacy elements are structurally out-of-scope and unsupported by `libcambrian_cobol`:

* **Interactive Terminals:** All `ACCEPT`, `DISPLAY` terminal screens, and `SCREEN SECTION` descriptions are disabled.
* **Dynamic Code Alteration:** The legacy COBOL `ALTER` verb is structurally unsupported.
* **No Frontend Linter:** There is no custom parser or static analysis pass checking source code for unsupported syntax; source translation relies entirely on the output capabilities and constraints of standard `cobc -C`.