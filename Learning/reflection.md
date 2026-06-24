## Proper prompt building
- Using hedging language or omitting definite words invite the LLM to treatcertin rules liberally. This will impact the strictness in which they follow the prompt, which is dangerous for a tool that need s to follow important guidelines that may have ethics legal or financial implications.

##  context leakage and shortcut reasoning
Document context already provided the date (year), hence despite calls to ignore any data the LLM thinks it knows and to use tools for realtime time updates, it captures that the date from the contect

## LLM Compliance Ensurance
- Explcit numbering for sequential rules, bold,and clear language can help bring weight to important rules.

## Compartmentalising instructions 
- Separating the prompt template between system (instructions for all engagements with users and set as baselines behavior) and human (for immediatetask/query)

## Language Model Limitations
- Llama 3.1 8B does not reliably support structured tool calling, esepcially in LangChain's OpenAI-style schema
- Swapped out with Qwen2.5 that is knwown to have better tool hadnling, with similar import process as Ollama 3.1

## Specify prompt templates for different jobs
- Current workflow invokes LLM twice for each query, one for generating possible trools to call and one for final answer
- It is important that botht heir template must be differentiated to allow for LLM at erach stage to be more focused on their specific roles.

## Tool Description
- Tool planner LLM should always receive the decription of the tools inpout and output other than its name, so that i understands what the tools can do.

## Blocking Hallucinations
- LLMs will ALWAYS prefer guessing unless you aggressively block it.

## Iterative Reasoning
- Best to use LangGraph for iterative reasoning, when input into one tool requires LLM reasoning and result from another tool.
- LangGraph helps to manage iterative loops and states. 

## LLM Confidence
- LLM is csometimes too confident in its own calculative and reasoning abilities, which makes it neglect tools that can give a more precise answer for self estimation provides inaccurate replies. 
- LLM have to be explcitedly curbed in its prompt to stop it from doing so.