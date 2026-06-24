# ADR-001: HR Policy Tool Returns Retrieved Documents Instead of Performing Internal RAG

## Status

Accepted

---

# Context

This project is an HR assistant built using LangChain and a custom iterative tool-calling architecture.

The overall pipeline consists of:

1. User question
2. Planner LLM selects tools
3. Tool execution loop
4. Tool outputs accumulated into state
5. Final answer LLM synthesizes a response

The system currently contains:

- Tool planner
- Tool execution loop
- Shared state and memory
- Date and holiday tools
- Safety/compliance tools
- HR document vector database (Chroma)
- Retriever over HR policy documents
- Final answer generation LLM

The HR policy capability can be implemented in two different ways.

---

# Option 1: Full RAG Inside the Tool

```text
User
 ↓
Planner
 ↓
hr_policy_tool
 ↓
Retriever
 ↓
Internal LLM (RAG)
 ↓
Tool output (answer)
 ↓
Final LLM
 ↓
User response
```

Example:

```python
@tool
def hr_policy_tool(query):
    docs = retriever.invoke(query)
    answer = rag_chain.invoke(...)
    return answer
```

### Advantages

- Tool behaves like a self-contained question-answering system.
- Final LLM receives concise summaries.
- Easier for simple FAQ systems.

### Disadvantages

- Two LLM passes are performed.

```text
Retriever
→ RAG LLM
→ Final LLM
```

This introduces:

- Increased latency.
- Additional token usage.
- Double abstraction.
- Loss of traceability.
- More opportunities for hallucination.
- Difficulty debugging information flow.

The final LLM no longer sees the original evidence, only another model's interpretation.

---

# Option 2: Retriever-Only Tool

```text
User
 ↓
Planner
 ↓
hr_policy_tool
 ↓
Retriever
 ↓
Raw documents
 ↓
Final LLM
 ↓
User response
```

Example:

```python
@tool
def hr_policy_tool(query):
    docs = retriever.invoke(query)
    return format_docs(docs)
```

### Advantages

- Only one reasoning step occurs.
- Lower latency.
- Lower token consumption.
- Final LLM has direct access to source evidence.
- Easier debugging.
- More transparent.
- Better tool composability.
- Reduced hallucination risk.

### Disadvantages

- Final prompt becomes more important.
- Larger context window usage.
- Final LLM must perform synthesis itself.

---

# Decision

The HR policy tool shall perform retrieval only.

The tool will:

1. Search the vector database.
2. Return retrieved document chunks.
3. Return source metadata where available.
4. Avoid any internal LLM calls.

Reasoning and answer synthesis are delegated to the final answer generation stage.

Example:

```python
@tool
def hr_policy_retriever(query):
    docs = retriever.invoke(query)

    return {
        "sources": [...],
        "documents": [...]
    }
```

---

# Rationale

The overall architecture already contains a final answer LLM.

Introducing another LLM inside the HR tool creates:

```text
LLM → LLM
```

which results in multiple layers of abstraction.

Since each LLM compresses and summarizes information, information fidelity decreases with every layer.

The final answer model would be reasoning over another model's output instead of reasoning over the original documents.

This makes hallucinations harder to detect and harder to debug.

Using retrieval only preserves the original evidence until the final reasoning stage.

---

# Insight: Tools Should Prefer Facts Over Interpretations

A useful design principle is:

> Tools should provide facts. LLMs should provide interpretations.

Examples:

### Good tools

Date tool:

```python
get_current_date()
→ "2026-06-22"
```

Holiday tool:

```python
public_holiday_checker()
→ {
    "holiday": "Christmas",
    "date": "2026-12-25"
}
```

HR retriever:

```python
hr_policy_retriever()
→ raw document chunks
```

Calculator:

```python
days_between()
→ 186
```

These tools return objective data.

---

### Bad tools

Tools that internally reason:

```python
hr_policy_tool()
→ "I think the employee should receive..."
```

Such outputs are already interpretations and become another source of hallucination.

---

# Architectural Principle

Prefer:

```text
Tools
 ↓
Facts
 ↓
Final LLM
 ↓
Answer
```

Over:

```text
Tools
 ↓
LLM interpretation
 ↓
Final LLM interpretation
 ↓
Answer
```

---

# Benefits for This Project

Because this project uses an iterative planner and tool execution loop, returning raw retrieved documents provides several advantages:

## Better Observability

State history contains actual evidence rather than summarized answers.

```python
state["history"]
```

can show exactly what was retrieved.

---

## Better Debugging

It becomes possible to inspect:

- Which documents were retrieved.
- Why a certain answer was produced.
- Whether retrieval or reasoning caused an error.

---

## Better Future Expansion

Future tools can consume the retrieved documents.

Examples:

```text
hr_policy_retriever
        ↓
leave_comparison_tool
        ↓
compliance_checker
        ↓
final_llm
```

without introducing additional LLM layers.

---

## Lower Cost

One LLM call is removed.

This reduces:

- token consumption
- latency
- complexity

---

## Reduced Hallucination Risk

Reasoning occurs only once.

Instead of:

```text
Retriever
 ↓
RAG LLM
 ↓
Final LLM
```

the system becomes:

```text
Retriever
 ↓
Final LLM
```

This preserves source fidelity and minimizes information loss.

---

# Conclusion

The project adopts a single-reasoning architecture:

```text
User
 ↓
Planner
 ↓
Tools
 ↓
Facts
 ↓
Final LLM
 ↓
Answer
```

The HR policy tool is responsible only for retrieval.

All interpretation, comparison, summarization, and response generation are centralized in the final answer LLM.

This decision prioritizes:

- transparency
- debuggability
- composability
- lower latency
- lower cost
- reduced hallucination risk

and aligns with the principle:

> Tools provide facts. The LLM provides understanding.