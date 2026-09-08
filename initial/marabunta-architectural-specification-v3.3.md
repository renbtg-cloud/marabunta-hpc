# MARABUNTA MESH & CAMBRIAN COBOL ARCHITECTURAL SPECIFICATION (v3.3)
*A Brutalist Engineering Blueprint for Distributed Legacy Batch Processing, Polyglot Hybrid Execution, State Isolation, Pragma-Level Concurrency Strategies, and Runtime Monopoly Disruption*

---

## 1. Executive Summary & Market Disruption Thesis

The enterprise cloud mainframe modernization sector is cornered by an extractive monopoly. Enterprises migrating off physical mainframes are funneled into ecosystems like AWS Mainframe Modernization, where they are locked into proprietary managed runtimes (predominantly Rocket Software / Micro Focus Enterprise Server). 

This infrastructure imposes a predatory, recurring software tax: **$5.55 per CPU core per hour for the runtime**, and **$2.14 per core per hour for developer tooling**. For a mid-market enterprise running an 8-core batch production cluster 24/7, this generates tens of thousands of dollars in pure software licensing fees monthly, completely decoupled from underlying cloud compute costs.

### The Cambrian Counter-Architecture
`cambrian_cobol` and `Marabunta` eliminate this economic extraction through four foundational pillars:
1. **Zero Runtime Tax:** Compiling legacy COBOL code into lean, statically linked native binaries that execute without metering agents, license keys, or cloud marketplace tollbooths.
2. **GnuCOBOL Integration:** Leveraging GnuCOBOL as the rock-solid transpiler (translating COBOL to C), which is compiled via standard toolchains (GCC/Clang) linked against the optimized `cambrian_cobol` runtime.
3. **Decentralized Mesh Sharding (`Marabunta`):** Breaking the sequential bottleneck of overnight batch windows by sharding iteration blocks and datasets across cheap, commodity node clusters (bare-metal, Hetzner, or spot instances) rather than forcing expensive vertical scaling on oversized cloud infrastructure.
4. **Polyglot Hybrid Extension (Python & Fortran Bridges):** Allowing engineers to retain deterministic record-level COBOL while cleanly delegating high-level orchestration, vector math (NumPy), or heavy numerical/actuarial subroutines (Fortran) via explicit, human-audited pragmas and zero-copy FFI bindings.

---

## 2. End-to-End System Architecture Flow

```
[Legacy COBOL Source Tree (.cob)]
               │
               ▼
   [cambrian-scan CLI Tool] ──► Injects Draft Pragmas (*> @marabunta-distribute? / @cambrian-call-*)
               │
               ▼
   [Human Engineer Audit]  ──► Confirms/Edits to Active Pragmas & Authors External Logic (.py / .f90)
               │
               ▼
      [GnuCOBOL Transpiler] ──► Converts COBOL AST to Optimized Intermediate C Code
               │
               ▼
      [GCC / Clang Build]   ──► Links against cambrian_cobol Runtime, mbunta-isam & FFI Bridges
               │
               ▼
   [Native Execution Binary] ──► Orchestrated Locally, via Marabunta Mesh, or Hybrid Polyglot Runtimes
```

---

## 3. `cambrian_cobol` Runtime Engine

`cambrian_cobol` is not a bloated enterprise compiler; it is a high-performance, low-level execution and data-handling layer engineered for deterministic execution speed and absolute memory discipline.

### 3.1 Memory Discipline & Data Layout
* **Zero-Copy Memory Mapping (`mmap`):** Large sequential files and indexed tables are mapped directly into virtual memory, avoiding expensive user-space buffer allocations and copy overhead.
* **Packed Decimal (`COMP-3`) Native Handling:** Arithmetic and logical operations on packed decimal fields execute via high-performance bitwise shifting and raw byte manipulation, matching IBM mainframe math precision without floating-point rounding errors.
* **Stack & Static Allocation:** Variables and working storage blocks are pre-allocated or managed with strict stack discipline. Garbage collection pauses are structurally impossible.

### 3.2 The `mbunta-isam` Data Layer
Legacy COBOL systems rely heavily on ISAM (Indexed Sequential Access Method) files (`.idx`, `.dat`) which break down in distributed environments.
* **Streamable Key-Value Backend:** Replaces legacy index structures with a high-throughput, log-structured storage engine optimized for sequential batch read/write blocks.
* **File Status Mapping:** Strictly maps standard COBOL return codes (`FILE-STATUS 00`, `23`, `35`) to ensure complete compatibility with existing error-handling paragraphs (`INVALID KEY`, `AT END`).

### 3.3 Subroutine Linkage (`CALL` Mechanics)
* **Static Resolution:** When the target of a `CALL` statement is known at compile time, the runtime resolves it directly to a native function pointer, eliminating lookup overhead.
* **Dynamic Hash Maps:** For dynamic calls (`CALL ws-program-name`), the runtime utilizes an optimized internal symbol hash map to resolve entry points instantly without OS-level registry thrashing.
* **Parameter Transfer (`USING`):**
  * `BY REFERENCE`: Passes raw mutable memory pointers (`&mut [u8]`) directly into the subprogram with zero copying.
  * `BY CONTENT`: Allocates an isolated immutable snapshot on the stack or local scratchpad buffer prior to execution.

---

## 4. `Marabunta` Decentralized Mesh Fabric

`Marabunta` is a leaderless, fault-tolerant execution fabric engineered to coordinate parallel batch workloads across distributed compute nodes without centralized bottlenecks.

### 4.1 Topology & Communication
* **Gossip Protocol:** Nodes maintain cluster awareness through lightweight background gossip exchanges, tracking node health, partition assignments, and resource utilization.
* **Zero Central Master:** There is no single point of failure or master coordinator node. Any node can ingest a job, partition the payload, and distribute shards across the mesh.

### 4.2 Payload Sharding & Execution
* **Context Serialization:** Iteration bounds, loop indices, linkage parameters, and subset data keys are serialized into compact binary frames.
* **Peer Offloading:** Shards are transmitted over TCP/QUIC to peer nodes, which execute the localized compiled binary slice against their assigned data partition.
* **State Reconciliation:** Upon completion, nodes return execution status codes and modified data diffs back to the originating coordinator, ensuring atomic batch synchronization.

---

## 5. Pragma Parsing & Human-in-the-Loop Workflow

To eliminate "verification debt" and prevent automated tooling from introducing catastrophic logic errors into mission-critical financial software, **all mesh and polyglot optimizations are strictly opt-in and human-verified**. The machine **never** generates calculation code or business logic autonomously.

### 5.1 The `cambrian-scan` CLI Tool
An independent Rust-based CLI utility that scans legacy COBOL codebases to identify optimization vectors:
* Scans control flow graphs for heavy nested loops (`PERFORM VARYING`).
* Identifies mathematical bottlenecks, actuarial accumulators, and candidate file reads.

### 5.2 Draft Pragma Taxonomy
When potential optimizations are detected, the tool injects commented draft suggestions directly into the source code:

```cobol
      *> [CAMBRIAN SUGGESTION] -> Heavy calculation/iteration block detected.
      *> @cambrian-call-python? [REVIEW REQUIRED: Verify array boundaries and memory mapping]
          PERFORM VARYING IDX FROM 1 BY 1 UNTIL IDX > 1000000
              COMPUTE WS-INTEREST-AMT(IDX) = WS-PRINCIPAL(IDX) * WS-RATE
          END-PERFORM.
```

### 5.3 Human Audit & Activation
1. **Review:** The enterprise engineer reviews the code, confirming safety and writing the corresponding external module (`.py` or `.f90`).
2. **Commit:** The engineer edits the comment into an active pragma (`*> @cambrian-call-python "math.py:func"`) or deletes it if side effects require local execution.
3. **Execution:** GnuCOBOL transpiles the code, and the runtime respects verified active pragmas while ignoring unvetted drafts.

---

## 6. Polyglot Hybrid Extension (Python & Fortran Bridges)

Legacy financial and actuarial systems often require heavy mathematical crunching or matrix operations that crawl inside procedural COBOL scalar loops. Rather than rewriting entire systems, `cambrian_cobol` exposes explicit FFI bridge pragmas to link user-authored **Python** (NumPy/Pandas) and **Fortran** (`.f90`) high-performance modules.

### 6.1 The Ironclad Rule
**The build toolchain never generates calculation code.** The user writes the external script or Fortran subroutine; the pragma strictly establishes the linkage interface.

### 6.2 COMP-3 FFI Type Parity Rule (v3.3 Update)
Packed decimal (`COMP-3`) data structures store values as Binary Coded Decimal (BCD) nibbles. Python, NumPy, and standard C-ABI structures have no native representation for BCD nibble layouts. 
* **Rule:** All FFI parameter buffers passing numeric data across language boundaries **must be explicitly typed as binary integers (`COMP`, `COMP-5`, or `C int/long`) or display numeric (`PIC X` / `PIC 9`)**. If legacy business logic requires `COMP-3`, the runtime must pass it through a native unpacking macro into a temporary binary buffer prior to bridge invocation.

### 6.3 Python Integration Vector
* **Use Case:** High-level batch orchestration, parallel data grouping, and vectorized array calculations via NumPy.
* **Mechanism:** Embedded interpreter or zero-copy buffer passing via `ctypes`/`cffi`.
* **Pragma Syntax Example:**
  ```cobol
      *> @cambrian-call-python "vector_math.py:compute_portfolio_risk"
          PERFORM VARYING IDX FROM 1 BY 1 UNTIL IDX > 500000
              COMPUTE WS-RISK-SCORE(IDX) = ...
          END-PERFORM.
  ```

### 6.4 Fortran Integration Vector
* **Use Case:** Ultra-fast numerical computing, actuarial mortality tables, and legacy scientific/financial matrix math where Fortran's native array syntax outperforms Python.
* **Mechanism:** Fortran subroutines are compiled into native shared libraries (`.so`) using standard compilers (gfortran) and linked directly through C-compatible interfaces (`ISO_C_BINDING`).
* **User-Authored Fortran Subroutine (`actuarial_core.f90`):**
  ```fortran
  subroutine calc_mortality(buffer_ptr, count) bind(c, name="calc_mortality")
      use iso_c_binding
      type(c_ptr), value :: buffer_ptr
      integer(c_int32_t), value :: count
      ! High-performance vectorized numerical math...
  end subroutine calc_mortality
  ```
* **Pragma Syntax Example:**
  ```cobol
      *> @cambrian-call-fortran "libactuarial_core.so:calc_mortality"
          PERFORM VARYING IDX FROM 1 BY 1 UNTIL IDX > 100000
              COMPUTE WS-MORTALITY-VAL(IDX) = ...
          END-PERFORM.
  ```

---

## 7. Advanced Control Flow: Hybrid Nested Loop Mechanics & Re-entrancy

When handling complex multi-tier nested loops where outer control flow or math is delegated to Python/Fortran while inner record-level business logic remains native COBOL, architectural separation is mandatory.

### 7.1 Subroutine Re-entrancy via `LOCAL-STORAGE`
In GnuCOBOL, standard `WORKING-STORAGE` inside subprograms retains state across multiple `CALL` statements unless declared `RECURSIVE` or `INITIAL`. To guarantee zero-initialization and eliminate dirty values across repeated external driver calls, all transient loop indices and scratchpad buffers within encapsulated subroutines **must use the `LOCAL-STORAGE SECTION` instead of `WORKING-STORAGE`**.

### 7.2 Code Architecture Example

**The User-Authored Python Driver with Parallel Shard Dispatch (v3.3 Update):**
```python
import ctypes
from concurrent.futures import ProcessPoolExecutor

# Load compiled COBOL inner execution block
cobol_lib = ctypes.CDLL("./libinner_batch.so")

def execute_region_shard(reg_idx):
    c_index = ctypes.c_int(reg_idx)
    cobol_lib.EXECUTE_REGION_BLOCK(ctypes.byref(c_index))
    return f"Region {reg_idx} complete."

def run_parallel_outer_loop(max_regions):
    # Utilizing ProcessPoolExecutor to bypass CPython GIL bottlenecks
    with ProcessPoolExecutor(max_workers=4) as executor:
        futures = [executor.submit(execute_region_shard, i) for i in range(1, max_regions + 1)]
        results = [f.result() for f in futures]
    return results
```

**The Corresponding Pragma-Annotated COBOL Program:**
```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN-BATCH-PROCESS.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01 REG-IDX          PIC 9(4) COMP.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Starting parallel hybrid batch run..."

           *> @cambrian-call-python-loop "region_driver.py:run_parallel_outer_loop"
           *> [EXPLICIT OVERRIDE: Outer loop handled externally via multi-processing, inner execution remains COBOL]
               PERFORM VARYING REG-IDX FROM 1 BY 1 UNTIL REG-IDX > 5
                   PERFORM EXECUTE-REGION-BLOCK-INTERNAL
               END-PERFORM.

           STOP RUN.
```

**The Compiled COBOL Subroutine (`EXECUTE-REGION-BLOCK-INTERNAL`):**
```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EXECUTE-REGION-BLOCK-INTERNAL.
       DATA DIVISION.
       LOCAL-STORAGE SECTION.
       *> LOCAL-STORAGE ensures clean re-entrancy and zero-initialization per call entry
       01 REP-IDX          PIC 9(4) COMP.

       PROCEDURE DIVISION.
           *> Native record-level business logic executing at raw compiled speed
           PERFORM VARYING REP-IDX FROM 1 BY 1 UNTIL REP-IDX > 100
               PERFORM PROCESS-REP-RECORD
           END-PERFORM.
           GOBACK.

       PROCESS-REP-RECORD.
           CONTINUE.
```

---

## 8. Residual Engineering Risks & Architectural Countermeasures (v3.3 Update)

To guarantee absolute safety when moving from single-machine legacy execution to a distributed mesh and polyglot runtime, `cambrian_cobol` and `Marabunta` enforce strict compile-time and runtime invariants addressing fundamental procedural failure modes.

### 8.1 `WORKING-STORAGE` Global State Isolation & Concurrency Strategies
* **The Vulnerability:** Standard COBOL `WORKING-STORAGE` is globally allocated and static by default. In a distributed mesh sharding model, executing parallel loops across nodes or threads introduces race conditions if subroutines rely on implicit global state or shared temporary flags (`WS-FLAG`, shared accumulators).
* **Architectural Countermeasure (Strict Enforcement & Bounded Pragmas):**
  1. **Default Immutability:** By default, the `cambrian-scan` utility treats working storage as read-only during sharded mesh passes. Any routine modifying global variables outside its local execution frame without an explicit pragma is flagged as `UNSAFE FOR MESH` and forced to execute locally in serial order.
  2. **Commutative Reductions (`@cambrian-strategy reduce-*`):** For bounded financial totals or counters, developers can annotate loops with reduction strategies (e.g., `*> @cambrian-strategy reduce-sum "WS-TOTAL-BATCH"`). The mesh permits independent shards to maintain local isolated scratchpad accumulators, merging them deterministically via atomic reduction during final state reconciliation without network locking.
  3. **Workspace Cloning (`@cambrian-strategy clone-workspace`):** For temporary flags and ephemeral buffers confined strictly to an iteration's lifecycle, this pragma instructs the runtime to instantiate a thread/shard-local memory clone of the working-storage block for every parallel execution slice.
  4. **Read-Mostly Reference Tables (`@cambrian-strategy shared-readonly`):** Static lookup matrices (zip-code tax tables, mortality rates) populated at initialization are mapped across mesh nodes via zero-copy `mmap` regions, bypassing immutability blocks safely since subsequent writes are structurally prohibited.

### 8.2 Distributed ISAM Write Synchronization & Secondary Index Rebuilding
* **The Vulnerability:** Legacy ISAM architecture (`.idx`, `.dat`) assumes a single-machine paradigm with local file locking. Allowing independent nodes in a `Marabunta` mesh to simultaneously write to overlapping key spaces or handle alternate indexes introduces index corruption.
* **Architectural Countermeasure (Orthogonal Partitioning & Post-Reconciliation Index Rebuild):**
  1. **Mathematically Orthogonal Partition Keys:** `Marabunta` prohibits arbitrary distributed record locking. Sharding requires strict partition-key design (e.g., Node A owns Region 1 key ranges; Node B owns Region 2 key ranges). Key spaces must be mutually exclusive across mesh peers.
  2. **Append-Only Log Segments (`mbunta-isam`):** During the distributed batch phase, nodes bypass live distributed lock managers entirely, writing modifications to isolated, append-only log segments.
  3. **Deterministic Single-Pass Merge & Secondary Index Rebuild:** Once parallel mesh nodes report completion and return status codes (`FILE-STATUS 00`), the coordinator executes a deterministic merge phase. **For legacy schemas containing alternate keys (`ALTERNATE RECORD KEY ... WITH DUPLICATES`), secondary index trees are fully rebuilt post-reconciliation** rather than relying on raw log append merging to prevent sequence collisions.

---

## 9. Observability and Brutalist Logging

Enterprise monitoring suites are frequently bloated, expensive SaaS traps. `cambrian_cobol` and `Marabunta` reject this complexity entirely:
* **Structured NDJSON Streams:** Node logs, mesh status checks, and batch execution metrics are emitted as high-speed Newline-Delimited JSON streams directly to standard output (`stdout`).
* **Zero Telemetry Tax:** No background monitoring agents, no external metric-collection tolls, and no cloud-vendor telemetry lock-in. Logs integrate seamlessly with any open-source collection stack (Grafana/Loki/Promtail) or write straight to local storage files.