---
name: code-to-doc-generator
description: |
  Generates onboarding-friendly internal documentation drafts from code.
  Supports AI Agent, ML Model, Data Pipeline, and utility function modules written in Python or Java.
  Activate when user pastes code and asks for documentation, module explanation,
  or onboarding notes.
triggers:
  - "generate doc"
  - "document this"
  - "write documentation"
---

## Step 1 — Clarify inputs

Before generating, check if the user has provided:

1. **Module type**: Agent / ML Model / Pipeline / Function
   - If not stated, infer from code. If still uncertain, ask.
2. **Business purpose** (1–2 sentences): What problem does this solve?
   - If not stated, ask: "What problem does this solve?"

**If the user provides a GitHub URL instead of pasting code:**
- Fetch the raw URL to get clean code text.
- If multiple files are needed, ask the user to provide each file's URL one by one.
- Private repos cannot be accessed — inform the user if fetch fails.

If the user says "just generate", skip asking and mark uncertain fields with `[TODO: verify]`.

---


## Step 2 — Generate draft

All templates share this opening:

```
# [Module Name]
[give GitHub url if user provided]

## What it does
[1–2 sentences. What is the outcome of using this module? Start with the business purpose, not the implementation.]

## Inputs
| Name | Type | Description |
|------|------|-------------|
| ...  | ...  | ...         |

## Outputs / Returns
[What does this produce? Describe format and how to interpret the result.]

## Dependencies
- ...
```

Then append the type-specific section below.

---

### Agent (append after shared sections)

```
## When it runs
[What event or condition triggers this agent?]

## Decision logic
[Key steps in order.]
1. ...
2. ...
```

---

### ML Model (append after shared sections)

```
## Interpreting results
[Base model, Parameter setting, Thresholds, confidence scores, label meanings — how should the caller act on the output?]

## Performance considerations
[Latency, batch size, memory constraints.]
[TODO: verify if not in code]

## Evaluation

## Decision Logic
[Order key steps to explain how the output generated.]
```

---

### Data Pipeline (append after shared sections)

```
## How it's triggered
[Scheduled / event-driven / manual?]

## Data flow
[Source] → [Step 1] → [Step 2] → [Destination]

## Transformation steps
1. ...
2. ...

## Error handling
[Retries, alerts, dead-letter behaviour.]
[TODO: verify if not in code]
```

---

### Utility Function (append after shared sections)

```
## When to use it
[Concrete scenario — when would a developer reach for this?]

## How to use it
[Explain the decision logic in order]

```

## Edge cases and gotchas
- ...
```

---

## Step 3 — Output rules

- Target reader: new technical team member — smart but unfamiliar with this codebase
- Always write *what* before *how*. Never open a section with "this function calls X"
- Mark every uncertain field with `[TODO: verify]`
- Write in English unless the user writes in Chinese, then write in Chinese
- Output as a md file
- End every draft with:

```
---
Draft generated from code only. Please verify: [list all TODO items]
```