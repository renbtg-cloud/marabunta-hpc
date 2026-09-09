# Annex B: Normative Struct Layout Specifications (`cambrian_cobol_annex-struct_layouts.md`)

* **Component Owner:** Cambrian Radiation Core Engineering
* **Target Compatibility:** GnuCOBOL 3.2 Generated C Intermediate Code (`cobc -C`)
* **Linkage Constraint:** Zero `-lcob` Dependency (Clean-Room Memory Mirroring)
* **Status:** Draft / Active / Normative

---

## 1. Memory Alignment & Platform Layout Principles

To ensure zero-copy binary compatibility with C source files emitted by upstream `cobc -C`, all memory structures implemented within the Rust/C runtime core **MUST** be mapped using strict C-compatible memory layouts (`#[repr(C)]` in Rust). 

* **Platform Bitness:** Structures are designed for 64-bit target architectures (LP64 / LLP64), matching native pointer sizes and alignment padding rules enforced by modern GCC and Clang toolchains.
* **Mutability Constraints:** Pointers passing across the FFI threshold must maintain strict mutability rules as expected by the generated intermediate code.

---

## 2. Field Attribute Layout (`CobFieldAttr`)

The `CobFieldAttr` structure defines the metadata type, scale, precision, and flag attributes associated with a COBOL data item.

    #[repr(C)]
    #[derive(Debug, Copy, Clone)]
    pub struct CobFieldAttr {
        pub type_code: u16,    // COBOL data type identifier (numeric, alphanumeric, packed-decimal)
        pub digits: u16,       // Total number of digits (precision)
        pub scale: i16,        // Scaling factor / decimal places (implied decimal point / V clause)
        pub flags: u32,        // Bitfield flags (signed, synchronized, blank-when-zero, etc.)
    }

### Field Attribute Bitmask Constants (`flags`)
* `COB_FLAG_HAVE_SIGN` (0x0001): Field contains an explicit or implicit algebraic sign.
* `COB_FLAG_SIGN_SEPARATE` (0x0002): Sign character occupies a separate character position (Leading/Trailing Separate).
* `COB_FLAG_SIGN_REVERSE` (0x0004): Sign is stored in the high-order nibble/character rather than low-order.
* `COB_FLAG_BLANK_ZERO` (0x0008): Field displays as spaces if numeric value is zero (`BLANK WHEN ZERO`).
* `COB_FLAG_JUSTIFIED` (0x0010): Alphanumeric field is right-justified (`JUSTIFIED RIGHT`).
* `COB_FLAG_REAL_BINARY` (0x0020): Binary integer uses full native word width (`COMP-5`).
* `COB_FLAG_IS_CONSTANT` (0x0040): Field is read-only literal or constant workspace storage.

### Type Code Constants
* `COB_TYPE_BOOL` (0x01): Conditional / boolean switch
* `COB_TYPE_A` (0x02): Alphabetic text field
* `COB_TYPE_X` (0x03): Alphanumeric string field
* `COB_TYPE_AN` (0x04): Alphanumeric edited / general data
* `COB_TYPE_NUM_DISPLAY` (0x10): Numeric display (zoned decimal)
* `COB_TYPE_NUM_BINARY` (0x11): Native binary integer (`COMP`, `COMP-4`, `COMP-5`)
* `COB_TYPE_NUM_PACKED` (0x12): Packed decimal (`COMP-3`)
* `COB_TYPE_NUM_FLOAT` (0x13): Floating point representation (`COMP-1`, `COMP-2`)

---

## 3. Field Reference Layout (`CobField`)

The `CobField` structure represents an active data descriptor passed to runtime manipulation routines (`cob_move`, `cob_cmp`, arithmetic operations).

    #[repr(C)]
    #[derive(Debug, Copy, Clone)]
    pub struct CobField {
        pub size: usize,               // Total byte length of the field data buffer
        pub data: *mut u8,             // Raw pointer to the underlying mutable byte storage
        pub attr: *const CobFieldAttr, // Pointer to immutable field attribute metadata
    }

---

## 4. File Control Block Layout (`CobFile`)

The `CobFile` structure mirrors the file control block managed by GnuCOBOL and intercepted by the `mbunta-isam` storage subsystem.

    #[repr(C)]
    pub struct CobFile {
        pub filename: *const std::os::raw::c_char,      // Physical or logical file path string
        pub cob_name: *const std::os::raw::c_char,      // Program-defined SELECT identifier
        pub file_status: *mut u8,                       // Pointer to 2-byte status code buffer ("00", "23", etc.)
        pub organization: i32,                          // Sequential, Line-Sequential, Relative, Indexed
        pub access_mode: i32,                           // Sequential, Random, Dynamic
        pub record_min: usize,                          // Minimum record byte length
        pub record_max: usize,                          // Maximum record byte length
        pub record: *mut CobField,                      // Pointer to current active record buffer field
        pub nkeys: i32,                                 // Total number of keys defined for indexed files
        pub keys: *mut CobFileKey,                      // Pointer to array of file key descriptor structures
        pub internal_handle: *mut std::os::raw::c_void, // Engine-internal routing handle (mbunta-isam context)
    }

### Organization Constants
* `COB_ORG_SEQUENTIAL` (1)
* `COB_ORG_LINE_SEQUENTIAL` (2)
* `COB_ORG_RELATIVE` (3)
* `COB_ORG_INDEXED` (4)

### Access Mode Constants
* `COB_ACCESS_SEQUENTIAL` (1)
* `COB_ACCESS_RANDOM` (2)
* `COB_ACCESS_DYNAMIC` (3)

---

## 5. Indexed File Key Descriptors (`CobFileKey` & `CobKeySegment`)

For indexed file organizations (`INDEXED`), GnuCOBOL maps primary and alternate keys via explicit key descriptor structures to handle multi-segment record keys and duplication permissions.

    #[repr(C)]
    #[derive(Debug, Copy, Clone)]
    pub struct CobKeySegment {
        pub offset: i32,    // Byte offset of the key segment within the record buffer
        pub size: i32,      // Byte length of the key segment
    }

    #[repr(C)]
    pub struct CobFileKey {
        pub flag: i32,               // Key flags (e.g., duplicates allowed flag)
        pub nsegs: i32,              // Number of segments composing this key
        pub segments: *mut CobKeySegment, // Pointer to array of key segments
    }

### Key Flags
* `COB_KEY_DUPLICATES` (0x01): Key permits duplicate values (`WITH DUPLICATES`)

---

## 6. Decimal Arithmetic Workspace Layout (`CobDecimal`)

The `CobDecimal` structure handles arbitrary-precision packed decimal and numeric evaluations, preventing overflow during complex calculations.

    #[repr(C)]
    pub struct CobDecimal {
        pub scale: i16,          // Current decimal scale factor
        pub precision: i16,      // Total significant digits
        pub value: [u8; 64],     // Internal normalized digit / sign storage array
    }