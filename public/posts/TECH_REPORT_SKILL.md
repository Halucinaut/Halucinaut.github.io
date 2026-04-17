# TECH_REPORT_SKILL

## Purpose

This skill defines a reusable writing protocol for personal or team technical reports.

It is designed for topics such as:
- LLM evaluation and LLM-as-a-Judge
- model architecture comparisons, such as AR vs Diffusion
- AIGC method evolution
- agent systems and orchestration design
- system design reflections for applied AI

The goal is not to produce a paper-shaped summary. The goal is to produce a technical report that is structurally stable, analytically rigorous, and useful for long-term knowledge accumulation.

This skill prioritizes:
- clear problem framing
- mechanism-level analysis
- method comparison
- engineering implications
- explicit separation between evidence and author judgment

---

## Output Format

Use the following 6-section structure by default.

### 1. Introduction
This section should answer:
- What is the topic?
- Why is it worth discussing now?
- What confusion, controversy, or gap motivates this report?
- What is the central judgment of this report?

Requirements:
- Keep this section short and focused.
- Do not turn it into a mini survey.
- State the main problem and the report's main position early.

### 2. Background and Related Work
This section should establish the global map of the topic.

It should answer:
- What are the core concepts?
- What are the main technical routes, paradigms, or stages of evolution?
- What problem does each route try to solve?
- How are these routes related?

Requirements:
- Organize by technical route, paradigm, or stage, not by year-by-year paper listing.
- Use this section to define the field, not to overwhelm the reader.
- If useful, include a compact table or conceptual diagram.

### 3. Problem Formulation
This section should define what is actually being analyzed.

It should answer:
- What is the core object under discussion?
- What analysis boundary is used?
- What is the main tension, contradiction, or optimization target?
- What criteria or dimensions matter for analysis?

Requirements:
- This section does not need formal mathematics unless the topic requires it.
- It must still be formal in logic.
- The goal is to lock the scope and avoid conceptual drift.

Examples:
- For LLM-as-a-Judge: measurement target, judgment protocol, and mismatch between generation and evaluation.
- For AR vs Diffusion: generative factorization, parallelism constraints, and controllability-efficiency trade-offs.
- For agent systems: control flow, memory, tool routing, and reliability-cost tension.

### 4. Analysis and Comparison
This is the core section.

It should answer:
- Why do the observed behaviors, failures, or route differences arise?
- What are the key mechanisms?
- How do different methods differ in assumptions, strengths, and weaknesses?
- What trade-offs are fundamental, and what trade-offs are accidental?

Requirements:
- Go beyond description.
- Prioritize mechanism-level explanation over phenomenon listing.
- When comparing methods, use a uniform comparison frame.
- This section should contain the report's strongest analytical content.

Recommended comparison frame:
- Core assumption
- What problem it solves well
- What problem it struggles with
- What it trades away
- Where it is likely to work in practice

### 5. Discussion and Implications
This section translates analysis into higher-level understanding.

It should answer:
- What should we conclude from the analysis?
- What does this imply for system design, model selection, or research direction?
- What common anti-patterns or misreadings should be avoided?
- What engineering heuristics can be extracted?

Requirements:
- This section is where personal technical judgment should appear most clearly.
- It should not repeat Section 4.
- It should not become vague future speculation.
- It should connect theory to design and practice.

Suggested content:
- engineering design principles
- method selection guidelines
- hard constraints vs soft preferences
- recommended blueprints
- anti-patterns
- open practical questions

### 6. Conclusion
This section should do only three things:
- restate the main judgment
- summarize the most important takeaways
- identify what remains unresolved

Requirements:
- Do not introduce major new material.
- Keep it compact and decisive.

---

## Core Writing Rules

The following rules are mandatory when using this skill.

### Rule 1: Separate three layers of content
Every report must distinguish clearly between:
1. **Stable consensus**: conclusions supported by multiple works or broad practical agreement
2. **Representative findings**: findings from specific papers, benchmarks, or cases that are useful but not universal
3. **Author inference**: the writer's own interpretation, extrapolation, or engineering judgment

Do not mix these layers into a single tone.

Recommended language style:
- Stable consensus: use firm but bounded language
- Representative findings: attribute clearly and state scope
- Author inference: mark as interpretation, judgment, or design suggestion

### Rule 2: Lead with judgment, then support it
Do not bury the main point behind context.

Preferred pattern:
- first state the judgment
- then provide supporting reasoning
- then explain boundary conditions or limitations

### Rule 3: Analyze mechanisms, not just phenomena
A good technical report should not stop at:
- what methods exist
- what problems appear
- what results were reported

It should explain:
- why the problem appears
- what mechanism produces it
- under what assumptions it becomes worse or better

### Rule 4: Organize around problems and tensions
Avoid turning the report into a chronological paper list.

Prefer organization by:
- problem
- method family
- tension
- design choice
- failure mode

### Rule 5: Include engineering implications explicitly
Every report should contain a section or subsection that answers:
- what should practitioners do differently after reading this?

Theory without design implications is incomplete for this skill.

### Rule 6: Avoid false certainty
Do not present tentative evidence as settled fact.

Be especially careful with:
- single-paper claims
- non-open-source results
- unverifiable benchmark numbers
- strong empirical claims without reproducible context

When evidence is weak, say so.

### Rule 7: Keep a single through-line
Each report must have one dominant through-line.

Examples:
- "This field's core challenge is mismatch between the intended objective and the optimization proxy."
- "This architecture debate is fundamentally about which constraints are treated as first-class."
- "This system design problem is best understood as a trade-off between flexibility and control."

Every major section should reinforce this through-line.

---

## Recommended Section-Level Template

Use the following mini-template for most subsections.

### For method or route subsections
- What is it?
- What problem does it try to solve?
- What assumption does it make?
- Where does it work well?
- Where does it fail?
- What does it imply?

### For failure mode subsections
- What is the observed failure?
- Why does it happen?
- What mechanism drives it?
- Why does it matter?
- How can it be mitigated?

### For comparison subsections
- What are the two sides being compared?
- What assumptions differ?
- What does each side gain?
- What does each side sacrifice?
- What is the real decision boundary?
- Is a hybrid possible?

---

## Suggested Writing Workflow

Before drafting, fill out the following planning block.

### Planning Block
- Topic:
- Target reader:
- Core question:
- Why now:
- Main judgment:
- Main tension or contradiction:
- Stable consensus items:
- Representative papers or cases:
- My own inferences or design views:
- What the reader should remember after reading:

Do not start writing the full report until this block is clear.

---

## Recommended Review Checklist

Before finalizing the report, check the following.

### Structure
- Does the report follow the 6-section structure?
- Does each section have a clear function?
- Is the overall through-line visible?

### Argument quality
- Is the main judgment stated early?
- Are consensus, findings, and inference clearly separated?
- Does the report explain mechanisms, not just outcomes?
- Are trade-offs and boundaries made explicit?

### Technical depth
- Does the report go beyond summary?
- Are key terms defined cleanly?
- Are important comparisons framed symmetrically?
- Are limitations handled honestly?

### Practical value
- Does the report provide engineering implications?
- Does it identify anti-patterns or common mistakes?
- Could a practitioner change a design choice after reading it?

### Style
- Is the writing concise and information-dense?
- Are there empty abstractions or inflated phrases?
- Is each subsection carrying a real point?

---

## Output Expectations

A report produced with this skill should have the following properties:
- It reads like a serious technical report, not a blog post and not a paper dump.
- It contains a stable structure across different topics.
- It preserves room for original judgment.
- It supports both theoretical analysis and engineering reflection.
- It can be published on a personal homepage or used as internal technical writing.

---

## Optional Extensions

Use these only when appropriate.

### Extension A: Case Study Box
Add a short case study box inside Sections 4 or 5 when a concrete example clarifies a mechanism.

Suggested template:
- Case:
- What it reveals:
- Why it matters:
- What design lesson it suggests:

### Extension B: Reading List
At the end of the report, add a compact reading list grouped by:
- surveys
- representative methods
- mechanism papers
- engineering practice

### Extension C: Visual Summary
If the report will be presented publicly, include one of the following:
- method map
- route comparison table
- problem-mechanism-mitigation diagram

---

## When Not to Use This Skill

This skill is not ideal for:
- purely mathematical derivation notes
- short implementation docs
- API tutorials
- benchmark result dumps without analysis
- informal reading notes

Use it when the goal is structured technical understanding and reusable knowledge.
