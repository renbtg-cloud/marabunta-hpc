# Annex E: Decimal Arithmetic Precision & Rounding Norms (`cambrian_cobol_annex-arithmetic.md`)

* **Component Owner:** Cambrian Radiation Core Engineering
* **Target Compatibility:** GnuCOBOL 3.2 Generated C Intermediate Code (`cobc -C`)
* **Linkage Constraint:** Zero `-lcob` Dependency (Clean-Room Memory Mirroring)
* **Status:** Draft / Active / Normative

---

## 1. Overview & Architectural Bound

The `cambrian_cobol` runtime core implements an arbitrary-precision decimal arithmetic engine to support COBOL computational clauses (`COMP-3`, `PACKED-DECIMAL`, `DISPLAY` numeric items) and arithmetic verbs (`ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE`, `COMPUTE`, exponentiation)[cite: 5]. 

To ensure exact parity with GnuCOBOL 3.2 behavior without relying on external libraries like `-lcob`, all calculations operate on normalized byte arrays via the `CobDecimal` workspace structure[cite: 5]. 

To prevent CPU stalling, memory churn, domain panics, stack overflows, ABI unwinding hazards, and aliasing bugs during heavy batch workloads, arithmetic execution enforces strict performance mitigations, thread isolation guards, and correctness constraints[cite: 5].

---

## 2. Stack-Framed Scratchpad Arena, RAII Guards, Error Isolation, & Concurrency Build Variants

To eliminate heap allocation churn in inner loops and prevent workspace aliasing during nested complex expressions, compound `COMPUTE` statements, and recursive intrinsic function calls, flat thread-local pointers are prohibited in favor of stack-framed scoping[cite: 5].

* **Scoped RAII Arena Specifications & Lifetime Branding:** 
  To ensure memory safety, prevent dangling or cross-thread arena contamination, and prohibit out-of-order drop sequences, the runtime utilizes a lightweight scoped RAII guard struct (`ScratchpadArenaGuard<'a>`) branded with invariant lifetime markers (`PhantomData<*mut &'a ()>`)[cite: 5]. 
  This guarantees rigid Last-In, First-Out (LIFO) drop order bound directly to the execution stack[cite: 5]. 
  Every entry into an arithmetic expression kernel executes a push operation via the guard, scoping temporary workspaces to the active frame and instantly reclaiming memory upon drop[cite: 5].

* **Frame-Scoped Error Registers & Shadow Stacks:** 
  To eliminate cross-thread race conditions and state pollution of execution error flags across re-entrant FFI boundaries, global or unstructured error variables are strictly prohibited[cite: 5]. 
  The active error status register and exception codes are bound directly to the stack-framed `ScratchpadArenaGuard` struct, ensuring each execution frame maintains its own isolated error context[cite: 5]. 
  Furthermore, to prevent re-entrant callbacks, user-defined functions, or declarative exception procedures (`USE AFTER STANDARD ERROR`) from overwriting parent error statuses, the guard implements an internal error register push/pop shadowing stack that isolates inner execution frames and restores outer status codes upon exit[cite: 5].

* **Arena Stack Depth Hard Cap:** 
  To guard stack-framed scratchpads against infinite recursive intrinsic calls or excessively nested compound statements, the runtime enforces a strict ceiling of 256 nested scratchpad frames within the context[cite: 5]. 
  Any attempt exceeding this depth boundary immediately aborts execution via a zero-panic status code return (`CobStatus::StackOverflow`) and telemetry dumps[cite: 5].

* **Workspace Structure & Two-Tier Scale Limits:**

    pub struct CobDecimal {
        pub scale: i32,                 // Widened scale factor (transient up to ±256, stored clamped at ±128)
        pub precision: i32,             // Total significant digits tracked (up to 38 digits)
        pub value: SmallVec<[u8; 128]>, // Hybrid high-density inline/spillover normalized packed BCD byte buffer
    }

* **Two-Tier Scale Operational Clamp, Scale-Delta Guardrails, & AST Root Deferral Trade-Offs:** 
  To prevent unbounded memory consumption or allocation denial-of-service from runaway scale inflation during pathological compound operations while avoiding premature failures on mathematically valid intermediate states, scale attributes utilize a robust **two-tier scale model**[cite: 5]. 
  During multi-term intermediate normalization and factor alignment phases, working scales are permitted a transient workspace ceiling of $\pm 256$[cite: 5]. 
  To prevent spurious intermediate overflow exceptions during complex multi-term compound expressions where an intermediate scale temporarily breaches $\pm 128$ before a subsequent division operation scales it back into valid bounds, the hard normative operational scale clamp ($\pm 128$) is **deferred until the entire parent compound expression tree finishes evaluation**, allowing working scales to expand up to $\pm 256$ across inner nodes. 
  * *Trade-Off & Caveat:* Deferring the clamp increases register pressure and memory footprint inside the scratchpad arena as working scales transiently expand. The stack-framed arena and SmallVec buffers must allocate sufficient headroom (`SmallVec<[u8; 128]>` with pool spillover) to accommodate these wider transient representations without triggering unexpected heap spillover or block-pool exhaustion during heavy recursive computations. 
  Any terminal scale exceeding $\pm 128$ after rounding concludes still triggers a normative overflow exception[cite: 5]. 
  Furthermore, pre-flight scale delta verification checks operand scale disparities prior to buffer allocation, preventing unexpected heap spillage and cache-busting allocations during extreme scale alignment shifts, utilizing checked primitives (`i32::checked_add` and `i32::checked_sub`) to eliminate integer wrap-around vulnerabilities[cite: 5].

* **Pooled High-Density Small-Vector Allocation & Infrastructure Fault Isolation:** 
  The internal value buffer utilizes an expanded small-vector architecture maintaining an inline stack-allocated buffer of 128 bytes (`SmallVec<[u8; 128]>`), ensuring that standard intermediate expressions and division series up to 31 decimal places remain stack-allocated for optimal L1/L2 cache locality[cite: 5]. 
  To prevent raw stack footprint spikes across the 256-frame depth ceiling, temporary workspace slots are pooled within each stack-framed arena guard[cite: 5]. 
  Extreme multi-term scale expansions spill over into pre-allocated fixed-capacity block pools managed by the arena; if exhausted, the allocator safely returns `CobStatus::OutOfMemory`[cite: 5]. 
  Crucially, this infrastructure fault is cleanly decoupled from business-level arithmetic exceptions and never maps to standard COBOL `ON SIZE ERROR` handling, triggering an immediate fatal stack unwind and telemetry dump instead[cite: 5].

* **Schema-Aware Packed-Decimal Metadata Validation:** 
  To handle legacy EBCDIC and ASCII migration datasets without breaking on platform variants, signed packed-decimal (`COMP-3`) sign nibbles accept an extended matrix (`C`, `A`, `F`, `E` as valid positive indicators; `D`, `B` as valid negative indicators)[cite: 5]. 
  Furthermore, layout descriptors inject schema metadata (`is_signed: bool`)[cite: 5]. 
  For fields explicitly declared as *unsigned*, the decoder bypasses sign-nibble checks entirely and treats all nibbles (including trailing fillers or `F` nibbles) strictly as numeric data, preventing false parity corruption[cite: 5].

### Concurrency Deployment Variants: Build Option A vs. Option B
Depending on the host deployment topology and execution scheduling architecture, `cambrian_cobol` offers two distinct compile-time build artifacts[cite: 5]:

* **Option A (Synchronous Core-Pinned Build — Default & Recommended):** 
  This is the default and most recommended build variant for running full, traditional COBOL programs compiled via GnuCOBOL 3.2 (`cobc -C`)[cite: 5]. 
  Under this mode, the `ScratchpadArenaGuard` is compiled with strict `non-Send` and `non-Sync` semantics, binding workspaces directly to the execution stack of the calling OS thread[cite: 5]. 
  Because standard GnuCOBOL generated C code is synchronous and blocking, Option A incurs zero synchronization overhead and maximizes raw instruction throughput on dedicated OS threads[cite: 5].

* **Option B (Asynchronous Coroutine-Safe Build with Lock-Free Atomic State Transitions):** 
  This variant is engineered for modern hybrid runtimes, asynchronous host environments, and specialized distributed node architectures—such as **Marabunta-mesh-extracted loops**—where lightweight computational tasks may yield control and migrate across worker threads[cite: 5]. 
  Under Option B, all transient computational state—including subnormal square root shift factor memoization caches and frame error registers—is anchored atomically within the stack-framed `ScratchpadArenaGuard` rather than thread-local storage (TLS)[cite: 5]. 
  Furthermore, the runtime enforces strict **non-yielding synchronous execution invariants**: any arithmetic expression running within a `ScratchpadArenaGuard` must complete synchronously on the active thread stack and cannot cross an async `.await` yield point mid-evaluation[cite: 5]. 
  To protect against uncooperative async executor runtimes that abandon futures mid-execution without polling them to completion, Option B implements a **lock-free atomic state transition model** (`AtomicU8` status flags) and **thread-local block pools** in `ScratchpadArenaGuard`: 
  1. **Primary Path:** The future polls to completion, resetting the atomic state cleanly. 
  2. **Drop-Time Fail-Safe:** If `Drop` executes while the atomic flag still indicates active ownership (meaning the future was abandoned unpolled), the guard fires an immediate fail-safe routine: it updates the atomic status to `TaskYieldAborted`, flushes structural telemetry metrics, and zero-fills the arena block slice using thread-local pools, completely eliminating mutex convoy contention during mass coroutine cancellations across Marabunta mesh nodes.

---

## 3. Intermediate Calculation Precision & Scaling Rules

Arithmetic operations involving multiple data items dynamically derive intermediate precision and scaling to prevent premature truncation or overflow[cite: 5]:

* **Unified Precision Tier for Mixed Operations:** 
  To prevent premature precision truncation when chaining operations with differing working limits across mixed compound expressions spanning multiple arithmetic verbs, all intermediate workspace nodes inherit the highest precision tier (38 significant digits), restricting the 31-decimal constraint strictly to transcendental series convergence buffers[cite: 5].

* **Addition & Subtraction (`ADD`, `SUBTRACT`):** 
  The operational scale of the intermediate workspace is automatically set to the maximum scale among all operands involved in the expression, computed via checked scale validation[cite: 5]:
  $$\text{Scale}_{\text{inter}} = \max(\text{scale}_1, \text{scale}_2, \dots, \text{scale}_n)$$

* **Multiplication (`MULTIPLY`) & Widened Exponent Tracking:** 
  Prior to computing product scales, a checked addition is executed using `i32::checked_add`[cite: 5]:
  $$\text{Scale}_{\text{inter}} = \text{scale}_1 + \text{scale}_2$$
  If the computed sum breaches the transient $\pm 256$ working scale ceiling, or if an overflow occurs during checked addition, multiplication is bypassed instantly, triggering a size error flag and returning `CobStatus::SizeError` without corrupting the workspace[cite: 5]. 
  To prevent the standard 38-digit intermediate precision ceiling from prematurely truncating raw product growth, raw multiplication of two 38-digit values dynamically expands workspace precision to a double-precision 76-digit scratchpad[cite: 5]. 
  To prevent extreme scale alignment shifts near the boundaries from colliding with standard limits, the internal exponent tracking tier for this 76-digit expansion workspace is temporarily widened during raw multiplication and factor alignment[cite: 5]. 
  Once rounding concludes, the resulting value is checked against the normative $\pm 128$ boundary before final assignment[cite: 5].

* **Division (`DIVIDE`) & Unassigned Scale Derivation:** 
  Division operations expand intermediate scales to ensure maximum retention of fractional digits[cite: 5]. 
  For unassigned intermediate division chains (such as compound statements without an immediate assignment target), working scales are explicitly derived via checked arithmetic primitives[cite: 5]:
  $$\text{scale}_{\text{inter}} = \max\left(\text{scale}_{\text{dividend}}, \text{scale}_{\text{divisor}} + \max(0, 31 - \text{precision}_{\text{dividend}})\right)$$
  This value is clamped strictly within the operational limits, protecting against runaway scale inflation and infinite recurring decimal loops (hard-capped at 31 decimal places)[cite: 5].

* **Look-Ahead Expression Scale Propagation & Recursive AST Tree Traversal for Compound Division:** 
  To prevent premature fractional precision stripping during multi-term compound expressions and deeply nested or left-heavy parenthetical division trees (such as `COMPUTE X = (A / B) / (C / D)`), compiler-generated AST evaluation passes recursively pass down the maximum precision tier (38 significant digits) across all nested sub-branches[cite: 5]. 
  Non-terminal child division nodes do not default to isolated working scales, ensuring intermediate division results bypass the 31-decimal hard cap until final assignment or explicit rounding occurs[cite: 5].

---

## 4. Exponentiation Performance Architecture & Correctness Guards (`**` Operator)

To prevent linear time traps ($O(N)$), transcendental performance degradation, and domain corruption during power calculations, exponentiation routines adhere to strict execution rules[cite: 5]:

* **Integer Exponents & Magnitude Pre-Checks:** 
  All integer powers are evaluated via square-and-multiply binary exponentiation algorithms ($O(\log N)$)[cite: 5]. 
  Prior to loop entry, magnitude validation checks the exponent value; if an integer exponent is large enough to guarantee intermediate workspace expansion past the 38-digit precision cap, execution aborts immediately with `CobStatus::SizeError`[cite: 5].

* **Explicit Odd/Even Sign Parity Rules for Negative Bases:** 
  When raising a negative base ($\text{sign} < 0$) to a whole integer exponent ($\text{scale} = 0$), algebraic sign bits are explicitly forced[cite: 5]:
  * **Even Exponent ($n \bmod 2 == 0$):** Workspace sign bit is forced positive ($\text{sign} > 0$)[cite: 5].
  * **Odd Exponent ($n \bmod 2 \neq 0$):** Workspace sign bit is forced negative ($\text{sign} < 0$)[cite: 5].

* **Financial Fast-Paths & Bounded Square Root Extraction:** 
  Common financial and structural powers ($x^2$, $x^3$) bypass generalized multiplication routines, hitting optimized inline squaring routines directly within the scratchpad arena[cite: 5]. 
  Square root extraction fast-paths utilize high-precision Newton-Raphson iteration governed by strict safety bounds: pre-iteration checks ensure argument $S > 0$[cite: 5]. 
  To eliminate calculation drift from static subnormal floor clamping, subnormal arguments are evaluated with a rapid convergence pre-check; if within the standard window ($\epsilon_m \le S \le 10^{38}$), dynamic scaling is bypassed entirely[cite: 5]. 
  For clustered subnormal inputs ($S < \epsilon_m$), the computed shift factor $k$ is memoized directly within the active stack-framed `ScratchpadArenaGuard` (guaranteeing task-migration safety under Build Option B) and processed via an **even-power-of-10 dynamic scaling wrapper** ($S' = S \times 10^{2k}$), pre-checked against operational scale bounds[cite: 5]. 
  The Newton-Raphson iteration executes on $S'$ with standard precision, and the resulting root is scaled back down ($result = \text{root}' / 10^k$)[cite: 5]. 
  The iteration loop is strictly capped at 32 cycles with residual error bound $\epsilon = 10^{-31}$ and non-monotonic delta validation[cite: 5]. 
  A trailing delta plateau detector terminates execution with structural convergence if the delta magnitude remains flat across 3 consecutive cycles, even if hovering just above $\epsilon$[cite: 5]. 
  An explicit zero-derivative guard inside the loop body immediately short-circuits execution and returns `CobStatus::ZeroDivide` if any intermediate denominator evaluates to absolute zero[cite: 5].

* **Zero Base with Non-Positive Exponents ($0^0$ or $0^{-n}$):** 
  If the base evaluates to absolute zero and the exponent is zero, the engine returns an exact integer `1` per COBOL standards[cite: 5]. 
  If the exponent is negative ($< 0$), transcendental processing is bypassed, triggering a zero-division status code (`CobStatus::ZeroDivide`), setting the size error flag, and routing execution to the `ON SIZE ERROR` handler[cite: 5].

* **Domain Trap Isolation for Fractional Exponents:** 
  When evaluating $base^{exponent}$, if the base is negative ($\text{sign} < 0$) and the exponent contains a fractional component ($\text{scale} > 0$), transcendental series approximations are strictly prohibited[cite: 5]. 
  The runtime core immediately bypasses expansion, sets the size error flag, registers exception code `EC-ARITHMETIC-DOMAIN` (`0x4201`), and routes control to the `ON SIZE ERROR` handler[cite: 5].

* **Exact Rational Power-of-10 Pre-Check, Padé Acceleration with Hermite Smoothstep, & Relaxed Curvature Convergence:** 
  Prior to executing Cody-Waite argument reduction or Taylor series logarithmic approximation for $a^b$, the engine evaluates a strict rational pre-check[cite: 5]. 
  If base $a$ is a clean canonical power of 10 ($\exists k \in \mathbb{Z}, a = 10^k$), transcendental series evaluation is bypassed entirely[cite: 5]. 
  The logarithm $\log_{10}(a)$ returns exact integer $k$, eliminating floating-point series drift on exact decimal scaling factors[cite: 5]. 
  For non-trivial fractional powers, evaluation proceeds via base-10 logarithmic and exponential series approximations via $a^b = 10^{b \cdot \log_{10}(a)}$, scaled to 31 decimal places[cite: 5]. 
  For bases lying far outside the local unity window, the engine applies Cody-Waite style argument reduction by decomposing the base into mantissa and exponent ($a = m \times 10^e$), computing $\log_{10}(a) = \log_{10}(m) + e$ with $m$ strictly normalized into the convergence window before series evaluation, eliminating irrational constant multiplication chatter entirely[cite: 5]. 
  For normalized base inputs falling within the critical window near unity ($1.0 - \epsilon_r < \alpha < 1.0 + \epsilon_r$), the engine avoids slow linear convergence and catastrophic cancellation by deploying a **rational Padé approximant fallback** ($\frac{P(x)}{Q(x)}$) accelerated and smoothed by a **Hermite cubic smoothstep interpolation function** ($s(x)$) across the boundary band[cite: 5]:
  $$s(x) = 3\left(\frac{x - (1.0 - \epsilon_r)}{2\epsilon_r}\right)^2 - 2\left(\frac{x - (1.0 - \epsilon_r)}{2\epsilon_r}\right)^3$$
  This guarantees strict mathematical $\mathcal{C}^1$ continuity by matching both value and first-derivative slopes ($f'(x) = 0$) at the boundaries, entirely eliminating micro-kinks and cumulative cent-level audit drift[cite: 5]. 
  Taylor/Maclaurin series convergence loops enforce a **dual-check convergence condition**: a primary iteration ceiling of 64 cycles combined with an absolute epsilon threshold ($\epsilon = 10^{-31}$)[cite: 5]. 
  If convergence stalls just outside standard tracking when hitting the 64-cycle ceiling, a grace extension evaluates a monotonic delta check ($\delta < 10^{-28}$) combined with a relaxed **second-derivative curvature evaluation** ($\frac{d^2\delta}{dx^2} \le 0$) over the final 4 consecutive cycles, ensuring that flat inflection plateaus or zero-acceleration points near the precision limit are accepted as valid structural convergence rather than falsely rejected. 
  * *Trade-Off & Caveat:* Permitting a zero second-derivative compromises absolute mathematical strictness by allowing flatlined, non-decreasing delta streams to pass through. Therefore, this relaxation **must** remain strictly coupled with the absolute epsilon threshold ($\epsilon = 10^{-31}$); if the absolute delta remains outside acceptable tolerance, an unconstrained flatline still aborts execution with a domain error. 
  If both hold, structural convergence is accepted and the result routes to the rounding engine; if oscillating or diverging, execution aborts immediately with `CobStatus::DomainError`[cite: 5].

---

## 5. Division-by-Zero & Remainder Sign-Correction Engine

* **Division-by-Zero Interception:** 
  Prior to executing division kernels, the divisor's normalized value is checked against absolute zero[cite: 5]. 
  If zero, execution sets the size error flag, registers a division-by-zero status code, and routes control to the `ON SIZE ERROR` handler or suppresses target modification[cite: 5].

* **Normative Remainder Sign-Correction & Precision Insulation:** 
  For all `DIVIDE` statements utilizing the `REMAINDER` clause, unsigned or absolute-value modulus operations are prohibited[cite: 5]. 
  The remainder is computed explicitly via[cite: 5]:
  $$\text{Remainder} = \text{Dividend} - \left( \text{Trunc}(\text{Quotient}) \times \text{Divisor} \right)$$
  To prevent accumulation drift and maintain regulatory auditing parity, the product term $(\text{Trunc}(\text{Quotient}) \times \text{Divisor})$ must execute entirely within an un-truncated 38-digit intermediate workspace[cite: 5]. 
  Crucially, `Trunc` applies strictly to the integer quotient extraction step required for constructing the product term; the intermediate quotient and subsequent multiplication retain full un-truncated precision and fractional scaling until the subtraction from the dividend is fully completed[cite: 5]. 
  To prevent intermediate scale alignment bit-loss prior to multiplication, raw multiplication of the truncated quotient and divisor executes **unscaled** inside the 38-digit buffer; all scale alignment shifts and factor adjustments are strictly deferred until *after* the product term is fully subtracted from the dividend, eliminating 1-unit rounding mismatches. 
  `Trunc` is explicitly defined as truncation toward zero (discarding fractional digits without flooring toward negative infinity) to ensure exact execution parity with GnuCOBOL 3.2 integer division routines regardless of operand signs[cite: 5]. 
  The sign bit of the resulting remainder variable is then forced via bitwise assignment to match the algebraic sign bit of the original Dividend (defaulting to positive if absolute zero)[cite: 5].

---

## 6. Rounding Mode Norms

When results are moved from high-precision `CobDecimal` intermediate workspaces into target `CobField` variables with smaller scales or tighter digit limits, the runtime core applies explicit rounding modes based on compiler directives or standard COBOL defaults[cite: 5]:

* **Truncation (Standard Default / `REMAINDER` omission):** 
  Discards excess digits beyond the target scale without adjustment[cite: 5].

* **Nearest-Away-From-Zero (`ROUNDED`):** 
  Rounds the least significant retained digit upward if the discarded fraction is greater than or equal to 0.5[cite: 5].

* **Nearest-Even (`ROUNDED MODE IS NEAREST-EVEN`):** 
  Rounds toward the nearest even digit if the discarded fraction is exactly 0.5[cite: 5]. 
  To eliminate branch-heavy conditional overhead and instruction cache bloat during heavy financial batch runs, nearest-even tie-breaking executes via dedicated vector intrinsics (AVX2 for x86_64, NEON for aarch64) guarded by runtime CPU feature detection (`is_x86_feature_detected!("avx2")`)[cite: 5]. 
  Because `SmallVec<[u8; 128]>` and general COBOL workspace buffers maintain a **native packed BCD layout** (two decimal digits per byte) without strict 32-byte or 16-byte memory alignments, unaligned load primitives guarantee safe byte scanning across arbitrary offsets without triggering hardware segmentation faults (`SIGSEGV` or `SIGBUS`), falling back cleanly to unrolled scalar blocks on legacy hardware without triggering illegal instruction faults (`SIGILL`)[cite: 5]. 
  **Length-Gated SIMD Dispatch, AVX2 16-Byte Chunking, & Hardware Table Lookup Acceleration:** To prevent vector instruction overhead on narrow data and avoid performance cliffs, vector routines enforce a strict **length-gated SIMD dispatch rule**: vectorization triggers *only* when field lengths exceed 64 bytes ($\ge 128$ digits); shorter fields route through optimized unrolled scalar blocks. 
  Furthermore, to prevent AVX2 YMM registers (strictly 32 bytes / 256 bits) from overflowing when unpacking packed BCD digits (which doubles the memory footprint from input to unpacked lanes), AVX2 vector chunk sizes are strictly restricted to **16 bytes of packed BCD per iteration** (expanding to 32 bytes of unpacked lanes). 
  * *Trade-Off & Caveat:* Restricting AVX2 chunks to 16 bytes prevents register overflow but **halves parallel throughput per instruction cycle** compared to ideal 32-byte loads. To recapture maximum performance, the engine utilizes dynamic runtime dispatch to selectively leverage AVX-512 for 32-byte chunks where ZMM register width permits, while gracefully falling back to 16-byte AVX2 chunks or unrolled scalar blocks on legacy nodes. 
  When SIMD dispatch triggers, raw bytes utilize hardware-accelerated table lookups (`_mm256_shuffle_epi8`) combined with explicit nibble-lane expansion *before* applying extraction bitmasks or parity evaluations[cite: 5]. 
  Furthermore, to prevent the scanning engine from misinterpreting the trailing sign nibble (`C`, `D`, `F`, `A`, `E`, `B`) as a numeric data digit, the algorithm explicitly partitions high-order data nibbles from the low-order sign nibble: total nibble length is computed via $\text{length} = \text{byte\_len} \times 2 - 1$, and the scanning cursor terminates traversal strictly at nibble index $\text{length} - 2$[cite: 5]. 
  Finally, tie-breaking checks apply explicit vector bitmasks (`0x0F` for lower lanes, `0xF0` for upper lanes) on the expanded element vectors before evaluating odd/even parity[cite: 5].

* **Toward-Zero (`ROUNDED MODE IS TOWARD-ZERO`):** 
  Truncates directly regardless of sign[cite: 5].

* **Up / Down Directed Rounding:** 
  Enforces ceiling or floor adjustments based on algebraic sign constraints[cite: 5].

---

## 7. Overflow, Underflow, FFI Status Code Interception, & Negative Sign-Flip Safeguards

When an arithmetic operation produces a result exceeding the capacity of the target data item or handles sub-resolution conditions, the following sequence occurs[cite: 5]:

1. **Zero-Panic FFI Status Return & Unwind Protection (`#[repr(C)]`):** 
   All runtime FFI entry points exposed to GnuCOBOL's generated C code are protected by an explicit `std::panic::catch_unwind` safety boundary and return explicit `#[repr(C)] CobStatus` enums (e.g., `Success`, `SizeError`, `DomainError`, `ZeroDivide`, `StackOverflow`, `OutOfMemory`, `InternalError`) instead of allowing raw Rust panics to cross the ABI boundary[cite: 5]. 
   If any internal assertion failure or unexpected panic occurs, the safety wrapper intercepts it instantly, records `CobStatus::InternalError` into the frame-scoped error register, and returns the status code safely to the C caller without violating ABI contracts or triggering undefined behavior via cross-language stack unwinding[cite: 5]. 
   GnuCOBOL's generated C code directly inspects these status codes via the frame-scoped error register to branch natively into procedural `ON SIZE ERROR` blocks[cite: 5].

2. **Rounding-Aware Micro-Fractional Underflow Policy, Sign-Preserving Normalization, & Sign-Flip Safeguards:** 
   To prevent numerical discontinuities, micro-fractional underflows do not blindly truncate to absolute zero[cite: 5]. 
   Instead, sub-resolution values are routed through the active rounding mode engine (Section 6) and evaluated via a **sign-preserving underflow policy**: the normalization routine retains the existing workspace sign bit during sub-resolution conversion, outputting a signed zero (`-0` or `+0`) to preserve audit trail parity unless `--strict-underflow` forces strict positive normalization. 
   To eliminate negative sign-flip hazards during nearest-away-from-zero or directed rounding on sub-resolution negative quantities, all rounding adjustments enforce strict atomic algebraic evaluation that preserves the active workspace sign bit[cite: 5]. 
   Negative values rounding upward remain bounded at negative zero or trigger size-error interception rather than silently flipping to a positive unit[cite: 5]. 
   If strict auditing is mandated via compiler flags (`--strict-underflow`), rounding is bypassed and underflow conditions trigger size-error interception immediately[cite: 5].

3. **Size Error Flag & Conditional Branching:** 
   The runtime sets the active frame's execution error flag[cite: 5]. 
   If the statement includes an explicit `ON SIZE ERROR` phrase, control jumps to the designated error block, bypassing standard procedural flow[cite: 5]. 
   If omitted, target modification is suppressed and execution proceeds with warnings logged to telemetry[cite: 5].