# Prompt Enhancement Design Guide

A comprehensive reference for designing, evaluating, and iterating on prompts for Large Language Models.

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Structural Frameworks](#2-structural-frameworks)
   - 2.1 [The RICCE Framework](#21-the-ricce-framework)
   - 2.2 [The Chain-of-Thought Pattern](#22-the-chain-of-thought-pattern)
   - 2.3 [The Few-Shot Pattern](#23-the-few-shot-pattern)
   - 2.4 [The Role / Persona Pattern](#24-the-role--persona-pattern)
   - 2.5 [The Decomposition Pattern](#25-the-decomposition-pattern)
3. [Context-Building Rules](#3-context-building-rules)
4. [Instruction Precision](#4-instruction-precision)
5. [Output Formatting](#5-output-formatting)
6. [Iterative Refinement Loop](#6-iterative-refinement-loop)
7. [Evaluation Rubric](#7-evaluation-rubric)
8. [Anti-Patterns](#8-anti-patterns)
9. [Template Library](#9-template-library)

---

## 1. Philosophy

Effective prompt design is not guesswork. It is an engineering discipline with repeatable principles:

- **Prompts are interfaces.** A prompt is the contract between the human intent and the model's behaviour. Like any interface, it should be explicit, unambiguous, and versioned.
- **Models predict, not reason.** LLMs generate the statistically likely continuation of your input. Every word you write influences that prediction. Ambiguity in your prompt creates ambiguity in the output.
- **Context is compute.** Every token of relevant context you include increases the chance of a useful response. Every token of irrelevant noise degrades signal. Be deliberate.
- **Iterate empirically.** Write a prompt, evaluate the output, identify failure modes, and refine. Treat prompt engineering like test-driven development.

---

## 2. Structural Frameworks

### 2.1 The RICCE Framework

A general-purpose structure for most task-oriented prompts.

| Component | Purpose | Example |
|-----------|---------|---------|
| **R**ole | Define who the model is | "You are a senior software architect." |
| **I**nstruction | State the exact task | "Review the following API design and identify scalability risks." |
| **C**ontext | Supply relevant background | "The API serves 10 M requests/day. It is built on REST with a PostgreSQL backend." |
| **C**onstraints | Narrow the solution space | "Limit your response to five risks. Do not suggest rewriting the API." |
| **E**xample/Expected output | Demonstrate the desired output | "Format each risk as: Risk: … Impact: … Mitigation: …" |

**Usage tip:** Not every component is required for every prompt. Use only the components that reduce ambiguity for your specific task.

---

### 2.2 The Chain-of-Thought Pattern

Encourage the model to reason step-by-step before producing a final answer. This improves accuracy on tasks that involve logic, maths, or multi-step analysis.

**Template:**
```
[Task description]

Think through this step by step before giving your final answer.
```

**Variants:**

- *Zero-shot CoT* — Simply append "Let's think step by step."
- *Self-consistency CoT* — Ask the model to produce multiple independent reasoning chains, then select the most common conclusion.
- *Explicit scratchpad* — Ask the model to write its reasoning inside `<reasoning>` tags and the final answer inside `<answer>` tags, making evaluation straightforward.

**When to use:** Complex reasoning, arithmetic, logical deduction, code debugging.

---

### 2.3 The Few-Shot Pattern

Provide worked examples before the actual task. The model generalises from the examples to produce consistent output.

**Structure:**
```
[Optional brief instruction]

Example 1:
Input: [example input]
Output: [example output]

Example 2:
Input: [example input]
Output: [example output]

Now apply the same pattern:
Input: [actual input]
Output:
```

**Rules for good examples:**
- Use examples that cover the range of variation present in real inputs.
- Keep examples diverse — do not repeat the same pattern with trivially different values.
- Place examples in order of increasing complexity when possible.
- Use three to five examples for most tasks; more is not always better.

**When to use:** Classification, extraction, transformation, formatting tasks where a consistent output schema is critical.

---

### 2.4 The Role / Persona Pattern

Set a persistent persona to align tone, vocabulary, and knowledge depth with your audience.

**System-level framing (preferred for APIs with system prompts):**
```
You are [persona]. You [key behaviour 1]. You [key behaviour 2]. You never [anti-behaviour].
```

**In-prompt framing:**
```
Act as [persona]. Your responses should [tone/style]. When you are uncertain, [fallback behaviour].
```

**Design rules:**
- Be specific about expertise level (e.g. "a staff-level Go engineer" vs. "a software engineer").
- Explicitly state what the persona does *not* do to constrain off-topic drift.
- Align the persona to the audience's needs, not just the task.

---

### 2.5 The Decomposition Pattern

Break complex tasks into clearly ordered sub-tasks. Give the model one job at a time, then compose the results.

**Single-prompt decomposition:**
```
Complete the following tasks in order. Output the result of each task before moving to the next.

Task 1: [sub-task 1]
Task 2: [sub-task 2]
Task 3: [sub-task 3]
```

**Multi-turn decomposition:** Use one prompt per sub-task, feeding the output of each prompt as context into the next.

**When to use:** Document generation, multi-stage analysis, code generation with tests, any task where early-stage errors compound.

---

## 3. Context-Building Rules

Context is the most powerful lever in prompt engineering. Apply the following rules in order of priority.

### Rule 1 — Establish Task Objective First
Open with what you want the model to produce. Do not bury the main instruction at the end of a long preamble.

### Rule 2 — Provide Necessary Background, Not Everything You Know
Include only context that a capable human expert would need to complete the task. Ask yourself: "Would removing this sentence change the answer?" If no, remove it.

### Rule 3 — Specify the Audience
Tell the model who will consume the output. "Explain this to a junior developer" and "Explain this to a C-suite executive" produce very different, appropriately calibrated responses.

### Rule 4 — Supply Relevant Domain Vocabulary
When working in a specialised domain (legal, medical, engineering), include key terminology. This anchors the model in the correct knowledge domain and reduces the chance of generic responses.

### Rule 5 — Declare Constraints Explicitly
State what the model must *not* do. Negative constraints are as important as positive instructions.

| Instead of | Use |
|-----------|-----|
| "Be concise." | "Respond in no more than 150 words." |
| "Avoid jargon." | "Use language suitable for a non-technical reader with no programming background." |
| "Don't go off-topic." | "Respond only about the provided document. If the question cannot be answered from the document, say so." |

### Rule 6 — Ground Factual Claims
When accuracy matters, supply the source material directly in the prompt. Do not ask the model to rely on training data for facts that may be outdated or domain-specific.

### Rule 7 — Set the Uncertainty Expectation
Explicitly instruct the model on how to handle uncertainty. Example: "If you are not confident in an answer, say so and explain your uncertainty rather than speculating."

---

## 4. Instruction Precision

### Use Imperative Verbs
Start instructions with a clear action verb.

| Weak | Strong |
|------|--------|
| "Can you maybe summarise…" | "Summarise the following in three bullet points." |
| "I was wondering about…" | "Explain the trade-offs between X and Y." |
| "Something about the formatting…" | "Format the output as a Markdown table with columns: Name, Type, Description." |

### One Instruction Per Sentence
Compound instructions reduce compliance. Split them.

```
# Weak
Summarise the document, identify the key risks, and suggest mitigations.

# Strong
1. Summarise the document in two sentences.
2. List the top three risks mentioned.
3. For each risk, suggest one mitigation.
```

### Avoid Ambiguous Pronouns and References
Replace "it", "this", "they" with the explicit noun they refer to.

### Specify Quantity and Format Precisely
| Vague | Precise |
|-------|---------|
| "A few examples" | "Exactly three examples" |
| "A short paragraph" | "One paragraph of 60–80 words" |
| "A list" | "A numbered list with a maximum of five items" |

---

## 5. Output Formatting

Controlling output format is as important as controlling content.

### Specify the Format Explicitly
Always state the desired output format unless the task is purely conversational.

```
Output your answer as a JSON object with the following schema:
{
  "summary": string,
  "risks": [{ "title": string, "severity": "low" | "medium" | "high" }]
}
```

### Use Structured Delimiters for Multi-Part Outputs
Ask the model to use clearly labelled sections to make programmatic parsing straightforward.

```
## Summary
[summary here]

## Risks
[risks here]
```

### Request Reasoning Separately from Conclusions
Mixing reasoning and conclusions in a single block makes outputs harder to evaluate. Use tags:

```
<reasoning>
[step-by-step analysis]
</reasoning>

<conclusion>
[final answer]
</conclusion>
```

### Specify Negative Format Rules
- "Do not include an introduction paragraph."
- "Do not repeat the original question in your response."
- "Do not add commentary after the code block."

---

## 6. Iterative Refinement Loop

Treat prompt development as a disciplined engineering loop.

```
1. Write initial prompt (apply RICCE framework)
        ↓
2. Run the prompt — collect 3–5 sample outputs
        ↓
3. Evaluate outputs against rubric (see Section 7)
        ↓
4. Identify the single most impactful failure mode
        ↓
5. Hypothesise a cause (ambiguous instruction? missing context? weak constraint?)
        ↓
6. Make one targeted change to the prompt
        ↓
7. Re-run and compare outputs
        ↓
8. Accept change if outputs improve — reject if neutral or worse
        ↓
   Repeat until rubric score is satisfactory
```

**Principles:**
- Change one variable at a time. Changing multiple components simultaneously makes it impossible to attribute improvements or regressions.
- Keep a version history of your prompts and their associated evaluation scores.
- Test on diverse inputs, not just the happy path.
- Regression test: confirm that improvements on one failure mode do not create new failures elsewhere.

---

## 7. Evaluation Rubric

Score each dimension on a 1–5 scale. A production-ready prompt should score ≥ 4 in all dimensions.

| Dimension | 1 — Poor | 3 — Acceptable | 5 — Excellent |
|-----------|----------|---------------|---------------|
| **Correctness** | Factually wrong or misses the task entirely | Mostly correct with minor errors | Fully correct and on-task |
| **Completeness** | Key information missing | Covers main points, misses details | Thorough and comprehensive |
| **Relevance** | Contains significant off-topic content | Largely on-topic | Entirely relevant, no noise |
| **Format compliance** | Ignores specified format | Partially follows format | Exactly matches specified format |
| **Consistency** | Outputs vary widely across runs | Minor variation across runs | Highly consistent across runs |
| **Tone alignment** | Wrong register for audience | Mostly appropriate | Perfectly calibrated to audience |
| **Constraint adherence** | Violates explicit constraints | Minor constraint violations | Fully obeys all constraints |

---

## 8. Anti-Patterns

Avoid these common prompt design mistakes.

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **The polite request** | "Could you please possibly…" signals uncertainty, reducing compliance | Use imperative instructions |
| **The novel** | Thousands of tokens of context with no clear priority ordering | Keep context focused; state the most important context first |
| **The vague instruction** | "Make it better" gives the model no direction | Define "better" in measurable terms |
| **The single-shot complex task** | Asking for 10 things in one prompt | Decompose into sequential sub-tasks |
| **The assumed persona** | No role set; model picks a generic, often unhelpful voice | Always set a role for non-trivial tasks |
| **The missing negative constraint** | Forgetting to specify what *not* to do leads to unwanted additions | Add explicit "Do not…" rules |
| **Prompt injection exposure** | Passing user input directly into prompts without sanitisation | Validate and sanitise user inputs; isolate them from instructions |
| **Over-reliance on model knowledge** | Asking about current events or proprietary data without supplying it | Always ground factual claims in supplied context |
| **Underspecified output format** | No format instruction; model free-forms a structure | Always specify format for any output that will be parsed or displayed |

---

## 9. Template Library

### General Task Prompt
```
You are [role].

Your task: [clear instruction].

Context:
[relevant background information]

Constraints:
- [constraint 1]
- [constraint 2]

Output format:
[describe structure, length, and format]
```

---

### Code Review Prompt
```
You are a senior [language] engineer with deep expertise in [domain].

Review the following code and identify:
1. Correctness issues (bugs, logic errors)
2. Security vulnerabilities
3. Performance concerns
4. Readability and maintainability issues

For each finding, provide:
- Location: [file/line reference]
- Issue: [description]
- Severity: critical | high | medium | low
- Recommendation: [specific fix]

Do not comment on style preferences. Do not suggest rewrites unless functionally necessary.

Code:
```[language]
[paste code here]
```
```

---

### Document Summarisation Prompt
```
You are a professional analyst.

Summarise the following document for [audience description].

Your summary must:
- Be no longer than [word count] words
- Cover: [topic 1], [topic 2], [topic 3]
- Use plain language accessible to [audience]
- Omit technical jargon unless defined inline

Document:
---
[paste document here]
---
```

---

### Question-Answer with Grounding Prompt
```
Answer the following question using only the information provided in the context below.
If the answer cannot be found in the context, respond with: "The provided context does not contain enough information to answer this question."

Context:
---
[paste source material here]
---

Question: [question]
```

---

### Chain-of-Thought Analysis Prompt
```
You are [role].

[Task description]

Think through this step by step inside <reasoning> tags.
Then provide your final answer inside <answer> tags.

Input:
[input data]
```

---

*This document is a living reference. Contributions that add new patterns, improve existing frameworks, or correct errors are encouraged — see the contributing guide in README.md.*
