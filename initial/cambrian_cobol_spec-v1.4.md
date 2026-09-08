# Module A Specification: `cambrian_cobol` Runtime & Transpilation Integration Engine (`cambrian_cobol_spec.md`)

* **Version:** 1.4.0-RELEASE (GnuCOBOL 3.2 ABI-Locked, Clean-Room Rust Runtime)
* **Target Architecture:** Rust / C11 / POSIX Compliant Runtimes, Linked via Clean-Room Static/Dynamic Object Harnesses (Zero `-lcob` Dependency).
* **Preferred Implementation Language:** Rust (with C-language only where 100% necessary for FFI memory layout alignment)
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
 [cambrian_cobol ABI Compatibility Layer] (libcambrian_cobol drop-in replacement)
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
* **Phase 2 (ABI Interception):** Rather than linking against standard `libcob`, the binary links against `libcambrian_cobol`, which implements the exact public function entry points and data structures required by GnuCOBOL 3.2 cleanly.  
* **Phase 3 (Engine Routing):** Standard file operations and sorts called by the generated C code are intercepted at the ABI boundary and redirected to our custom high-performance `mbunta-isam` (Cambrian's/Marabunta's own ISAM) and memory sort engines.  

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

### 2.2 Resolution of the Linkage Paradox (Clean-Room Standalone ABI)
To permanently satisfy the LGPL-avoidance mandate, `libcambrian_cobol` functions as a pure clean-room standalone runtime replacement.
* **Zero `-lcob` Dependency:** `libcambrian_cobol` **MUST NOT** link against upstream `-lcob`.
* **Direct Symbol Export:** The library acts as the primary provider of the `cob_*` symbol namespace. Compilation requires only header definitions for struct definitions (`struct cob_file`, `cob_field`) extracted via host-system inclusion at build time, but all function execution logic is natively implemented within the Rust/C `cambrian_cobol` engine codebase.

---

## 3. Packed Decimal (`COMP-3`) FFI & Memory Layout Subsystem

Packed decimal data types must maintain strict alignment and bitwise integrity across memory boundaries, mirroring GnuCOBOL 3.2 internal field arithmetic layouts.  

### 3.1 BCD Layout Specifications & Sign Rules
* Each byte holds two decimal digits (high nibble and low nibble).  
* The final low-order nibble of the field is reserved for the algebraic sign:  
  * **Positive values:** `0xC` (Default generated sign), `0xF`, `0xA`, `0xE`
  * **Negative values:** `0xD` (Default generated sign), `0xB`

### 3.2 Type-Safe Implementation (C11 `static inline`)
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

## 4. ABI Compatibility Contract & Core Symbol Surface

Generated C code emitted by upstream `cobc -C` relies on a well-defined subset of runtime entry points. `libcambrian_cobol` implements and exports the precise ABI surface required to support structured batch execution, file handling, and arithmetic.

### 4.1 Lifecycle & Version Control Entry Points
* `int cob_init(int argc, char **argv)`: Initializes memory workspaces, exception registries, and thread-local state.
* `void cob_tidy(void)`: Flushes buffers, unlinks temporary sort run files, and releases global runtime allocations.
* `void cob_stop_run(int status)`: Terminates execution, returning the standard POSIX process exit status.
* `int cob_check_version(unsigned int major, unsigned int minor, unsigned int subminor)`: Version handshake verifying compatibility against GnuCOBOL 3.2 expectations (returns 0 on successful match, non-zero on structural divergence).

### 4.2 Core Verb-to-Runtime Mapping Matrix

| COBOL Verb / Construct | Emitted Runtime Symbol / Hook | Engine Routing Target |
| :--- | :--- | :--- |
| **OPEN, CLOSE, READ, WRITE, REWRITE, DELETE, START** | `cob_open`, `cob_close`, `cob_read`, `cob_write`, `cob_rewrite`, `cob_delete`, `cob_start` | `mbunta-isam` |
| **SORT, MERGE** | `cob_s_init`, `cob_s_add`, `cob_s_sort`, `cob_s_finish` | `cambrian_sort` |
| **MOVE, INITIALIZE** | Direct memory layout copy / `cob_move` | In-memory SIMD / primitive copy |
| **ADD, SUBTRACT, MULTIPLY, DIVIDE** | `cob_add`, `cob_sub`, `cob_mul`, `cob_div` | Native Rust arithmetic / `mb_unpack_comp3` |
| **CALL, CANCEL** | `cob_call`, `cob_cancel` | Static symbol resolver / Program registry |

---

## 5. Rust-First Architecture & FFI Boundary Specification

In accordance with the component preference for Rust (with C reserved exclusively for low-level FFI bindings where mandatory), the core runtime logic, memory management, ISAM interface wrappers, and sort routines are implemented in Rust.  

### 5.1 Memory Layout & Struct Mirroring (`#[repr(C)]`)
To guarantee safe, bitwise-identical interaction with upstream C code generated by `cobc -C`, Rust components mirror all required GnuCOBOL data structures using strict C-compatible memory layouts:

```rust
#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct CobFieldAttr {
    pub type_code: u16,    // COBOL data type (numeric, alphanumeric, packed-decimal)
    pub digits: u16,       // Total number of digits
    pub scale: i16,        // Scaling factor / decimal places (V clause)
    pub flags: u32,        // Field flags (signed, synchronized, etc.)
}

#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct CobField {
    pub size: usize,             // Byte length of the field data
    pub data: *mut u8,           // Raw pointer to the underlying buffer memory
    pub attr: *const CobFieldAttr, // Pointer to type/precision attributes
}

#[repr(C)]
pub struct CobFile {
    pub filename: *const std::os::raw::c_char, // Physical or logical file path
    pub cob_name: *const std::os::raw::c_char, // Program-defined SELECT name
    pub file_status: *mut u8,                  // 2-byte status code buffer ("00", "23")
    pub organization: i32,                     // Sequential, Indexed, Relative, Line-Sequential
    pub access_mode: i32,                      // Sequential, Random, Dynamic
    pub record_min: usize,                     // Minimum record length
    pub record_max: usize,                     // Maximum record length
    pub record: *mut CobField,                 // Pointer to current record buffer field
    pub internal_handle: *mut std::os::raw::c_void, // Engine internal routing handle
}
```

### 5.2 FFI Safety, Panic Handling, and Symbol Export
Because Rust panics across an FFI boundary result in undefined behavior, every exported `extern "C"` function entry point wraps its core logic inside an unwinding guard:

```rust
use std::panic::catch_unwind;

#[no_mangle]
pub extern "C" fn cob_init(argc: std::os::raw::c_int, argv: *mut *mut std::os::raw::c_char) -> std::os::raw::c_int {
    let result = catch_unwind(|| {
        0
    });

    match result {
        Ok(code) => code,
        Err(_) => -1,
    }
}
```

### 5.3 Threading & Concurrency Model
* **Single-Process & Multi-Sharded Execution Context:** GnuCOBOL programs traditionally assume a process-wide mutable state. `libcambrian_cobol` encapsulates state within thread-safe Rust constructs (`Arc<Mutex<T>>` or thread-local contexts) to enable multi-sharded mesh execution, parallel API request handling, and concurrent batch pipelines without race conditions.  

---

## 6. Embedded External Merge Sort Engine (`cambrian_sort`) Contract

`cambrian_sort` integrates directly with GnuCOBOL's table and file sort hooks to service native COBOL `SORT` and `MERGE` verbs without spawning external cluster nodes.  

### 6.1 GnuCOBOL Sort Hook Interception
Generated C code interfaces with sorting via explicit entry points:
* `cob_s_init`: Allocates the memory workspace (default 512MB) and configures internal sorting contexts.  
* `cob_s_add`: Streams records directly from `USING` procedures.  
* `cob_s_sort`: Dispatches GnuCOBOL-compatible comparator function pointers evaluating key byte lengths, positions, and explicit `COLLATING SEQUENCE` overrides.  
* `cob_s_finish`: Resolves the multi-way merge phase and streams results to a `GIVING` file target or an `OUTPUT PROCEDURE` coroutine.  

### 6.2 Spill, Merge, and Stability Architecture
* **Spill-to-Disk Pipeline:** When the allocation budget saturates, records quicksort in-memory and flush out to temporary unlinked run files (`/tmp/mb_sort_[PID]_[RUN].tmp`).  
* **Min-Heap Merge:** Employs a multi-way binary min-heap priority queue to merge temporary runs concurrently.  
* **Sort Stability:** Enforces stable sorting guarantees for rows containing identical keys.  

---

## 7. `mbunta-isam` Storage & File I/O Subsystem Contract

To replace standard file handling systems without external LGPL dependencies, `mbunta-isam` implements native file organization and access semantics mapped directly from GnuCOBOL file control blocks.

### 7.1 Supported File Organizations & Access Modes
* **File Organizations:** `SEQUENTIAL`, `LINE SEQUENTIAL`, `RELATIVE`, and `INDEXED` (supporting primary and alternate record keys).  
* **Access Modes:** `SEQUENTIAL`, `RANDOM`, and `DYNAMIC`.  

### 7.2 SELECT/ASSIGN and File Status Code Mapping Matrix
Every file operation updates a 2-byte status buffer in the `cob_file` control block mapped by `mbunta-isam`:  

| Status Code | Condition / Meaning | Trigger Scenario |
| :--- | :--- | :--- |
| **00** | Success | Operation completed successfully with no anomalies. |
| **02** | Success (Duplicate) | Successful write/read on an indexed file with duplicate alternate keys allowed. |
| **10** | End of File | Sequential read reached physical end of file. |
| **22** | Duplicate Key Error | Attempt to write a record whose primary or unique key already exists. |
| **23** | Record Not Found | Read or start operation failed to locate a record matching the target key. |
| **24** | Boundary Violation | Write beyond file size limits or relative file boundary rules. |
| **35** | File Not Found | OPEN request failed because the target file does not exist on disk. |
| **46** | Invalid Sequential Read | Read sequential attempted past end-of-file condition. |
| **49** | Mandatory Lock Failure | File lock collision during multi-process concurrent access. |

---

## 8. Operational Boundaries & Structural Scope

To maintain bare-metal execution speed, the following legacy elements are structurally out-of-scope and unsupported by `libcambrian_cobol`:  
* **Interactive Terminals:** All `ACCEPT`, `DISPLAY` terminal screens, and `SCREEN SECTION` descriptions are disabled.
* **Dynamic Code Alteration:** The legacy COBOL `ALTER` verb is structurally unsupported.
* **No Frontend Linter:** There is no custom parser or static analysis pass checking source code for unsupported syntax; source translation relies entirely on the output capabilities and constraints of standard `cobc -C`.