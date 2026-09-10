# Annex E: Decimal Arithmetic Precision & Rounding Normals (`cambrian_cobol_annex-arithmetic.md`)

* **Component Owner:** Cambrian Radiation Core Engineering[cite: 12]
* **Target Compatibility:** GnuCOBOL 3.2, specified dialect/profile, compiler configuration (`cobc -C`), with explicit arithmetic mode (defaulting to standard intermediate arithmetic behavior verified via differential testing against GnuCOBOL 3.2 reference targets, with `-farithmetic-osvs` explicitly documented when toggled)[cite: 12]
* **Linkage Constraint:** Zero `-lcob` Dependency (Clean-Room Memory Mirroring)[cite: 12]
* **Status:** Draft / Active / Normative[cite: 12]

---

## 1. Overview, Governing Invariant, & Architectural Bound

The `cambrian_cobol` runtime core implements a **bounded high-precision decimal engine** (utilizing standard 38-digit semantic precision tiers and 76-digit widening for multiplication scratchpads) to support COBOL computational clauses (`COMP-3`, `PACKED-DECIMAL`, `DISPLAY` numeric items) and arithmetic verbs (`ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE`, `COMPUTE`, exponentiation)[cite: 12]. 

To ensure exact parity with GnuCOBOL 3.2 behavior without relying on external libraries like `-lcob`, all calculations operate on normalized byte arrays via the `CobDecimal` workspace structure[cite: 12]. 

### The Governing Invariant Rule
> **The scalar decimal implementation defines the normative arithmetic behavior. Every optimized implementation path (including SIMD kernels, arena scratchpads, and fast-paths) is permitted only when it demonstrably preserves the scalar implementation's observable result and the applicable COBOL/GnuCOBOL compatibility invariants. Differential and property testing serve as mandatory empirical evidence of equivalence, not the definition of correctness itself[cite: 12].**

To prevent CPU stalling, memory churn, domain panics, stack overflows, ABI unwinding hazards, and aliasing bugs during heavy batch workloads, arithmetic execution enforces strict performance mitigations, thread isolation guards, and correctness constraints[cite: 12].

---

## 2. Stack-Framed Scratchpad Arena, RAII Guards, Error Isolation, & Concurrency Build Variants

To eliminate heap allocation churn in inner loops and prevent workspace aliasing during nested complex expressions, compound `COMPUTE` statements, and recursive intrinsic function calls, flat thread-local pointers are prohibited in favor of stack-framed scoping[cite: 12].

* **Scoped RAII Arena Specifications & Lifetime Branding:**
  To ensure memory safety, prevent dangling or cross-thread arena contamination, and prohibit out-of-order drop sequences, the runtime utilizes a lightweight scoped RAII guard struct (`ScratchpadArenaGuard<'a>`) branded with invariant lifetime markers (`PhantomData<*mut &'a ()>`)[cite: 12]. 
  The guard's ownership and lifetime structure is designed to enforce Last-In, First-Out (LIFO) release; the implementation must reject constructions that permit frames to outlive or out-order their parents[cite: 12]. 
  Every entry into an arithmetic expression kernel executes a push operation via the guard, scoping temporary workspaces to the active frame and instantly reclaiming memory upon drop[cite: 12].

* **Frame-Scoped Error Registers & Shadow Stacks:**
  To eliminate cross-thread race conditions and state pollution of execution error flags across re-entrant FFI boundaries, global or unstructured error variables are strictly prohibited[cite: 12]. 
  The active error status register and exception codes are bound directly to the stack-framed `ScratchpadArenaGuard` struct, ensuring each execution frame maintains its own isolated error context[cite: 12]. 
  Furthermore, to prevent re-entrant callbacks, user-defined functions, or declarative exception procedures (`USE AFTER STANDARD ERROR`) from overwriting parent error statuses, the guard implements an internal error register push/pop shadowing stack that isolates inner execution frames and restores outer status codes upon exit[cite: 12].

* **Arena Stack Depth as a Runtime Resource Limit (Non-Semantic Boundary):**
  To guard stack-framed scratchpads against infinite recursive intrinsic calls or excessively nested compound statements, the runtime enforces a strict operational ceiling of 256 nested scratchpad frames within the context[cite: 12]. 
  *Crucial Semantic Distinction:* This depth cap is strictly an **infrastructure resource limit**, not an arithmetic property[cite: 12]. Compiler lowering passes that restructure deeply nested parenthetical ASTs into flatter sequences do not alter semantic correctness; any attempt exceeding this structural depth boundary immediately aborts execution via a zero-panic status code return (`CobStatus::StackOverflow`) and telemetry dumps rather than triggering a business-level COBOL `ON SIZE ERROR`[cite: 12].

* **Workspace Structure & Terminology Decoupling:**
  To eliminate ambiguity across scale representations, the engine enforces strict semantic separation of scale attributes[cite: 12]:
  * `stored_scale`: The explicit scale factor recorded in persistent layout descriptors or schema definitions[cite: 12].
  * `working_scale`: The active intermediate operational scale within the transient evaluation workspace[cite: 12].
  * `field_scale`: The target variable's receiving scale constraint[cite: 12].
  * `decimal_exponent`: The scientific exponent tracking factor used during transcendental scaling and normalizations[cite: 12].
  * `physical_bcd_alignment`: The byte-offset and nibble packing layout mapping[cite: 12].

    pub struct CobDecimal {
        pub working_scale: i32,         // Active intermediate operational scale (transient up to ±256, terminal clamped at ±128)
        pub precision: i32,             // Total significant digits tracked (up to 38 digits)
        pub value: SmallVec<[u8; 128]>, // Hybrid high-density inline/spillover normalized packed BCD byte buffer
    }

* **Two-Tier Scale Operational Clamp, Scale-Delta Guardrails, & AST Root Deferral Trade-Offs:**
  To prevent unbounded memory consumption or allocation denial-of-service from runaway scale inflation during pathological compound operations while avoiding premature failures on mathematically valid intermediate states, scale attributes utilize a robust **two-tier scale model**[cite: 12]. 
  During multi-term intermediate normalization and factor alignment phases, working scales are permitted a transient workspace ceiling of $\pm 256$[cite: 12]. 
  To prevent spurious intermediate overflow exceptions during complex multi-term compound expressions where an intermediate scale temporarily breaches $\pm 128$ before a subsequent division operation scales it back into valid bounds, the hard normative operational scale clamp ($\pm 128$) is **deferred until the entire parent compound expression tree finishes evaluation**, allowing working scales to expand up to $\pm 256$ across inner nodes[cite: 12]. 
  * Trade-Off & Caveat:* Deferring the clamp increases register pressure and memory footprint inside the scratchpad arena as working scales transiently expand[cite: 12]. The stack-framed arena and SmallVec buffers must allocate sufficient headroom (`SmallVec<[u8; 128]>` with pool spillover) to accommodate these wider transient representations without triggering unexpected heap spillover or block-pool exhaustion during heavy recursive computations[cite: 12]. 
  *Unambiguous Terminal Scale Boundary Rule:* Intermediate `working_scale` values that exceed $\pm 128$ during inner node evaluation do not trigger terminal scale-overflow conditions; the $\pm 128$ terminal check occurs **strictly and exclusively at the defined terminalization and storage boundary**[cite: 12]. 
  Any terminal working scale exceeding $\pm 128$ after rounding concludes triggers a normative workspace overflow exception (`CobStatus::WorkspaceScaleOverflow`), which remains distinct from target-PIC capacity failures and transient workspace ceilings[cite: 12]. Target-PIC field capacity is evaluated separately during receiving-field conversion[cite: 12]. 
  Furthermore, pre-flight scale delta verification checks operand scale disparities prior to buffer allocation, preventing unexpected heap spillage and cache-busting allocations during extreme scale alignment shifts, utilizing checked primitives (`i32::checked_add` and `i32::checked_sub`) to eliminate integer wrap-around vulnerabilities[cite: 12].

* **Pooled High-Density Small-Vector Allocation & Infrastructure Fault Isolation:**
  The internal value buffer utilizes an expanded small-vector architecture maintaining an inline stack-allocated buffer of 128 bytes (`SmallVec<[u8; 128]>`), ensuring that standard intermediate expressions and division series remain stack-allocated for optimal L1/L2 cache locality[cite: 12]. 
  To prevent raw stack footprint spikes across the 256-frame depth ceiling, temporary workspace slots are pooled within each stack-framed arena guard[cite: 12]. 
  Extreme multi-term scale expansions spill over into pre-allocated fixed-capacity block pools managed by the arena; if exhausted, the allocator safely returns `CobStatus::OutOfMemory`[cite: 12]. 
  Crucially, this infrastructure fault is cleanly decoupled from business-level arithmetic exceptions and never maps to standard COBOL `ON SIZE ERROR` handling, triggering an immediate fatal stack unwind and telemetry dump instead[cite: 12].

* **Schema-Aware Packed-Decimal Metadata Validation & Strict Unsigned Guardrails:**
  To handle legacy EBCDIC and ASCII migration datasets without breaking on platform variants, signed packed-decimal (`COMP-3`) sign nibbles accept an extended matrix (`C`, `A`, `F`, `E` as valid positive indicators; `D`, `B` as valid negative indicators)[cite: 12]. 
  Furthermore, layout descriptors inject schema metadata (`is_signed: bool`)[cite: 12]. 
  For fields explicitly declared as *unsigned*, the decoder explicitly distinguishes between decimal payload nibbles, which must contain decimal digits `0` through `9`, and the reserved final low-order sign/unsigned indicator nibble, which is not part of the decimal digit payload[cite: 12]. The decoder does not interpret the final indicator nibble as a decimal digit, preserving the fact that unsigned packed-decimal representation uses the final indicator nibble, and accepts or emits the GnuCOBOL unsigned indicator value `F` as required by the compatibility profile[cite: 12]. 
  *Strict Schema Separation:* Unsigned field decoders operating under explicit schema rules reject malformed decimal payload nibbles rather than silently coercing them, preventing legacy compatibility modes from accidentally admitting corrupt data streams that standard runtimes reject[cite: 12]. Differential property test suites must explicitly validate signed/unsigned variations, odd/even digit counts, valid/malformed sign nibbles, zeros, single-digit fields, and maximum-size fields against GnuCOBOL 3.2 reference outputs[cite: 12].

### Concurrency Deployment Variants: Build Option A vs. Option B
Depending on the host deployment topology and execution scheduling architecture, `cambrian_cobol` offers two distinct compile-time build artifacts[cite: 12]:

* **Option A (Synchronous Core-Pinned Build — Default & Recommended):**
  This is the default and most recommended build variant for running full, traditional COBOL programs compiled via GnuCOBOL 3.2 (`cobc -C`)[cite: 12]. 
  Under this mode, the `ScratchpadArenaGuard` is compiled with strict `non-Send` and `non-Sync` semantics, binding workspaces directly to the execution stack of the calling OS thread[cite: 12]. 
  Because standard GnuCOBOL generated C code is synchronous and blocking, Option A incurs zero synchronization overhead and maximizes raw instruction throughput on dedicated OS threads[cite: 12].

* **Option B (Asynchronous Coroutine-Safe Build with Audited Ownership & Publication Protocols):**
  This variant is engineered for modern hybrid runtimes, asynchronous host environments, and specialized distributed node architectures—such as **Marabunta-mesh-extracted loops**—where incomplete tasks may suspend and later be polled sequentially on a different worker thread[cite: 12]. 
  A **poll-return boundary** is any return from the current `Future::poll` invocation, whether `Poll::Pending` or `Poll::Ready`; one poll may contain zero, one, or multiple synchronous arithmetic intervals, and every stack-nested `ScratchpadArenaGuard` created during those intervals MUST complete teardown before either kind of poll return[cite: 12]. A live guard therefore cannot survive an actual suspending `.await`, cross a suspending executor handoff, or migrate between workers; an `.await` that resolves immediately without suspension does not itself create such a handoff[cite: 12]. 
  Thread-local block pools remain permitted only as worker-local allocation/recycling caches used during synchronous arithmetic on the current worker[cite: 12]. Before any poll-return boundary, every TLS-affine borrow, pointer/reference/slice, block loan, and applicable cleanup, return, release, disposal, or zeroization obligation for the completed arithmetic interval MUST have terminated under the existing allocator policy on the worker where that affinity is valid; this does not require every loan to be literally returned to a cache and does not impose universal zeroization on ordinary teardown[cite: 12]. No TLS-affine resource or cleanup obligation may be detached, migrated with the future, or deferred to future `Drop`.
  State that must remain meaningful after a suspending executor handoff belongs instead to migration-safe cross-handoff task-control state, stored in the future/task, an existing task-control object, or another existing Rust-safe shared ownership/reference form as appropriate; purely frame-local protocol bookkeeping may remain guard-local if it is dead by guard teardown[cite: 12]. The surviving portion of the existing `AtomicU8` ownership/publication/abandonment/cancellation/future-drop protocol belongs in this cross-handoff domain; any frame-active task-level state MUST transition to a non-frame-active state by guard teardown and before the poll-return boundary, and `TaskYieldAborted`, if retained, denotes task-level cancellation/abandonment rather than ownership of a guard that survived suspension. On successful `Poll::Ready` completion, task-level ownership/publication protocol state MUST be finalized or reset into its completed/non-active state under the existing verified publication rules; because guard-local and TLS-affine teardown is already complete before the poll-return boundary, no frame-active state, live guard, or TLS-affine resource remains at completion. 
  *Strict Execution Invariant & Scope of Atomicity:* Option B permits sequential worker migration only after a suspending `Poll::Pending` handoff: worker A completes all guard-local and TLS-affine teardown, the executor/runtime receives `Pending`, and a later poll may run on worker B with fresh guards and worker B's own TLS pool[cite: 12]. The same future MUST NOT be concurrently polled on multiple workers. Task state that actually transfers between workers must satisfy the applicable `Send` requirement; exclusively owned sequentially moved state does not require `Sync` solely because of migration, while shared cross-thread references or objects must satisfy the applicable `Sync` and aliasing requirements. Task migratability does not imply that a live `ScratchpadArenaGuard` must be `Send`, and this patch does not mandate changing the guard's existing `Send`/`Sync` trait status.
  `AtomicU8` coordinates the existing task-control protocol but does not make its enclosing object automatically `Send` or `Sync`, does not make unrelated reachable state migration-safe, does not publish the whole object, and does not make the arithmetic arena concurrently mutable[cite: 12]. The executor/runtime's existing task-transfer mechanism supplies the legal sequential ownership/synchronization handoff; the atomic protocol is not a substitute for that boundary.
  Cancellation signalling, if supported while `poll` is active, MUST use an already-existing thread-safe task-control/runtime mechanism and MUST NOT cross-thread access the live arithmetic guard or polling worker's TLS. Actual destruction of the same future MUST occur only when no `poll` invocation is active; an incomplete future may be dropped after `Poll::Pending` and before a later poll, and a completed future may be destroyed after `Poll::Ready`. If such an incomplete future is abandoned/dropped after a suspending `Poll::Pending` handoff and before a later poll, the task-level cross-handoff protocol MUST transition to `TaskYieldAborted` if that existing status remains in use and MUST flush structural telemetry for the abandonment according to the existing telemetry policy. Future `Drop` may act only on state legitimately reachable through the future/task's cross-handoff ownership/control representation and MUST NOT recover a previous poll's guard, access previous-worker TLS, release or zero a previous poll's TLS-affine arena loan, or depend on worker identity remaining stable. Correctness-critical guard-local and TLS-affine cleanup is therefore complete before the corresponding poll-return boundary and future-drop correctness is independent of previous-worker TLS[cite: 12]. All ownership, publication, frame-active/non-frame-active, cancellation/abandonment, completion, and future-drop transitions under Option B are formal protocol transitions subject to concurrency auditing and validation rather than informal assumptions.
---

## 3. Intermediate Calculation Precision & Scaling Rules

Arithmetic operations involving multiple data items dynamically derive intermediate precision and scaling to prevent premature truncation or overflow[cite: 12]:

* **Unified Precision Tier for Mixed Operations & Precision Distinction:**
  Ordinary intermediate arithmetic uses the 38-significant-digit tier, established as a Cambrian compatibility-profile constraint and implementation rule verified via differential testing against GnuCOBOL 3.2, ensuring that mixed compound expressions spanning multiple arithmetic verbs inherit consistent operational precision[cite: 12]. 
  Multiplication may use the explicitly defined 76-digit widened scratch workspace; however, this 76-digit product workspace represents widened multiplication scratch precision and must not be interpreted as changing the ordinary 38-digit semantic precision tier[cite: 12].

* **Addition & Subtraction (`ADD`, `SUBTRACT`):**
  The operational working scale of the intermediate workspace is automatically set to the maximum scale among all operands involved in the expression, computed via checked scale validation[cite: 12]:
  $$\text{working\_scale}_{\text{inter}} = \max(\text{working\_scale}_1, \text{working\_scale}_2, \dots, \text{working\_scale}_n)$$

* **Multiplication (`MULTIPLY`) & Widened Exponent Tracking:**
  Prior to computing product scales, a checked addition is executed using `i32::checked_add`[cite: 12]:
  $$\text{working\_scale}_{\text{inter}} = \text{working\_scale}_1 + \text{working\_scale}_2$$
  If the computed sum breaches the transient $\pm 256$ working scale ceiling, or if an overflow occurs during checked addition, multiplication is bypassed instantly, returning `CobStatus::WorkspaceScaleOverflow` rather than corrupting the workspace[cite: 12]. 
  To prevent the standard 38-digit intermediate precision ceiling from prematurely truncating raw product growth, raw multiplication of two 38-digit values dynamically expands workspace precision to a double-precision 76-digit scratchpad[cite: 12]. 
  *Widened Product Lifetime Invariant:* The widened 76-digit product is retained with its full mathematical value until the specified normalization and rounding boundary[cite: 12]. It must not be silently truncated or rounded to 38 digits merely because the ordinary `CobDecimal` descriptor field has its `precision` attribute set to 38[cite: 12]. This preservation is vital for `REMAINDER` when that widened multiplication path is sufficient, and for other compound operations that consume the widened product before terminal rounding; however, the separate `REMAINDER` exactness contract is not bounded to 76 digits and may require another finite, bounded exact method for otherwise-valid supported operands[cite: 12]. Widened scratch precision must not be conflated with the ordinary 38-digit semantic precision tier[cite: 12]. 
  To prevent extreme scale alignment shifts near the boundaries from colliding with standard limits, the internal exponent tracking tier for this 76-digit expansion workspace is temporarily widened during raw multiplication and factor alignment[cite: 12]. 
  Once rounding concludes, the resulting value is checked against the normative $\pm 128$ boundary before final assignment[cite: 12].

* **Explicit Scope of Division Policies & 31-Decimal Semantic Precision:**
  To eliminate implementer ambiguity regarding division scaling limits, division operations are governed by precise, non-contradictory policies[cite: 12]:
  1. **Unassigned Intermediate Division Chains & Standalone Divisions:** For compound statements without an immediate assignment target or standalone division expressions, working scales are explicitly derived via checked arithmetic primitives subject to a bounded scale-derivation formula[cite: 12]:
     $$\text{working\_scale}_{\text{inter}} = \max\left(\text{working\_scale}_{\text{dividend}}, \text{working\_scale}_{\text{divisor}} + \max(0, 31 - \text{precision}_{\text{dividend}})\right)$$
     After successful operand ingestion and validation, every integer addition or subtraction used in this working-scale derivation MUST use checked integer arithmetic. If checked integer arithmetic overflows, or if the successfully derived intermediate `working_scale` lies outside the transient $\pm 256$ workspace range, division MUST be bypassed before workspace allocation or arithmetic execution and return `CobStatus::WorkspaceScaleOverflow`. The derived scale MUST NOT be wrapped, saturated, or clamped to $\pm 256$, and this deterministic scale-derivation failure MUST NOT be mapped to `CobStatus::SizeError`, `CobStatus::OutOfMemory`, or `CobStatus::ConvergenceExhausted`; actual allocation or pool exhaustion remains `CobStatus::OutOfMemory`, receiving-field / target-PIC capacity overflow remains `CobStatus::SizeError`, and the terminal $\pm 128$ check remains at the already-defined terminalization and storage boundary.
     *Precise Semantic Definition of the 31-Based Scale-Derivation Factor & Scale Architecture:* Under the Cambrian compatibility profile, the constant `31` governs the precision-derived fractional-scale extension term `max(0, 31 - precision_dividend)` within the **intermediate working scale** (`working_scale`) derivation for unassigned division operations; it is **strictly neither** a universal 31-significant-digit precision limit nor a blanket 31-digit constraint on every individual division node[cite: 12]. Significant precision tracking remains bounded by the engine's standard 38-digit semantic precision tier (`precision`), while the final `working_scale` is the result of the complete normative formula and may legitimately exceed 31; it MUST NOT be clamped merely because it exceeds 31 and remains subject instead to the separately defined transient $\pm 256$ workspace boundary before subsequent terminal rounding, terminal scale clamping ($\pm 128$), or assignment truncation into the target field scale[cite: 12].
     *Compatibility Profile Status:* This 31-based scale-derivation factor applied to unassigned division chains is an explicit **Cambrian compatibility-profile and implementation constraint** designed to prevent infinite recurring decimal loops, rather than a universal COBOL standard requirement or an absolute quotient-scale ceiling[cite: 12]. Its correctness and parity must be established and verified via differential testing against GnuCOBOL 3.2 reference outputs[cite: 12].
  2. **Non-Terminal Parenthetical AST Nodes:** Non-terminal child division nodes evaluated within deeply nested or left-heavy parenthetical trees (e.g., `COMPUTE X = (A / B) / (C / D)`) are explicitly not constrained by the designated 31-based scale-derivation policy; instead, they inherit the unified 38-digit intermediate precision tier across all nested sub-branches until final assignment or explicit rounding occurs[cite: 12].

---

## 4. Exponentiation Performance Architecture & Correctness Guards (`**` Operator)

To prevent linear time traps ($O(N)$), transcendental performance degradation, and domain corruption during power calculations, exponentiation routines adhere to strict execution rules[cite: 12]. All exponent classification and dispatch decisions are governed by a canonical predicate: `ExponentClass := Integral | Fractional`, determined strictly from the true mathematical value ($value \pmod 1 == 0 \implies \text{Integral}$)[cite: 12].

* **Integer Exponents & Magnitude Pre-Checks (Status Decoupling):**
  All integer powers are evaluated via square-and-multiply binary exponentiation algorithms ($O(\log N)$)[cite: 12]. 
  Prior to loop entry, magnitude validation checks the exponent value using a mathematically sufficient determination of whether the required intermediate or result representation exceeds permitted representation or workspace bounds[cite: 12]. *Constraint:* Exponent magnitude alone must never be used as a proxy for overflow; an early workspace-overflow decision may be made only when the implementation can establish that the required computation necessarily exceeds permitted representation and workspace bounds[cite: 12]. 
  *Strict Status Decoupling:* If an integer exponent or exponentiation operation causes representation limits to breach, the error status is routed strictly based on the failure boundary:
  - **Scale / Representation-Boundary Failure (`CobStatus::WorkspaceScaleOverflow`):** Used for transient working scale exceeding $\pm 256$, terminal working scale exceeding $\pm 128$, or checked integer overflow while deriving required scale metadata[cite: 12].
  - **Actual Memory / Allocation Resource Failure (`CobStatus::OutOfMemory`):** Used for failure to obtain required arena storage, fixed-capacity block-pool exhaustion, or allocation/resource exhaustion required by exponentiation[cite: 12].
  - **Frame-Depth Resource Failure (`CobStatus::StackOverflow`):** Used for scratchpad frame-depth exhaustion.
  - **Receiving-Field / Business Capacity Failure (`CobStatus::SizeError`):** Used for target-field / target-PIC business-capacity overflow (triggering COBOL `ON SIZE ERROR`)[cite: 12].
  Actual allocation or pool exhaustion during exponentiation is not `WorkspaceScaleOverflow` merely because the failure occurs while computing an exponentiation intermediate. Internal infrastructure or workspace limits are never permitted to masquerade as COBOL receiving-field size errors and MUST NOT be reported as `CobStatus::SizeError`[cite: 12].

* **Explicit Odd/Even Sign Parity Rules for Negative Bases:**
  When raising a negative base ($\text{sign} < 0$) to an exponent classified as `ExponentClass::Integral`, algebraic sign bits are explicitly forced based on the canonical integer parity of the evaluated value[cite: 12]:
  * **Even Integer Exponent:** Workspace sign bit is forced positive ($\text{sign} > 0$)[cite: 12].
  * **Odd Integer Exponent:** Workspace sign bit is forced negative ($\text{sign} < 0$)[cite: 12].

* **Financial Fast-Paths & Bounded Square Root Extraction:**
  Common financial and structural powers ($x^2$, $x^3$) bypass generalized multiplication routines, hitting optimized inline squaring routines directly within the scratchpad arena[cite: 12]. 
  Square root extraction fast-paths utilize high-precision Newton-Raphson iteration governed by strict safety bounds: pre-iteration checks evaluate the argument $S$: if $S < 0$, execution immediately bypasses transcendental approximation loops and registers a domain error (`CobStatus::DomainError`)[cite: 18]; if $S = 0$, transcendental iteration is bypassed instantly and the square root returns an exact normalized zero (`+0`) with scale preservation[cite: 12], whereas for $S > 0$, execution proceeds with the Newton-Raphson iteration[cite: 12]. 
  To eliminate calculation drift from static subnormal floor clamping, subnormal arguments are evaluated with a rapid convergence pre-check; if within the standard window ($\epsilon_m \le S \le 10^{38}$), dynamic scaling is bypassed entirely[cite: 12]. 
  *Definition of Decimal Representation Threshold ($\epsilon_m$):* Here, $\epsilon_m = 10^{-38}$ represents the strict **Cambrian decimal representation/profile threshold** defining the lower baseline precision floor of the engine's supported decimal range, rather than an IEEE-754 binary floating-point subnormal number[cite: 12]. It is an explicitly decimal parameter and must not be interpreted or handled as a binary floating-point subnormal quantity[cite: 12]. 
  For clustered subnormal inputs ($S < \epsilon_m$), the computed shift factor $k$ is memoized directly within the active stack-framed `ScratchpadArenaGuard` (guaranteeing task-migration safety under Build Option B) and processed via an **even-power-of-10 dynamic scaling wrapper** ($S' = S \times 10^{2k}$), pre-checked against operational scale bounds[cite: 12]. 
  The Newton-Raphson iteration executes on $S'$ with standard precision, and the resulting root is scaled back down ($result = \text{root}' / 10^k$)[cite: 12]. 
  *Iteration Limit & Convergence Resource Handling:* The iteration loop is bound by a resource ceiling of 32 cycles[cite: 12]. **Maximum iteration counts serve strictly as resource exhaustion limits, not arithmetic correctness criteria[cite: 12].** If an internal Newton iteration encounters a zero denominator or fails to converge within 32 cycles due to pathological plateaus without establishing the required error bound, execution yields a defined internal convergence/resource failure (`CobStatus::ConvergenceExhausted`) rather than falsely flagging a numeric size error, domain error, or an operand-level COBOL divide-by-zero[cite: 12].

* **Zero Base with Non-Positive Exponents ($0^0$ or $0^{-n}$):**
  If the base evaluates to absolute zero and the exponent is zero, the engine returns an exact integer `1` under the Cambrian compatibility profile (established as a profile-specific implementation rule matching common COBOL conventions, subject to dialect verification)[cite: 12]. 
  If the exponent is negative ($< 0$), transcendental processing is bypassed, registering a division-by-zero exception code (`CobStatus::ZeroDivide`) and routing execution according to the exception classification table (Section 7)[cite: 12].

* **Domain Trap Isolation via Canonical Exponent Classification:**
  When evaluating $base^{exponent}$, the engine queries `ExponentClass` to determine whether an exponent is fractional ($value \pmod 1 \neq 0$), never inspecting representation metadata scales[cite: 12]. Thus, integer values stored with non-zero representation scales (e.g., `12.0 ** 2.0`) evaluate as `ExponentClass::Integral` and pass cleanly[cite: 12]. 
  If the base is negative ($\text{sign} < 0$) and `ExponentClass` evaluates to `Fractional`, transcendental series approximations are strictly prohibited[cite: 12]. 
  The runtime core immediately bypasses expansion, registers exception code `EC-ARITHMETIC-DOMAIN` (`0x4201`, a Cambrian compatibility-profile exception identifier), and routes control according to the exception classification table (Section 7)[cite: 12].

* **Exact Rational Power-of-10 Pre-Check, Padé Acceleration with Hermite Smoothstep, & Normative Error-Bound Contract:**
  Prior to executing Cody-Waite argument reduction or Taylor series logarithmic approximation for $a^b$, the engine evaluates a strict rational pre-check[cite: 12]. 
  If base $a$ is a clean canonical power of 10 ($\exists k \in \mathbb{Z}, a = 10^k$), transcendental series evaluation is bypassed entirely[cite: 12]. 
  The logarithm $\log_{10}(a)$ returns exact integer $k$, eliminating floating-point series drift on exact decimal scaling factors[cite: 12]. 
  For non-trivial fractional powers, evaluation proceeds via base-10 logarithmic and exponential series approximations via $a^b = 10^{b \cdot \log_{10}(a)}$, scaled to 31 decimal places[cite: 12]. 
  For bases lying far outside the local unity window, the engine applies Cody-Waite style argument reduction by decomposing the base into mantissa and exponent ($a = m \times 10^e$), computing $\log_{10}(a) = \log_{10}(m) + e$ with $m$ strictly normalized into the convergence window before series evaluation, eliminating irrational constant multiplication chatter entirely[cite: 12]. 
  For normalized base inputs falling within the critical window near unity ($1.0 - \epsilon_r < \alpha < 1.0 + \epsilon_r$), the engine avoids slow linear convergence and catastrophic cancellation by deploying a **rational Padé approximant fallback** ($\frac{P(x)}{Q(x)}$) accelerated and smoothed by a **Hermite cubic smoothstep interpolation function** ($s(x)$) across the boundary band[cite: 12]:
  $$s(x) = 3\left(\frac{x - (1.0 - \epsilon_r)}{2\epsilon_r}\right)^2 - 2\left(\frac{x - (1.0 - \epsilon_r)}{2\epsilon_r}\right)^3$$
  This guarantees strict mathematical $\mathcal{C}^1$ continuity at the boundaries[cite: 12]. 
  *Normative Transcendental Error-Bound Contract:* Taylor/Maclaurin series convergence loops are bound by a resource ceiling of 64 cycles[cite: 12]. Every normative transcendental approximation used by the implementation (including logarithm, exponential, Padé fallback, and smoothstep/blended regions) must be governed by an **approximation-specific, formally documented error-bound contract covering the complete returned approximation (including interpolation and blending where applicable)**[cite: 12]. The contract must explicitly define[cite: 12]:
  1. The explicit error bound formula $E_{bound}(x)$ derived from rigorous series remainder estimation theorems and interval analysis;
  2. The mathematical theorem and interval domain justifying $E_{bound}(x)$;
  3. How the runtime computes $E_{bound}(x)$ dynamically using available decimal arithmetic;
  4. Complete coverage proving that component approximation error plus interpolation/blending error satisfies the total condition $E_{bound}(x) \le 10^{-31}$. 
  Runtime acceptance and termination require that this computable upper bound satisfies $E_{bound}(x) \le 10^{-31}$[cite: 12]. Arbitrary iteration-count heuristics, unverified convergence tests, or float-based termination checks are strictly prohibited[cite: 12]. Approximation-specific bound formulas serve as normative contracts that must be formally documented and independently validated before an approximation is considered conforming[cite: 12]. If detailed formulas belong in a future approximation-specific specification, those normative bounds and documentation requirements must exist and be validated[cite: 12]. If an iteration sequence flattens out or exhausts its 64-cycle ceiling without establishing the required computable error bound $E_{bound}(x) \le 10^{-31}$, execution yields `CobStatus::ConvergenceExhausted`[cite: 12]. 
  *Strict Error Distinction:* Algorithmic inability to establish the required error bound within resource ceilings yields `CobStatus::ConvergenceExhausted`, which remains strictly distinct from a `DomainError` (reserved exclusively for mathematically invalid inputs such as negative bases raised to fractional powers). Conflating algorithmic convergence exhaustion with mathematical domain invalidity is strictly prohibited[cite: 12].

---

## 5. Division-by-Zero & Remainder Sign-Correction Engine

* **Division-by-Zero Interception:**
  Prior to executing division kernels, the divisor's normalized value is checked against absolute zero[cite: 12]. 
  If zero, execution registers a division-by-zero exception code (`CobStatus::ZeroDivide`) and routes control according to the exception classification table (Section 7)[cite: 12].

* **Normative Remainder Sign-Correction, Quotient Hardening, & Exact-Zero Sign Rule:**
  For all `DIVIDE` statements utilizing the `REMAINDER` clause, unsigned or absolute-value modulus operations are prohibited[cite: 12]. 
  The remainder is computed explicitly via[cite: 12]:
  $$\text{Remainder} = \text{Dividend} - \left( \text{Trunc}(\text{Quotient}) \times \text{Divisor} \right)$$
  To prevent accumulation drift and maintain regulatory auditing parity, ordinary multiplication of two ordinary 38-significant-digit operands continues to use the engine's widened **76-digit multiplication workspace**; `REMAINDER` MAY use that widened multiplication path when the exact quotient/product representation required by the chosen algorithm fits it without loss[cite: 12]. 
  *Widened Product Isolation & Non-Truncation Rule:* The 76-digit multiplication scratch result is **not** semantically rounded or truncated to 38 digits merely because the ordinary enclosing `CobDecimal.precision` field is set to 38[cite: 12]. The full widened mathematical product remains available and active until the specified normalization and rounding boundary whenever that multiplication path is used[cite: 12]. `REMAINDER` exactness is not semantically capped by 76 digits: if the exact truncated quotient or an explicitly materialized product would exceed that scratch tier, the implementation MUST use another formally validated exact method—such as exact aligned-integer division/remainder, exact divmod over bounded scale-expanded integer representations, chunked exact quotient/remainder computation, directed comparisons with exact subtraction against integer multiples, or another exact method that never depends on an inexact quotient—without rounding or truncating the quotient/product merely to force it into 76 digits[cite: 12]. Any such wider temporary representation remains finite and bounded by the engine's existing supported operand precision, scale, workspace, and resource limits; actual failure to obtain required bounded storage remains `CobStatus::OutOfMemory`, while a valid computation MUST NOT return `CobStatus::WorkspaceScaleOverflow` or `CobStatus::SizeError` merely because an internal exact quotient/product representation exceeds 76 digits, provided the existing scale boundaries and receiving-field rules are otherwise satisfied. Widened scratch precision must not be conflated with the ordinary 38-digit semantic precision tier[cite: 12].
  *Quotient Hardening & Exact Quotient Determination Invariant:* Let `q = Trunc(Dividend / Divisor)` denote the exact mathematical truncation-toward-zero quotient. To prevent finite approximation errors from landing on the wrong side when the mathematical quotient lies extremely close to an integer boundary, the implementation MUST determine the remainder so that its effective mathematical quotient is exactly `q`; this does not require physically materializing the complete integer `q` or the product `q × Divisor` when another formally validated exact method produces the identical mathematical remainder[cite: 12]. 
  *Implementation-Independent Correctness Contract:* To satisfy this invariant robustly across architectures without prescribing a single rigid implementation algorithm, implementations may establish the exact mathematical `q`/remainder relationship using any formally validated method, whether or not the complete quotient or product is materialized, such as[cite: 12]:
  - high-precision guard digits exceeding operand precision;
  - interval bounds and rigorous remainder sign checks;
  - directed rational comparisons against exact integer multiples;
  - exact rational decimal quotient/remainder computation loops;
  - or any other formally validated method ensuring exact mathematical truncation.
  The normative requirement is strictly functional[cite: 12]: the remainder MUST be mathematically identical to $\text{Dividend} - (q \times \text{Divisor})$, where `q` is the true mathematical integer truncation-toward-zero result, even when finite numerical approximations lie arbitrarily close to an integer boundary[cite: 12]. If the chosen implementation materializes `q`, that value MUST equal the exact mathematical truncation-toward-zero quotient; if it materializes `q × Divisor`, that product MUST be exact for the chosen path and MUST NOT be rounded or truncated merely to fit the ordinary 76-digit scratch tier. 
  `Trunc` is explicitly defined as truncation toward zero (discarding fractional digits without flooring toward negative infinity) to ensure exact execution parity with GnuCOBOL 3.2 integer division routines regardless of operand signs[cite: 12]. 
  *Exact-Zero Remainder Sign Rule & Audit Semantics:* The exact-zero remainder sign rule applies explicitly and exclusively when the **exact mathematical remainder**—as evaluated directly from $\text{Dividend} - \left( \text{Trunc}(\text{Quotient}) \times \text{Divisor} \right)$ prior to any subsequent rounding, target-scale reduction, receiving-field conversion, or underflow handling—is equal to zero[cite: 12]. When this exact mathematical remainder is zero, the internal audit representation **retains the algebraic sign of the original Dividend (`+0` or `-0`)** to ensure audit trail fidelity[cite: 12]. Conversely, **COBOL-observable output semantics normalize the remainder to canonical positive zero (`+0`)** unless explicit signed picture formatting rules govern the receiving field[cite: 12]. If an exact mathematical remainder is non-zero but subsequently evaluates to zero as a result of rounding, scale reduction, receiving-field conversion, or underflow handling, that result is governed strictly by standard rounding and underflow semantics and MUST NOT be classified or rehandled as an exact-zero `REMAINDER`[cite: 12]. 
  The sign bit of non-zero remainder results is forced via bitwise assignment to match the algebraic sign bit of the original Dividend[cite: 12].

---

## 6. Rounding Mode Norms & SIMD Optimization Boundary

When results are moved from high-precision `CobDecimal` intermediate workspaces into target `CobField` variables with smaller scales or tighter digit limits, the runtime core applies explicit rounding modes based on compiler directives or standard COBOL defaults[cite: 12]. All rounding tie-breaks and threshold evaluations must be computed via **direct decimal digit integer comparisons**, completely eliminating binary floating-point conversion drift[cite: 12].

* **Truncation (Standard Default / `REMAINDER` omission):**
  Discards excess digits beyond the target scale without adjustment[cite: 12].

* **Nearest-Away-From-Zero (`ROUNDED`):**
  Rounds the least significant retained digit upward if the discarded fraction is greater than or equal to 0.5[cite: 12].

* **Nearest-Even (`ROUNDED MODE IS NEAREST-EVEN`):**
  Rounds toward the nearest even digit based on the discarded fraction[cite: 12]:
  * If discarded fraction $< 0.5$: round down (truncate excess)[cite: 12].
  * If discarded fraction $> 0.5$: round up[cite: 12].
  * If discarded fraction exactly $= 0.5$: round to the nearest even retained digit (inspecting the parity of the least significant retained integer/significand digit)[cite: 12]. 
  To eliminate branch-heavy conditional overhead and instruction cache bloat during heavy financial batch runs, nearest-even tie-breaking may execute via dedicated vector intrinsics (AVX2 for x86_64, NEON for aarch64) guarded by runtime CPU feature detection (`is_x86_feature_detected!("avx2")`)[cite: 12]. 
  Because `SmallVec<[u8; 128]>` and general COBOL workspace buffers maintain a **native packed BCD layout** (two decimal digits per byte) without strict 32-byte or 16-byte memory alignments, unaligned load primitives guarantee safe byte scanning across arbitrary offsets without triggering hardware segmentation faults (`SIGSEGV` or `SIGBUS`), falling back cleanly to unrolled scalar blocks on legacy hardware without triggering illegal instruction faults (`SIGILL`)[cite: 12]. 
  **SIMD Dispatch & Hardware Table Lookup Acceleration:** Vector routines may employ length-gated dispatch; the specific byte-length crossover threshold is implementation-defined and benchmarked per target architecture, while the correctness boundary remains strictly normative[cite: 12]. 
  Furthermore, to prevent AVX2 YMM registers (strictly 32 bytes / 256 bits) from overflowing when unpacking packed BCD digits, AVX2 vector chunk sizes are strictly restricted to **16 bytes of packed BCD per iteration** (expanding to 32 bytes of unpacked lanes)[cite: 12]. 
  *Normative Oracle Validation:* Per the Governing Invariant, **the scalar decimal implementation serves as the absolute correctness oracle[cite: 12].** All SIMD optimization paths must pass exhaustive differential property testing against the scalar reference across adversarial boundaries (e.g., carry-chains of 9s, exact halves, subnormal floors) before being certified for production deployment[cite: 12].

* **Toward-Zero (`ROUNDED MODE IS TOWARD-ZERO`):**
  Truncates directly regardless of sign[cite: 12].

* **Up / Down Directed Rounding:**
  Enforces ceiling or floor adjustments based on algebraic sign constraints[cite: 12].

---

## 7. Pipeline Hypothesis, Overflow, Underflow, FFI Status Code Interception, & Exception Compatibility Mapping

* **Execution Pipeline Hypothesis:**
  Arithmetic execution follows a strictly ordered compatibility pipeline[cite: 12]:
  $$\text{Operand Ingestion} \longrightarrow \text{Mathematical Operation} \longrightarrow \text{Rounding Point Application} \longrightarrow \text{Receiving-Field Mapping} \longrightarrow \text{Size-Error Evaluation}$$
  This pipeline structure acts as a compatibility hypothesis, validated continuously via differential test suites against GnuCOBOL 3.2 reference outputs[cite: 12].

* **Zero-Panic FFI Status Return & Unwind Protection (`#[repr(C)]`):**
  All runtime FFI entry points exposed to GnuCOBOL's generated C code are protected by an explicit `std::panic::catch_unwind` safety boundary and return explicit `#[repr(C)] CobStatus` enums (e.g., `Success`, `SizeError`, `DomainError`, `ZeroDivide`, `StackOverflow`, `OutOfMemory`, `ConvergenceExhausted`, `InternalError`) instead of allowing raw Rust panics to cross the ABI boundary[cite: 12]. 
  If any internal assertion failure or unexpected panic occurs, the safety wrapper intercepts it instantly, records `CobStatus::InternalError` into the frame-scoped error register, and returns the status code safely to the C caller without violating ABI contracts or triggering undefined behavior via cross-language stack unwinding[cite: 12]. 
  GnuCOBOL's generated C code directly inspects these status codes via the frame-scoped error register to branch natively into procedural error blocks[cite: 12].

* **Explicit Separation of Workspace Scale Boundaries, Transient Workspaces, & Target-PIC Capacity:**
  The normative specification enforces strict mutual exclusivity and operational decoupling across failure classes[cite: 12]:
  1. **Transient Internal Working Scale Exceedance ($\pm 256$ workspace range):** Triggered when internal intermediate scales breach the transient $\pm 256$ ceiling during inner-node multi-term expression evaluation. After successful operand ingestion and validation, checked integer overflow while deriving required arithmetic `working_scale` metadata is likewise a deterministic workspace/representation-boundary failure and maps to `CobStatus::WorkspaceScaleOverflow`, as does a successfully derived transient working scale outside $\pm 256$; these failures are detected before allocation and MUST NOT be wrapped, saturated, clamped, or reclassified as `CobStatus::SizeError` or `CobStatus::OutOfMemory`. This constitutes a deterministic scale boundary violation and maps to `CobStatus::WorkspaceScaleOverflow`, **never** a COBOL business-field `SizeError` or an infrastructure fault like `CobStatus::OutOfMemory`[cite: 12].
  2. **Terminal Workspace Scale Overflow ($\pm 128$ terminal limit):** Triggered when the final terminal working scale exceeds the $\pm 128$ boundary after compound expression evaluation concludes. This is an engine representation fault (`CobStatus::WorkspaceScaleOverflow`)[cite: 12].
  3. **Target-PIC Capacity Overflow:** Triggered when a successfully computed workspace result exceeds the digit or scale capacity of the receiving field's `PICTURE` clause during receiving-field conversion (`CobStatus::SizeError`)[cite: 12]. 
  These three failure classes are mutually exclusive and never conflated[cite: 12].

* **Explicit Exception Compatibility Mapping Table (Validated via Differential Harness):**
  To ensure precise alignment with GnuCOBOL runtime behavior and prevent ambiguity between size failures and domain violations, arithmetic anomalies map to specific status codes and handling paths as follows (with native GnuCOBOL conditions versus Cambrian-defined compatibility profile identifiers explicitly distinguished[cite: 12]):

| Condition / Error Class | GnuCOBOL / Runtime Status | Default Handling Path |
| :--- | :--- | :--- |
| **Divide by Zero** | `CobStatus::ZeroDivide` (`EC-SIZE-ZERO-DIVIDE` [Native GnuCOBOL])[cite: 12] | Triggers `ON SIZE ERROR` if declared; otherwise raises runtime exception signal[cite: 12]. |
| **Invalid Negative Fractional Exponent** | `CobStatus::DomainError` (`EC-ARITHMETIC-DOMAIN` [Cambrian-defined])[cite: 12] | Raises runtime exception signal per COBOL dialect rules; bypasses `ON SIZE ERROR`[cite: 12]. |
| **Arithmetic Overflow (Target PIC)** | `CobStatus::SizeError` (`EC-SIZE-OVERFLOW` [Native GnuCOBOL])[cite: 12] | Triggers `ON SIZE ERROR` if declared; suppresses target update if omitted[cite: 12]. |
| **Workspace Scale Overflow (transient ±256 / terminal ±128 limits)** | `CobStatus::WorkspaceScaleOverflow` [Cambrian-defined][cite: 12] | Triggers structural error interlock (engine boundary violation)[cite: 12]. |
| **Conversion Overflow** | `CobStatus::SizeError` (`EC-SIZE-CONVERSION` [Native GnuCOBOL])[cite: 12] | Triggers `ON SIZE ERROR`[cite: 12]. |
| **Underflow (Standard Mode)** | Retains internal signed zero (`+0` / `-0`)[cite: 12] | Standard procedural continuation with telemetry logging[cite: 12]. |
| **Underflow (`--strict-underflow` extension)** | `CobStatus::SizeError`[cite: 12] | Bypasses rounding and triggers immediate size-error interception (Cambrian strict audit extension)[cite: 12]. |
| **Internal Convergence Exhaustion** | `CobStatus::ConvergenceExhausted` (`EC-CONVERGENCE-EXHAUSTED` [Cambrian-defined])[cite: 12] | Yields convergence resource fault; **bypasses `ON SIZE ERROR`** (treated as an infrastructure/resource limit under the Cambrian compatibility profile rather than a business arithmetic size error)[cite: 12]. |
| **Internal Resource / Workspace Exhaustion** | `CobStatus::OutOfMemory` / `StackOverflow`[cite: 12] | Unwinds execution; bypasses `ON SIZE ERROR` (infrastructure fault)[cite: 12]. |

* **Separation of Internal Signed-Zero Preservation from Observable COBOL Semantics:**
  To maintain rigorous internal audit trails, the runtime engine preserves signed zeros (`+0` / `-0`) internally during sub-resolution underflows and exact-zero remainder calculations[cite: 12]. However, **observable COBOL field representation semantics strip internal sign distinction during final `DISPLAY` or formatted output assignment** unless explicitly governed by signed picture clause formatting rules, ensuring full compatibility with ANSI/GnuCOBOL expectations while retaining audit fidelity in raw logs[cite: 12]. When the explicit compiler extension flag `--strict-underflow` is enabled (a deliberate Cambrian compilation profile extension rather than standard COBOL semantics), rounding is bypassed, and underflow conditions trigger size-error interception immediately, changing runtime arithmetic semantics intentionally for high-assurance audit environments[cite: 12]. 
  *Note on GnuCOBOL Compatibility Claims:* Compatibility claims regarding GnuCOBOL 3.2 execution parity and dialect behavior are verified via empirical differential testing against GnuCOBOL 3.2 reference targets under specified compiler configurations (`cobc -C`), with explicit flags such as `-farithmetic-osvs` documented when toggled[cite: 12].