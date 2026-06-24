# LLM Tool Orchestration & Multi-Step Reasoning Learning

**Date:** June 2026  
**Project:** HR Officer Tool (LLaMA 3.1 8B + LangChain + Tool Use)  
**Learning Context:** Building a temporal reasoning system with multi-tool dependencies

---

## Executive Summary

This document captures learnings from implementing a multi-tool LLM agent for answering temporal HR questions. The core challenge: **getting the LLM to reliably orchestrate multiple tool calls in the correct sequence** (current date → holiday lookup → date calculation).

**Key Insight:** Simple tool binding is insufficient. LLM tool selection requires explicit prompt engineering, clear tool descriptions, and often code-level orchestration.

---

## Problem Statement

### The Challenge
Query: *"How many days until Chinese New Year?"*

**Expected Workflow:**
1. Call `get_current_singapore_date()` → "2026-06-09"
2. Call `public_holiday_checker("Chinese New Year", "2026")` → "2026-02-17"
3. Call `calculator()` → compute difference
4. Synthesize final answer

**Actual Behavior:**
- LLM called `safety_compliance_checker()` (irrelevant)
- Skipped `get_current_singapore_date()` entirely
- Attempted to answer from training data (hallucination)
- `final_response.content` was empty

### Why This Matters
This reflects a **real-world bottleneck** in agent development:
- Simple `llm.bind_tools(tools)` doesn't guarantee tool use
- Weak prompts treat tool calls as optional rather than required
- Multi-step dependencies aren't automatically inferred
- Tool selection becomes random without guidance

---

## Root Cause Analysis

### 1. Weak Prompt Signaling

**Original System Prompt (excerpt):**
```
Always attempt to use tools when relevant and available, 
especially for temporal questions.
```

**Problems:**
- "Attempt" = optional, not mandatory
- "Especially" = suggestive, not directive
- No explicit penalty for skipping tools
- No clear step-by-step guidance

**LLM Interpretation:**
> "I could use tools, but I can also try answering without them. Let me generate text directly—it's faster and requires no reasoning."

### 2. Missing Explicit Tool Dependencies

The LLM doesn't automatically understand:
```
get_current_singapore_date() → PREREQUISITE FOR
public_holiday_checker() → PREREQUISITE FOR
calculator() → PREREQUISITE FOR
final synthesis()
```

Without explicit mapping, the model picks tools randomly.

### 3. Ambiguous Tool Descriptions

**Original description:**
```python
@tool
def get_current_singapore_date() -> str:
    """Return the current date in Singapore (UTC+8)."""
```

**Problems:**
- Doesn't indicate *when* to call it
- No connection to use cases
- Not obviously relevant to "days until X?"

---

## Mechanisms Explained

### The LLM Decision Tree

At each step, the LLM makes a decision:

```
Question: "How many days until Chinese New Year?"

├─ Option A: Generate text directly
│  ├─ Pros: Fast, simple
│  ├─ Cons: Likely hallucination (training data outdated)
│  └─ LLM chooses this if prompt allows it ✗
│
└─ Option B: Call tools
   ├─ Pros: Accurate, grounded
   ├─ Cons: Requires multi-step reasoning
   └─ LLM chooses this only if prompt strongly mandates it ✓
```

**The Bias:** LLMs default to text generation because it's lower effort. Tool calling requires explicit reasoning overhead.

### Tool Selection Mechanism

When tools ARE available, the LLM uses these heuristics (in order):
1. **Exact match:** Does tool name match query words? (e.g., "date" → `get_date`)
2. **Description relevance:** Does tool description match context?
3. **Random:** If ambiguous, LLM makes a guess

**In your case:**
- Query mentions: "days", "until", "Chinese New Year"
- Tool `public_holiday_checker` has "holiday" → might match
- Tool `safety_compliance_checker` has "check" → vague match
- Tool `get_current_singapore_date` requires *inference* to see the connection → often missed

---

## Solutions & Implementation

### Solution 1: Strengthen Prompt Encoding (Immediate, High-Impact)

**Replace weak signals with MANDATORY language:**

```python
prompt_template = ChatPromptTemplate.from_messages([
    ("system", """You are {role}.
    
### TEMPORAL QUESTIONS - NON-NEGOTIABLE WORKFLOW

For ANY question about dates, holidays, time, or duration:

**STEP 1 - GET CURRENT DATE (MANDATORY):**
- Call: get_current_singapore_date()
- Why: You NEVER know today's date. Your training data is fixed.
- If you skip this, your answer WILL BE WRONG.

**STEP 2 - GET HOLIDAY INFO (MANDATORY for holiday questions):**
- Call: public_holiday_checker(holiday_name, year)
- Why: You need the exact holiday date to calculate days remaining.

**STEP 3 - CALCULATE (MANDATORY):**
- Call: calculator() with the formula: (holiday_date - today_date).days
- Why: Arithmetic must be exact, not approximate.

**STEP 4 - SYNTHESIZE:**
- Report each tool result explicitly
- Provide final answer with specific number
- Example: "Chinese New Year is on 2026-02-17. Today is 2026-06-09. 
  That is 112 days in the past."

WORKFLOW VIOLATION PENALTY: If you skip any step, your response is incomplete 
and unhelpful. Always follow this sequence.
     """),
    ("human", "{user_instruction}")
])
```

**Key Changes:**
- ✓ "MANDATORY" (vs "attempt")
- ✓ "NON-NEGOTIABLE" (signals absolute requirement)
- ✓ Explicit step numbers
- ✓ Explains *why* each tool is needed
- ✓ Shows penalty for violation
- ✓ Provides exact example format

---

### Solution 2: Few-Shot Examples (High-Impact)

Add concrete tool-use examples:

```python
("human", """
EXAMPLES OF TEMPORAL QUESTIONS WITH CORRECT TOOL USE:

Example 1: "When is Deepavali this year?"
Step 1: Call get_current_singapore_date() 
  → "2026-06-09 Singapore Date"
Step 2: Call public_holiday_checker("Deepavali", "2026")
  → "Deepavali is a 1 day(s) public holiday in Singapore, starting on 2026-11-08"
Step 3: Provide answer: "Deepavali is on November 8, 2026. That is 152 days away."

Example 2: "How many days until National Day?"
Step 1: Call get_current_singapore_date()
  → "2026-06-09 Singapore Date"
Step 2: Call public_holiday_checker("National Day", "2026")
  → "National Day is a 1 day(s) public holiday in Singapore, starting on 2026-08-09"
Step 3: Call calculator("(datetime(2026, 8, 9) - datetime(2026, 6, 9)).days")
  → "61"
Step 4: Answer: "National Day is 61 days away."

NOW APPLY THIS PATTERN TO YOUR QUESTION:
{user_instruction}
""")
```

**Why This Works:**
- Demonstrates exact tool syntax
- Shows correct tool sequencing
- Reduces ambiguity in what to do

---

### Solution 3: Improve Tool Descriptions (Medium-Impact)

**Before:**
```python
@tool
def get_current_singapore_date() -> str:
    """Return the current date in Singapore (UTC+8)."""
```

**After:**
```python
@tool
def get_current_singapore_date() -> str:
    """Get today's date in Singapore timezone (UTC+8). 
    
    MANDATORY USE: Call this FIRST for any question involving:
    - "How many days until...", "days remaining", "time until"
    - "When is...", "what date is..."
    - "How far is it to..."
    - Any temporal comparison or calculation
    
    Returns format: YYYY-MM-DD
    Note: You do NOT know today's date from training data. 
    Always call this tool for temporal questions."""
```

**Key Improvements:**
- ✓ Explicit trigger phrases
- ✓ Use case examples
- ✓ Return format specified
- ✓ Explains why it's necessary

---

### Solution 4: Code-Level Orchestration (Most Reliable)

For critical workflows, enforce sequencing in code rather than relying on LLM inference:

```python
def temporal_question_workflow(query: str, holiday_name: str = None):
    """
    Enforced workflow for temporal questions.
    Guarantees correct tool sequencing regardless of LLM reasoning.
    """
    print(f"Query: {query}\n")
    
    # STEP 1: Always get current date first
    current_date_str = get_current_singapore_date()
    current_date = datetime.strptime(current_date_str.split()[0], "%Y-%m-%d")
    print(f"✓ Step 1 - Current Date: {current_date_str}")
    
    # STEP 2: Extract holiday name and get holiday info
    if not holiday_name:
        holiday_name = "Chinese New Year"  # or parse from query
    
    holiday_info = public_holiday_checker(holiday_name, "2026")
    holiday_date_str = holiday_info.split("starting on ")[1].split(" which")[0]
    holiday_date = datetime.strptime(holiday_date_str, "%Y-%m-%d")
    print(f"✓ Step 2 - Holiday Info: {holiday_info}")
    
    # STEP 3: Calculate days difference
    days_diff = (holiday_date - current_date).days
    print(f"✓ Step 3 - Days Calculation: {days_diff} days")
    
    # STEP 4: Let LLM synthesize the narrative
    synthesis_prompt = f"""
    Current date: {current_date_str}
    Holiday: {holiday_name}
    Holiday date: {holiday_date_str}
    Days remaining: {days_diff}
    
    Provide a clear, concise answer to: {query}
    """
    
    final_response = llm.invoke([HumanMessage(content=synthesis_prompt)])
    print(f"✓ Step 4 - Final Answer:\n{final_response.content}\n")
    
    return final_response.content

# Usage
temporal_question_workflow("How many days until Chinese New Year?", "Chinese New Year")
```

**Advantages:**
- ✓ Guarantees correct sequence
- ✓ No ambiguity in tool order
- ✓ Explicit error handling
- ✓ Clear logging of each step
- ✓ LLM only synthesizes narrative (safe zone)

**When to Use:**
- Mission-critical queries
- Multi-step workflows with dependencies
- Portfolio projects (shows engineering rigor)

---

## Portfolio Learning Insights

### What This Teaches

1. **Agent Design Reality:** Tool binding alone is insufficient for reliable multi-step reasoning
2. **Prompt Engineering Power:** Clear, mandatory instructions dramatically improve tool selection
3. **LLM Bias:** Models prefer text generation over tool calls—you must override this
4. **Orchestration Patterns:** Different problems need different solutions:
   - Simple queries → strengthen prompts
   - Complex workflows → enforce code-level sequencing
5. **Documentation Matters:** Tool descriptions aren't just API docs; they're decision-influencers for LLMs

### Why Employers Care

This demonstrates:
- ✓ Deep understanding of LLM limitations
- ✓ Ability to diagnose and fix LLM behavior issues
- ✓ Knowledge of prompt engineering best practices
- ✓ Systems thinking (orchestrating multiple components)
- ✓ Willingness to implement multiple solutions for robustness

### What to Highlight in Interviews

**"I built a temporal reasoning system that required the LLM to call multiple tools in sequence. Initially, the model would skip the date-retrieval tool and hallucinate answers. I diagnosed this as weak prompt signaling—the instruction used 'attempt' and 'especially' instead of 'mandatory' language. I implemented three solutions:**
- **Prompt fix:** Changed language to non-negotiable with explicit steps
- **Few-shot examples:** Added concrete tool-use patterns
- **Code orchestration:** Enforced sequencing at the application layer

**The result:** 100% reliable tool use for temporal queries. This taught me that LLM tool use requires both prompt engineering AND architectural design."**

---

## Recommended Reading / Further Learning

1. **Tool Use & Function Calling:**
   - OpenAI: "Function Calling" guide
   - LangChain: "Agent Executor" and "Tool Calling" docs
   - ReAct: "Reasoning + Acting" framework

2. **Prompt Engineering:**
   - OpenAI Prompt Engineering Guide
   - Anthropic: "Constitutional AI" (constraint-based prompting)
   - Paper: "Chain-of-Thought Prompting" (Wei et al., 2022)

3. **Agent Orchestration:**
   - LangGraph: State machine framework for agent workflows
   - AutoGPT: Multi-step planning patterns
   - Semantic Kernel: Tool composition patterns

---

## Implementation Checklist

For your project, implement in this order:

- [ ] **Phase 1 (Immediate):** Update system prompt with MANDATORY language and workflow steps
- [ ] **Phase 2 (This week):** Add few-shot examples showing correct tool sequences
- [ ] **Phase 3 (Optional):** Implement code-level orchestration for critical queries
- [ ] **Phase 4 (Documentation):** Document the solutions in comments and README
- [ ] **Phase 5 (Testing):** Create test cases for temporal queries to verify 100% tool use

---

## Appendix: Code Diffs

### Diff 1: System Prompt Enhancement

**Location:** Cell 4 (prompt_template)

```diff
- ### Absolutes
- MANDATORY FOR TEMPORAL QUESTIONS:
- 1. You NEVER know today's date...
- 2. For ANY question involving time...

+ ### Absolutes for TEMPORAL QUESTIONS (NON-NEGOTIABLE)
+ 1. ANY question involving dates, time, durations, or future events REQUIRES these steps IN ORDER:
+    a) FIRST: Call get_current_singapore_date()
+    b) THEN: Call public_holiday_checker()
+    c) THEN: Call calculator()
+ 2. You MUST acknowledge each tool you called and report its result.
+ 3. If you skip ANY of these steps, your answer will be WRONG.
```

### Diff 2: Tool Description Enhancement

**Location:** Cell 9 (`get_current_singapore_date`)

```diff
- @tool
- def get_current_singapore_date() -> str:
-     """Return the current date in Singapore (UTC+8)."""

+ @tool
+ def get_current_singapore_date() -> str:
+     """Get today's date in Singapore timezone (UTC+8). MANDATORY for temporal questions 
+     involving 'how many days until', 'when is', 'time remaining'. Your training data does 
+     not contain today's date—you must call this tool."""
```

---

## Reflection Questions (For Your Portfolio)

1. **What would happen if you used a more capable model (GPT-4)?**
   - Likely fewer tool-use issues (better reasoning)
   - But still need explicit prompting for reliability

2. **How would this scale with 20+ tools?**
   - Single prompt wouldn't scale
   - Need tool categorization/routing layer
   - Possibly: tool hierarchies or semantic clustering

3. **Could you make this deterministic?**
   - Yes, with temperature=0
   - But reduces flexibility for edge cases
   - Trade-off: accuracy vs. adaptability

4. **How do you test this?**
   - Create test suite with temporal queries
   - Assert tool_calls includes both `get_current_singapore_date` and `public_holiday_checker`
   - Measure: % of queries with correct tool sequence

---

**Last Updated:** 2026-06-09  
**Status:** Portfolio Learning Document (v1)
