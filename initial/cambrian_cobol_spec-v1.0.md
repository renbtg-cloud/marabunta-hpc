# Module A Specification: `cambrian_cobol` Runtime & Transpilation Integration Engine (`cambrian_cobol_spec.md`)

* **Version:** 1.1-RELEASE
* **Target Architecture:** C11 / POSIX Compliant Runtimes, Linked via Static/Dynamic Object Harnesses.
* **Component Owner:** Cambrian Radiation Core Engineering

---

## 1. Architectural Scope & Translation Pipeline

Writing a custom COBOL lexer and syntax parser from scratch introduces massive fragility when dealing with legacy enterprise dialects. `cambrian_cobol` avoids parser bloat by decoupling syntax translation from runtime execution.
[Legacy COBOL Source (.cbl)]
│
▼
[Upstream Translator: cobc -C] (Generates intermediate C source)
│
▼
[cambrian_cobol Header & Runtime Injection] (libcambrian_cobol)
│
▼
[Native Compiler: GCC / Clang -O3]
│
▼
[High-Performance Standalone Binary / Mesh Shard]

1. **Phase 1 (Source Translation):** Enterprise COBOL source files are passed through standard robust translators (such as the GnuCOBOL transpiler backend configured with `-C`) to emit clean, standards-compliant intermediate C code.
2. **Phase 2 (Runtime Header Binding):** Standard upstream runtime calls are intercepted and re-mapped via header injection to `libcambrian_cobol`, routing file operations, sorts, and data type conversions directly into our custom high-performance engines.
3. **Phase 3 (Native Compilation):** The resulting C code is compiled directly via GCC or Clang with aggressive optimizations (`-O3`, link-time optimization), producing a bare-metal executable entirely free of VM interpretation overhead.

---

## 2. The `libcambrian_cobol` Runtime API & File Control Blocks

Every transpiled COBOL file control entry (`SELECT ... ASSIGN TO`) initializes a runtime File Control Block (FCB) managed by `libcambrian_cobol`.

### 2.1 FCB Structure Definition (C11)
```c
typedef enum {
    MB_FILE_SEQ = 1,
    MB_FILE_IDX = 2,
    MB_FILE_REL = 3
} mb_file_org_t;

typedef struct {
    char            dsname[256];        // Physical dataset or ISAM path
    mb_file_org_t   organization;       // Sequential, Indexed, or Relative
    int             access_mode;        // Sequential, Dynamic, or Random
    int             open_mode;          // Input, Output, I/O, Extend
    uint32_t        record_length;      // Fixed length or max length
    int             file_status;        // 2-byte COBOL status string as integer
    void*           vfs_handle;         // Underlying pointer to mbunta-isam driver
} mb_fcb_t;
```

### 2.2 Deterministic File Status Code Emulation
The runtime intercepts all I/O anomalies and maps them directly to strict COBOL file status return codes, ensuring existing business logic error paragraphs (INVALID KEY, AT END) execute without modification:

| Status Code | Condition Description | Action Taken by `libcambrian_cobol` |
| :--- | :--- | :--- |
| **00** | Success | Operation completed normally. |
| **02** | Success - Duplicate Key | Write/Rewrite succeeded, alternate key duplication detected. |
| **22** | Duplicate Key Error | Primary key violation on write/rewrite; execution routes to `INVALID KEY`. |
| **23** | Key Not Found | Read/Start target key does not exist; routes to `INVALID KEY`. |
| **35** | File Not Found | Open input failed because physical `.dat`/`.idx` path is missing. |
| **41** | File Already Open | Attempted to open a file descriptor already marked active in the FCB. |

---

## 3. Packed Decimal (COMP-3) FFI & Memory Layout Subsystem

Packed decimal data types must maintain strict alignment and bitwise integrity across memory boundaries, especially when interfacing with external C libraries or Python zero-copy views.

### 3.1 BCD Layout Specifications
* Each byte holds two decimal digits (high nibble and low nibble).
* The final low-order nibble of the field is reserved for the algebraic sign:
  * **Positive values:** `0xC`, `0xF`, `0xA`, `0xE`
  * **Negative values:** `0xD`, `0xB`

### 3.2 Type-Safe Implementation (C11 static inline)
```c
#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>

/**
 * Unpacks a Micro Focus / ANSI COMP-3 (Packed Decimal) buffer into a native 64-bit signed integer.
 * 
 * @param src Pointer to the packed BCD byte array.
 * @param len Exact byte length of the COMP-3 field.
 * @param dest Pointer to the destination int64_t variable.
 * @return 0 on success, -1 on invalid sign nibble or BCD format.
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
            
            if (high > 9) return -1; // Invalid BCD digit
            accum = (accum * 10) + high;

            if (low == 0x0D || low == 0x0B) {
                negative = true;
            } else if (low != 0x0C && low != 0x0F && low != 0x0A && low != 0x0E) {
                return -1; // Invalid sign code
            }
        } else {
            uint8_t high = (b >> 4) & 0x0F;
            uint8_t low = b & 0x0F;
            
            if (high > 9 || low > 9) return -1; // Invalid BCD digit
            accum = (accum * 10) + high;
            accum = (accum * 10) + low;
        }
    }

    *dest = negative ? -accum : accum;
    return 0;
}

/**
 * Packs a native 64-bit signed integer into a COMP-3 (Packed Decimal) buffer.
 * 
 * @param val Native signed 64-bit integer value.
 * @param dest Pointer to destination BCD byte buffer.
 * @param len Exact byte length of the target field.
 * @return 0 on success, -1 on buffer overflow.
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
        return -1; // Overflow: Field too small for value
    }

    return 0;
}
```

---

## 4. Embedded External Merge Sort Engine (`cambrian_sort`)

`cambrian_sort` is compiled directly into `libcambrian_cobol` to service native COBOL `SORT` and `MERGE` verbs without spawning external processes or relying on cluster nodes for single-node workloads.

### 4.1 Execution Lifecycle
1. **Allocation & Buffering:** The runtime allocates a dedicated memory workspace (`SORT-CORE-SIZE`, default 512MB). Input records from `USING` clauses stream into this workspace.
2. **Key Comparison Dispatch:** Sorting routines use a specialized comparison function generated per-sort based on COBOL `ASCENDING`/`DESCENDING KEY` definitions, evaluating positions, byte lengths, and explicit `COLLATING SEQUENCE` overrides.
3. **Spill and Merge Pipeline:**
   * When the memory workspace saturates, records are quicksorted in-memory and flushed to temporary unlinked files (`/tmp/mb_sort_[PID]_[RUN].tmp`).
   * Once input is exhausted, a multi-way binary min-heap priority queue opens all temporary runs concurrently, streaming sorted output directly to the `GIVING` target file or an `OUTPUT PROCEDURE` coroutine.

---

## 5. Operational Boundaries & Unsupported Constructs

To maintain execution speed and eliminate runtime bloat, the following legacy elements are explicitly unsupported by `libcambrian_cobol`. Attempting to transpile source code utilizing these elements will result in explicit build-time or link-time rejection:

* **Interactive Terminals:** All `ACCEPT` and `DISPLAY` statements interacting with user terminals, as well as complete `SCREEN SECTION` descriptions, are disabled. Batch logging is strictly redirected to file descriptors or `stderr`.
* **Dynamic Code Alteration:** The COBOL `ALTER` verb (modifying procedure paragraphs dynamically at runtime) is structurally blocked. All control graphs are statically resolved at compile time.
* **Unstructured Control Flow:** Unstructured `GO TO` statements crossing paragraph or section boundaries are rejected by the translator front-end. Code must adhere to structured control paradigms (`PERFORM`, `IF/ELSE`, `EVALUATE`).