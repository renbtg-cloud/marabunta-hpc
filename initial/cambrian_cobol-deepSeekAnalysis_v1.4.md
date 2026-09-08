# DeepSeek Gap Analysis: `cambrian_cobol_spec-v1.4.md`

## Bottom line

The spec is an **architecture and routing intent**, not yet an implementation-ready ABI contract. DeepSeek can implement modules, but cannot currently produce a correct `libcambrian_cobol` that generated GnuCOBOL 3.2 C code will actually link and run against, because the **exact ABI surface, struct layouts, and operational semantics are mostly unspecified**.

Below is what is missing, organized by severity.

---

## 1. Missing: a complete, normative `cob_*` symbol manifest

Section 4 names only a handful of entry points. GnuCOBOL 3.2 generated C code can reference a much larger runtime surface. Without a complete manifest, implementation is guesswork.

The spec must add, per function:

- exact C signature and calling convention
- return value semantics
- whether the function is provided by GnuCOBOL 3.2 generated code directly, or only implicitly referenced
- panic/error behavior across FFI

Missing areas include at least:

- lifecycle: `cob_init`, `cob_tidy`, `cob_stop_run`, `cob_check_version`, `cob_exit`
- field/memory: `cob_malloc`, `cob_free`, `cob_field_*`, `cob_move`, `cob_cmp`, `cob_store`
- decimal arithmetic: `cob_decimal_*`
- file: `cob_open`, `cob_close`, `cob_read`, `cob_write`, `cob_rewrite`, `cob_delete`, `cob_start`, plus sequential/random next-file variants
- sort: `cob_s_init`, `cob_s_add`, `cob_s_sort`, `cob_s_finish`, and any key/config helpers
- CALL/CANCEL: `cob_call`, `cob_cancel`, program registry internals
- exception handling: `cob_set_exception`, `cob_set_cancel`, `cob_raise_exception`, `cob_exception_code`
- unsupported-feature stubs: `cob_display`, `cob_accept`, screen-related symbols, `cob_alter` if generated code ever references it

The current "Core Verb-to-Runtime Mapping Matrix" is a routing table, not an ABI contract. It is not sufficient to implement against.

---

## 2. Missing: exact GnuCOBOL 3.2 struct layouts

The Rust mirror in §5.1 is simplified and likely incorrect at the ABI level.

Specific problems:

- `CobFieldAttr` is missing GnuCOBOL's `pic` pointer and may have incorrect flag/type widths.
- `CobFile` has far more members in real GnuCOBOL 3.2 than the nine fields shown. It normally carries key definitions, lock state, open mode, file-function pointers, report/line-sequential state, etc.
- `CobFile.record` is modeled as `*mut CobField`, but the record buffer in the runtime file structure is almost certainly not a `CobField *`; this needs to be corrected against the target headers.
- There is no definition for:
  - `cob_file_key`
  - `cob_decimal`
  - exception structures
  - select/assign structures
  - alternate key descriptors
  - file lock descriptors

The spec needs an **ABI Annex** that defines these structs byte-for-byte, or explicitly says "taken from host headers at build time" for each one.

---

## 3. Missing: semantic behavior of each function

Even where symbol names are listed, the spec mostly says what it should do, not how it behaves at the ABI boundary.

Examples:

- `cob_init` currently stubs to `0` and ignores arguments. The spec says it must initialize memory workspaces, exception registries, and thread-local state, but never defines those structures or initialization requirements.
- `cob_stop_run` is described as returning a POSIX exit status, but GnuCOBOL's actual behavior may be process termination or longjmp-like control transfer.
- `cob_check_version` has no accepted version tuple, no success/failure convention, and no defined handling when `COB_LIBRARY_PATH` or runtime options are present.
- `cob_move` is mentioned but not specified: padding rules, numeric alignment, category conversion, overlapping buffers, and status/exception effects are undefined.
- CALL/CANCEL has no program lookup rules, symbol naming convention, dynamic library search path, or cancellation semantics.

---

## 4. Missing: `mbunta-isam` is named but not specified

Section 7 is only a high-level contract. To implement the file subsystem, the following are missing:

- exact `cob_open`, `cob_read`, etc. signatures tied to `cob_file`
- indexed file on-disk format
- primary and alternate key index structure
- duplicate-key handling and overflow chains
- record addressing and locking protocol
- full status-code behavior, not just the eight codes listed
- persistence/recovery behavior
- interaction with `SELECT/ASSIGN` environment variables such as `COB_FILE_PATH`, `DD_*`, or `COB_ASSIGN`

The status-code table is incomplete for a realistic GnuCOBOL runtime. Missing/at-risk statuses include `04`, `05`, `07`, `14`, `30`, `34`, `37`, `38`, `39`, `41`, `42`, `43`, `47`, `48`, and the OS-error `5x`/`9x` families.

---

## 5. Missing: `cambrian_sort` hook contract

The sort section describes behavior but not the implementation ABI:

- exact signatures for `cob_s_init`, `cob_s_add`, `cob_s_sort`, `cob_s_finish`
- key descriptor layout: byte offset, length, direction, collating sequence
- comparator function pointer signature
- how records are represented when passed to/from the sort engine
- spill run file format
- memory budget configuration surface
- behavior when `cob_s_finish` feeds an output procedure or `GIVING` file

---

## 6. Missing: arithmetic/decimal contract

The spec routes `ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE` to `cob_add`, `cob_sub`, `cob_mul`, `cob_div`, but actual GnuCOBOL generated arithmetic is commonly expressed through `cob_decimal_*` helper functions. This must be verified and specified exactly.

Also missing:

- `COMP`, `COMP-4`, `COMP-5`, and binary field handling
- decimal rounding, scaling, and `ON SIZE ERROR` semantics
- status/exception behavior for overflow and divide-by-zero
- how `cob_move` interacts with numeric/packed and numeric-display fields

---

## 7. Missing: concurrency model decision

The spec says `Arc<Mutex<T>>` **or** thread-local contexts, which is an unresolved design choice, not a specification.

It must define:

- process-wide state vs per-shard state
- what `cob_init` does under concurrent usage
- whether file handles can be shared across threads
- how GnuCOBOL's inherently global generated-C state is isolated per shard
- lock ordering for file and sort operations

---

## 8. Missing: unsupported-feature behavior

The spec says `ACCEPT`, `DISPLAY`, `SCREEN SECTION`, and `ALTER` are unsupported, but it does not define what happens when generated C still references their runtime symbols.

This is important because the project does not have its own frontend linter, so unsupported source can still reach translation. The spec must define one of:

- explicit stub symbols and return codes
- compile-time translation failures
- runtime fatal error behavior

---

## 9. Missing: acceptance/validation criteria

The spec has no ABI-conformance test requirement:

- no set of sample COBOL programs
- no expected outputs
- no "golden" `cobc -C` output to compare against
- no status-code conformance matrix
- no version-handshake acceptance test

DeepSeek can implement the code, but cannot demonstrate that the clean-room runtime is ABI-correct without this.

---

## 10. Specific defects/inconsistencies already present

These should be corrected before implementation:

1. **§2.1 vs §2.2 contradiction:**  
   §2.1's example links with `-lcob`; §2.2 explicitly prohibits `-lcob`. The final link strategy must be unambiguous.

2. **`mb_pack_comp3` / `mb_unpack_comp3` are broken for even-digit PICs:**  
   For a 4-digit field containing `1234`, the current packing example produces `00 23 4C` rather than the GnuCOBOL-compatible `12 34 0C`. The algorithm only works for odd digit counts such as `9(5)`.

3. **Naming inconsistency:**  
   §1 says `mb_sort`; §4 says `cambrian_sort`; §6 says `cambrian_sort`; §7 says `mbunta-isam`. One canonical name per component is needed.

4. **`cob_init` return type/prototype:**  
   The spec uses `int cob_init(...)`. Actual GnuCOBOL 3.2 compatibility must be verified from the target headers before locking this ABI.

5. **Rust/C placement is unclear:**  
   §3 is C11 `static inline`, but §5 is Rust-first. The spec should state whether COMP-3 packing is implemented in Rust or in a required C shim.

6. **Arithmetic symbol names are uncertain:**  
   `cob_add`, `cob_sub`, etc. may not be the real generated-code hooks; the spec must confirm or replace them with the actual GnuCOBOL decimal symbols.

---

## What should be added to the spec before implementation

1. **Annex A — ABI Manifest:** every exported symbol, signature, return convention, and constant value.
2. **Annex B — Struct Layouts:** exact `cob_field`, `cob_field_attr`, `cob_file`, `cob_file_key`, `cob_decimal`, and related types.
3. **Annex C — Function Semantics:** pre/postconditions, errors, exceptions, stubs.
4. **Annex D — `mbunta-isam` design:** index format, locking, status codes, persistence.
5. **Annex E — `cambrian_sort` hook and spill format.**
6. **Annex F — Concurrency and sharding state model.**
7. **Annex G — Acceptance test corpus.**

The most urgent missing item is Annex A + Annex B: **the exact GnuCOBOL 3.2 ABI surface**. Everything else can be designed, but without that, the exported library cannot be trusted to link or run generated C.