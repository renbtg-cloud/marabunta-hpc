# Annex D: File Status Mapping & Error Subsystem (`cambrian_cobol_annex-file_status.md`)

* **Component Owner:** Cambrian Radiation Core Engineering
* **Target Compatibility:** GnuCOBOL 3.2 Generated C Intermediate Code (`cobc -C`)
* **Linkage Constraint:** Zero `-lcob` Dependency (Clean-Room Memory Mirroring)
* **Status:** Draft / Active / Normative

---

## 1. Overview & Architectural Bound

The `mbunta-isam` storage subsystem interacts with the generated COBOL program via the `CobFile` control block defined in Annex B. Every file input/output operation (`OPEN`, `READ`, `WRITE`, `REWRITE`, `DELETE`, `START`, `CLOSE`) produces an integer return code from the storage engine. 

The runtime core **MUST** translate these internal engine return codes into a standard two-character COBOL file status buffer (`file_status` pointer in `CobFile`), allowing generated code to evaluate success, `INVALID KEY`, `AT END`, or permanent physical errors.

---

## 2. File Status Key Class Architecture

A COBOL file status consists of a 2-byte alphanumeric field (`XX`). The first digit (key class) determines the high-level category of the operation result:

* `0x30` (`'0'`) — **Successful Completion:** The operation executed without anomalies.
* `0x31` (`'1'`) — **At End:** Sequential read encountered logical end-of-file.
* `0x32` (`'2'`) — **Invalid Key:** Key violation (duplicate key, record not found, boundary violation).
* `0x33` (`'3'`) — **Permanent Error:** Hardware failure, missing file, permission denied, or physical corruption.
* `0x34` (`'4'`) — **Logic Error:** Sequence or boundary violation inconsistent with file organization.
* `0x39` (`'9'`) — **Implementer-Defined:** Subsystem-specific extensions or fatal engine faults.

---

## 3. Normative Translation Table (`mbunta-isam` -> COBOL Status)

The following table defines the normative mapping from `mbunta-isam` internal error codes to standard COBOL two-byte status strings.

| Internal Engine Code (`i32`) | COBOL File Status (`[u8; 2]`) | Category / Description |
| :--- | :--- | :--- |
| `0` | `"00"` | Success: Successful completion of I/O verb. |
| `2` | `"02"` | Success: Duplicate key accepted (`WITH DUPLICATES`). |
| `4` | `"04"` | Success: Record length mismatch tolerated / truncated. |
| `7` | `"07"` | Success: File close with no rewind / reel status. |
| `10` | `"10"` | At End: Logical end of file reached during sequential read. |
| `22` | `"22"` | Invalid Key: Duplicate key value on write/rewrite. |
| `23` | `"23"` | Invalid Key: Record not found on read, start, or delete. |
| `24` | `"24"` | Invalid Key: Boundary violation / key sequence error. |
| `30` | `"30"` | Permanent Error: Physical I/O error or media failure. |
| `34` | `"34"` | Permanent Error: Boundary write failure / disk full. |
| `35` | `"35"` | Permanent Error: Mandatory file not found (`OPEN INPUT`). |
| `37` | `"37"` | Permanent Error: Permission denied or open mode conflict. |
| `38` | `"38"` | Permanent Error: Closed file violation or lock conflict. |
| `39` | `"39"` | Permanent Error: Attribute conflict (record length / key mismatch). |
| `41` | `"41"` | Logic Error: File already open. |
| `42` | `"42"` | Logic Error: File not open. |
| `43` | `"43"` | Logic Error: Read/Write sequence error. |
| `44` | `"44"` | Logic Error: Record size mismatch during operation. |
| `90` | `"90"` | System Error: General implementer-defined `mbunta-isam` fault. |
| `91` | `"91"` | System Error: Decentralized mesh synchronization timeout. |

---

## 4. Exception Handling & Declaratives Integration

When an operation returns a non-zero status code, the runtime core updates the status buffer and evaluates the following control flow rules:

* **Automatic Status Propagation:** The 2-byte code is copied directly into the memory address referenced by `file_status` in the active `CobFile` structure.
* **Conditional Branching:** If the generated code includes explicit `INVALID KEY` or `AT END` phrases, the runtime sets internal exception flags verified by generated conditional jumps.
* **USE AFTER ERROR Declaratives:** If a file error triggers a status class `'3'` or `'9'`, the runtime core intercepts execution and dispatches control to registered `DECLARATIVES` error procedures if defined within the source program.