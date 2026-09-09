# Cambrian Implementation-Recipe & Adversarial Provenance Workflow

## 1. Purpose

Cambrian normative specifications define **what observable behavior is required**.

Where useful, a normative specification MAY reference small companion `.md` files containing implementation-oriented material such as:

- deliberately simple pseudocode;
- explicit algorithms;
- loops and branches;
- heuristics;
- decision tables;
- state machines;
- invariants;
- implementation recipes;
- pathological examples;
- reference test vectors;
- expected failure conditions;
- resource-limit handling;
- optimization constraints.

These companion files exist to reduce implementation ambiguity and to reduce the amount of mathematical, architectural, or algorithmic invention required from implementation agents such as DeepSeek, Codex, or future coding systems.

The intended implementation-agent task should increasingly resemble:

> Translate this approved algorithm into safe, idiomatic, efficient Rust while preserving its defined behavior.

rather than:

> Read this large specification, infer the intended mathematics, invent an algorithm, design an architecture, implement it, and determine whether the result is correct.

The closer Cambrian can move the implementation task toward faithful translation, the cheaper, more reproducible, and more auditable AI-assisted development becomes.

---

# 2. Authority Hierarchy

Cambrian engineering artifacts have an explicit authority hierarchy.

In descending order:

1. **Normative specification**
2. **Approved implementation recipe / algorithm**
3. **Approved invariants and test requirements**
4. **Production implementation**
5. **Optimization implementation**
6. **Comments and explanatory material**

The normative specification defines required semantics.

An implementation recipe defines an approved method for realizing those semantics.

Production code implements the approved method.

Tests provide evidence that the implementation behaves correctly.

No lower-level artifact may silently redefine a higher-level artifact.

---

# 3. Normative Conflict Is a Hard Interlock

The implementation recipe is **not the normative specification**.

If an implementation recipe conflicts with, contradicts, or cannot be reconciled unambiguously with the normative specification, **code generation MUST stop at the point where the conflict is detected**.

The implementation agent MUST:

1. identify the conflicting normative requirement;
2. identify the conflicting implementation-recipe instruction;
3. describe the mismatch clearly in conversational output;
4. refrain from guessing;
5. refrain from silently choosing one interpretation;
6. refrain from silently modifying the normative requirement;
7. refrain from silently modifying the implementation recipe;
8. produce no code whose correctness depends upon resolving the conflict.

The implementation agent MAY continue working on unrelated code paths whose behavior does not depend upon the unresolved conflict.

Code generation for the affected path may resume only after the normative specification or implementation recipe has been explicitly corrected or clarified.

**Ambiguity is an error condition, not an invitation to creativity.**

---

# 4. Overall Development Pipeline

The preferred Cambrian workflow is:

    Normative semantics
            ↓
    Deliberately simple implementation recipe
            ↓
    Adversarial algorithm review
            ↓
    Algorithm correction
            ↓
    Provenance / implementation-similarity review
            ↓
    Prior-art / algorithm-ancestry review
            ↓
    Patent / FTO preliminary triage
            ↓
    Approved implementation recipe
            ↓
    Implementation agent
          /       \
    Production    Decision /
       code       provenance log
          \       /
            ↓
    Independent implementation audit
            ↓
    Unit / property / adversarial tests
            ↓
    Differential reference testing
            ↓
    Correct scalar implementation
            ↓
    Optimization
            ↓
    Scalar-vs-optimized differential testing

The objective is to make creative freedom progressively smaller as an artifact approaches production.

---

# 5. Suggested Repository Structure

A possible layout is:

    specs/
        cambrian_cobol_annex-arithmetic.md

    algorithms/
        decimal-addition.md
        decimal-subtraction.md
        decimal-multiplication.md
        decimal-division.md
        remainder.md
        rounding.md
        integer-power.md
        fractional-power.md

    src/
        addition.rs
        addition.rs.decisions.md

        multiplication.rs
        multiplication.rs.decisions.md

        division.rs
        division.rs.decisions.md

        remainder.rs
        remainder.rs.decisions.md

        rounding.rs
        rounding.rs.decisions.md

The exact repository organization is implementation-defined.

The important structural idea is that important production modules have traceable relationships to:

- normative requirements;
- implementation recipes;
- tests;
- implementation decisions;
- provenance information.

---

# 6. Implementation Recipe Header

Every implementation-recipe file SHOULD clearly identify its status and authority.

Example:

    # Exact Remainder Quotient Algorithm

    Status:
    NON-NORMATIVE IMPLEMENTATION RECIPE

    Normative authority:
    specs/cambrian_cobol_annex-arithmetic.md §5

    Implements:
    - ARITH-REM-001
    - ARITH-REM-002
    - ARITH-REM-006

    Conflict rule:
    If this recipe conflicts with the normative specification,
    implementation MUST stop for the affected path and report
    the mismatch. No dependent code may be generated until
    the conflict is resolved.

Requirement identifiers MAY be introduced where useful to improve traceability.

---

# 7. Pseudocode Should Be Deliberately Simple

Implementation recipes SHOULD favor **obviousness over elegance**.

The recipe does not need to resemble idiomatic Rust.

It does not need to impress anyone.

It needs to make the algorithm inspectable.

For example:

    if discarded_fraction < half:
        keep_value()

    else if discarded_fraction > half:
        increment_retained_digit()

    else:
        if retained_digit_is_even:
            keep_value()
        else:
            increment_retained_digit()

Likewise:

    while carry != 0:
        process_next_digit()

or:

    if sign_a == sign_b:
        result_sign = positive
    else:
        result_sign = negative

Explicit loops, repetitive branches, temporary variables, and mechanically obvious control flow are acceptable.

The pseudocode MAY be ugly.

It SHOULD NOT be clever merely for elegance.

The primary question is:

> Can a competent reviewer easily inspect the control flow and determine what the algorithm actually does?

Ugly but obvious is preferable to elegant but opaque.

---

# 8. Reference Algorithms and Optimized Algorithms

For difficult kernels Cambrian MAY maintain separate recipes:

    algorithms/reference/foo.md
    algorithms/optimized/foo.md

The reference implementation SHOULD prioritize:

- simplicity;
- direct correspondence with normative semantics;
- auditability;
- deterministic behavior;
- clear invariants;
- obvious control flow.

The optimized implementation MAY introduce:

- better asymptotic algorithms;
- specialized integer representations;
- SIMD;
- vectorization;
- lookup tables;
- caching;
- memoization;
- arena allocation;
- platform-specific instructions;
- branch elimination;
- specialized fast paths.

However:

> Every optimized path must demonstrably preserve the observable behavior of the approved scalar/reference implementation and the normative specification.

Optimization is subordinate to correctness.

---

# 9. Algorithm-Level Invariants

Implementation recipes SHOULD state important invariants explicitly.

Examples:

    abs(remainder) < abs(divisor)

    quotient == mathematical_truncation_toward_zero(dividend / divisor)

    optimized_result == scalar_result

    carry ∈ {0, 1}

    digit ∈ 0..9

    no binary floating-point conversion occurs

    no receiving-field mutation occurs after SizeError

An invariant is useful when it makes incorrect implementations easier to detect.

Important invariants SHOULD also be represented in tests or assertions where practical.

---

# 10. Adversarial Algorithm Review

Before production implementation, each non-trivial implementation recipe SHOULD be attacked independently.

The purpose of review is **not consensus**.

The purpose is to find failures.

Useful review prompts include:

> Assume this algorithm is wrong. Find counterexamples.

> Find inputs for which this recipe violates the normative specification.

> Look specifically for boundary failures, sign errors, scale errors, precision loss, incorrect termination, integer overflow, incorrect rounding, aliasing assumptions, invalid resource handling, and hidden binary-floating-point assumptions.

> Find cases where the implementation recipe permits behavior prohibited by the normative specification.

> Do not redesign the architecture unless redesign is required to demonstrate a defect.

Independent models SHOULD preferably perform independent reviews rather than merely accepting another reviewer's conclusions.

For example:

    Gemini:
        attack mathematics and algorithm assumptions

    ChatGPT:
        attack specification compliance, edge cases,
        contradictions, and missing invariants

    DeepSeek or another model:
        independently search for counterexamples

LLM agreement is not proof.

Executable evidence and authoritative reference behavior outrank LLM consensus.

---

# 11. Algorithm Correction Loop

When an adversarial reviewer finds a valid defect:

1. correct the implementation recipe;
2. rerun adversarial review;
3. update associated tests;
4. repeat until no known material defect remains.

The coding agent SHOULD receive the algorithm only after the algorithm has reached an approved implementation state.

This deliberately moves expensive conceptual work **before** production implementation.

---

# 12. Provenance / Implementation-Similarity Audit

After an algorithm stabilizes, Cambrian SHOULD perform a separate adversarial provenance review.

The question is:

> Does this implementation recipe appear suspiciously derived from an identifiable external implementation?

Reviewers should search for distinctive similarities such as:

- unusual operation ordering;
- distinctive constants;
- unusual lookup tables;
- characteristic branch structures;
- peculiar normalization procedures;
- matching magic values;
- unusual special-case handling;
- recognizable internal terminology;
- unusual performance heuristics;
- strongly matching state machines;
- recognizable code structure surviving conversion into pseudocode.

A useful adversarial prompt is:

> Attempt to identify existing source code from which this algorithm could plausibly have been derived. Search aggressively for distinctive structural similarities. Do not assume independent development merely because identifiers or language syntax differ.

A superficial similarity to a standard mathematical algorithm is not itself evidence of copying.

The goal is to discover candidates deserving investigation.

---

# 13. Prior-Art / Algorithm-Ancestry Review

A separate review SHOULD ask:

> What known algorithms, publications, standards, libraries, or historical techniques does this recipe resemble?

Relevant sources may include:

- academic papers;
- textbooks;
- standards;
- numerical-analysis literature;
- compiler literature;
- historical algorithms;
- public implementations;
- runtime systems;
- expired patents;
- active patents.

Possible findings include:

    Cambrian algorithm appears equivalent to known Algorithm X,
    combined with Cambrian-specific COBOL scale and
    terminalization semantics.

This is useful information.

Independently rediscovering an established algorithm is not necessarily undesirable.

An established algorithm MAY be preferable if it is:

- better analyzed;
- easier to prove correct;
- better documented;
- more efficient;
- more portable;
- easier to test.

Cambrian MAY deliberately replace a homegrown algorithm with a known established algorithm after suitable provenance, license, and patent review.

---

# 14. Patent / Freedom-to-Operate Preliminary Triage

Patent analysis MUST remain separate from copyright provenance and license compatibility.

Independent invention does not necessarily eliminate patent risk.

LLMs SHOULD NOT be asked merely:

> Does this infringe any patents?

A simple affirmative or negative answer has little value.

Instead, the review should be adversarial:

> Search for the strongest active patent families whose claims could plausibly read onto this implementation recipe.

For each plausible candidate, the reviewer SHOULD attempt to identify:

1. patent/application identifier;
2. patent family;
3. assignee;
4. relevant jurisdictions;
5. priority dates;
6. apparent legal status;
7. relevant independent claims;
8. dependent claims that materially narrow the technique;
9. each relevant claim limitation;
10. corresponding Cambrian implementation element, if any;
11. claim elements clearly absent;
12. uncertain mappings;
13. reasons the patent may or may not warrant specialist review.

The desired result is not:

> No infringement detected.

The desired result is:

> These are the strongest candidates found, these are their relevant claim elements, this is where Cambrian appears to overlap, this is where Cambrian appears not to overlap, and these areas remain uncertain.

LLM-based patent review is **preliminary triage only**.

It is not a legal freedom-to-operate opinion.

Material unresolved candidates SHOULD be reviewed by qualified patent counsel before relevant commercial deployment.

---

# 15. Keep Four IP Questions Separate

Cambrian reviews MUST distinguish at least four different questions.

## 15.1 Source Provenance / Copyright

Did Cambrian copy or closely reproduce protected implementation expression from another source?

## 15.2 License Compatibility

If external code, pseudocode, tables, or other implementation material was intentionally used, are the associated license terms compatible with Cambrian's intended use and distribution?

## 15.3 Prior Art / Algorithm Ancestry

Is the technique already known, standardized, published, historically established, or widely implemented?

## 15.4 Patent Freedom-to-Operate

Could an enforceable patent claim cover what Cambrian intends to make, use, distribute, license, or sell in a relevant jurisdiction?

These questions are related but fundamentally different.

They MUST NOT be collapsed into a generic "IP check."

---

# 16. Suggested Provenance Section in Algorithm Files

Each significant recipe MAY eventually contain:

    ## Provenance

    Normative source:
    - cambrian_cobol_annex-arithmetic.md §X.Y

    Development origin:
    - independently derived from normative requirements
    - refined through Cambrian multi-LLM review

    Known related algorithms:
    - Algorithm A
    - Algorithm B

    External source code intentionally consulted:
    - none

    External publications intentionally consulted:
    - ...

    Distinctive Cambrian requirements:
    - ...
    - ...

    Provenance review:
    - date:
    - reviewers:
    - suspicious implementation matches:
    - disposition:

    Prior-art review:
    - date:
    - known algorithm ancestry:
    - relevant publications:

    Patent/FTO preliminary triage:
    - date:
    - jurisdictions searched:
    - potentially relevant patent families:
    - unresolved candidates:
    - specialist review required: yes/no

This creates an intellectual chain of custody around implementation decisions.

---

# 17. Implementation-Agent Contract

Once semantics and algorithms are approved, the implementation agent SHOULD receive tightly bounded instructions.

Example:

    Implement algorithms/remainder.md in Rust.

    Normative authority:
    specs/cambrian_cobol_annex-arithmetic.md §5.

    Requirements:

    - Preserve the normative semantics.
    - Preserve the functional behavior of the approved algorithm.
    - Do not redesign arithmetic semantics.
    - Do not silently substitute another algorithm.
    - Use safe idiomatic Rust where practical.
    - Report conflicts between recipe and specification.
    - Report any case where the recipe cannot be implemented
      as written.
    - Abort generation for any path affected by an unresolved
      specification/recipe conflict.
    - Add tests exercising every meaningful branch and
      documented boundary.
    - Run relevant tests.
    - Record material implementation decisions.
    - Record deviations from the recipe.
    - Record materially influential external sources.
    - Do not optimize before scalar correctness is established.

The implementation agent is primarily being hired as:

> a top-quality implementation programmer with adequate mathematical competence.

It is not being asked to act simultaneously as:

- product architect;
- COBOL archaeologist;
- numerical-method inventor;
- specification author;
- standards interpreter;
- patent lawyer.

---

# 18. Implementation Decision Logs

Important production modules SHOULD have a companion decision file.

Example:

    src/rounding.rs
    src/rounding.rs.decisions.md

Markdown is preferred initially because it is easy for:

- humans;
- LLMs;
- auditors;
- source-control review.

A structured format such as JSON MAY later be generated mechanically if automated tooling requires it.

Cambrian SHOULD avoid maintaining both `.md` and `.json` manually because duplicated decision records can diverge.

---

# 19. What Belongs in a Decision Log

The decision log SHOULD document **material implementation decisions**, not trivial syntax choices.

Examples of material decisions:

- algorithm selection;
- representation choice;
- precision strategy;
- scale strategy;
- rounding implementation;
- overflow handling;
- resource-limit handling;
- allocation strategy;
- unsafe-code use;
- external dependency choice;
- portability compromise;
- synchronization strategy;
- performance/correctness trade-off;
- recipe deviation;
- optimization;
- interpretation of implementation freedom;
- compatibility behavior.

The log SHOULD NOT contain useless bureaucracy such as:

> Used a `for` loop because Rust supports loops.

The question is:

> Did the implementor have meaningful freedom here?

If yes, the decision may deserve recording.

---

# 20. Suggested Decision Record

Example:

    ## DEC-001 — Exact half comparison uses decimal digits

    Decision:
    Compare discarded decimal digits directly.

    Normative authority:
    arithmetic annex §6.

    Recipe authority:
    algorithms/rounding.md.

    Reason:
    Binary floating-point conversion would violate the normative
    requirement for exact decimal threshold evaluation.

    Alternatives considered:
    - f64 conversion
    - approximate floating threshold

    Alternatives rejected because:
    They cannot guarantee exact decimal tie detection.

    Implementation:
    - round_nearest_even()
    - round_nearest_away()

    Evidence:
    - rounding_exact_half_positive
    - rounding_exact_half_negative
    - rounding_below_half
    - rounding_above_half

---

# 21. Required Deviation Records

If the implementation differs materially from the approved recipe, the implementation agent MUST record the deviation.

Example:

    ## DEV-001

    Recipe:
    algorithms/division.md specifies ...

    Production implementation:
    division.rs instead performs ...

    Reason:
    ...

    Why observable behavior is unchanged:
    ...

    Invariants preserved:
    ...

    Evidence:
    - ...
    - ...

    Review status:
    pending

A deviation is not automatically prohibited.

Cambrian explicitly permits implementors to improve implementation quality.

However:

> A material deviation must be visible.

This reflects the cultural rule:

> Do as asked. Improve it if you can. Tell us exactly where you stopped doing as asked.

---

# 22. Conflicts Discovered During Implementation

If an implementation agent discovers a conflict, the decision log SHOULD record it.

Example:

    ## CONFLICT-001

    Normative specification:
    §X requires ...

    Implementation recipe:
    algorithms/foo.md requires ...

    Conflict:
    These requirements cannot both hold because ...

    Impact:
    Code path foo() depends upon the unresolved behavior.

    Action:
    IMPLEMENTATION ABORTED FOR THIS PATH.

    Dependent code generated:
    No.

    Resolution:
    Pending specification clarification.

This creates a durable record rather than leaving the contradiction buried in a chat transcript.

---

# 23. External-Material Record

The implementation decision file SHOULD contain an external-material section.

Example:

    ## External Material

    External implementations consulted:
    - none

    External algorithm descriptions consulted:
    - Knuth, TAOCP Vol. 2, Algorithm D
    - Rust standard-library documentation

    External source code copied:
    - none

    External source code adapted:
    - none

    Materially influential sources:
    - ...

The purpose is not bureaucratic purity.

The purpose is traceability.

---

# 24. No Retrospective Rationalization

Implementation agents MUST NOT invent beautiful explanations after the fact merely to justify existing code.

Material decisions SHOULD be recorded when reasonably close to the time they are made.

If a decision was accidental, inherited, or its rationale is genuinely unknown, the record SHOULD state that.

For example:

    Reason:
    Unknown. This structure emerged during implementation and
    requires independent review before being considered intentional.

This is preferable to fabricated certainty.

LLMs are particularly capable of producing plausible retrospective rationalizations.

Cambrian therefore treats unexplained decisions as legitimate review targets rather than asking models to invent explanations.

---

# 25. Independent Implementation Audit

After implementation, an independent reviewer SHOULD inspect:

    normative specification
            +
    implementation recipe
            +
    production code
            +
    decision log
            +
    deviations
            +
    tests
            +
    external-material record

The reviewer SHOULD ask:

> Does the production implementation actually implement the approved algorithm?

> Does the approved algorithm actually satisfy the normative requirement?

> Did the implementation make decisions that were not recorded?

> Did the implementation silently deviate from the recipe?

> Are any recorded rationales unsupported?

> Are any tests merely reproducing the implementation rather than independently checking semantics?

> Are important branches or invariants untested?

> Did external implementation structure appear unexpectedly in the production code?

This review SHOULD preferably be performed by models other than the original implementor.

---

# 26. Multi-LLM Division of Labor

Cambrian MAY use different LLMs for different roles.

A possible low-cost workflow is:

    DeepSeek
        primary implementation agent

    Gemini
        mathematical / algorithmic adversarial reviewer

    ChatGPT
        normative-specification / implementation-compliance reviewer

    Codex
        escalation implementation agent for difficult Rust,
        debugging, optimization, repository-level work, or
        implementation failures that cheaper models cannot resolve

The roles are not permanent.

The important property is **independence of review**.

An implementation agent SHOULD NOT be treated as the final authority on its own work.

---

# 27. Reviewers Are Critics, Not Co-Authors by Default

When auditing an implementation, reviewers SHOULD normally be instructed:

> Do not redesign for style.

> Do not rewrite merely because you prefer another abstraction.

> Find demonstrable defects, missing cases, specification mismatches, unjustified assumptions, unsafe behavior, or missing tests.

Otherwise multiple LLMs may waste time endlessly "improving" each other's architecture.

Architectural redesign SHOULD be requested explicitly when desired.

---

# 28. Testing Authority

Neither pseudocode nor production Rust proves correctness.

Correctness evidence SHOULD come from multiple layers:

    normative specification
            +
    reviewed implementation recipe
            +
    branch tests
            +
    boundary tests
            +
    unit tests
            +
    property tests
            +
    adversarial test vectors
            +
    invariant checks
            +
    scalar-vs-optimized differential testing
            +
    external reference-runtime differential testing
              where applicable

Tests themselves are not automatically correct.

Where possible, test expectations SHOULD derive independently from normative requirements or trusted reference behavior rather than simply reproducing implementation logic.

---

# 29. Differential Reference Testing

Where a selected authoritative or compatibility reference exists, Cambrian SHOULD compare its behavior empirically.

A disagreement MUST be investigated.

It MUST NOT automatically be resolved by assuming:

> Cambrian must be wrong.

Nor:

> The reference must be wrong.

Instead determine:

- whether the normative specification is wrong;
- whether the implementation recipe is wrong;
- whether the production implementation is wrong;
- whether the test harness is wrong;
- whether configuration differs;
- whether the reference behavior is dialect-specific;
- whether the reference contains an implementation quirk Cambrian intentionally does or does not reproduce.

Differential testing provides evidence.

It does not replace semantic reasoning.

---

# 30. Optimization Rule

The normal sequence is:

    Make semantics explicit.

    Make the algorithm obvious.

    Attack the algorithm.

    Establish provenance.

    Implement the scalar version.

    Test the scalar version.

    Differentially validate it.

    Then make it fast.

A fast incorrect arithmetic engine is worthless.

An ugly correct scalar implementation is an asset.

Optimized paths should therefore be introduced only after an acceptable scalar correctness oracle exists.

---

# 31. Optimization-Agent Contract

When requesting an optimization, the agent SHOULD receive instructions such as:

    Optimize src/rounding.rs.

    The current scalar implementation is the correctness oracle.

    Observable behavior MUST remain identical.

    You MAY change:
    - representation
    - loop structure
    - vectorization
    - memory access strategy
    - allocation behavior

    You MAY NOT change:
    - normative semantics
    - rounding decisions
    - error behavior
    - observable status codes

    Any material change must be documented in
    rounding.rs.decisions.md.

    Add scalar-vs-optimized differential tests.

This keeps performance creativity subordinate to semantic stability.

---

# 32. Cost-Control Objective

A central reason for implementation recipes is to reduce expensive coding-agent reasoning.

Without recipes:

    specification
        ↓
    coding agent must:
        interpret semantics
        invent algorithm
        choose representation
        decide edge cases
        write code
        test code
        debug assumptions

With recipes:

    specification
        ↓
    approved algorithm
        ↓
    implementation agent:
        translate
        integrate
        test
        report deviations

This allows cheaper implementation models to perform a larger proportion of the work.

More capable or expensive models can be reserved for:

- pathological arithmetic failures;
- difficult Rust ownership problems;
- concurrency;
- formal numerical analysis;
- optimization;
- repo-wide debugging;
- discrepancies that cheaper models cannot resolve.

---

# 33. Bounded Work Units

Implementation tasks SHOULD be kept small and independently understandable.

Prefer:

> Implement `algorithms/rounding.md`.

over:

> Implement the arithmetic annex.

Prefer:

> Implement exact remainder quotient determination and its tests.

over:

> Fix division.

Prefer:

> Implement the Java JNI wrapper for these already-established calls.

over:

> Add Java support.

Small work units reduce:

- context size;
- architectural wandering;
- accidental redesign;
- debugging surface;
- token usage;
- cross-model confusion.

---

# 34. Ownership of Bounded Modules

If multiple implementation models are used, Cambrian SHOULD prefer assigning whole bounded modules rather than handing partially implemented algorithms back and forth unnecessarily.

For example:

    DeepSeek owns:
    - rounding
    - COMP-3 decoding
    - integer exponentiation

rather than:

    DeepSeek writes half of division,
    Codex writes another third,
    DeepSeek fixes Codex,
    another model rewrites everything.

Cross-model handoffs create additional context and understanding costs.

Escalation is useful when a module becomes genuinely difficult.

---

# 35. Failure Escalation

A possible escalation ladder is:

    1. Cheap implementation model attempts bounded task.

    2. Independent models audit result.

    3. Original implementation model fixes concrete findings.

    4. Tests and differential harness rerun.

    5. If the subsystem remains unresolved, escalate the
       entire bounded subsystem to a stronger coding model.

The stronger model SHOULD receive:

- normative specification;
- approved recipe;
- existing implementation;
- decision log;
- failing tests;
- independent audit findings.

It should not have to rediscover the entire history from scratch.

---

# 36. Security Against LLM Confidence

LLM confidence is not evidence.

Cambrian SHALL assume that any model may:

- hallucinate;
- misunderstand the specification;
- invent unsupported mathematical claims;
- rationalize accidental code;
- overlook boundary conditions;
- reproduce familiar implementations unintentionally;
- confidently misstate patent status;
- create tests that merely confirm its own bug.

Therefore:

> No LLM is trusted merely because it sounds certain.

The workflow is designed around cross-checking, bounded artifacts, provenance, and executable evidence.

---

# 37. Internal Knowledge vs Public Documentation

These implementation recipes, decision logs, adversarial test corpora, provenance records, and detailed semantic notes MAY constitute valuable internal Cambrian knowledge.

They do not automatically need to be published merely because production source code is published.

Cambrian MAY distinguish between:

## Public material

- API documentation;
- usage documentation;
- compatibility claims;
- ordinary tests;
- examples;
- supported behavior;
- documented limitations;
- open-source production implementation where applicable.

## Internal material

- exhaustive semantic decision tables;
- adversarial implementation recipes;
- complete differential corpora;
- vendor-specific behavioral discoveries;
- internal proofs;
- exhaustive pathological cases;
- unpublished compatibility observations;
- detailed implementation provenance;
- patent/FTO review records;
- complete semantic archaeology.

Open source does not require publishing every laboratory notebook that produced the source.

---

# 38. Intellectual Chain of Custody

The desired result for every important algorithm is a traceable chain:

    Normative requirement
            ↓
    Approved implementation recipe
            ↓
    Adversarial algorithm review
            ↓
    Provenance / prior-art / patent triage
            ↓
    Production implementation
            ↓
    Decision log
            ↓
    Deviations
            ↓
    Tests
            ↓
    Independent implementation audit
            ↓
    Differential evidence

This allows a future reviewer to answer:

> Why does this code behave this way?

without relying upon:

> Because an LLM wrote it three years ago and nobody remembers.

---

# 39. Core Philosophy

Cambrian implementation recipes exist to make difficult reasoning **small, explicit, inspectable, attackable, and reproducible**.

They do not need to be beautiful.

They need to be:

- bounded;
- explicit;
- understandable;
- adversarially reviewable;
- testable;
- traceable;
- replaceable;
- subordinate to normative semantics.

The intended division of labor is:

> **Humans and LLMs argue about meaning.**

> **Normative specifications define what must happen.**

> **Simple pseudocode freezes an approved way to make it happen.**

> **Adversarial reviewers try to destroy that algorithm.**

> **Provenance review looks for suspicious implementation ancestry.**

> **Prior-art review determines what we may have reinvented.**

> **Patent/FTO triage looks for the strongest plausible claim overlap rather than seeking reassurance.**

> **Implementation agents turn approved recipes into production code.**

> **Decision logs expose the material choices they made.**

> **Deviations are permitted but must not be hidden.**

> **Conflicts stop generation rather than invite improvisation.**

> **Independent reviewers attack the resulting implementation.**

> **Tests and reference behavior provide evidence.**

> **Optimization happens only after correctness has a boring place to stand.**

The objective is not to eliminate creativity.

The objective is to use creativity **where creativity is useful**, then progressively remove ambiguity as the software approaches production.

For high-assurance closed-world computation, that is a feature rather than a limitation.