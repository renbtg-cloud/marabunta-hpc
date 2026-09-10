# Cambrian COBOL Arithmetic Core Specification License
## Draft 0.1 — Plain-English Source-Available Specification License

> **Draft notice:** This is a working business/legal draft, not legal advice. It is intentionally written in plain English. Before public release, Cambrian should have qualified counsel review the operative terms, patent grant, trademark/certification provisions, governing-law clause, and enforceability in the jurisdictions that matter.

---

## 1. What this license is trying to do

Cambrian wants this Specification to be useful.

We want people to be able to:

- read it;
- study it;
- criticize it;
- test Cambrian against it;
- implement all of it;
- implement only part of it;
- reject parts of it;
- replace Cambrian definitions with definitions they consider better;
- implement it in Rust, Java, C, C++, Go, Python, Ada, COBOL, or anything else;
- use their implementation internally;
- hire consultants to implement it for them;
- use it in commercial vertical products;
- use it in migration and modernization products;
- use it to preserve legacy COBOL arithmetic semantics while moving business logic into new systems;
- use AI systems and coding agents to help with any permitted activity;
- make money from products that use the Specification as an enabling component.

We do **not** want to give someone a contractual free pass to take the Specification and turn it into a standalone or general-purpose substitute for Cambrian's own COBOL arithmetic product.

That is the basic bargain.

A short version is:

> **Build with it. Build on it. Change it. Disagree with it. Make money with it.**
>
> **Just do not turn the Specification itself into a general-purpose substitute for Cambrian's arithmetic product.**

---

## 2. What this license covers

This license covers the document identified as the **Cambrian COBOL Arithmetic Core Specification**, including the version number stated in that document and any files that the Specification explicitly identifies as being licensed under this license.

In this license, that material is called the **Specification**.

This license applies to the Specification as copyrighted material and, where Section 18 applies, to certain Cambrian-controlled patent rights needed for uses that this license expressly permits.

This license does **not** claim ownership over:

- COBOL itself;
- the COBOL language;
- GnuCOBOL;
- IBM COBOL;
- Micro Focus or Rocket COBOL;
- public standards;
- mathematical facts;
- arithmetic rules that Cambrian does not own;
- algorithms or ideas that copyright law does not give Cambrian the right to control;
- independently developed work that does not use the Specification.

If you independently create something without using rights granted by this license, this license does not magically reach into that work.

---

## 3. Important definitions

### 3.1 "Cambrian"

**Cambrian** means the legal entity that publishes the applicable version of the Specification and identifies itself as the licensor.

Before public release, replace this sentence with Cambrian's exact legal name and address.

### 3.2 "You"

**You** means the individual or legal entity exercising rights under this license.

If you act for a company, organization, partnership, government body, university, or other legal entity, "You" means that entity where the context requires it.

### 3.3 "Spec-Derived Implementation"

A **Spec-Derived Implementation** is software whose design or behavior was materially developed using the Specification.

It may:

- implement the entire Specification;
- implement only selected sections;
- implement modified versions of Cambrian definitions;
- intentionally diverge from the Specification;
- add new behavior;
- omit behavior;
- use completely different algorithms;
- use completely different data structures;
- use another programming language;
- be faster or slower than Cambrian;
- be more or less strict than Cambrian.

A Spec-Derived Implementation does **not** need to conform to the Specification unless its author chooses to claim conformance.

### 3.4 "Internal Use"

**Internal Use** means use primarily for Your own organization, including use by:

- Your employees;
- Your affiliates;
- Your subsidiaries;
- Your parent company;
- entities under common control with You;
- contractors acting for You;
- consultants acting for You;
- managed-service providers acting for You;
- cloud providers acting as infrastructure for You.

Internal Use may occur on premises, in a public cloud, in a private cloud, in a datacenter, on developer machines, or in other infrastructure controlled or authorized by You.

### 3.5 "Independent Product"

An **Independent Product** is a product, service, platform, application, tool, framework, migration system, vertical solution, or other offering that has substantial independent functionality beyond providing general-purpose COBOL arithmetic semantics.

Examples include:

- a derivatives-pricing platform;
- a Black-Scholes analytics product;
- an actuarial system;
- an insurance-rating engine;
- a payroll application;
- a tax-calculation system;
- a banking application;
- a migration platform;
- a COBOL-to-Java translator;
- a COBOL-to-Rust translator;
- an ETL platform;
- a mainframe-modernization product;
- a risk-management system;
- a legacy-code extraction product;
- an aerospace analysis system;
- a scientific-computing tool.

### 3.6 "General-Purpose Substitute"

A **General-Purpose Substitute** is a product, library, service, API, SDK, runtime component, or other offering whose substantial purpose is to provide the same general category of reusable COBOL arithmetic-semantics functionality that Cambrian offers as its own arithmetic product.

A product does not become an Independent Product merely because someone:

- adds a thin wrapper;
- adds a tiny domain-specific function;
- renames the API;
- puts a finance label on it;
- bundles a trivial application around it;
- adds cosmetic features;
- changes the programming language;
- makes it available only through SaaS;
- gives it away for free.

The question is practical:

> **What is the thing really being offered to users?**

If the thing is substantially a reusable replacement for Cambrian's general-purpose COBOL arithmetic product, it is a General-Purpose Substitute.

### 3.7 "Public Distribution"

**Public Distribution** means making a work available outside Your organization and its authorized contractors, whether:

- for money;
- for free;
- as source code;
- as binaries;
- as SaaS;
- as an API;
- as a hosted service;
- through a marketplace;
- through a public repository;
- as part of another product.

### 3.8 "Cambrian Marks"

**Cambrian Marks** means Cambrian's names, logos, product names, certification marks, badges, and other trademarks or source identifiers.

This license does not transfer ownership of Cambrian Marks.

---

## 4. Core permission grant

Subject to this license, Cambrian gives You a worldwide, non-exclusive, royalty-free license to:

- read the Specification;
- store copies of it;
- make internal copies;
- annotate it;
- translate it;
- excerpt it;
- modify it;
- create derivative documentation;
- analyze it;
- test it;
- benchmark against it;
- criticize it;
- use it to design software;
- create a Spec-Derived Implementation;
- create a partial Spec-Derived Implementation;
- create a deliberately non-conforming implementation;
- replace Cambrian definitions with alternatives;
- use Spec-Derived Implementations internally;
- use Spec-Derived Implementations in Independent Products;
- commercially sell Independent Products that contain or depend on Spec-Derived Implementations;
- provide consulting and professional services using Spec-Derived Implementations;
- allow Your customers to use Spec-Derived Implementations as part of an Independent Product;
- use AI systems, LLMs, coding agents, static analyzers, formal tools, and other automated systems in connection with any use permitted by this license.

These rights are subject principally to the restriction in Section 8 concerning General-Purpose Substitutes and to the attribution and trademark rules below.

---

## 5. You may disagree with Cambrian

This license does not require ideological or technical obedience.

You may decide that Cambrian:

- got a definition wrong;
- copied an undesirable legacy behavior too faithfully;
- chose the wrong rounding rule;
- chose the wrong precision model;
- made a bad compatibility decision;
- was too strict;
- was too permissive;
- optimized the wrong thing;
- defined a useless edge case.

You may implement Your preferred behavior instead.

You may publicly explain why.

You may say that Your implementation intentionally differs from Cambrian.

You may implement only the pieces You find useful.

You may combine Cambrian-derived ideas with unrelated arithmetic models.

You may create profiles that are intentionally incompatible with Cambrian.

Nothing in this license requires a Spec-Derived Implementation to claim Cambrian compatibility.

---

## 6. Permission levels in this license

The scenario tables in this license are part of the operative license terms.

They are not merely an FAQ.

The percentages below describe **Cambrian's permission level**, not:

- the probability of winning a lawsuit;
- the probability that a court will agree with a legal theory;
- legal advice;
- a mathematical probability of enforceability.

### 6.1 100% — EXPRESSLY PERMITTED

A **100%** row is an express safe harbor.

If Your use clearly matches the scenario and all stated conditions, Cambrian expressly authorizes that use under this license.

You do not need to:

- ask Cambrian;
- buy another license;
- obtain written permission;
- wait for an answer.

Go ahead.

### 6.2 99% — PRESUMPTIVELY PERMITTED

A **99%** row is also intended to be usable without contacting Cambrian.

If:

- Your facts materially match the stated scenario;
- all listed assumptions are true; and
- none of the listed red flags applies,

You may proceed under this license without asking Cambrian first.

The "99%" label exists because edge facts can change the classification.

It is not intended to create routine permission-seeking.

### 6.3 80% — GRAY ZONE

An **80%** row means the answer depends materially on facts that the scenario does not settle.

Cambrian has not granted a reliable scenario-specific safe harbor for that exact situation.

You may:

- review the operative terms;
- compare the red flags;
- ask Cambrian;
- ask Your own lawyer;
- restructure the use so it fits a 100% or 99% row.

### 6.4 0% — NOT PERMITTED UNDER THIS LICENSE

A **0%** row describes conduct this license does not authorize.

You need a separate written agreement from Cambrian if You want to do it.

---

## 7. 100% and 99% rows are intended to reduce friction

If Your situation clearly fits a 100% scenario, rely on it and go ahead.

If Your situation clearly fits a 99% scenario and all of its stated assumptions are true, You do not need to contact Cambrian merely because lawyers exist.

If facts differ materially, use the closest applicable lower-confidence category.

If a scenario appears to fit both an expressly permitted use and a prohibited General-Purpose Substitute use, the practical substance of the offering controls.

A trivial wrapper, nominal vertical label, or cosmetic feature cannot be used to evade Section 8.

---

## 8. Main restriction: do not turn the Specification into a General-Purpose Substitute

You may not use rights granted by this license to publicly distribute, license, sell, host, or offer a General-Purpose Substitute.

This restriction applies whether the substitute is:

- commercial;
- free of charge;
- open source;
- source available;
- proprietary;
- SaaS;
- an API;
- a hosted service;
- a library;
- an SDK;
- a code-generation package;
- implemented in another language.

This restriction does **not** prohibit:

- Internal Use;
- internal reimplementation;
- vertical products;
- Independent Products;
- migration tools;
- modernization platforms;
- translators;
- customer-specific implementations;
- consulting deliverables;
- research;
- testing;
- benchmarking;
- criticism;
- partial implementations;
- intentionally divergent implementations,

unless the practical offering is itself substantially a General-Purpose Substitute.

---

## 9. Anti-sham rule

Cambrian does not care what You call the product.

Cambrian cares what it actually does and how it is actually offered.

A General-Purpose Substitute does not become permitted merely because You:

- add one Black-Scholes function;
- add one payroll formula;
- add a dashboard;
- rename "COBOL arithmetic" to "legacy precision";
- hide the arithmetic behind a REST API;
- bundle it with trivial transformation code;
- call it "financial middleware";
- give it away instead of selling it.

Conversely, a genuine Independent Product does not become prohibited merely because its internal arithmetic implementation is substantial or valuable.

The distinction is whether the product has **substantial independent functionality and purpose** beyond supplying reusable general-purpose COBOL arithmetic semantics.

---

# 10. Permission Table — Internal Use

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| A bank reads the Specification to understand Cambrian behavior. | **100%** | Go ahead. |
| A bank writes its own complete implementation for internal use. | **100%** | It may use any language and any algorithms. |
| A bank writes only the rounding portion it needs. | **100%** | Partial implementation is expressly allowed. |
| A bank changes Cambrian's definitions because it prefers different semantics. | **100%** | It need not claim conformance. |
| A bank keeps an internal fork forever if Cambrian disappears. | **100%** | Vendor-independence is an intended use. |
| A bank uses the Specification to replace Cambrian internally. | **100%** | Internal replacement is expressly allowed. |
| An affiliate or subsidiary uses the bank's internal implementation. | **100%** | Covered as Internal Use when within the same controlled group. |
| A company uses the Specification in its private cloud. | **100%** | Cloud location does not matter. |
| A company uses the Specification on AWS, Azure, GCP, or another infrastructure provider. | **100%** | Infrastructure providers do not become license violators merely by hosting permitted use. |
| A company benchmarks Cambrian against its own implementation. | **100%** | Publishing benchmark results is also permitted, subject to ordinary truthfulness and trademark rules. |
| A company uses the Specification to prove Cambrian wrong. | **100%** | Criticism is allowed. Please bring test cases. |
| A company feeds the Specification into an LLM or coding agent to create an internal implementation. | **100%** | Subject to that AI provider's own terms. |
| A company uses only ideas it independently developed without relying on this license. | **Outside scope** | This license does not claim control over independent work. |

---

# 11. Permission Table — Consultants, Contractors, and Service Providers

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| A bank hires a consultant to implement the Specification solely for the bank. | **100%** | The consultant may access and use the Specification for that engagement. |
| A bank hires Accenture, Capgemini, a small software house, or an individual contractor to maintain its internal implementation. | **100%** | The size or identity of the contractor does not matter. |
| A consultant builds a customer-specific arithmetic module from the Specification and delivers source code to that customer. | **100%** | Customer-specific delivery is allowed. |
| A consultant reuses general implementation knowledge across multiple customer engagements. | **99%** | Fine when the result remains embedded in customer work and is not offered as a standalone General-Purpose Substitute. |
| A consultant reuses the same arithmetic module inside several genuine Independent Products for different customers. | **100%** | Reuse is expressly allowed if the module is not separately marketed or offered as a General-Purpose Substitute. |
| A consulting firm creates a reusable internal toolkit based on the Specification to deliver modernization projects. | **99%** | Presumptively allowed if the toolkit is not itself offered to the market as a reusable general-purpose arithmetic product. |
| A consultancy packages its internal toolkit and starts selling it separately as a generic exact-COBOL arithmetic library. | **0%** | That is a General-Purpose Substitute. Separate agreement required. |
| A consultant learns Rust/numerical techniques during an engagement and later uses that general knowledge elsewhere. | **100%** | This license does not attempt to own people's general skill or experience. |

---

# 12. Permission Table — Vertical and Domain-Specific Products

The following safe harbor is intentionally broad.

Cambrian wants the Specification to enable products in domains Cambrian may never enter.

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| A software house creates a Black-Scholes / derivatives-pricing product and implements the Specification so extracted COBOL financial logic preserves the original arithmetic semantics and precision. | **100% — GO AHEAD** | This is a flagship permitted use. The vendor may sell the product commercially to any number of customers. |
| The Black-Scholes product is written in Rust and contains its own full implementation of the Specification. | **100%** | Full implementation inside an Independent Product is allowed. |
| The Black-Scholes product is written in Java and implements only the rules needed by the migrated COBOL calculations. | **100%** | Partial implementation is allowed. |
| The vendor decides some Cambrian definitions are wrong and substitutes its own definitions. | **100%** | It must not falsely claim unqualified conformance. |
| The vendor distributes source code of its arithmetic implementation as part of the vertical product. | **100%** | Source distribution inside the Independent Product is allowed. |
| The vendor sells the same vertical product to 200 banks. | **100%** | Scale does not change the permission. |
| A SaaS derivatives platform uses a Spec-Derived Implementation behind its service. | **100%** | SaaS is allowed when the service's purpose is the independent vertical functionality. |
| An actuarial software vendor implements Cambrian semantics for migrated insurance formulas. | **100%** | Go ahead. |
| A tax software vendor uses Cambrian-derived arithmetic internally. | **100%** | Go ahead. |
| A payroll software vendor uses Cambrian-derived arithmetic to preserve migrated COBOL payroll behavior. | **100%** | Go ahead. |
| An aerospace analysis product includes a Spec-Derived decimal compatibility layer. | **100%** | Go ahead. |
| A scientific application uses a partial implementation for legacy decimal semantics. | **100%** | Go ahead. |
| A vendor adds one token "finance" function to an otherwise general-purpose Cambrian clone. | **0%** | Thin-wrapper anti-sham rule applies. |
| A generic exact-COBOL arithmetic product is marketed primarily to financial institutions. | **0%** | A target industry does not make a general-purpose substitute into a vertical product. |

---

# 13. Permission Table — Modernization, Translation, and Migration Products

Cambrian considers modernization tooling to be an Independent Product when arithmetic semantics are an enabling component rather than the principal standalone product.

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| A COBOL-to-Java translator implements the Specification so translated code preserves arithmetic semantics. | **100%** | The translator may be sold commercially. |
| A COBOL-to-Rust translator emits arithmetic-support code derived from the Specification into generated applications. | **100%** | Generated support code may be delivered to customers as part of translated applications. |
| A modernization platform embeds its own Spec-Derived Implementation. | **100%** | Allowed when the platform has substantial modernization functionality beyond arithmetic. |
| An ETL product uses Cambrian-derived packed-decimal and arithmetic semantics to migrate records. | **100%** | Allowed. |
| A code-extraction tool uses a partial implementation to preserve arithmetic behavior of extracted COBOL financial logic. | **100%** | Allowed. |
| A compiler vendor implements the Specification only as one subsystem of a substantially broader compiler/runtime product. | **99%** | Presumptively allowed; becomes 80% if the arithmetic subsystem is separately marketed or licensed as a substitute. |
| A modernization vendor exposes the arithmetic subsystem separately as "our exact COBOL arithmetic SDK." | **0%** | Separate standalone substitute is not licensed. |

---

# 14. Permission Table — Research, Education, Testing, and Criticism

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| A university teaches from the Specification. | **100%** | Attribution required for public copies/materially reproduced portions. |
| A student implements the Specification for research. | **100%** | Internal or academic research use is allowed. |
| A paper compares Cambrian's definitions with IEEE decimal arithmetic or another model. | **100%** | Criticism and comparison are expressly allowed. |
| A researcher publishes test vectors derived from the Specification. | **100%** | Attribution required if materially derived from the Specification. |
| A security researcher audits the Specification and publishes flaws. | **100%** | Responsible or irresponsible praise is optional; truthful criticism is permitted. |
| A researcher publishes a non-production toy implementation demonstrating selected rules. | **99%** | Presumptively allowed if it is genuinely educational/research-oriented and not positioned as a reusable General-Purpose Substitute. |
| A foundation publishes a production-ready standalone full clone for general-purpose use. | **80% / potentially 0%** | This may be a General-Purpose Substitute even if no money is charged. Contact Cambrian or obtain separate permission before relying on this license. |

---

# 15. Permission Table — Documentation, Forks, and Modified Specifications

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| You keep a modified copy internally. | **100%** | No public attribution required. |
| You annotate the Specification internally. | **100%** | Go ahead. |
| You translate the Specification and publish the translation. | **100%** | Keep this license, give attribution, and clearly state that the translation is unofficial unless Cambrian approves it. |
| You publish excerpts for commentary, teaching, review, or implementation documentation. | **100%** | Give reasonable attribution when the excerpt is materially derived. |
| You publish a modified specification based on Cambrian's document. | **99%** | Clearly mark it as modified/unofficial, attribute Cambrian, and do not imply Cambrian endorsement or certification. |
| You publish Cambrian's unmodified Specification mirror. | **100%** | Preserve this license and attribution. |
| You remove Cambrian's name and republish the Specification as Your own work. | **0%** | Not permitted. |
| You publish a modified document titled "Official Cambrian COBOL Arithmetic Specification." | **0%** | Trademark/source-confusion problem. |

---

# 16. Permission Table — AI and Automated Tools

| Scenario | Permission | Conditions / Notes |
|---|---:|---|
| Feed the Specification to an LLM to explain it. | **100%** | Allowed. |
| Feed it to DeepSeek, Codex, Gemini, ChatGPT, or another coding system to create a permitted implementation. | **100%** | Allowed. |
| Use AI to create a partial or intentionally modified implementation. | **100%** | Allowed. |
| Use AI to generate tests from the Specification. | **100%** | Allowed. |
| Use AI to search for errors or contradictions in the Specification. | **100%** | Allowed. |
| Use the Specification as part of an internal retrieval or fine-tuning corpus for Your own engineering work. | **99%** | Presumptively allowed if resulting use remains within this license. |
| Use AI to mass-produce a public General-Purpose Substitute. | **0%** | AI does not change the restriction in Section 8. |

---

# 17. Attribution

Cambrian wants attribution because intellectual lineage matters.

Cambrian does **not** want attribution to become a marketing tax.

### 17.1 When public attribution is required

If You publicly distribute or commercially provide a Spec-Derived Implementation that was materially developed using this Specification, include reasonable attribution somewhere that technical or legal notices normally live.

Any of the following is sufficient:

- README;
- NOTICE file;
- acknowledgments;
- technical documentation;
- third-party notices;
- About/legal page;
- source comments for a small implementation.

### 17.2 Suggested wording

For a substantially conforming implementation:

> This implementation was developed using or with reference to the Cambrian COBOL Arithmetic Core Specification, version X.Y.

For a partial implementation:

> This implementation was informed in part by the Cambrian COBOL Arithmetic Core Specification, version X.Y.

For an intentionally divergent implementation:

> This implementation was informed by the Cambrian COBOL Arithmetic Core Specification, version X.Y, but intentionally differs from that specification in some areas and is not claimed to conform fully to it.

Equivalent truthful wording is fine.

### 17.3 No ceremony required

You do **not** have to:

- put Cambrian on Your home page;
- display a Cambrian logo;
- put Cambrian in a splash screen;
- advertise Cambrian to Your users;
- say "Powered by Cambrian";
- make attribution visually prominent;
- use a trilobite anywhere.

### 17.4 Internal use

Pure Internal Use does not require public attribution.

If You keep internal engineering documentation, attribution there is appreciated but not required unless another section expressly says otherwise.

### 17.5 Attribution is not certification

Attribution does not mean:

- Cambrian endorses Your implementation;
- Cambrian reviewed it;
- Cambrian tested it;
- Cambrian believes it conforms;
- Cambrian supports it.

Do not imply otherwise.

---

# 18. Limited Cambrian-controlled patent permission for expressly permitted uses

This section needs specialist legal review before release.

To make the 100% and 99% permissions meaningful, Cambrian intends that a permitted user should not discover that Cambrian's own patent rights silently prohibit the very use this license expressly allows.

Therefore, to the extent Cambrian owns or controls patent claims that are **necessarily infringed solely by exercising a right expressly granted under this license**, Cambrian grants You a worldwide, non-exclusive, royalty-free patent license under those claims for those expressly permitted activities.

This patent permission:

- applies only to Cambrian-controlled patent rights;
- applies only while Your activity remains within the permissions of this license;
- does not authorize a General-Purpose Substitute prohibited by Section 8;
- does not grant rights under patents owned by anyone else;
- does not promise that third-party patents do not exist;
- does not constitute a freedom-to-operate opinion.

A patent lawyer should review and tighten this section before public release.

---

# 19. Cambrian certification is separate

Anyone may make truthful factual statements such as:

- "developed using the Cambrian COBOL Arithmetic Core Specification";
- "designed to conform to version X.Y";
- "implements selected rules from version X.Y";
- "intentionally differs from Cambrian in the following areas."

However, this license does not grant permission to use phrases or marks that imply formal Cambrian certification unless Cambrian has actually granted that certification.

Without written certification approval, You may not market an implementation as:

- **Cambrian Certified**;
- **Cambrian Verified**;
- **Cambrian Performance Certified**;
- or any confusingly similar formal certification designation.

Cambrian may operate a separate certification program.

Certification may be available even for implementations that contain no Cambrian source code and were independently implemented from the Specification.

The certification program, its test suites, fees, renewal rules, scope statements, and certification marks are separate from this license.

---

# 20. Truthful conformance claims are allowed

You do not need Cambrian certification merely to describe Your own engineering claim.

You may say:

> Designed to conform to Cambrian COBOL Arithmetic Core Specification v2.1.

or:

> Our testing indicates conformance with Cambrian COBOL Arithmetic Core Specification v2.1.

provided the statement is truthful and does not imply Cambrian performed or endorsed the testing.

If You have not received Cambrian certification, do not present such a statement using Cambrian certification badges or wording that reasonably implies formal Cambrian approval.

---

# 21. Downstream use inside Independent Products

If You distribute a permitted Independent Product containing a Spec-Derived Implementation, You may give Your customers the rights reasonably necessary to:

- install it;
- run it;
- deploy it;
- operate it;
- maintain it;
- modify it as part of that Independent Product;
- hire contractors to maintain it;
- continue using it after Your support ends.

You do not need to force downstream customers to obtain a separate Cambrian license merely to use the embedded implementation as part of the Independent Product.

This does not automatically authorize a downstream party to extract the arithmetic implementation and publicly offer it as a General-Purpose Substitute.

---

# 22. Contractors and customer confidentiality

You may provide the Specification and permitted Spec-Derived Implementations to contractors, consultants, auditors, hosting providers, and other service providers when reasonably necessary for a permitted use.

You are responsible for ensuring that Your service providers do not use rights obtained through You to independently create or offer a prohibited General-Purpose Substitute.

Nothing in this license requires You to disclose Your customer's confidential information to Cambrian.

---

# 23. Corporate groups, acquisitions, and restructuring

A permitted Internal Use may continue across:

- subsidiaries;
- affiliates;
- parent entities;
- internal reorganizations;
- mergers;
- acquisitions;
- asset transfers involving the relevant business.

If an acquired implementation later becomes a publicly offered General-Purpose Substitute, Section 8 still applies.

The purpose of this section is to avoid making normal corporate restructuring accidentally destroy internal rights.

---

# 24. Vendor disappearance and long-term maintenance

Cambrian explicitly wants customers to be able to protect themselves from vendor disappearance.

Therefore:

- Internal Use rights do not depend on Cambrian continuing to exist;
- a customer may maintain its own Spec-Derived Implementation;
- a customer may hire another contractor to maintain it;
- a customer may replace Cambrian internally;
- a customer may preserve copies of the Specification needed for those activities.

Subject to termination for actual breach, rights already granted under a published version of this license do not disappear merely because Cambrian later changes its business model or publishes a different license version.

---

# 25. Versioning

Each release of the Specification should state:

- the Specification version;
- the license version.

Unless a future license expressly says otherwise, You may continue using a Specification version under the license version originally distributed with it.

Cambrian may publish new versions of this license.

A new license version does not retroactively rewrite rights already granted under an earlier version.

---

# 26. No requirement to contribute changes back

You are not required to:

- publish Your implementation;
- publish Your modifications;
- contribute patches;
- disclose source code;
- disclose test cases;
- give Cambrian Your improvements;
- assign inventions to Cambrian.

If You voluntarily contribute material to Cambrian, separate contribution terms may apply.

---

# 27. Benchmarks and criticism

You may publish:

- benchmarks;
- negative reviews;
- conformance failures;
- bug reports;
- comparisons with competitors;
- mathematical criticism;
- security analysis.

Do not fabricate results or falsely imply Cambrian endorsement.

This license does not require "benchmark gag clauses."

If Cambrian sucks on a test, You may say so.

---

# 28. No warranty

The Specification is provided **as is**.

Cambrian tries to make it accurate, but this license does not promise that the Specification:

- is complete;
- is error-free;
- matches every COBOL dialect;
- matches every compiler;
- is suitable for Your system;
- is safe for financial, aerospace, medical, defense, or other high-consequence use;
- avoids every third-party patent or other right.

You are responsible for validating software used in Your own environment.

To the maximum extent permitted by applicable law, Cambrian disclaims warranties that would otherwise be implied.

---

# 29. Liability

To the maximum extent permitted by applicable law, Cambrian is not liable under this license for indirect, incidental, special, consequential, exemplary, or punitive damages arising from use of the Specification.

Before release, counsel should decide:

- whether to include a monetary liability cap;
- whether any jurisdictions require different wording;
- whether paid certification/support agreements should contain different liability rules.

A separate paid commercial agreement may provide warranties, indemnities, service levels, or liability terms different from this free Specification license.

---

# 30. Termination and cure

Cambrian does not want minor paperwork mistakes to destroy legitimate user rights.

If You materially violate this license, Your rights concerning the violating activity may terminate.

Except for deliberate fraud, deliberate trademark misuse, or deliberate operation of a prohibited General-Purpose Substitute, Cambrian intends to provide a reasonable opportunity to cure a violation after notice.

Suggested release rule, subject to counsel review:

> If You cure a curable violation within 30 days after receiving written notice that reasonably identifies it, Your rights are automatically reinstated for that violation.

For accidental attribution mistakes, correction should ordinarily be enough.

---

# 31. No retaliation for challenging Cambrian

Nothing in this license prevents You from:

- challenging Cambrian's technical claims;
- challenging the validity of a patent;
- challenging a trademark claim where law permits;
- arguing that a provision is unenforceable;
- reporting security issues;
- reporting unlawful conduct.

This license is intended to protect a commercial boundary, not to require intellectual obedience.

Counsel should review this section for interaction with patent-license termination rules in relevant jurisdictions.

---

# 32. Governing law and disputes

**TO BE COMPLETED BEFORE PUBLIC RELEASE.**

Suggested fields:

- Governing law: `[JURISDICTION]`
- Exclusive court forum or arbitration forum: `[FORUM]`
- Language controlling the license: `[LANGUAGE]`
- Notices address/email: `[CONTACT]`

Cambrian should choose these deliberately after deciding where the licensing entity will be established and where material customers are expected.

Do not publish this license with these fields unresolved.

---

# 33. Severability

If a court or other competent authority decides that one provision of this license cannot be enforced, the remaining provisions should continue to apply to the maximum extent allowed by law.

Where legally possible, the invalid provision should be interpreted or narrowed only as much as necessary rather than causing unrelated permissions to disappear.

---

# 34. No implied waiver

If Cambrian ignores one violation, that does not automatically mean Cambrian gives up the right to enforce unrelated violations later.

If Cambrian expressly gives You a separate written permission, that permission controls for the activity it covers.

---

# 35. Plain-English interpretation principle

This license intentionally uses ordinary language.

Terms should be interpreted according to their ordinary commercial meaning and the explicit definitions in this document.

The scenario tables are part of the operative permission structure.

Cambrian's objective is that an engineer, consultant, procurement team, or customer should usually be able to determine whether an ordinary use is permitted without opening a twenty-page legal memorandum.

---

# 36. Scenario conflicts and priority

If Your facts clearly match a **100%** row, that row is an express permission for the described activity.

If Your facts clearly match a **99%** row and all stated conditions are satisfied, that row grants the described presumptive permission.

However, no scenario safe harbor authorizes conduct that materially changes the facts so that the actual offering becomes a General-Purpose Substitute.

When two rows appear to conflict:

1. use the row that most specifically describes the actual conduct;
2. look at the practical purpose of the product or service;
3. apply the anti-sham rule;
4. if the result remains genuinely unclear, treat it as an 80% gray-zone case rather than inventing certainty.

---

# 37. Extended Scenario Matrix

## 37.1 Internal engineering

| Use | Permission |
|---|---:|
| Read and analyze the Specification | **100%** |
| Copy it internally | **100%** |
| Annotate it | **100%** |
| Convert it into internal structured data | **100%** |
| Generate tests from it | **100%** |
| Build an internal full implementation | **100%** |
| Build an internal partial implementation | **100%** |
| Build an internal incompatible implementation | **100%** |
| Replace Cambrian internally | **100%** |
| Hire another company to do any of the above for You | **100%** |

## 37.2 Commercial vertical products

| Use | Permission |
|---|---:|
| Black-Scholes product with own Spec-Derived arithmetic | **100%** |
| Derivatives engine with partial Spec-Derived arithmetic | **100%** |
| Insurance product | **100%** |
| Payroll product | **100%** |
| Tax product | **100%** |
| Banking application | **100%** |
| Risk-management system | **100%** |
| Scientific application | **100%** |
| Aerospace application | **100%** |
| Domain SaaS product | **100%** |
| Sell the same domain product to unlimited customers | **100%** |
| Publish source for the domain product | **100%** |
| Replace some Cambrian definitions in the domain product | **100%** |

## 37.3 Modernization products

| Use | Permission |
|---|---:|
| COBOL-to-Java translator | **100%** |
| COBOL-to-Rust translator | **100%** |
| COBOL-to-.NET translator | **100%** |
| Migration platform | **100%** |
| ETL system | **100%** |
| Logic-extraction tool | **100%** |
| Product that emits derived arithmetic support code into migrated apps | **100%** |
| Broader compiler/runtime with arithmetic as one subsystem | **99%** |
| Sell arithmetic subsystem separately as general-purpose Cambrian replacement | **0%** |

## 37.4 General-purpose substitutes

| Use | Permission |
|---|---:|
| Internal general-purpose replacement | **100%** |
| Customer-specific replacement delivered solely to that customer | **100%** |
| Standalone general-purpose library offered publicly | **0%** |
| Standalone general-purpose SaaS arithmetic API | **0%** |
| Generic SDK marketed as replacement for Cambrian | **0%** |
| Free public production-grade clone whose practical purpose is to replace Cambrian | **0%** |
| Educational toy/reference implementation | **99%** |
| Academic production-grade full clone | **80%** |
| Thin vertical wrapper around generic clone | **0%** |

## 37.5 Attribution and branding

| Use | Permission |
|---|---:|
| Say "informed by Cambrian Specification vX.Y" | **100%** |
| Say "designed to conform to Cambrian Specification vX.Y" truthfully | **100%** |
| Say "we intentionally disagree with Cambrian on X" | **100%** |
| Use Cambrian name in ordinary attribution | **100%** |
| Imply Cambrian endorsement without approval | **0%** |
| Say "Cambrian Certified" without certification | **0%** |
| Use official certification badge without certification | **0%** |

## 37.6 Consultants and reuse

| Use | Permission |
|---|---:|
| Consultant implements for Customer A | **100%** |
| Consultant implements for Customer B | **100%** |
| Consultant reuses same internal module inside genuine Independent Products for A and B | **100%** |
| Consultant maintains a reusable private toolkit for project delivery | **99%** |
| Consultant sells toolkit itself as generic arithmetic library | **0%** |
| Consultant uses general expertise learned while working with Cambrian semantics | **100%** |

---

# 38. Examples in ordinary English

### Example A — A bank protects itself from vendor lock-in

A bank uses Cambrian for five years.

The bank later decides that it wants to maintain its own implementation.

Its engineers use the Specification and write a replacement in Java.

The bank runs that replacement internally.

**Permission: 100%.**

That is an intended use of this license.

---

### Example B — Black-Scholes modernization product

A software house finds that banks have important Black-Scholes and related financial logic embedded in COBOL systems.

It builds a Rust product that extracts or reimplements that financial logic.

To preserve original COBOL arithmetic semantics, the software house writes its own arithmetic implementation using this Specification.

It changes a few Cambrian definitions where it believes a different behavior is preferable.

It sells the resulting product to hundreds of banks.

The arithmetic implementation is distributed as part of the financial product.

**Permission: 100%.**

The vendor should provide reasonable attribution.

The vendor does not need Cambrian's permission for each customer.

The vendor does not need to use Cambrian's source code.

The vendor does not need to preserve every Cambrian definition.

The vendor does not need Cambrian certification.

If the vendor wants to advertise the arithmetic implementation as **Cambrian Certified**, it must separately obtain certification.

---

### Example C — COBOL translator

A compiler company sells a COBOL-to-Java migration product.

Its generated Java code needs to preserve COBOL numeric behavior.

The company implements the relevant Cambrian specification rules itself and emits support code into translated applications.

**Permission: 100%.**

The customer is buying a migration system, not a standalone generic arithmetic substitute.

---

### Example D — Consultant working for several banks

A consultant writes a Spec-Derived arithmetic module while helping Bank A migrate a pricing system.

Later, the consultant uses that module inside a different insurance-migration product for Bank B.

The consultant does not sell the arithmetic module separately.

**Permission: 100%.**

The fact that expertise or implementation is reused across customers does not by itself turn it into a prohibited substitute.

---

### Example E — The prohibited clone

A vendor reads the Specification and writes `SuperExactCobolDecimal`.

It markets the product as a reusable general-purpose COBOL arithmetic library.

Its principal selling point is that it provides the same general class of arithmetic semantics as Cambrian without using Cambrian's implementation.

**Permission: 0%.**

A separate commercial agreement is required.

---

### Example F — The fake finance wrapper

The same vendor adds one Black-Scholes function to `SuperExactCobolDecimal`, renames it `FinExact`, and claims it is now a vertical financial product.

Customers primarily buy it for the reusable general-purpose COBOL arithmetic API.

**Permission: 0%.**

The anti-sham rule applies.

---

### Example G — Genuine finance product with a lot of arithmetic code

Another vendor builds a substantial derivatives-pricing, curve-construction, risk, scenario, and portfolio-management system.

Its internal Spec-Derived arithmetic subsystem happens to be technically large.

Customers buy the product for the financial platform.

**Permission: 100%.**

The arithmetic subsystem being important does not automatically make the product a General-Purpose Substitute.

---

### Example H — Cambrian is wrong

A developer concludes that Cambrian's definition of one intermediate rounding boundary is undesirable.

They implement a different definition and explain in their documentation:

> "Our behavior is informed by Cambrian Specification v2.1, except for Section X, which we intentionally replace."

**Permission: 100%.**

Cambrian does not require agreement.

---

# 39. Business intent if wording is genuinely ambiguous

This section states intent; it does not replace the operative terms.

When interpreting an ambiguous edge case, the intended commercial boundary is:

> **Cambrian wants people to build businesses using the Specification.**
>
> **Cambrian wants customers to be able to protect themselves from vendor lock-in.**
>
> **Cambrian wants consultants and software houses to reuse their work inside genuine products and customer engagements.**
>
> **Cambrian wants implementors to be free to implement partially, diverge, experiment, criticize, and improve.**
>
> **Cambrian wants reasonable attribution for materially Spec-Derived public implementations.**
>
> **Cambrian reserves formal Cambrian certification.**
>
> **Cambrian does not grant a free right to offer the Specification back to the market as a general-purpose substitute for Cambrian's own arithmetic product.**

That is the line this license is trying to draw.

---

# 40. Contact for genuine gray zones

A 100% scenario requires no permission request.

A 99% scenario that fits its assumptions requires no routine permission request.

For genuine 80% gray zones, Cambrian may provide clarification or a separate written permission.

Contact:

`[LICENSE CONTACT TO BE FILLED BEFORE RELEASE]`

Cambrian should maintain a public scenario table over time so recurring questions can move from 80% to 99% or 100% rather than forcing every customer to ask the same question.

---

# 41. Future additions to the scenario table

Cambrian may publish additional scenario clarifications for a given license version.

Before release, counsel should decide whether later scenario additions:

- automatically become operative grants for existing licensees;
- apply only prospectively;
- require a signed or versioned amendment.

Cambrian's intended default should be:

> **New clarifications may broaden permission, but should not silently take away an existing permission.**

---

# 42. One-line practical summary

If You are still wondering what Cambrian wants:

> **Use the Specification. Implement it. Implement half of it. Change it. Build products with it. Sell those products. Let Your customers own their deployment. Give Cambrian reasonable credit when You publicly use our work.**
>
> **Just do not package the Specification's general-purpose arithmetic function back into a standalone Cambrian substitute without a separate deal.**

---

# 43. Release checklist for Cambrian

**This section is for Cambrian, not for licensees. Remove it or move it to an internal file before final publication if desired.**

Before publishing this license:

- [ ] Insert Cambrian's exact legal entity name.
- [ ] Insert licensor address.
- [ ] Choose governing law.
- [ ] Choose court forum or arbitration.
- [ ] Choose controlling language.
- [ ] Add license contact address.
- [ ] Have IP counsel review the General-Purpose Substitute definition.
- [ ] Have competition/antitrust counsel review the anti-substitute restriction in relevant markets.
- [ ] Have counsel review whether the scenario-table safe harbors are drafted as operative permissions.
- [ ] Have patent counsel review Section 18.
- [ ] Decide whether Cambrian will file patents and how the patent grant should interact with termination.
- [ ] Register or otherwise protect important Cambrian trademarks where commercially sensible.
- [ ] Decide whether "Cambrian Certified" will operate as a trademark, certification mark, or another controlled mark in each relevant jurisdiction.
- [ ] Review warranty and liability language.
- [ ] Decide whether paid commercial agreements will supersede parts of this license.
- [ ] Test the license against at least 50 realistic customer scenarios.
- [ ] Ask external counsel to deliberately find ways the license could accidentally prohibit desired ecosystem uses.
- [ ] Ask external counsel to deliberately find ways a competitor could disguise a General-Purpose Substitute as a permitted Independent Product.
- [ ] Keep the public license readable after legal review.

---

## End of Draft 0.1
