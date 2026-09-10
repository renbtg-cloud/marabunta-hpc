# FIX REQUEST — TRANSCENDENTAL ERROR-BOUND EVALUATION MUST REMAIN A CONSERVATIVE UPPER BOUND

Apply exactly the change described below to the current `cambrian_cobol_annex-arithmetic.md`.

Do not redesign the transcendental algorithms.

Do not change the accepted absolute acceptance target:

    E_bound(x) <= 10^-31

Do not change the 64-cycle resource ceiling.

Do not change the approximation families, Padé fallback, smoothstep/blending architecture, `ConvergenceExhausted`, `DomainError`, or any arithmetic status mapping.

This patch fixes one correctness hole inside the existing normative transcendental error-bound contract:

> **The specification requires a rigorous mathematical upper bound `E_bound(x)`, but does not explicitly require the finite decimal computation of that bound to preserve the upper-bound property under rounding.**

Everything unrelated to that defect is frozen.

---

# DEFECT

The current contract correctly requires:

- an explicit approximation-specific error-bound formula `E_bound(x)`;
- a theorem and interval domain justifying it;
- runtime dynamic computation using decimal arithmetic;
- total coverage of approximation plus interpolation/blending error;
- acceptance only when `E_bound(x) <= 10^-31`.

However, the contract currently says only:

> “How the runtime computes `E_bound(x)` dynamically using available decimal arithmetic”

It does not state how finite-precision rounding during that computation is handled.

That omission is dangerous.

Suppose the mathematically exact bound is slightly above the acceptance threshold:

    true E_bound = 1.000...1 × 10^-31

If an intermediate or final bound calculation is rounded downward to:

    computed E_bound = 1.000...0 × 10^-31

or otherwise underestimates the exact mathematical bound, the implementation could accept an approximation that has NOT established the required guarantee.

A quantity used as a normative **upper bound** must never be allowed to become smaller than the mathematically justified bound because of finite arithmetic.

---

# REQUIRED SEMANTIC RESULT

Modify only the local normative transcendental error-bound contract so that all of the following are explicit.

1. The runtime value used as `E_bound(x)` for conformance/termination decisions MUST be a **certified conservative upper enclosure** of the mathematically justified total approximation error.

2. Finite decimal evaluation of the bound MUST NOT underestimate that mathematical upper bound.

3. Every arithmetic operation used to compute the certified runtime bound MUST therefore use one of the following correctness-equivalent approaches:
   - exact arithmetic for that bound operation; or
   - directed/outward rounding that preserves an upper enclosure; or
   - interval arithmetic whose returned upper endpoint is itself conservatively rounded outward; or
   - another formally validated method proving that the runtime value is greater than or equal to the true mathematical bound.

4. Ordinary round-to-nearest evaluation is NOT sufficient by itself unless the implementation separately proves that the resulting runtime value still cannot underestimate the exact bound.

5. The comparison:

       E_bound(x) <= 10^-31

   may succeed only using the certified conservative runtime upper bound.

6. If finite bound evaluation cannot establish a conservative upper enclosure, the approximation MUST NOT be accepted.

7. Failure to establish such a conservative bound within the already-defined resource ceiling remains:

       CobStatus::ConvergenceExhausted

   under the existing convergence/resource rules.

8. This patch does NOT change:
   - the threshold `10^-31`;
   - the mathematical approximation;
   - the number of permitted iterations;
   - the underlying returned numerical result when the bound is successfully certified;
   - domain-error semantics;
   - target-PIC semantics;
   - workspace/resource status semantics.

---

# IMPORTANT DISTINCTION — MATHEMATICAL BOUND VS. COMPUTED BOUND

The resulting text MUST distinguish these concepts:

- the **mathematically justified exact upper bound** derived from the theorem/error analysis; and
- the **finite runtime representation of that upper bound** used to make the acceptance decision.

The runtime representation must conservatively enclose the mathematical bound from above.

Do not say merely that the bound formula is “rigorous.”

The formula can be mathematically rigorous while an incorrectly rounded implementation of the formula is not.

Do not permit an implementation to rely on ordinary rounding and assume the upper-bound property survives.

---

# COMPOSITION / BLENDING REQUIREMENT

The existing contract already requires the bound to cover component approximation error plus interpolation/blending error.

Preserve that.

The conservative-evaluation rule applies to the **complete composed bound**, including all arithmetic used to combine:

- component approximation error;
- argument-reduction error where included by the existing approximation-specific contract;
- Padé approximation error;
- interpolation/blending contribution;
- any other component already required by the existing documented total-error contract.

Do NOT invent new error components in this patch.

The point is simply that whatever components the existing formal contract requires must be combined without losing the upper-bound guarantee through inward rounding.

---

# PATCH LOCATION

Change only the local portion of Section 4 beginning with:

`*Normative Transcendental Error-Bound Contract:*`

and ending immediately before:

`*Strict Error Distinction:*`

A tiny grammatical dependency in `*Strict Error Distinction:*` may be edited only if absolutely necessary to preserve wording consistency, but its semantics MUST NOT change.

Everything else is frozen.

Prefer to clarify item 3 and the runtime acceptance paragraph rather than rewriting the whole subsection.

Do not change:

- the 64-cycle ceiling;
- the list of approximation families;
- the threshold `10^-31`;
- the requirement for approximation-specific formulas;
- formal documentation requirements;
- `ConvergenceExhausted`;
- `DomainError`;
- any other section.

---

# DO NOT WRITE THE APPROXIMATION ALGORITHMS YET

This annex is still defining the correctness contract.

Do NOT prescribe:

- a particular interval library;
- a particular decimal limb format;
- a specific directed-rounding implementation;
- exact guard-digit counts;
- a specific Padé degree;
- a specific Taylor truncation order;
- pseudocode for the approximation routines.

The later algorithm specification can choose those mechanisms.

This patch requires only the invariant:

> **The runtime acceptance value must be proven not to underestimate the mathematically justified error bound.**

# FIX THE CONTRACT. DO NOT WRITE THE FUCKING NUMERICAL-ANALYSIS LIBRARY YET.

---

# SOURCE-MARKER INTEGRITY

Bracketed source markers are hard literal document content.

The current complete document contains exactly:

- `[cite: 12]` = **189 occurrences**
- `[cite: 18]` = **1 occurrence**

The current `*Normative Transcendental Error-Bound Contract:*` paragraph through its runtime acceptance paragraph contains exactly:

- `[cite: 12]` = **8 occurrences**
- `[cite: 18]` = **0 occurrences**

If the immediately following `*Strict Error Distinction:*` sentence is included in the local edit boundary, the combined local region contains exactly:

- `[cite: 12]` = **9 occurrences**
- `[cite: 18]` = **0 occurrences**

The resulting document MUST preserve the applicable inventories exactly.

Do NOT:

- add source markers;
- remove source markers;
- renumber source markers;
- merge or split source markers;
- globally regenerate source markers;
- redistribute unrelated markers.

Keep each rewritten cited claim's marker attached to the semantically corresponding claim.

Do not invent a new marker solely for the conservative-rounding clarification.

---

# MINIMUM-DIFF RULE

The intended repair is small.

The existing contract already has the right architecture.

Add only the missing numerical-soundness rule:

- `E_bound` is an actual conservative runtime upper enclosure;
- its finite computation cannot round inward below the justified bound;
- acceptance uses that conservative value;
- inability to certify it means no acceptance and eventually `ConvergenceExhausted` under the existing resource rule.

Do not rewrite the whole transcendental section.

Do not change the target tolerance.

Do not “improve” neighboring terminology.

Do not touch `REMAINDER`.

If you notice another issue:

# KEEP YOUR FUCKING HANDS OFF IT.

---

# ACCEPTANCE TEST

After the patch, all of the following MUST be unambiguous.

1. Is the mathematically derived `E_bound(x)` still required?  
   **Yes.**

2. Must the runtime value used for acceptance be at least as large as the mathematically justified true bound?  
   **Yes.**

3. May finite decimal rounding cause the runtime bound to underestimate the mathematical bound?  
   **No.**

4. Is ordinary round-to-nearest automatically sufficient for evaluating an upper bound?  
   **No, not unless separately proven conservative for that evaluation.**

5. May exact arithmetic be used for the bound calculation?  
   **Yes.**

6. May directed/outward rounding be used?  
   **Yes.**

7. May interval arithmetic with a conservatively rounded upper endpoint be used?  
   **Yes.**

8. Is one particular implementation technique mandated?  
   **No.**

9. Does the acceptance comparison still use `E_bound(x) <= 10^-31`?  
   **Yes.**

10. May that comparison use an inward-rounded underestimate?  
    **No.**

11. If the implementation cannot certify a conservative bound within the existing resource ceiling, may it accept the approximation anyway?  
    **No.**

12. What status ultimately applies when the required bound cannot be established within the existing convergence resource ceiling?  
    **`CobStatus::ConvergenceExhausted`.**

13. Is the 64-cycle ceiling changed?  
    **No.**

14. Is the `10^-31` threshold changed?  
    **No.**

15. Are `DomainError` semantics changed?  
    **No.**

16. Does the complete document still contain exactly 189 `[cite: 12]` and 1 `[cite: 18]`?  
    **Yes.**

If any answer above is unclear after the edit, the patch is incomplete.

---

# OUTPUT

Return the complete updated `cambrian_cobol_annex-arithmetic.md` only.

Preserve everything outside the authorized local error-bound contract exactly.

# AN UPPER BOUND THAT ROUNDS DOWN IS NOT AN UPPER BOUND.
# CERTIFY THE RUNTIME ENCLOSURE.
# ACCEPT ONLY AGAINST THE CONSERVATIVE VALUE.
# SAME 10^-31 TARGET.
# SAME 64-CYCLE CEILING.
# NO PSEUDOCODE YET.
# DO NOT FUCK WITH ANYTHING ELSE.
