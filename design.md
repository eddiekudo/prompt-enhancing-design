# Prompt Enhancement Design Guide

> Structural frameworks, context-building rules, output-formatting patterns, and best practices for writing highly effective LLM prompts.

---

## Table of Contents

1. [Prompt Anatomy](#1-prompt-anatomy)
2. [Structural Frameworks](#2-structural-frameworks)
   - 2.1 [Zero-Shot Prompting](#21-zero-shot-prompting)
   - 2.2 [Few-Shot Prompting](#22-few-shot-prompting)
   - 2.3 [Chain-of-Thought (CoT)](#23-chain-of-thought-cot)
   - 2.4 [ReAct (Reason + Act)](#24-react-reason--act)
   - 2.5 [Role-Based Prompting](#25-role-based-prompting)
   - 2.6 [Structured Task Decomposition](#26-structured-task-decomposition)
3. [Context-Building Rules](#3-context-building-rules)
4. [Output Formatting Patterns](#4-output-formatting-patterns)
5. [Constraint & Safety Patterns](#5-constraint--safety-patterns)
6. [Iterative Refinement Strategies](#6-iterative-refinement-strategies)
7. [Anti-Patterns](#7-anti-patterns)
8. [Evaluation Checklist](#8-evaluation-checklist)

---

## 1. Prompt Anatomy

Every high-quality prompt is composed of distinct layers. Not every layer is required for every task, but being intentional about each one dramatically improves output quality.

```
┌─────────────────────────────────────────────────────────┐
│  SYSTEM / ROLE LAYER                                    │
│  Who is the model? What persona, expertise, or          │
│  behavioral constraints govern its responses?           │
├─────────────────────────────────────────────────────────┤
│  CONTEXT LAYER                                          │
│  Background information the model needs to reason       │
│  accurately. Documents, facts, prior conversation,      │
│  or domain knowledge.                                   │
├─────────────────────────────────────────────────────────┤
│  INSTRUCTION LAYER                                      │
│  The explicit task. What should the model do?           │
│  Clear, imperative sentences work best.                 │
├─────────────────────────────────────────────────────────┤
│  CONSTRAINT LAYER                                       │
│  Guardrails on content, tone, length, format,           │
│  or topics the model must avoid.                        │
├─────────────────────────────────────────────────────────┤
│  OUTPUT FORMAT LAYER                                    │
│  The exact structure expected: JSON, Markdown,          │
│  numbered list, table, prose paragraph, etc.            │
└─────────────────────────────────────────────────────────┘
```

### Layer Ordering Guidance

- Place the **role/system** instruction at the very top (or in the system message field if the API supports it) so it frames all subsequent content.
- Place **context** immediately after the role — before the instruction — so the model has background before it sees the task.
- Place the **instruction** after context so the model can apply what it just read.
- Place **constraints and format** at the end of the instruction, or in a clearly labelled section, so they are the freshest information when the model begins generating.

---

## 2. Structural Frameworks

### 2.1 Zero-Shot Prompting

**When to use:** The task is simple, well-defined, and the model is likely to have seen many similar examples during training.

**Template:**

```
[ROLE]
You are a [domain] expert.

[INSTRUCTION]
[Single, clearly worded task sentence.]

[CONSTRAINTS]
- [Constraint 1]
- [Constraint 2]

[OUTPUT FORMAT]
Respond in [format].
```

**Example:**

```
You are a senior software engineer specializing in Python.

Explain what a Python decorator is and when you would use one.

- Use plain language suitable for a developer with 6 months of experience.
- Keep the response under 200 words.

Respond in two short paragraphs.
```

**Strengths:** Low token cost, fast iteration.  
**Weaknesses:** Can fail on complex reasoning tasks that benefit from worked examples.

---

### 2.2 Few-Shot Prompting

**When to use:** The desired output style, tone, or format is difficult to describe but easy to demonstrate. Especially effective for classification, extraction, and transformation tasks.

**Template:**

```
[ROLE]
You are a [domain] assistant.

[INSTRUCTION]
[Task description.]

[EXAMPLES]
Input: [example input 1]
Output: [example output 1]

Input: [example input 2]
Output: [example output 2]

[TASK]
Input: [actual input]
Output:
```

**Guidelines for writing good examples:**

1. **Diversity** — examples should cover different cases, not just variations of the same one.
2. **Consistency** — all examples must follow exactly the same input/output format.
3. **Relevance** — examples should be drawn from the same domain and complexity level as the actual task.
4. **Number** — 2–5 examples is usually optimal; more than 8 rarely improves performance and wastes tokens.

**Example (sentiment classification):**

```
You are a sentiment analysis assistant.

Classify the sentiment of each customer review as Positive, Negative, or Neutral.

Input: "The shipping was fast and the product works perfectly."
Output: Positive

Input: "Arrived broken and customer support was unhelpful."
Output: Negative

Input: "The package arrived on time."
Output: Neutral

Input: "I love how lightweight this laptop is, but the battery life is disappointing."
Output:
```

---

### 2.3 Chain-of-Thought (CoT)

**When to use:** Tasks involving multi-step reasoning, math, logic, planning, or any problem where the intermediate steps matter as much as the final answer.

**Template:**

```
[ROLE]
You are a [domain] expert.

[INSTRUCTION]
[Task description.]

[COT TRIGGER]
Think through the problem step by step before giving your final answer.

[TASK]
[Actual input / question]
```

**Variants:**

| Variant | How to trigger it |
|---|---|
| **Standard CoT** | "Think step by step." |
| **Zero-shot CoT** | "Let's think through this carefully." |
| **Self-consistency CoT** | Ask the same question N times and aggregate the majority answer. |
| **Least-to-most** | "First break this problem into sub-problems, then solve each in order." |

**Example:**

```
You are a financial analyst.

A company had revenue of $4.2M in Q1 and $3.8M in Q2. Operating costs were 60% of revenue each quarter. Calculate the total operating profit for H1.

Think through the problem step by step before giving your final answer.
```

**Why it works:** CoT forces the model to generate intermediate reasoning tokens, which reduces the chance of the final answer being a low-probability shortcut.

---

### 2.4 ReAct (Reason + Act)

**When to use:** Agentic workflows where the model must interleave reasoning with tool calls (search, code execution, API requests).

**Template:**

```
[ROLE]
You are an agent that can use the following tools: [tool list].

[INSTRUCTION]
Answer the user's question by reasoning about what information you need,
calling the appropriate tool, observing the result, and continuing until
you can give a complete answer.

[FORMAT]
Thought: [reasoning step]
Action: [tool name]
Action Input: [tool input]
Observation: [tool output — filled in by the system]
... (repeat as needed)
Final Answer: [answer to the original question]

[TASK]
[User question]
```

**Key principles:**
- Each `Thought` step should state *why* the next action is needed.
- The model should stop and provide a `Final Answer` only when it has sufficient information.
- Keep tool descriptions concise and unambiguous to reduce mis-routing.

---

### 2.5 Role-Based Prompting

**When to use:** Tasks that benefit from a specific perspective, expertise level, or communication style (technical writing, teaching, code review, legal summarization, etc.).

**Guidelines:**

1. **Be specific about expertise level.** "You are a senior database administrator with 10 years of PostgreSQL experience" produces more targeted output than "You are a database expert."
2. **Define the audience.** "Explain this to a non-technical executive" shapes vocabulary, analogies, and level of detail.
3. **State behavioral norms.** "You are direct, concise, and never use filler phrases like 'Great question!'" helps control tone.
4. **Avoid conflicting roles.** Don't instruct the model to be both "a cautious lawyer" and "a creative risk-taker" in the same prompt.

**Anti-pattern:** Vague role assignment

```
❌  You are a helpful assistant. Answer my question.
```

**Good practice:** Specific role with audience and behavioral norms

```
✅  You are a senior DevOps engineer. The person asking is a junior developer
    who has just started using Kubernetes. Explain concepts without assuming
    prior knowledge of container orchestration. Be thorough but avoid jargon.
```

---

### 2.6 Structured Task Decomposition

**When to use:** Long, multi-part tasks where a single prompt would be too complex and prone to partial completion or drift.

**Pattern:** Break the overall goal into a sequence of focused sub-prompts, where each prompt takes the output of the previous one as part of its context.

```
Prompt 1 → [outline]
Prompt 2 → [draft section 1, given outline]
Prompt 3 → [draft section 2, given outline]
Prompt 4 → [review and edit full draft]
```

**Benefits:**
- Easier to debug: when something goes wrong, you can identify which step failed.
- Outputs are verifiable at each stage before proceeding.
- Reduces the risk of the model losing track of requirements in a very long single prompt.

---

## 3. Context-Building Rules

Context is everything the model reads before it generates an answer. The following rules govern how to construct context that is both efficient and effective.

### Rule 1 — Include Only What the Model Needs

Every sentence of context costs tokens and competes for the model's attention. Ask: *"Would removing this sentence change the model's answer?"* If no, remove it.

**Before:**
```
Our company was founded in 2010. We have 500 employees across 12 countries.
We primarily sell B2B SaaS products. Last year we had $50M in ARR.
Please summarize the following customer support ticket: [ticket text]
```

**After:**
```
You are a customer support specialist. Summarize the following support ticket
in two sentences, focusing on the customer's core issue and desired resolution.

[ticket text]
```

### Rule 2 — Put High-Priority Information First (and Last)

Research on transformer attention suggests that models pay more attention to content at the beginning and end of a prompt (primacy and recency effects). Place the most critical constraints and instructions in these positions.

```
[Critical constraint at top]
[Background context in the middle]
[Critical instruction / output format at bottom — nearest to generation]
```

### Rule 3 — Separate Context from Instruction Clearly

Use delimiters to help the model distinguish between reference material and the actual task. Ambiguity here is a leading cause of the model incorporating source text verbatim instead of reasoning about it.

**Good delimiters:**
- Triple backticks ` ``` `
- XML-style tags: `<document>...</document>`, `<instruction>...</instruction>`
- Clear headers: `### Background ###`, `### Task ###`

**Example:**

```
### Background ###
<document>
[Pasted article text here]
</document>

### Task ###
Based solely on the document above, answer the following question in one paragraph:
What are the three main causes of the trend described?
```

### Rule 4 — State What the Model Does Not Know

When providing partial context, explicitly tell the model what information is missing so it does not hallucinate missing details.

```
✅  The following transcript is a partial excerpt. You do not have access to
    the earlier parts of the conversation. Base your analysis only on what
    is provided below.
```

### Rule 5 — Use Grounding Anchors for Factual Tasks

For tasks requiring factual accuracy, anchor the model to the provided source and prohibit independent knowledge retrieval.

```
✅  Answer the question using ONLY the information in the document below.
    If the answer is not contained in the document, say "The document does
    not contain enough information to answer this question."
```

### Rule 6 — Manage Token Budget Proactively

When context is long, summarize or truncate before injecting. For retrieval-augmented generation (RAG) workflows:

1. Retrieve the top-k chunks ranked by semantic similarity.
2. Order chunks by relevance (most relevant closest to the instruction).
3. Trim chunks that push the total context beyond ~75% of the model's context window to leave room for the completion.

### Rule 7 — Version Your Context Templates

When a prompt is used repeatedly in production, treat it like code:
- Store it in version control.
- Document what each section does.
- Record which model version it was tuned for.

---

## 4. Output Formatting Patterns

Explicit format instructions dramatically reduce post-processing effort and make outputs machine-readable.

### 4.1 Structured Data (JSON / YAML)

```
Respond with a valid JSON object. Do not include any text outside the JSON block.

{
  "summary": "<one-sentence summary>",
  "key_points": ["<point 1>", "<point 2>", "<point 3>"],
  "sentiment": "positive | negative | neutral",
  "confidence": <float between 0 and 1>
}
```

**Tips:**
- Provide a schema or example object so the model knows field names and types.
- Instruct the model to output *only* the JSON (no preamble) if downstream parsing is automated.
- For complex schemas, use JSON Schema notation in the prompt.

### 4.2 Markdown

```
Format your response as a Markdown document with:
- A level-2 heading (##) for each main section
- Bullet points for lists of items
- Bold text (**...**) for key terms
- A code block (```python) for any code snippets
```

### 4.3 Tables

```
Present your findings in a Markdown table with the following columns:
| Feature | Benefit | Trade-off |
```

### 4.4 Prose with Length Control

```
Write three paragraphs, each between 60 and 80 words.
```

Or:

```
Respond in no more than 150 words. Do not use bullet points.
```

### 4.5 Step-by-Step Instructions

```
Format the instructions as a numbered list. Each step should:
- Begin with an imperative verb (e.g., "Open", "Click", "Enter").
- Be a single action — do not combine multiple actions in one step.
- Be no longer than two sentences.
```

---

## 5. Constraint & Safety Patterns

Constraints prevent the model from drifting into undesired territory. Write constraints as explicit prohibitions or explicit permissions — do not rely on implicit assumptions.

### 5.1 Topic Constraints

```
Answer only questions related to [topic]. If the user asks about anything
outside [topic], respond: "I can only help with [topic]-related questions."
```

### 5.2 Tone and Style Constraints

```
- Do not use technical jargon. Write for a general audience.
- Do not use first person ("I", "we").
- Do not speculate. If uncertain, say so explicitly.
- Do not use filler phrases such as "Certainly!", "Of course!", or "Great question!".
```

### 5.3 Source Constraints (Anti-Hallucination)

```
Base your response exclusively on the provided document. Do not use any
information from your training data. If the answer cannot be found in the
document, state that explicitly.
```

### 5.4 Length and Scope Constraints

```
- Your response must be between 100 and 200 words.
- Include exactly three recommendations, no more, no fewer.
- Do not summarize the entire document — focus only on the financial data.
```

### 5.5 Format Constraints

```
Do not use bullet points or numbered lists. Respond in continuous prose only.
```

### 5.6 Layering Constraints Effectively

Order constraints from most critical to least critical. When there are many constraints, group them by category:

```
CONTENT CONSTRAINTS:
- Only discuss topics related to data privacy.
- Do not provide legal advice.

TONE CONSTRAINTS:
- Use a formal, professional tone.
- Do not use humor.

FORMAT CONSTRAINTS:
- Respond in plain text (no Markdown).
- Limit to 3 sentences.
```

---

## 6. Iterative Refinement Strategies

No prompt is perfect on the first attempt. Use these strategies to improve a prompt systematically.

### 6.1 The Delta Method

Run the same prompt on 5–10 representative inputs. Note which outputs are wrong or low quality. Change exactly **one thing** per iteration and re-run. This isolates the variable that is causing failures.

### 6.2 Targeted Follow-Up Prompts

After an initial response, use a follow-up prompt to correct or expand rather than rewriting the whole prompt from scratch:

```
Your previous answer listed three benefits but did not explain the third one.
Expand on benefit three in two to three sentences, keeping the same formal tone.
```

### 6.3 Self-Critique and Revision

Ask the model to critique its own output before finalizing:

```
Review your response above. Check for:
1. Factual accuracy based only on the provided document.
2. Adherence to the requested format.
3. Any claims that are vague or unsupported.

If you find issues, revise the response. If not, confirm it is correct.
```

### 6.4 Prompt Regression Testing

Maintain a set of golden input/output pairs (test cases) for any prompt used in production. When modifying the prompt, run all test cases and verify that previously correct outputs are not degraded.

---

## 7. Anti-Patterns

The following patterns consistently produce poor results. Avoid them.

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **Vague instructions** — "Write something about X." | Model has too many degrees of freedom; output is generic. | Specify task, audience, length, and format explicitly. |
| **Negative-only constraints** — "Don't be too technical." | Models respond better to positive instructions. | State what you *do* want: "Use plain language suitable for a high-school student." |
| **Overloaded single prompt** — Asking the model to do 5 unrelated tasks in one prompt. | Model prioritizes some tasks over others; outputs are uneven. | Split into separate prompts. |
| **Contradictory instructions** — "Be concise" and "cover every detail." | Model is forced to choose; result is unpredictable. | Decide which constraint takes priority. |
| **Implicit format expectations** — No format instruction, but expecting JSON. | Model returns prose. | Always state the output format explicitly. |
| **Prompt injection via unvalidated user input** — Inserting raw user text into a system prompt without sanitization. | User can override system instructions. | Clearly delimit user content and validate inputs before injection. |
| **Relying on implicit knowledge** — "Use the standard approach." | "Standard" is ambiguous; model may choose inappropriately. | Describe the approach explicitly or provide an example. |
| **Missing stop conditions** — Asking for "a list" without specifying how many items. | Model produces variable-length outputs that are hard to process. | Always specify an exact count or a bounded range. |
| **Role without audience** — Assigning a role but not specifying who the model is speaking to. | Response calibration is off (too technical, too simple). | Define both the role and the audience. |
| **Long context with no anchoring** — Pasting a document and asking a question without directing the model to use the document. | Model may ignore the document and use training knowledge instead. | Explicitly anchor: "Based solely on the document above…" |

---

## 8. Evaluation Checklist

Use this checklist before deploying any prompt to production or sharing it as a reference.

### Clarity

- [ ] The instruction is a clear, imperative statement of what the model should do.
- [ ] There are no ambiguous pronouns or vague qualifiers ("it", "some", "appropriate").
- [ ] Each constraint is stated as a specific, testable rule.

### Context

- [ ] All background information the model needs is present.
- [ ] Information the model does not have access to is explicitly flagged.
- [ ] Context is separated from the instruction using clear delimiters.
- [ ] No irrelevant background information is included.

### Format

- [ ] The expected output format is explicitly stated.
- [ ] Length or scope boundaries are defined (word count, number of items, etc.).
- [ ] If structured output (JSON, YAML, table) is needed, a schema or example is provided.

### Constraints

- [ ] Constraints are stated as positive instructions where possible.
- [ ] No two constraints contradict each other.
- [ ] Safety / grounding constraints are present for factual tasks.

### Robustness

- [ ] The prompt has been tested on at least 3–5 diverse inputs.
- [ ] Edge cases (empty input, very short input, off-topic input) have been tested.
- [ ] A regression test set exists if the prompt is used in production.

### Efficiency

- [ ] Token count is as low as possible while still meeting quality requirements.
- [ ] Redundant or duplicated instructions have been removed.
- [ ] Context has been trimmed to the minimum necessary.

---

*This document is a living reference. Contributions and improvements are welcome — see [`README.md`](./README.md) for contribution guidelines.*
