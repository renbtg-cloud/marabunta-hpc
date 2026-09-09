# Annex A: Normative ABI Symbol Manifest (`cambrian_cobol_annex-symbol_manifest.md`)

* **Component Owner:** Cambrian Radiation Core Engineering
* **Target Compatibility:** GnuCOBOL 3.2 Generated C Intermediate Code
* **Linkage Constraint:** Zero `-lcob` Dependency (Clean-Room Standalone Provider)
* **Status:** Draft / Active / Normative

---

## 1. Symbol Export Policy & Calling Conventions

All symbols defined within this normative manifest **MUST** be exported with standard C linkage (`extern "C"`) and platform-default calling conventions to ensure binary compatibility with GCC/Clang-compiled object files emitted by upstream `cobc -C`.

* **FFI Boundary Unwinding Safety:** Every exported entry point acting as an FFI boundary **MUST** wrap its inner execution logic within a safe unwinding panic barrier (e.g., Rust's `std::panic::catch_unwind` or equivalent C-level isolation) to strictly prohibit Rust/runtime panics from unwinding across the FFI threshold.
* **Namespace Exclusivity:** `libcambrian_cobol` serves as the sole provider of the `cob_*` symbol namespace for target binaries, completely superseding and replacing upstream `libcob`.

---

## 2. Core Lifecycle & Version Control Manifest

### `cob_init`
* **Signature:** `extern "C" int cob_init(int argc, char **argv);`[cite: 4]
* **Purpose:** Initializes core runtime execution workspaces, thread-local storage parameters, exception register arrays, and file locking tables.
* **Return Value:** `0` on successful allocation and initialization; non-zero error status code upon failure.

### `cob_tidy`
* **Signature:** `extern "C" void cob_tidy(void);`[cite: 4]
* **Purpose:** Flushes active I/O streams, unlinks temporary sort run files from disk, and tears down runtime memory pools.
* **Return Value:** `void`

### `cob_stop_run`
* **Signature:** `extern "C" void cob_stop_run(int status);`[cite: 4]
* **Purpose:** Terminates program execution context cleanly.
* **Return Value:** `void` (Forces process exit with the specified POSIX integer status code).

### `cob_check_version`
* **Signature:** `extern "C" int cob_check_version(unsigned int major, unsigned int minor, unsigned int subminor);`[cite: 4]
* **Purpose:** Performs a runtime binary compatibility handshake against GnuCOBOL version requirements.
* **Return Value:** `0` if target version constraints are satisfied; non-zero on structural or ABI version divergence.

---

## 3. Memory, Field, & Arithmetic Symbol Manifest

### Field Allocation and Memory Management
* **Signatures:**
  * `extern "C" void *cob_field_alloc(size_t size);`
  * `extern "C" void cob_field_free(void *ptr);`
* **Purpose:** Dynamic management of memory buffers associated with variable-length COBOL fields.

### `cob_move`
* **Signature:** `extern "C" void cob_move(const struct CobField *src, struct CobField *dest);`[cite: 4]
* **Purpose:** Executes data transfer, numeric decimal alignment, space/zero filling, truncation, and category conversion between fields.

### `cob_cmp`
* **Signature:** `extern "C" int cob_cmp(const struct CobField *f1, const struct CobField *f2);`[cite: 4]
* **Purpose:** Evaluates relational comparisons between two COBOL fields (alphanumeric or numeric).
* **Return Value:** `-1` if `f1 < f2`, `0` if `f1 == f2`, `1` if `f1 > f2`.

### Packed Decimal & Numeric Arithmetic (`cob_decimal_*`)
* **Signatures:**
  * `extern "C" void cob_decimal_init(struct CobDecimal *d);`
  * `extern "C" int cob_decimal_set_field(struct CobDecimal *d, const struct CobField *f);`
  * `extern "C" int cob_decimal_get_field(struct CobField *f, const struct CobDecimal *d, int mode);`
  * `extern "C" int cob_decimal_add(struct CobDecimal *d1, const struct CobDecimal *d2);`
  * `extern "C" int cob_decimal_sub(struct CobDecimal *d1, const struct CobDecimal *d2);`
  * `extern "C" int cob_decimal_mul(struct CobDecimal *d1, const struct CobDecimal *d2);`
  * `extern "C" int cob_decimal_div(struct CobDecimal *d1, const struct CobDecimal *d2);`
* **Purpose:** Handles arbitrary-precision decimal mathematics required for `COMP-3` and numeric display calculations, checking for size errors and scale adjustments.

---

## 4. File I/O Symbol Manifest (`mbunta-isam` Routing)

All file operations intercepted from generated C code route directly into the high-performance `mbunta-isam` storage subsystem[cite: 4].

### Open / Close Subsystem
* `extern "C" int cob_open(struct CobFile *file, int mode, int optional);`[cite: 4]
  * *Modes:* `COB_OPEN_INPUT` (1), `COB_OPEN_OUTPUT` (2), `COB_OPEN_I_O` (3), `COB_OPEN_EXTEND` (4).
* `extern "C" int cob_close(struct CobFile *file);`[cite: 4]
  * *Options:* `COB_CLOSE_NORMAL` (0), `COB_CLOSE_LOCK` (1), `COB_CLOSE_NO_REWIND` (2).

### Read / Write Operations
* `extern "C" int cob_read(struct CobFile *file, int options);`[cite: 4]
* `extern "C" int cob_read_next(struct CobFile *file, int options);`[cite: 4]
* `extern "C" int cob_write(struct CobFile *file);`[cite: 4]
* `extern "C" int cob_rewrite(struct CobFile *file);`[cite: 4]
* `extern "C" int cob_delete(struct CobFile *file);`[cite: 4]
* `extern "C" int cob_start(struct CobFile *file, int op, const struct CobField *key);`[cite: 4]
  * *Relational Operators:* `COB_COMPARE_EQ` (=), `COB_COMPARE_GT` (>), `COB_COMPARE_GE` (>=), `COB_COMPARE_NOT_LT` (!<).

---

## 5. Sort Engine Symbol Manifest (`cambrian_sort` Routing)

* **Initialization:** `extern "C" void cob_s_init(void);`[cite: 4]
* **Record Ingestion:** `extern "C" void cob_s_add(const void *record, size_t size);`[cite: 4]
* **Execution Dispatch:** `extern "C" int cob_s_sort(void *sort_context, int (*comparator)(const void *, const void *));`[cite: 4]
* **Finalization & Streaming:** `extern "C" void cob_s_finish(void *target_handle);`[cite: 4]
* **Procedure Release & Return Hooks:**
  * `extern "C" void cob_s_release(const struct CobField *record);`
  * `extern "C" int cob_s_return(struct CobField *record);`
  * *Purpose:* Services explicit `RELEASE` and `RETURN` verbs inside `INPUT PROCEDURE` and `OUTPUT PROCEDURE` coroutines during sorting operations.

---

## 6. Program Control, Call Registry, & Parameter Stacking (`CALL` / `CANCEL`)

* **Module Calling:** `extern "C" int cob_call(const char *name, int argc, ...);`[cite: 4]
  * Resolves static linkage symbols or dynamically loads module entry points within the runtime mesh.
* **Module Cancellation:** `extern "C" void cob_cancel(const char *name);`[cite: 4]
  * Unloads module states and releases workspace memory tied to the specified program identifier.
* **Parameter Stacking Helpers:**
  * `extern "C" void cob_parameter_add(struct CobField *field, int mode);`
  * `extern "C" void cob_parameter_clear(void);`
  * *Purpose:* Manages argument passing boundaries (`BY REFERENCE`, `BY CONTENT`, `BY VALUE`) before invoking `cob_call`.

---

## 7. String, Unstring, & Inspect Subsystem

* **String Manipulation (`STRING`):**
  * `extern "C" void cob_string_init(struct CobField *dest, struct CobField *pointer);`
  * `extern "C" void cob_string_delimited(struct CobField *delim);`
  * `extern "C" void cob_string_append(const struct CobField *src);`
  * `extern "C" int cob_string_finish(void);`
* **Unstring Operation (`UNSTRING`):**
  * `extern "C" int cob_unstring_init(const struct CobField *src, struct CobField *tally);`
  * `extern "C" void cob_unstring_delimited(const struct CobField *delim);`
  * `extern "C" int cob_unstring_field(struct CobField *dest, struct CobField *delim_found, struct CobField *count);`
* **Inspection Operation (`INSPECT`):**
  * `extern "C" void cob_inspect_start(struct CobField *field, int type);`
  * `extern "C" void cob_inspect_data(const struct CobField *pattern, const struct CobField *replace);`

---

## 8. Table Subscripting & Bounds Verification

* **Subscript Check Helpers:**
  * `extern "C" int cob_subscript_check(int val, int lower, int upper);`
  * *Purpose:* Evaluates runtime array indices against table bounds (`OCCURS` clauses), triggering sub-script range error exceptions if violated.

---

## 9. Exception Handling & Status Registers

* **Exception Setter:** `extern "C" void cob_set_exception(int exception_code);`
* **Exception Getter:** `extern "C" int cob_exception_code(void);`
* **Purpose:** Populates and reads internal status codes, triggering conditional branch behaviors for `INVALID KEY`, `AT END`, size error routines, and range violations.