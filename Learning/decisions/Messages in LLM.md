## Message Types in LLM Applications (LangChain / Tool-Using Agents)

When building LLM applications with tool use, message types are not interchangeable. Each message type represents a different role in the conversation state machine, and using the wrong one leads to confusing model behavior, hallucinations, or broken tool loops.

This document explains the four core message types:

HumanMessage
SystemMessage
AIMessage
ToolMessage

and when to use each.

1. Mental Model: Why multiple message types exist

LLM systems are not just “chat logs”.

They are a structured execution graph:

User → System rules → Model reasoning → Tool execution → Tool results → Model response

Each message type represents one layer of this pipeline.

2. HumanMessage
Purpose

Represents user input.

Meaning to the model

“This is what the user is asking.”

When to use
User queries
Instructions from end user
Inputs to the system
Example
HumanMessage(content="How many days until Christmas?")
Key properties
Always originates from the user
Treated as ground truth request
Never contains tool outputs or system logic
3. SystemMessage
Purpose

Defines rules, constraints, and behavior instructions for the model.

Meaning to the model

“These are the rules you must follow when responding.”

When to use
Safety constraints (PDPA, compliance)
Tool usage rules
Role definitions (“You are an HR assistant”)
Output format constraints
Example
SystemMessage(content="You are an HR assistant. Always use tools for dates.")
Key properties
Highest priority instruction layer
Not part of conversation content
Should NOT contain user data or tool results
4. AIMessage
Purpose

Represents model-generated output.

This includes:

Final answers
Tool call requests
Intermediate reasoning (depending on framework)
Meaning to the model

“This is what the model previously said.”

Two important uses
(A) Normal assistant response
AIMessage(content="Christmas is in 19 days.")
(B) Tool call request (important in agent systems)
AIMessage(
    content="",
    tool_calls=[
        {"name": "get_current_singapore_date", "args": {}}
    ]
)

This means:

“I want to call this tool next.”

Key properties
Represents model behavior
Can contain tool call requests
Used for conversation continuity
NOT used to store tool outputs or memory
5. ToolMessage
Purpose

Represents output from external tools after execution.

Meaning to the model

“This is the result returned by a tool you requested.”

Example
ToolMessage(
    content="2026-06-15 Singapore Date",
    tool_call_id="abc123"
)
Key properties
Always tied to a specific tool call
Bridges tool execution → model reasoning
Only used in reactive agent loops
NOT used in planner–executor architectures
6. When NOT to use ToolMessage (important for your system)

If your architecture is:

Planner → Executor → Final LLM

Then:

❌ You should NOT use ToolMessage

Because:

You are NOT doing back-and-forth tool calling
You already store tool results in state["memory"]
ToolMessage becomes redundant serialization
7. Why AIMessage is NOT used for tool results

A common mistake is:

AIMessage(content="", tool_calls=state["memory"]) ❌ WRONG
Why this is incorrect:

Because:

tool_calls = instructions to call tools
NOT storage for tool outputs
NOT memory representation

So this causes:

schema confusion
model misinterpretation
broken execution loops
8. Why SystemMessage or HumanMessage is used for tool results (in your architecture)

In a planner–executor system, tool outputs are:

external facts used for reasoning

They are best passed as:

Option A (recommended): SystemMessage
SystemMessage(content="Tool results:\n{...}")
Why:
Treated as ground truth context
Not confused with conversation turns
Stable for reasoning
Option B: HumanMessage (acceptable alternative)
HumanMessage(content="Tool results:\n{...}")
Why:
Simpler pipeline
Still readable as input context
Rule of thumb:
Type	Use for tool outputs?
SystemMessage	✅ Best
HumanMessage	⚠️ acceptable
AIMessage	❌ never
ToolMessage	❌ only for agent loops
9. Summary Table
Message Type	Purpose	Contains Tool Outputs?	Used in Your Architecture?
HumanMessage	User input	❌	✅
SystemMessage	Rules & constraints	❌	✅
AIMessage	Model output / tool calls	❌	⚠️ only for planning
ToolMessage	Tool execution results	✅	❌ (not needed)
10. Final Design Rule (for your system)

If you are using:

Planner → Executor → Final Answer LLM

Then:

✔ Use:
SystemMessage (rules)
HumanMessage (query + tool results)
state["memory"] (internal truth store)
❌ Avoid:
ToolMessage
misusing AIMessage.tool_calls for memory
duplicating state in multiple formats
11. Key Insight

ToolMessage exists only to support reactive agent loops.
If you are not running a looped tool-call conversation, you do not need it.