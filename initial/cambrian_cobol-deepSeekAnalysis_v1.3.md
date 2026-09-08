Gap analysis: what's missing before DeepSeek can implement cambrian_cobol_spec-v1.3.md
Analysis is based only on cambrian_cobol_spec-v1.3.md, ignoring all other files as requested. No build/tooling red tape included.

Overall verdict: v1.3 is a skeleton. The only implementation-concrete parts are the two int64_t COMP-3 helpers in Section 3 and the high-level sort pipeline in Section 4. The central deliverable — libcambrian_cobol as a libcob drop-in, and mbunta-isam — is named but never specified well enough to implement.

1. Critical contradiction that must be resolved first (Sections 1–2)
The spec says libcambrian_cobol is a drop-in replacement for libcob, implemented clean-room to avoid LGPL entanglements, but the example links it against the real library:


Apply
gcc -O3 -shared -fPIC src/cambrian_cobol.c -I/usr/include/cobol -lcob -o libcambrian_cobol.so
This is internally contradictory:

If libcambrian_cobol.so links -lcob, it is a shim/wrapper over libcob, not a replacement.
A drop-in replacement must not link against the library it is replacing.
The stated LGPL-avoidance goal (Section 2) is defeated by -lcob.
The spec must decide: pure clean-room replacement (remove -lcob, implement every required symbol), or interposer/wrapper (then the LGPL and licensing story needs to change). This decision changes the entire implementation surface.

2. The ABI layer itself is never specified (Section 2)
Section 2 says libcambrian_cobol "implements the exact public function entry points and data structures required by GnuCOBOL 3.2" — but never enumerates them.

Missing:

The list of cob_* symbols that the generated C actually references.
The supported subset of those symbols: cob_init, cob_tidy, cob_stop_run, cob_check_version, cob_open/cob_close/cob_read_next/cob_write/cob_rewind/cob_delete, cob_field accessors (cob_get_int, cob_get_uint, cob_get_decimal, cob_get_pic, cob_get_str, cob_put_*), cob_move, cob_add/cob_sub/cob_mul/cob_div/cob_compare, decimal arithmetic, intrinsic/functions entry points, sort entry points, call/cancel/program-registry entry points.
A verb → runtime-call mapping matrix for the COBOL source language. Without it, an implementer cannot know what must be implemented for ADD, MOVE, STRING, SEARCH ALL, CALL, SORT, READ, etc.
The cobc -C flag contract: which compiler flags/options are supported (-farithmetic=, -fnotrunc, -fbinary-size, -fstatic-call, -std=, EBCDIC/ASCII mode). These options change the emitted C and therefore change what the runtime receives.
Program lifecycle semantics: cob_init, cob_tidy, cob_stop_run, cancellation, ON EXCEPTION/EXCEPTION handling, cob_set_exception, language exception codes/status.
Version negotiation: how libcambrian_cobol responds when generated code calls cob_check_version for GnuCOBOL 3.2, and what happens on mismatch.
The headers only describe struct layouts and symbols; they do not state which subset this product supports. That is the single biggest missing item.

3. Rust-vs-C boundary is undecided (throughout)
The spec's preferred implementation language is Rust, yet:

The only implementation code is C11 static inline (Section 3).
The example builds src/cambrian_cobol.c, not Rust.
There is no architecture for the FFI boundary between Rust and the C ABI/GnuCOBOL headers.
An implementer needs:

Where bindgen/cbindgen or hand-written #[repr(C)] Rust struct mirrors of cob_field/struct cob_file live.
How Rust exports #[no_mangle] extern "C" entry points matching the cob_* ABI.
Whether the C COMP-3 helpers are compiled separately and called from Rust, or ported to Rust.
The panic/unwinding policy: Rust panics must be caught and converted into cob_set_exception/return codes at the FFI boundary.
The threading model: GnuCOBOL runtimes are historically single-threaded COBOL-process wide; Rust forces a decision on Send/Sync, per-thread cob_init, and static mutable runtime state.
4. Field/arithmetic layer is far too narrow (Section 3)
The COMP-3 section is int64_t-only, but GnuCOBOL COBOL arithmetic is cob_decimal (up to 38 significant digits).

Missing:

The strategy for PIC 9(19) through PIC 9(38). The spec never declares a precision ceiling nor defines arbitrary-precision compatibility with GnuCOBOL's cob_decimal layout/arithmetic.
Zoned decimal / DISPLAY numerics: PIC 9(n) DISPLAY, PIC S9(n) DISPLAY with embedded sign overpunch (F0–D0/C0), trailing/leading/separate SIGN IS variants.
Binary usage types: BINARY, COMP, COMP-5, PACKED-DECIMAL, FLOAT (COMP-1/COMP-2).
Implied decimal / scaling behavior: PIC 9(5)V99, PIC P scaling, rounding, truncation.
How the COMP-3 helpers connect to cob_field attributes (digits, scale, usage). Right now they consume only (src, len); the runtime needs to know the backing cob_field and its PIC so it can validate input and signal ON SIZE ERROR.
Error mapping: the helper returns -1 on invalid data, but there is no mapping from that return to COBOL SIZE ERROR/exception status.
5. mbunta-isam is mentioned but completely unspecified (Sections 1 and 3)
It appears as a routing target by name only. There is zero specification for:

The API contract between GnuCOBOL file I/O hooks and the ISAM engine.
Which file organizations are supported: SEQUENTIAL, LINE SEQUENTIAL, RELATIVE, INDEXED.
SELECT/ASSIGN mapping semantics: filenames, environment variables, device names.
Indexed-file constructs: RECORD KEY, ALTERNATE RECORD KEY, WITH DUPLICATES, ACCESS MODE IS DYNAMIC/RANDOM/SEQUENTIAL.
START semantics: equal/>=/>/NOT </NOT >/partial-key comparison.
FILE STATUS code behavior: 00, 02, 10, 22, 23, 24, 35, 46, 49, with exact trigger conditions.
Locking, record sharing, buffering, crash recovery, transactionality.
On-disk format (if any) for ISAM tables: B-tree/LSM, key compression, duplicates, free space management.
Without this, the "Engine Routing" claim in Section 3 is not implementable.

6. cambrian_sort is a pipeline sketch, not a contract (Section 4)
Missing items needed to implement it against GnuCOBOL 3.2:

The exact cob_* entry points generated by cobc for SORT/MERGE, e.g. cob_alloc_sort, comparator setup, release/return hooks. The spec says "integrates directly with GnuCOBOL's table and file sort hooks" but does not name them or define the argument/ABI contract.
Key descriptor format: how generated code passes ON ASCENDING/DESCENDING KEY — cob_field pointers, offsets, lengths, sign handling, multi-key order.
Comparator contract: what a GnuCOBOL-compatible comparator function pointer looks like, and how it receives collating sequence overrides.
COLLATING SEQUENCE representation: ALPHABET table layouts, default ASCII vs EBCDIC, locale/national collations, NATIVE/EBCDIC/STANDARD-1/2 semantics.
INPUT PROCEDURE / OUTPUT PROCEDURE coroutine integration, including the RELEASE/RETURN verb mappings.
Multiple USING / multiple GIVING behavior.
MERGE semantics: presorted input requirements, stability, ordering verification.
Stability guarantee: COBOL SORT is expected stable for equal keys; the spec never states this.
Spill/merge details: temp file format, run size caps, max merge fan-in, cleanup on process termination, I/O error path.
Memory budget: "default 512 MB" is stated, but no configuration override mechanism or behavior at saturation.
WITH DUPLICATES IN ORDER handling.
7. "Out of scope" boundaries are ambiguous (Section 5)
Section 5 disables some features, but the edges are unclear:

"All ACCEPT, DISPLAY terminal screens ... disabled." Ambiguous: are these supported or not?
DISPLAY to stdout/stderr (batch logging)
ACCEPT FROM ENVIRONMENT-VARIABLE
ACCEPT FROM DATE/TIME/DAY-OF-WEEK
DISPLAY ... UPON CONSOLE vs UPON SYSOUT
No supported-verb matrix: the spec never states whether it supports arithmetic verbs, MOVE, INITIALIZE, STRING, UNSTRING, INSPECT, SEARCH/SEARCH ALL, OCCURS DEPENDING ON, CALL/CANCEL/ENTRY, FUNCTION intrinsics, Report Writer, etc.
Dynamic CALL/program registry (COB_LIBRARY_PATH, nested programs, EXTERNAL, LOCAL-STORAGE) is never addressed.
STOP RUN / GOBACK / EXIT PROGRAM exit behavior and return codes are unspecified.
8. Cross-cutting runtime semantics missing
Program registry and dynamic program resolution: calls to cob_call, cob_cancel, cob_resolve, nested programs, EXTERNAL linkage.
Working-storage initialization: INITIALIZE, VALUE clause handling, initial program state, local storage per invocation.
Diagnostics/error reporting: what happens inside cambrian_cobol on invalid data, I/O errors, decimal overflow; where output goes (stderr, cob_set_exception, both).
Environment variable handling: which GnuCOBOL env vars are honored (COB_LIBRARY_PATH, COB_CONFIG_DIR, COB_SORT_MEMORY, etc.).
Conformance/acceptance criteria: what success looks like behaviorally is never defined. "Drop-in for libcob" is the only criterion, but with no symbol/behavior compatibility tests described.
9. Minor document issues
Filename says v1.3; the document header says Version: 1.2.1-RELEASE.
"mbunta-isam" appears only in Section 1 and Section 3 text, with no dedicated section — which feels like a copy/paste omission given how central it is.
Prioritized next-step plan
If you want, I can draft the missing spec content in the following order:

ABI Compatibility Contract — symbol support matrix, verb→runtime-call mapping, resolved -lcob contradiction, Rust/C FFI boundary.
Numeric field architecture — move beyond int64 COMP-3 to a declared precision model and the full usage-type set.
mbunta-isam engine contract — file organization matrix, FILE STATUS semantics, SELECT/ASSIGN mapping, index format.
cambrian_sort contract — named cob_* hooks, comparator/collation rules, key descriptors, spill/merge/error behavior.
Operational support matrix — batch ACCEPT/DISPLAY variants, CALL/CANCEL, and an explicitly supported COBOL verb set.
