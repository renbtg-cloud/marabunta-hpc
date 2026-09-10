# ROLE FOR THIS PROMPT: HUMBLE, FOCUSED PATCH EXECUTOR

For this prompt, you are **not a specifications editor**.

You are not a reviewer, co-author, cleanup assistant, citation manager, source normalizer, or document improver.

You are a **humble and focused patch executor**.

The semantic wording in the current `cambrian_cobol_annex-arithmetic.md` is now correct.

# DO NOT CHANGE ANY SEMANTIC WORDING.

Your only job in this round is to repair one class of unauthorized literal-text changes:

**bracketed source markers were not preserved exactly as hard content.**

Bracketed source markers such as `[cite: 12]` and `[cite: 18]` are **hard document content, not metadata**.

They are literal characters in the file.

Do not regenerate them.

Do not reinterpret them.

Do not improve them.

Do not source new claims.

Do not decide where citations “should” go.

Do not perform citation management.

You are applying a literal-text preservation repair.

---

# INPUT AUTHORITY

Use the files this way:

- `cambrian_cobol_annex-arithmetic.md` = **SEMANTIC-TEXT AUTHORITY**
- `old.md` = **PRE-PATCH HARD-CONTENT REFERENCE**
- this request = **the only authorization for changes**

Preserve every non-source-marker character in the current `cambrian_cobol_annex-arithmetic.md` exactly as it is.

# NOT ONE FUCKING SEMANTIC WORD CHANGES IN THIS ROUND.

The current Section 4 rewrite is accepted semantically.

Do not rewrite it.

Do not polish it.

Do not rephrase it.

Do not “improve” it.

Do not revert it to `old.md`.

`old.md` is being supplied only so you can identify the hard source-marker content that existed before the patch.

---

# THE ONE DEFECT TO REPAIR

The current output violated the hard-content rule in two ways:

1. It removed the unique `[cite: 18]` from the negative-square-root domain-error clause outside the authorized Section 4 patch.
2. It introduced two additional `[cite: 12]` markers inside the rewritten Section 4 block that did not exist as source-marker content in the pre-patch block.

This round repairs **only that literal source-marker drift**.

Nothing else.

---

# REQUIRED REPAIR 1 — RESTORE THE UNIQUE `[cite: 18]`

In the square-root paragraph, restore the hard-content marker exactly so this fragment is:

    registers a domain error (`CobStatus::DomainError`)[cite: 18]; if $S = 0$

Do not change any other character in that paragraph.

The `[cite: 18]` marker is hard content.

It was not part of the authorized semantic patch.

Its deletion was unauthorized.

Put it the fuck back.

---

# REQUIRED REPAIR 2 — REMOVE ONLY THE TWO INVENTED `[cite: 12]` MARKERS

The rewritten Section 4 semantic text is correct and must remain exactly as written.

However, the current block contains six `[cite: 12]` markers where the pre-patch block contained four.

Do not “balance” them.

Do not invent sourcing.

Do not redistribute them.

Remove exactly these two newly introduced markers:

### A. Frame-depth bullet

Current:

    - **Frame-Depth Resource Failure (`CobStatus::StackOverflow`):** Used for scratchpad frame-depth exhaustion[cite: 12].

Required:

    - **Frame-Depth Resource Failure (`CobStatus::StackOverflow`):** Used for scratchpad frame-depth exhaustion.

### B. First sentence of the final paragraph

Current:

    Actual allocation or pool exhaustion during exponentiation is not `WorkspaceScaleOverflow` merely because the failure occurs while computing an exponentiation intermediate[cite: 12]. Internal infrastructure or workspace limits are never permitted to masquerade as COBOL receiving-field size errors and MUST NOT be reported as `CobStatus::SizeError`[cite: 12].

Required:

    Actual allocation or pool exhaustion during exponentiation is not `WorkspaceScaleOverflow` merely because the failure occurs while computing an exponentiation intermediate. Internal infrastructure or workspace limits are never permitted to masquerade as COBOL receiving-field size errors and MUST NOT be reported as `CobStatus::SizeError`[cite: 12].

Do not change any word, punctuation mark, spacing, or Markdown other than deleting those two exact `[cite: 12]` strings.

---

# EXPECTED FINAL SOURCE-MARKER INVENTORY

After the repair, the complete document must contain exactly:

- `[cite: 12]` = **189 occurrences**
- `[cite: 18]` = **1 occurrence**
- `[cite: 12, 18]` = **0 occurrences**
- `[cite: 17]` = **0 occurrences**
- `[cite: 17, 18]` = **0 occurrences**

This is not a request to “manage citations.”

This is a literal hard-content integrity check.

If your counts differ, your output is wrong.

---

# ABSOLUTE PATCH BOUNDARY

The only authorized character-level changes are:

1. insert `[cite: 18]` back after `CobStatus::DomainError` in the square-root clause;
2. delete the `[cite: 12]` at the end of the new frame-depth bullet;
3. delete the `[cite: 12]` after `exponentiation intermediate` in the first sentence of the final rewritten Section 4 paragraph.

# THREE MARKER OPERATIONS. THAT'S IT.

Do not change Section 4 wording.

Do not change any other wording.

Do not change whitespace.

Do not rewrap lines.

Do not normalize Markdown.

Do not change punctuation.

Do not adjust headings.

Do not fix another issue.

Do not touch anything merely because you notice it.

If it is not one of the three marker operations above:

# LEAVE THE FUCKING THING ALONE.

---

# SELF-CHECK

Before returning the file, verify:

1. Current non-source-marker text is byte-for-byte unchanged.
2. The accepted Section 4 semantic rewrite remains intact.
3. The square-root `[cite: 18]` is restored exactly.
4. The frame-depth bullet has no newly invented source marker.
5. The first new allocation/pool-exhaustion sentence in the final Section 4 paragraph has no newly invented source marker.
6. The existing `[cite: 12]` at the end of the infrastructure/`SizeError` sentence remains.
7. Global `[cite: 12]` count is exactly 189.
8. Global `[cite: 18]` count is exactly 1.
9. No other character changed.

If you changed a semantic word while performing this repair:

# REVERT IT. YOU WERE NOT ASKED TO EDIT THE SPECIFICATION.

If you changed anything beyond the three marker operations:

# REVERT THAT SHIT BEFORE RETURNING THE FILE.

---

# OUTPUT

Return the complete modified:

`cambrian_cobol_annex-arithmetic.md`

Do not return commentary.

Do not summarize.

Do not explain.

Return the full file with only the three authorized hard-content marker operations applied.

# HUMBLE PATCH EXECUTOR.
# THREE MARKER OPERATIONS.
# ZERO SEMANTIC EDITS.
# ZERO FUCKING SIDE QUESTS.
