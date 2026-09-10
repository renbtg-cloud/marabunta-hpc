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
  This variant is engineered for modern hybrid runtimes, asynchronous host environments, and specialized distributed node architectures—such as **Marabunta-mesh-extracted loops**—where lightweight computational tasks may yield control and migrate across worker threads[cite: 12]. 
  Under Option B, all transient computational state—including subnormal square root shift factor memoization caches and frame error registers—is anchored atomically within the stack-framed `ScratchpadArenaGuard` rather than thread-local storage (TLS)[cite: 12]. 
  *Strict Execution Invariant & Scope of Atomicity:* Arithmetic execution running within a `ScratchpadArenaGuard` is strictly synchronous and **cannot cross an async `.await` yield point mid-evaluation**[cite: 12]. The `AtomicU8` state machine governs frame and task ownership, publication, abandonment, and future-drop behavior; **it does not make the arithmetic arena concurrently mutable[cite: 12].** Because arithmetic cannot cross `.await`, the atomic ownership state is strictly about future abandonment/drop and surrounding task ownership rather than concurrent multi-threaded data mutation of the arena itself[cite: 12]. 
  To protect against uncooperative async executor runtimes that abandon futures mid-execution without polling them to completion, Option B implements an explicit **ownership and publication state machine** (`AtomicU8` status flags) paired with thread-local block pools in `ScratchpadArenaGuard`[cite: 12]:
  1. **Primary Path:** The future polls to completion, resetting the atomic ownership state cleanly under verified publication rules[cite: 12]. 
  2. **Drop-Time Fail-Safe:** If `Drop` executes while the atomic flag still indicates active ownership (meaning the future was abandoned unpolled), the guard fires an immediate fail-safe routine: it updates the atomic status to `TaskYieldAborted`, flushes structural telemetry metrics, and zero-fills the arena block slice using thread-local pools, completely eliminating mutex convoy contention during mass coroutine cancellations across Marabunta mesh nodes[cite: 12]. All state transitions under this mode are treated as formal protocols subject to concurrency auditing rather than assumed correct by default[cite: 12].

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
  *Widened Product Lifetime Invariant:* The widened 76-digit product is retained with its full mathematical value until the specified normalization and rounding boundary[cite: 12]. It must not be silently truncated or rounded to 38 digits merely because the ordinary `CobDecimal` descriptor field has its `precision` attribute set to 38[cite: 12]. This preservation is vital for `REMAINDER` and other compound operations that consume the widened product before terminal rounding[cite: 12]. Widened scratch precision must not be conflated with the ordinary 38-digit semantic precision tier[cite: 12]. 
  To prevent extreme scale alignment shifts near the boundaries from colliding with standard limits, the internal exponent tracking tier for this 76-digit expansion workspace is temporarily widened during raw multiplication and factor alignment[cite: 12]. 
  Once rounding concludes, the resulting value is checked against the normative $\pm 128$ boundary before final assignment[cite: 12].

* **Explicit Scope of Division Policies & 31-Decimal Semantic Precision:**
  To eliminate implementer ambiguity regarding division scaling limits, division operations are governed by precise, non-contradictory policies[cite: 12]:
  1. **Unassigned Intermediate Division Chains & Standalone Divisions:** For compound statements without an immediate assignment target or standalone division expressions, working scales are explicitly derived via checked arithmetic primitives subject to a bounded scale-derivation formula[cite: 12]:
     $$\text{working\_scale}_{\text{inter}} = \max\left(\text{working\_scale}_{\text{dividend}}, \text{working\_scale}_{\text{divisor}} + \max(0, 31 - \text{precision}_{\text{dividend}})\right)$$
     *Precise Semantic Definition of the 31-Decimal Limit & Scale Architecture:* Under the Cambrian compatibility profile, the "31 decimal places" factor governs the **intermediate working scale** (`working_scale`) derivation for unassigned division operations to prevent infinite recurring decimal loops; it is **strictly neither** a universal 31-significant-digit precision limit nor a blanket 31-digit constraint on every individual division node[cite: 12]. Significant precision tracking remains bounded by the engine's standard 38-digit semantic precision tier (`precision`), while the working scale expansion is capped at 31 fractional decimal places for the designated intermediate quotient scale before subsequent terminal rounding, scale clamping ($\pm 128$), or assignment truncation into the target field scale[cite: 12].
     *Compatibility Profile Status:* This 31-decimal-place limit applied to unassigned division chains is an explicit **Cambrian compatibility-profile and implementation constraint** designed to prevent infinite recurring decimal loops, rather than a universal COBOL standard requirement[cite: 12]. Its correctness and parity must be established and verified via differential testing against GnuCOBOL 3.2 reference outputs[cite: 12].
  2. **Non-Terminal Parenthetical AST Nodes:** Non-terminal child division nodes evaluated within deeply nested or left-heavy parenthetical trees (e.g., `COMPUTE X = (A / B) / (C / D)`) are explicitly not constrained by the 31-decimal policy; instead, they inherit the unified 38-digit intermediate precision tier across all nested sub-branches until final assignment or explicit rounding occurs[cite: 12].

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
  Square root extraction fast-paths utilize high-precision Newton-Raphson iteration governed by strict safety bounds: pre-iteration checks evaluate the argument $S$: if $S < 0$, execution immediately bypasses transcendental approximation loops and registers a domain error (`CobStatus::DomainError`); if $S = 0$, transcendental iteration is bypassed instantly and the square root returns an exact normalized zero (`+0`) with scale preservation[cite: 12], whereas for $S > 0$, execution proceeds with the Newton-Raphson iteration[cite: 12]. 
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
  To prevent accumulation drift and maintain regulatory auditing parity, the product term $(\text{Trunc}(\text{Quotient}) \times \text{Divisor})$ must execute using the engine's widened **76-digit multiplication workspace**, preventing precision overflow when multiplying two 38-digit numeric operands[cite: 12]. 
  *Widened Product Isolation & Non-Truncation Rule:* The 76-digit multiplication scratch result is **not** semantically rounded or truncated to 38 digits merely because the ordinary enclosing `CobDecimal.precision` field is set to 38[cite: 12]. The full widened mathematical product remains available and active until the specified normalization and rounding boundary[cite: 12]. Specifically, `REMAINDER` calculations consume this full widened product before any 38-digit terminal collapse occurs[cite: 12]. Widened scratch precision must not be conflated with the ordinary 38-digit semantic precision tier[cite: 12].
  *Quotient Hardening & Exact Quotient Determination Invariant:* To prevent finite approximation errors from landing on the wrong side when the mathematical quotient lies extremely close to an integer boundary, the implementation must establish the **exact mathematical integer quotient correctly before computing the product term**[cite: 12]. 
  *Implementation-Independent Correctness Contract:* To satisfy this invariant robustly across architectures without prescribing a single rigid implementation algorithm, implementations may establish exact quotient truncation using any formally validated method, such as[cite: 12]:
  - high-precision guard digits exceeding operand precision;
  - interval bounds and rigorous remainder sign checks;
  - directed rational comparisons against exact integer multiples;
  - exact rational decimal quotient/remainder computation loops;
  - or any other formally validated method ensuring exact mathematical truncation.
  The normative requirement is strictly functional[cite: 12]: the computed truncated quotient used in $\text{Trunc}(\text{Quotient}) \times \text{Divisor}$ **must equal the true mathematical integer truncation-toward-zero result**, even when finite numerical approximations lie arbitrarily close to an integer boundary[cite: 12]. 
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
  1. **Transient Internal Working Scale Exceedance ($\pm 256$ workspace range):** Triggered when internal intermediate scales breach the transient $\pm 256$ ceiling during inner-node multi-term expression evaluation. This constitutes a deterministic scale boundary violation and maps to `CobStatus::WorkspaceScaleOverflow`, **never** a COBOL business-field `SizeError` or an infrastructure fault like `CobStatus::OutOfMemory`[cite: 12].
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