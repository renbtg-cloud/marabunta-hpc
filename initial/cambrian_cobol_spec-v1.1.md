# Module A Specification: `cambrian_cobol` Runtime & Transpilation Integration Engine (`cambrian_cobol_spec.md`)

* **Version:** 1.02-RELEASE (GnuCOBOL ABI Compatible)
* **Target Architecture:** C11 / POSIX Compliant Runtimes, Linked via Static/Dynamic Object Harnesses (`libcob` drop-in replacement).
* **Component Owner:** Cambrian Radiation Core Engineering

---

## 1. Architectural Scope & Translation Pipeline

Writing a custom COBOL lexer and syntax parser from scratch introduces massive fragility when dealing with legacy enterprise dialects. `cambrian_cobol` avoids parser bloat by decoupling syntax translation from runtime execution via GnuCOBOL's compiler frontend (GnuCOBOL version: 3.2).

```text
[Legacy COBOL Source (.cbl)]
│
▼
[Upstream Translator: cobc -C] (Emits standard libcob C source)
│
▼
[cambrian_cobol ABI Compatibility Layer] (libcambrian_cobol alias for libcob)
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

1. **Phase 1 (Source Translation):** Enterprise COBOL source files pass through GnuCOBOL (`cobc -C`) to emit clean, standards-compliant intermediate C source code.
2. **Phase 2 (ABI Interception):** Rather than linking against standard `libcob`, the binary links against `libcambrian_cobol`, which implements GnuCOBOL's exact public structures (`cob_file`, `cob_field`).
3. **Phase 3 (Engine Routing):** Standard file operations and sorts called by the generated C code are intercepted at the ABI boundary and redirected to our custom high-performance `mbunta-isam` and memory sort engines.

---

## 2. GnuCOBOL ABI Compatibility & File Control Blocks (`cob_file`)

To ensure seamless compilation of `cobc -C` output, the runtime implements GnuCOBOL's native file control structure (`cob_file`) and function signatures.

### 2.1 Native `cob_file` Structure Mapping
```c
// Aligned with GnuCOBOL libcob internal file descriptor layout
typedef struct cob_file {
    const char          *filename;
    unsigned char       *select_name;
    cob_field           *file_status;
    unsigned int        organization;
    unsigned int        access_mode;
    unsigned int        lock_mode;
    unsigned int        open_mode;
    unsigned int        flag_optional;
    unsigned int        flag_select_features;
    unsigned int        flag_needs_newline;
    unsigned int        flag_is_net_file;
    unsigned int        record_min;
    unsigned int        record_max;
    cob_field           *record;
    void                *vfs_handle;        // Mapped to mbunta-isam backend driver
} cob_file;
```

### 2.2 Intercepted libcob File I/O Entry Points
The following standard libcob functions are implemented directly in `libcambrian_cobol` to intercept runtime execution:

```c
int cob_open(cob_file *file, int mode, int optional);
int cob_close(cob_file *file, int option);
int cob_read(cob_file *file, cob_field *key, int options);
int cob_read_next(cob_file *file, int options);
int cob_write(cob_file *file, int options);
int cob_rewrite(cob_file *file, int options);
int cob_start(cob_file *file, cob_field *key, int op);
int cob_delete(cob_file *file);
```

---

## 3. Packed Decimal (COMP-3) FFI & Memory Layout Subsystem

Packed decimal data types must maintain strict alignment and bitwise integrity across memory boundaries, mirroring GnuCOBOL's internal field arithmetic layouts.

### 3.1 BCD Layout Specifications
* Each byte holds two decimal digits (high nibble and low nibble).
* The final low-order nibble of the field is reserved for the algebraic sign:
  * Positive values: `0xC`, `0xF`, `0xA`, `0xE`
  * Negative values: `0xD`, `0xB`

### 3.2 Type-Safe Implementation (C11 static inline)
```c
#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>

/**
 * Unpacks a GnuCOBOL / ANSI COMP-3 (Packed Decimal) buffer into a native 64-bit signed integer.
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
            accum = (accum * 10) + high;
            accum = (accum * 10) + low;
        }
    }

    *dest = negative ? -accum : accum;
    return 0;
}

/**
 * Packs a native 64-bit signed integer into a COMP-3 (Packed Decimal) buffer.
 */
static inline int mb_pack_comp3(int64_t val, uint8_t *dest, size_t len) {
    if (len == 0 || dest == NULL) {
        return -1;
    }

    uint8_t sign = (val < 0) ? 0x0D : 0x0C;
    if (val < 0) val = -val;

    dest[len - 1] = ((val % 10) << 4) | sign;
    val /= 10;

    for (size_t i = len - 1; i > 0; i--) {
        size_t idx = i - 1;
        uint8_t low = val % 10; 
        val /= 10;
        uint8_t high = val % 10; 
        val /= 10;
        dest[idx] = (high << 4) | low;
    }

    if (val > 0) {
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
'
## 5. Operational Boundaries & Unsupported Constructs

To maintain bare-metal execution speed, the following legacy elements are structurally blocked:

* **Interactive Terminals:** All `ACCEPT`, `DISPLAY` terminal screens, and `SCREEN SECTION` descriptions are disabled.
* **Dynamic Code Alteration:** The COBOL `ALTER` verb is rejected at compile time.
* **Unstructured Control Flow:** Unstructured `GO TO` statements crossing paragraph boundaries are rejected by the transpiler frontend.