Analysis: What's Missing or Incorrect in cambrian_cobol_spec-v1.1.md
The file describes a GnuCOBOL ABI drop-in replacement runtime, but it does not contain enough ABI, API, semantic, or build information for an implementation to be produced reliably. Several statements are also inaccurate or internally contradictory.

Below are the specific gaps and corrections needed.

1. Critical Version and Identity Problems
1.1 Inconsistent versioning
The file is named cambrian_cobol_spec-v1.1.md, but its header says:

Title references cambrian_cobol_spec.md
Version says 1.02-RELEASE
DeepSeek needs one unambiguous spec version. Right now it is impossible to know which document version is authoritative.

1.2 GnuCOBOL ABI version is not pinned
The spec says "GnuCOBOL ABI Compatible" and "libcob drop-in replacement", but never says which GnuCOBOL version:

GnuCOBOL 2.2
GnuCOBOL 3.1.2
GnuCOBOL 3.2
The cob_file layout, function prototypes, generated C code, and exported symbols all change between releases. Implementing an ABI-compatible libcob without pinning the upstream version is not possible.

Needed:
A fixed statement such as:

Target ABI: GnuCOBOL 3.1.2, 64-bit, C11, POSIX, configured with default --with-libcob ABI.

2. Architectural Contradiction
2.1 "No custom parser" vs "transpiler frontend rejection"
Section 1 says the project avoids writing a custom COBOL parser and relies on cobc -C.

But Section 5 says unsupported constructs are rejected by "the transpiler frontend":

ALTER is rejected at compile time
Unstructured GO TO crossing paragraph boundaries is rejected by the transpiler frontend
ACCEPT, DISPLAY, and SCREEN SECTION are disabled
There is no custom transpiler frontend in the proposed architecture. GnuCOBOL's cobc will happily compile GO TO, ALTER, ACCEPT, and DISPLAY.

Needed:
An actual implementation mechanism for enforcing those boundaries. For example:

A static source-lint/scan step before cobc
GnuCOBOL compiler configuration/warnings
Post-processing of cobc C output
Link-time stubs that fail noisily
DeepSeek cannot implement "compile-time rejection" from a policy sentence alone.

3. The ABI Surface Is Substantially Incomplete
3.1 cobc -C does not emit code that only calls cob_open, cob_read, etc.
The spec implies intercepting a handful of file I/O functions is enough. In reality, GnuCOBOL-generated C code calls a large runtime surface:

Initialization: cob_init, cob_tidy, cob_exit
Field allocation: cob_malloc, cob_free, cob_cache_alloc
Field operations: cob_field_fld, cob_field_size, cob_field_display
Data movement: cob_move, cob_fast_move
Arithmetic: cob_add, cob_sub, cob_mul, cob_div, cob_cmp, cob_decimal
Numeric conversion: cob_set_int, cob_get_int, cob_set_picx, etc.
Control flow/runtime: cob_set_location, cob_runtime_error, cob_setjmp, cob_longjmp
Program call: cob_call, cob_cancel
Display: cob_display, cob_accept
File I/O: a larger set than shown
Sort: the entire cob_file_sort* family
Environment/configuration: cob_getenv, cob_global
The spec only defines:

8 file functions
2 COMP-3 helpers
A prose description of sort
This is far too small to link any real cobc -C output.

Needed:
A complete symbol manifest obtained from a representative GnuCOBOL-generated object. DeepSeek should be given something like:


Apply
nm -u sample_module.o | grep cob_
Or a portability test harness that compiles representative COBOL programs with cobc -C, then links with the new runtime. The first acceptance test should be "the produced object links successfully", not behavioral correctness.

4. cob_file Definition Is Not GnuCOBOL ABI-Compatible
4.1 The structure is invented, not aligned
The spec says the shown cob_file is "aligned with GnuCOBOL libcob internal file descriptor layout". It is not.

Real GnuCOBOL's cob_file is much larger and contains fields such as:

FILE *fp
int fd
char *filename
cob_field *select_name
cob_field *file_status
organization, access, lock, open mode bitmasks
special flags
size_t record_min, record_max
cob_field *record
key definitions
sort-related pointers
internal mutexes, buffers, etc.
The struct in the spec omits most of these and appends a custom:


Apply
void *vfs_handle;
4.2 Appending fields is ABI-breaking
If generated C is compiled against the real GnuCOBOL headers, it expects the real struct size and offset layout. Appending vfs_handle changes sizeof(cob_file) and may conflict with or misalign later fields.

A drop-in replacement must either:

Include the exact upstream GnuCOBOL cob_file definition, or
Reuse the upstream public/internal headers wholesale
If vfs_handle is needed, it must map onto an existing reserved or unused upstream field, never be appended arbitrarily.

Needed:
The exact upstream cob_file definition from the pinned GnuCOBOL version, copied from its headers, with explicit documentation of which existing fields are repurposed.

5. Intercepted Function Prototypes Are Incorrect or Unspecified
The spec gives these prototypes:


Apply
int cob_open(cob_file *file, int mode, int optional);
int cob_close(cob_file *file, int option);
int cob_read(cob_file *file, cob_field *key, int options);
int cob_read_next(cob_file *file, int options);
int cob_write(cob_file *file, int options);
int cob_rewrite(cob_file *file, int options);
int cob_start(cob_file *file, cob_field *key, int op);
int cob_delete(cob_file *file);
These do not match GnuCOBOL's actual exported prototypes. For example, GnuCOBOL file APIs typically include additional arguments for sharing, disposition, lock options, and often have different record/key parameters.

Needed:
Exact prototypes copied from GnuCOBOL's libcob.h / fileio.h for the pinned version. Without this, the runtime will compile but will not link or will crash at the ABI boundary.

6. cob_field Is Referenced but Never Defined
The spec refers to cob_field * repeatedly but never defines:

Its structure layout
Its type codes
Its flag bits
Its size, digits, scale fields
Its data pointer
Its PIC information
Since cobc -C generated code manipulates cob_field directly, the runtime cannot be implemented without it.

Needed:
The complete cob_field definition and associated constants from the pinned GnuCOBOL headers.

7. COMP-3 Implementation Has Defects and Unspecified Behavior
The provided mb_unpack_comp3 and mb_pack_comp3 are useful, but not sufficient as a complete FFI subsystem.

7.1 Undefined behavior on INT64_MIN
This code:


Apply
if (val < 0) val = -val;
has undefined behavior when val == INT64_MIN.

7.2 Packed buffer may contain uninitialized bytes
mb_pack_comp3 does not zero the destination buffer before packing. If the field is larger than needed, high nibbles remain uninitialized.

7.3 No digit-capacity validation
The spec never states the digit capacity of a byte length. For example, a 4-byte COMP-3 field holds 7 digits plus sign. The code only detects overflow after the fact.

7.4 No overflow detection during unpack
accum = (accum * 10) + digit can overflow int64_t silently beyond the supported digit count.

7.5 Platform-specific sign preference is unspecified
The accepted sign nibbles are listed, but the default generated sign when packing positive/negative values should be explicitly specified. GnuCOBOL may use 0xC/0xD by default, but 0xF is also common.

7.6 Only COMP-3 is covered
GnuCOBOL-generated code also requires:

COMP/COMP-4
COMP-5
DISPLAY numeric
Binary and native-endian fields
Needed:
Corrections and additions:

Handle INT64_MIN safely
Zero-fill destination buffers
Define digit capacity clearly
Add range/overflow checks
Specify exact sign nibbles generated for positive/negative
Clarify whether COMP and COMP-5 conversion is in scope
Add endianness requirements
Add tests for boundary values: 0, positive max, negative max, invalid nibbles, invalid sign
8. Sort Engine Is Too Vague to Implement
Section 4 describes a merge sort, but no implementable ABI or API is given.

Missing details:
Which exact GnuCOBOL sort symbols must be intercepted
Function prototypes for those symbols
Comparator function signature
How ASCENDING/DESCENDING KEY descriptions are represented
How COLLATING SEQUENCE is represented
Struct definitions for sort keys
How to invoke USING procedures
How to invoke GIVING output
How to invoke OUTPUT PROCEDURE as a coroutine
Record layout in memory
Run file format
How variable-length records are stored in spill files
Whether sorting is single-threaded or parallel
Error handling on write failure
Configuration variable name for 512MB
What happens when MERGE has duplicate keys
Needed:
Concrete sort API prototypes and data structures, ideally taken from GnuCOBOL's fileio.h, plus an end-to-end example COBOL program with expected behavior.

9. File I/O Semantics and Status Codes Are Almost Entirely Missing
The spec says operations are intercepted, but does not define their semantics:

Record format: fixed, variable, line sequential
Blocking and deblocking rules
Indexed file key handling
Primary and alternate keys
Duplicate-key behavior
Dynamic/random access rules
Read next/previous semantics
Rewrite vs write differences
Delete semantics
File sharing/locking
File status code mapping
Needed:
A table mapping backend I/O outcomes to COBOL file status codes:

COBOL status	Meaning	Trigger
00	Success	normal completion
02	Duplicate alternate key	duplicate allowed
10	End of file	read next at EOF
22	Duplicate key	primary key violation
23	Key not found	read/start target missing
35	File not found	open input missing
39	Attribute mismatch	organization/access conflict
41	Already open	double open
9x	Implementation-defined error	backend failure
This was present in related versions of the specification, but is absent from the v1.1 file under analysis.

10. mbunta-isam / mb_sort Interfaces Are Referenced but Never Defined
The pipeline says file operations and sorts are redirected to:

mbunta-isam
mb_sort
But the compact document never defines:

The C API of mbunta-isam
How vfs_handle maps to an mbunta backend driver
How records and keys flow across that boundary
Which component owns buffers
Error translation rules
Thread-safety guarantees
Needed:
A separate interface contract is required before the runtime can be implemented. The spec should either:

Include the full mbunta-isam driver API, or
Treat mbunta-isam as an abstract interface and define its function table explicitly
11. No Build, Layout, or Link Specification
The spec never states:

Directory structure
Header file names
Library output name and SONAME
Whether the result is static, shared, or both
Build system: Make, CMake, Meson
Include paths for GnuCOBOL headers
Link flags
Symbol visibility rules
Version-script requirements
How to build against mbunta-isam
Required compiler flags for C11 and POSIX
Needed:
A minimal repository layout and build contract. For example:


Apply
include/cambrian_cobol/
src/
  fileio/
  decimal/
  sort/
  runtime/
tests/
  cobol_fixtures/
  abi_smoke/
CMakeLists.txt
And a link command that must succeed against real cobc -C output.

12. No Acceptance Criteria or Test Fixtures
The spec defines no way to verify the implementation.

Needed:
At minimum:

A representative COBOL program using sequential files
A representative COBOL program using indexed files
A representative COBOL program using SORT/MERGE
A representative COBOL program using COMP-3
Known expected output or file contents for each
A link-compatibility test proving all required GnuCOBOL symbols are resolved
Negative tests for unsupported constructs
13. Priority Summary for the Specification Update
Priority	Missing/corrected item
P0	Pin exact GnuCOBOL version
P0	Replace invented cob_file with upstream definition
P0	Add complete cob_field definition
P0	Replace incorrect function prototypes with upstream prototypes
P0	Provide full required-symbol manifest
P0	Resolve no-parser vs transpiler-frontend contradiction
P1	Define file I/O semantics and status mapping
P1	Define complete sort API and data structures
P1	Define mbunta-isam interface
P1	Fix COMP-3 undefined behavior and buffer initialization
P1	Specify build system, layout, and link flags
P2	Add acceptance tests and fixtures
P2	Clarify version/naming inconsistency
Bottom Line
The current cambrian_cobol_spec-v1.1.md is a conceptual sketch, not an implementation-ready specification. It would tell DeepSeek what the component should do in broad terms, but not how it must be built and verified.

The most important correction is architectural: the spec claims to be a GnuCOBOL ABI-compatible libcob drop-in replacement, yet it invents private structures and omits nearly the entire ABI. For DeepSeek to implement it correctly, the spec must first be expanded with the exact GnuCOBOL version, upstream header definitions, full symbol list, precise file and sort semantics, build targets, and test cases.