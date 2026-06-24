# Decision Document: Planner Architecture — Overview Planner vs Iterative Tool Loop Planner

## 1. Context of the Project

This project builds an LLM-powered system with:
- Tool use (date, holidays, HR policies, calculations, safety checks)
- RAG-based retrieval (ChromaDB + embeddings)
- A custom planner LLM that decides when and how tools should be used
- A final answer generation step that consumes tool outputs

The system evolved through multiple iterations:
1. Simple LLM-only answers  
2. RAG-only augmentation  
3. Tool-augmented LLM  
4. Hybrid system (RAG + tools + planner loop)

Two planner architectures emerged during experimentation.

---

## 2. Option A — Overview Planner (Single-Step Planning)

The planner produces a complete tool plan upfront.

Flow:
1. User Query → Planner
2. Planner outputs full tool plan (all required tools at once)
3. External executor runs all tools
4. Final LLM call consumes aggregated results

**Key characteristic:** planning is separated from execution.

---

## 3. Option B — Iterative Tool Loop Planner (Step-by-Step Execution)

The planner operates in a loop, selecting one tool at a time based on updated state.

Flow:
1. User Query → Planner
2. Planner selects ONE tool call
3. Tool executes
4. State updates
5. Planner is called again
6. Repeat until no tool call is returned

---

## 4. Observed Behaviour in This Project (Iterative Planner)

### 4.1 Tool repetition / looping issue

Observed during execution:

- The same tool was repeatedly called with identical arguments even after successful execution.

Example logs:
- `[public_holiday_checker(Christmas, 2026)]` repeated multiple times
- `[hr_policy_retriever(annual leave policy for full-time vs part-time staff)]` triggered repeatedly even after valid retrieval was completed

Even after adding mitigation strategies:
- `executed` set (signature-based deduplication)
- duplicate tool-call detection (`Skipping duplicate tool call: ...`)
- max step limit (`max_steps = 10`)

the planner still re-issued identical tool calls in some cases.

---

### 4.2 State not reliably respected by planner

Even when tool results were already stored in state:
- The planner still requested the same tool again
- Previously executed outputs were not consistently treated as authoritative

Observed pattern:
- Tool result exists in `state["memory"]`
- Planner ignores it and re-queries tool with same arguments

---

### 4.3 Excessive iterations despite complete information

Even when all required tool outputs were already available:
- The planner continued cycling through tool calls
- Required external termination conditions (`max_steps`, dedup checks) to stop execution

This indicates the planner does not reliably self-terminate.

---

### 4.4 Redundant tool calls caused by argument reformatting

In some runs:
- Same logical query produced slightly different argument ordering

Example observed:
- `{"holiday": "Christmas", "year": "2026"}`
- `{"year": "2026", "holiday": "Christmas"}`

This bypassed naive deduplication unless normalized.

---

### 4.5 Final answer sometimes weakly grounded in tool outputs

Even when tool outputs were correct:
- Final LLM response sometimes partially ignored structured results
- Reconstructed answers using general language patterns instead of strict grounding

---

## 5. Trade-off Analysis

### Option A — Overview Planner

**Strengths**
1. Single planning step reduces LLM calls
2. No risk of repeated tool loops
3. Deterministic execution flow
4. Easier debugging (linear trace)
5. Better grounding of final response (single aggregated context)

**Weaknesses**
1. Requires correct full plan upfront
2. Less adaptive if tool outputs change downstream reasoning

---

### Option B — Iterative Planner

**Strengths**
1. Adaptive reasoning based on intermediate tool outputs
2. Handles multi-step conditional logic
3. Flexible for dynamic workflows

**Weaknesses (observed directly in this project)**  
1. Tool repetition even after successful execution  
2. State not reliably respected by planner  
3. Requires external loop-breaking mechanisms (`max_steps`, deduplication)  
4. Argument reordering breaks naive dedup checks  
5. High latency due to repeated LLM invocations  
6. Final answer sometimes not strongly grounded in tool outputs  
7. Planner does not reliably self-terminate  

---

## 6. Key Insight From Experiments

The iterative planner behaves as if:
- It does not treat `state` as authoritative memory
- It re-evaluates tool needs from scratch each iteration
- It requires external enforcement to behave correctly

This results in:
- Engineering overhead shifting from LLM → control logic
- Increasing complexity in maintaining correctness

---

## 7. Decision Consideration

### When Iterative Planner is justified
1. Multi-step reasoning where each tool output changes next decision
2. Complex branching workflows
3. Research or exploratory pipelines

### When Overview Planner is preferred
1. Deterministic tools (date, holiday, calculators)
2. RAG retrieval + summarisation workflows
3. HR policy or document lookup systems
4. Systems requiring stable and predictable execution

---

## 8. Final Decision (For This Project)

**The Overview Planner is the preferred architecture.**

### Rationale:
1. Observed repeated tool call loops in iterative system
2. State is not consistently respected by planner
3. Deduplication logic is external and brittle
4. Latency increases significantly with iteration
5. Final answers are more stable when grounded in a single aggregated tool output stage

The iterative planner may still be retained for:
- experimental reasoning workflows
- future multi-agent or autonomous planning research

---

## 9. Summary

- Iterative planner: flexible but unstable in current implementation
- Overview planner: simpler, more reliable, and easier to control
- Empirical behavior strongly favors overview design for this project’s toolset

---