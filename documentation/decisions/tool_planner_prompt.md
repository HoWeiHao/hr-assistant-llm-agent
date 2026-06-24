# ADR-002: Tool Planner Prompt Design

## Status

Accepted

---

# Context

This project implements an iterative agent architecture instead of allowing the language model to answer directly.

Execution flow:

1. User
2. Planner LLM
3. Tool Execution Loop
4. Shared State
5. Final Answer LLM
6. User

The planner LLM has only one responsibility:

> Decide the next action.

It does not answer questions, perform calculations, or simulate tool outputs. It reasons only over the user query and existing tool state, and decides either:

- One next tool call
- No further tool calls

This prompt evolved through debugging multiple failure modes discovered during development.

---

# Prompt Sections and Their Purpose

## 1. Restricting the Planner Role

```text
You are a tool planner.

Your ONLY job is to select which tools to call.
```

### Problem Prevented

The model attempts to answer questions directly instead of using tools.

Example:

```text
User: How many days until Christmas?

LLM:
Christmas is on Dec 25, so about 180 days away.
```

No tools are used.

### Solution

Separate responsibilities:

1. Planner → chooses actions.
2. Executor → runs tools.
3. Final LLM → answers.

---

## 2. Preventing Simulated Tool Outputs

```text
You MUST NOT:
- answer the question
- explain reasoning
- simulate results
- execute tools
```

### Problem Prevented

The model hallucinates information that should come from tools.

Examples:

```text
Current year = 2026
```

or

```text
Christmas is 191 days away.
```

without actually calling tools.

### Solution

Strictly separate planning from execution.

---

## 3. Forcing One-Step Iterative Planning

```text
Based on user query and past tool calls, you are ONLY allowed to choose ONE best next action.

Return exactly one tool call OR final answer.

Do not plan future steps.
```

### Problem Prevented

Early versions produced:

```text
1. get_current_date()
2. public_holiday_checker()
3. date_difference()
```

all at once.

This prevented later tool outputs from influencing subsequent decisions.

### Solution

Force ReAct-style behavior:

1. Think.
2. Execute one tool.
3. Observe.
4. Execute next tool.
5. Observe.
6. Finish.

This allows placeholder resolution and state-dependent reasoning.

---

## 4. Reusing Existing Tool Results

```text
If required information is already present in tool results, return NO tool calls.
```

### Problem Prevented

Infinite loops.

Observed behavior:

```text
holiday_checker(Christmas,2026)

holiday_checker(Christmas,2026)

holiday_checker(Christmas,2026)
```

repeated forever.

### Solution

Treat state as authoritative memory.

Already-known facts should be reused rather than rediscovered.

---

## 5. Avoiding Unnecessary Current-Date Calls

```text
If the query depends on the CURRENT date, obtain current date first using tool.

If the query specifies an explicit year/date, do not call current date tools unless required.
```

### Problem Prevented

The planner wasted a step on current date even when the user already specified the year.

Example:

```text
Was Christmas a public holiday in 2023?
```

Unwanted sequence:

1. get_current_date()
2. holiday_checker(2023)

### Solution

Distinguish between:

Absolute queries:

```text
2023
December 25
Last year
```

and relative queries:

```text
Today
This year
How many days left
Next Christmas
```

Only relative queries require current-date tools.

---

## 6. Preventing Numeric Hallucinations

```text
YOU ARE STRICTLY FORBIDDEN FROM GUESSING OR INFERRING:
- years
- dates
- current time
- holidays
- durations
- numeric values
```

### Problem Prevented

Qwen occasionally guessed years and dates without tool usage.

Examples:

```text
2026
Christmas duration
Current date
```

### Solution

Numeric facts are unknown unless provided by:

- User input.
- Tool outputs.

---

## 7. Forcing Arithmetic into Tools

```text
You MUST use tools for:
- arithmetic
- date differences
- duration calculations

Never perform calculations yourself.
```

### Problem Prevented

LLMs perform unreliable arithmetic.

Example:

```text
June to December is about four months.
```

instead of calculating exact days.

### Solution

Use deterministic tools.

Principle:

> LLMs reason. Tools calculate.

---

## 8. Placeholder Mechanism

```text
If values are unknown:

$current_year
$current_date
$holiday_year
```

### Problem Prevented

Future values were guessed.

Example:

```text
holiday_checker(
    holiday="Christmas",
    year=2026
)
```

before the current year was established.

### Solution

Represent unknowns symbolically:

```text
holiday_checker(
    holiday="Christmas",
    year=$current_year
)
```

Then resolve them after:

1. get_current_date()
2. current year discovered.
3. placeholder substituted.
4. holiday checker executed.

---

## 9. Temporal Safety Rules

```text
MANDATORY FOR TEMPORAL QUESTIONS:
1. Use current date and time if not provided.
2. You NEVER know today's date.
3. Always establish current date first.
4. Never derive current date from document context.
```

### Problem Prevented

Model knowledge leakage.

The LLM sometimes inferred:

```text
Current year = 2026
```

from previous context or historical documents.

### Solution

Current time is runtime information and must originate from tools.

---

# Human Message

```python
("human",
"""
### User Instruction
{user_instruction}

### Tool State (authoritative facts only)
{state}

You MUST NOT infer missing values not present in this state.
"""
)
```

## Why State Is Included

State acts as working memory.

Example:

```python
{
    "current_year": "2026",
    "current_date": "2026-06-17",
    "holiday_info":
        "Christmas starts on 2026-12-25"
}
```

The planner reasons only over these facts.

### Problems Prevented

Repeated tool calls:

```text
get_current_date()
get_current_date()
```

Forgotten tool outputs:

1. get_current_date()
2. holiday_checker($current_year)

Hallucinated values that do not exist in state.

### Principle

State should contain facts, not interpretations.

Good:

```python
{
    "current_year": 2026
}
```

Bad:

```python
{
    "christmas_is_far_away": True
}
```

Interpretations belong to the final answer LLM.

---

# Architectural Philosophy

The planner behaves like a CPU scheduler.

It should never:

- Answer.
- Calculate.
- Summarize.

Its only responsibility is:

> What is the next action?

Tool outputs become the single source of truth.

Overall execution becomes:

1. User asks question.
2. Planner decides next tool.
3. Tool executes.
4. State is updated.
5. Planner decides again.
6. Final LLM synthesizes the answer.

This makes reasoning observable, execution deterministic, debugging easier, and hallucinations less likely, while preserving the benefits of a ReAct-style agent under manual control.