# FIX REQUEST — SQUARE-ROOT DYNAMIC SCALING MUST COVER BOTH SMALL AND LARGE OUT-OF-WINDOW INPUTS

Apply exactly the change described below to the current `cambrian_cobol_annex-arithmetic.md`.

Do not redesign square-root extraction.

Do not change Newton-Raphson.

Do not change the existing square-root domain/zero dispatch.

Do not change `epsilon_m = 10^-38`.

Do not change the existing standard no-scaling window:

    epsilon_m <= S <= 10^38

Do not change the square-root 32-cycle ceiling.

Do not change the bounded non-semantic proof/approximation precision architecture.

This patch fixes one incomplete magnitude-normalization branch:

> **The annex explicitly defines dynamic even-power-of-10 scaling for `S < epsilon_m`, but it does not define what happens for otherwise-valid positive square-root inputs with `S > 10^38`, even though the preceding sentence defines `epsilon_m <= S <= 10^38` as the window in which scaling is bypassed.**

Everything unrelated to that missing upper-side branch is frozen.

---

# DEFECT

The current square-root subsection says:

> if `epsilon_m <= S <= 10^38`, dynamic scaling is bypassed entirely.

It then says:

> for `S < epsilon_m`, use an even-power-of-10 dynamic scaling wrapper:

    S' = S * 10^(2k)

and later:

    result = root' / 10^k

But there is no corresponding rule for:

    S > 10^38

That leaves a valid positive-magnitude case outside the explicitly defined no-scaling window with no normative scaling/dispatch behavior.

The architecture already has the right mechanism.

The fix is to make the even-power-of-10 wrapper symmetric:

- small out-of-window values use a positive `k`;
- large out-of-window values use a negative `k`;
- in-window values use no scaling.

---

# REQUIRED SEMANTIC RESULT

Modify only the local square-root magnitude-normalization wording so that all of the following are unambiguous.

1. For:

       epsilon_m <= S <= 10^38

   dynamic magnitude scaling is bypassed exactly as today.

2. For:

       S < epsilon_m

   the existing even-power-of-10 scaling mechanism remains valid.

3. For:

       S > 10^38

   the same mathematical wrapper MUST also be available, using a suitable negative integer shift factor `k` so that:

       S' = S * 10^(2k)

   brings the scaled positive argument into the existing standard Newton window.

4. The chosen integer `k` for any out-of-window positive `S` MUST make the scaled value `S'` fall inside the already-defined standard window:

       epsilon_m <= S' <= 10^38

   unless a separately existing exact/specialized path makes scaling unnecessary.

5. Because the exponent applied to the input is `2k`, the square-root rescaling identity remains exact:

       sqrt(S) = sqrt(S') / 10^k

6. The existing result-rescaling sentence:

       result = root' / 10^k

   remains mathematically authoritative for both signs of `k`.

7. `k > 0` naturally scales very small values upward.

8. `k < 0` naturally scales very large values downward.

9. `k = 0` corresponds to the existing no-scaling case.

10. The implementation MUST use checked scale/exponent arithmetic and the already-existing workspace-boundary rules when deriving/applying the scaling factor.

11. Do NOT classify a valid large positive square-root operand as `DomainError` merely because it is above `10^38`.

12. Do NOT classify it as `ConvergenceExhausted` merely because the upper-side scaling branch was previously unspecified.

13. Actual scale-boundary failure remains governed by the existing `CobStatus::WorkspaceScaleOverflow` rules.

14. Actual scratch allocation/resource failure remains governed by the existing `CobStatus::OutOfMemory` rules.

15. Failure of Newton/certification within the existing 32-cycle ceiling remains `CobStatus::ConvergenceExhausted`.

---

# IMPORTANT DISTINCTION — `10^38` IS A NO-SCALING WINDOW BOUNDARY, NOT A UNIVERSAL OPERAND-MAGNITUDE CEILING

The resulting text MUST NOT imply:

    S > 10^38  => invalid square-root operand

That is not what the current architecture says.

The value `10^38` is the upper boundary of the square-root path's **standard no-scaling window**.

It is not, by itself, a target-PIC limit, a general decimal-engine magnitude limit, or a mathematical square-root domain boundary.

A positive operand above that window must be normalized when necessary using the existing even-power-of-10 mechanism, subject to the engine's already-defined scale/workspace/resource limits.

---

# REQUIRED EDIT LOCATION

Change only the local portion of:

`* **Financial Fast-Paths & Bounded Square Root Extraction:**`

Specifically, edit only the three adjacent magnitude-normalization sentences/clauses:

1. the sentence beginning:

   `To eliminate calculation drift from static subnormal floor clamping...`

2. the sentence beginning:

   `For clustered subnormal inputs (S < epsilon_m)...`

3. if strictly necessary for grammatical consistency, the immediately following sentence beginning:

   `The Newton-Raphson iteration executes on S'...`

Everything else in the square-root subsection is frozen.

Preserve exactly:

- `S < 0` → `DomainError`;
- `S = 0` → exact normalized `+0` with scale preservation;
- `S > 0` → Newton path;
- `epsilon_m = 10^-38`;
- the even-power-of-10 form `S' = S × 10^(2k)`;
- result rescaling `result = root' / 10^k`;
- the existing proof-precision/certification semantics;
- the 32-cycle ceiling.

---

# DO NOT TURN THIS INTO PSEUDOCODE

Do NOT prescribe:

- the exact algorithm for selecting `k`;
- whether `k` is minimal;
- a lookup table;
- a specific logarithm/exponent-estimation method;
- a new numeric type;
- arbitrary precision;
- a new scale limit;
- a new resource ceiling.

It is enough to require that a suitable finite integer `k` normalize an otherwise-valid out-of-window positive `S` into the existing standard window while respecting the already-defined checked scale/workspace rules.

The later algorithm specification can choose the concrete `k` derivation.

---

# OPTION B WORDING IS FROZEN IN THIS PATCH

The current nearby sentence mentions the active `ScratchpadArenaGuard` and Build Option B.

Do NOT use this patch to redesign or rewrite Option B.

Do NOT change task-migration semantics here.

Do NOT move `k` into task-control state.

The arithmetic interval remains governed by the already-defined synchronous guard-lifetime rules.

If you notice wording you dislike there:

# KEEP YOUR FUCKING HANDS OFF IT FOR THIS PATCH.

---

# SOURCE-MARKER INTEGRITY

Bracketed source markers are hard literal document content.

The current complete document contains exactly:

- `[cite: 12]` = **189 occurrences**
- `[cite: 18]` = **1 occurrence**

The current complete:

`* **Financial Fast-Paths & Bounded Square Root Extraction:**`

block contains exactly:

- `[cite: 12]` = **11 occurrences**
- `[cite: 18]` = **1 occurrence**

The resulting document MUST preserve those inventories exactly.

Do NOT:

- add source markers;
- remove source markers;
- renumber source markers;
- merge or split source markers;
- move unrelated source markers;
- regenerate source markers globally.

Keep each existing marker attached to the semantically corresponding rewritten claim.

---

# PATCH BOUNDARY

Only the local square-root magnitude-normalization wording identified above is editable.

Everything else is frozen.

Do not change:

- 38-digit semantic precision;
- 76-digit multiplication scratch;
- proof workspace;
- transformed-exponent semantics;
- `10^-31`;
- conservative `E_bound`;
- final-rounding certification;
- square-root 32-cycle ceiling;
- general transcendental 64-cycle ceiling;
- zero-base exponentiation;
- power-of-10 pre-check;
- division;
- `REMAINDER`;
- rounding modes;
- Option A;
- Option B;
- COMP-3;
- FFI;
- existing scale boundaries.

If you notice another defect:

# LEAVE IT THE FUCK ALONE.

---

# ACCEPTANCE TEST

After the patch, all of the following MUST be unambiguous.

1. Is `epsilon_m <= S <= 10^38` still the square-root no-scaling window?  
   **Yes.**

2. Does `S < epsilon_m` still use the existing even-power-of-10 scaling mechanism when needed?  
   **Yes.**

3. Is `S > 10^38` now explicitly covered?  
   **Yes.**

4. For a large positive out-of-window operand, may `k` be negative?  
   **Yes.**

5. For a small positive out-of-window operand, may `k` be positive?  
   **Yes.**

6. Must the selected scaling preserve:

       S' = S * 10^(2k)

   **Yes.**

7. Must the result still satisfy:

       result = root' / 10^k

   **Yes.**

8. Must the scaled operand be brought into the existing standard window when this normalization path is used?  
   **Yes.**

9. Does `S > 10^38` automatically mean `DomainError`?  
   **No.**

10. Does `S > 10^38` automatically mean `SizeError`?  
    **No.**

11. Are actual scale-boundary failures still `WorkspaceScaleOverflow`?  
    **Yes.**

12. Are actual allocation failures still `OutOfMemory`?  
    **Yes.**

13. Does inability to converge/certify within 32 Newton cycles remain `ConvergenceExhausted`?  
    **Yes.**

14. Is `epsilon_m = 10^-38` unchanged?  
    **Yes.**

15. Is the 32-cycle ceiling unchanged?  
    **Yes.**

16. Does this prescribe a concrete `k` algorithm?  
    **No.**

17. Does the complete document still contain exactly 189 `[cite: 12]` and 1 `[cite: 18]`?  
    **Yes.**

If any answer above is unclear after the edit, the patch is incomplete.

---

# OUTPUT

Return the complete updated `cambrian_cobol_annex-arithmetic.md` only.

Preserve everything outside the authorized square-root magnitude-normalization wording exactly.

# THE STANDARD WINDOW HAS TWO SIDES.
# SMALL VALUES SCALE UP.
# LARGE VALUES SCALE DOWN.
# `S > 10^38` IS NOT A FUCKING DOMAIN ERROR.
# KEEP THE EVEN-POWER-OF-10 IDENTITY.
# SAME 32-CYCLE CEILING.
# SAME SCALE/RESOURCE STATUSES.
# NO PSEUDOCODE YET.
# DO NOT FUCK WITH ANYTHING ELSE.
