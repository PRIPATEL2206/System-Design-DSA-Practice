# Phase 25 — LangGraph

## Overview

LangGraph is a framework for building **stateful, multi-actor AI workflows** as directed graphs. While LangChain chains are linear pipelines (A → B → C), LangGraph enables **cycles, conditional branching, persistence, human-in-the-loop**, and multi-agent coordination — the patterns needed for production-grade AI agents.

Think of it this way: LangChain LCEL is a conveyor belt (data flows one way). LangGraph is a **state machine** — nodes process data, edges route based on conditions, and the graph can loop back to re-process, wait for human input, or coordinate multiple agents.

LangGraph is essential when:
- An agent needs to **retry or self-correct** (cycles)
- You need **human approval** before an action (human-in-the-loop)
- Multiple agents collaborate on a task (multi-agent)
- State must be **persisted** across sessions (checkpointing)
- You need **streaming** of intermediate steps

---

## 1. Core Concepts

![LangGraph State Machine](diagrams/langgraph_state_machine.svg)

### Graph Structure

```
Nodes:  Functions that process state (LLM calls, tool execution, logic)
Edges:  Connections between nodes (unconditional or conditional)
State:  Typed dictionary that flows through the graph and accumulates data
```

### Why Graphs Over Chains?

| Capability | LCEL Chain | LangGraph |
|---|---|---|
| Linear pipeline | Yes | Yes |
| Conditional routing | Limited (RunnableBranch) | Full (conditional edges) |
| Cycles / loops | No | Yes |
| Human-in-the-loop | No | Yes (interrupt_before/after) |
| State persistence | No (stateless) | Yes (checkpointing) |
| Multi-agent | Manual | Native support |
| Streaming intermediate steps | Limited | Full |
| Error recovery / retry | Manual | Built-in via cycles |
| Parallel node execution | Yes | Yes |
| Time travel / replay | No | Yes (from any checkpoint) |

### Installation

```bash
pip install langgraph langchain-openai langchain-community
```

---

## 2. State & State Schema

State is the central concept in LangGraph — it's the data that flows through the graph, accumulated and transformed by each node.

```python
from typing import Annotated, TypedDict, Sequence
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

# ============================================================
# Define state schema using TypedDict
# ============================================================
class AgentState(TypedDict):
    """State that flows through the graph."""
    # Messages accumulate (each node can append)
    messages: Annotated[Sequence[BaseMessage], add_messages]
    # Other state fields
    current_step: str
    retry_count: int
    final_answer: str

# add_messages is a REDUCER — it appends new messages to existing ones
# Without a reducer, new values OVERWRITE old ones

# ============================================================
# Custom reducers
# ============================================================
from operator import add

class ResearchState(TypedDict):
    query: str                                          # Overwritten each time
    messages: Annotated[list, add_messages]             # Appended
    sources: Annotated[list[str], add]                  # Appended (list concat)
    documents: Annotated[list[str], add]                # Appended
    answer: str                                         # Overwritten
    iteration: int                                      # Overwritten
```

---

## 3. Building Your First Graph

### Simple Chatbot with Tool Use

```python
from langgraph.graph import StateGraph, END, START
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage
from langchain_core.tools import tool
from typing import Annotated, TypedDict, Sequence
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages


# Step 1: Define state
class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]


# Step 2: Define tools
@tool
def search(query: str) -> str:
    """Search the web for information."""
    return f"Search results for '{query}': [relevant information about {query}]"

@tool
def calculator(expression: str) -> str:
    """Calculate a mathematical expression."""
    return str(eval(expression))

tools = [search, calculator]


# Step 3: Define nodes
model = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

def call_model(state: State) -> dict:
    """Node: Call the LLM with current messages."""
    response = model.invoke(state["messages"])
    return {"messages": [response]}


# Step 4: Build the graph
graph_builder = StateGraph(State)

# Add nodes
graph_builder.add_node("agent", call_model)
graph_builder.add_node("tools", ToolNode(tools))

# Add edges
graph_builder.add_edge(START, "agent")  # Start → agent

# Conditional edge: if LLM called tools → go to tools node, else → END
graph_builder.add_conditional_edges(
    "agent",
    tools_condition,  # Built-in: checks if response has tool_calls
    {"tools": "tools", END: END}
)

# After tools execute, go back to agent (THE CYCLE!)
graph_builder.add_edge("tools", "agent")

# Compile
graph = graph_builder.compile()


# Step 5: Run
result = graph.invoke({
    "messages": [HumanMessage(content="What's 25 * 47 + 130?")]
})

# Print conversation
for msg in result["messages"]:
    print(f"{msg.type}: {msg.content}")
# human: What's 25 * 47 + 130?
# ai: [tool_call: calculator("25 * 47 + 130")]
# tool: 1305
# ai: 25 × 47 + 130 = 1,305


# Step 6: Stream intermediate steps
for event in graph.stream(
    {"messages": [HumanMessage(content="Search for the latest AI news")]},
    stream_mode="values"
):
    last_msg = event["messages"][-1]
    print(f"  [{last_msg.type}] {last_msg.content[:100]}")
```

### Visualize the Graph

```python
# Print ASCII graph
print(graph.get_graph().draw_ascii())

#        +-----------+
#        | __start__ |
#        +-----------+
#              |
#              v
#         +-------+
#    +--->| agent |
#    |    +-------+
#    |       |    |
#    |       |    v
#    |       |  +-----+
#    |       |  | END |
#    |       |  +-----+
#    |       v
#    |   +-------+
#    +---| tools |
#        +-------+

# Or generate a PNG/Mermaid diagram
graph.get_graph().draw_mermaid_png(output_file_path="graph.png")
```

---

## 4. Conditional Edges & Routing

```python
from langgraph.graph import StateGraph, END, START

class RouterState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    intent: str

def classify_intent(state: RouterState) -> dict:
    """Classify the user's intent."""
    last_message = state["messages"][-1].content
    
    # Use LLM to classify
    classification = classifier_llm.invoke(
        f"Classify this as 'technical', 'billing', or 'general': {last_message}"
    ).content.strip().lower()
    
    return {"intent": classification}

def handle_technical(state: RouterState) -> dict:
    """Handle technical questions with RAG."""
    response = technical_rag_chain.invoke(state["messages"][-1].content)
    return {"messages": [AIMessage(content=response)]}

def handle_billing(state: RouterState) -> dict:
    """Handle billing questions."""
    response = billing_chain.invoke(state["messages"][-1].content)
    return {"messages": [AIMessage(content=response)]}

def handle_general(state: RouterState) -> dict:
    """Handle general questions."""
    response = general_chain.invoke(state["messages"][-1].content)
    return {"messages": [AIMessage(content=response)]}

# Routing function
def route_by_intent(state: RouterState) -> str:
    """Route to the appropriate handler based on intent."""
    intent = state["intent"]
    if "technical" in intent:
        return "technical"
    elif "billing" in intent:
        return "billing"
    else:
        return "general"

# Build graph
builder = StateGraph(RouterState)

builder.add_node("classify", classify_intent)
builder.add_node("technical", handle_technical)
builder.add_node("billing", handle_billing)
builder.add_node("general", handle_general)

builder.add_edge(START, "classify")

# Conditional routing based on classification
builder.add_conditional_edges(
    "classify",
    route_by_intent,
    {
        "technical": "technical",
        "billing": "billing",
        "general": "general"
    }
)

# All handlers end the graph
builder.add_edge("technical", END)
builder.add_edge("billing", END)
builder.add_edge("general", END)

router_graph = builder.compile()
```

---

## 5. Human-in-the-Loop

The killer feature of LangGraph — pause execution, wait for human approval, then continue.

```python
from langgraph.graph import StateGraph, END, START
from langgraph.checkpoint.memory import MemorySaver

class ApprovalState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    action_plan: str
    approved: bool

def plan_action(state: ApprovalState) -> dict:
    """Agent plans an action (e.g., sending email, making purchase)."""
    plan = model.invoke(state["messages"]).content
    return {"action_plan": plan}

def execute_action(state: ApprovalState) -> dict:
    """Execute the approved action."""
    result = f"Executed: {state['action_plan']}"
    return {"messages": [AIMessage(content=result)]}

def check_approval(state: ApprovalState) -> str:
    """Route based on human approval."""
    if state.get("approved", False):
        return "execute"
    else:
        return "end"

# Build graph
builder = StateGraph(ApprovalState)
builder.add_node("plan", plan_action)
builder.add_node("execute", execute_action)

builder.add_edge(START, "plan")
builder.add_conditional_edges(
    "plan",
    check_approval,
    {"execute": "execute", "end": END}
)
builder.add_edge("execute", END)

# CRITICAL: Use checkpointer for persistence + interrupt
checkpointer = MemorySaver()

graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute"]  # Pause BEFORE execution!
)

# --- Usage ---
config = {"configurable": {"thread_id": "approval_1"}}

# Run until interrupt point
result = graph.invoke(
    {"messages": [HumanMessage(content="Send a refund email to customer@email.com for $50")]},
    config=config
)
print(f"Plan: {result['action_plan']}")
# Plan: "Send email to customer@email.com confirming $50 refund..."

# At this point, the graph is PAUSED. Show plan to human.
# Human approves...

# Resume with approval
graph.update_state(config, {"approved": True})
final_result = graph.invoke(None, config=config)  # Continue from checkpoint
print(f"Result: {final_result['messages'][-1].content}")
# Result: "Executed: Send email to customer@email.com confirming $50 refund..."
```

---

## 6. Persistence & Checkpointing

Checkpointing saves graph state so workflows can be paused, resumed, replayed, or time-traveled.

```python
from langgraph.checkpoint.memory import MemorySaver       # In-memory (dev)
from langgraph.checkpoint.sqlite import SqliteSaver       # SQLite (local)
from langgraph.checkpoint.postgres import PostgresSaver   # Postgres (production)

# ============================================================
# Production: PostgreSQL checkpointer
# ============================================================
from psycopg_pool import ConnectionPool

pool = ConnectionPool(conninfo="postgresql://user:pass@localhost:5432/langgraph_db")
checkpointer = PostgresSaver(pool)
checkpointer.setup()  # Create tables

graph = builder.compile(checkpointer=checkpointer)

# Every invoke automatically saves state
config = {"configurable": {"thread_id": "user_session_abc123"}}

# Turn 1
result1 = graph.invoke(
    {"messages": [HumanMessage(content="Hello, I need help with my order")]},
    config=config
)

# Turn 2 (continues from saved state)
result2 = graph.invoke(
    {"messages": [HumanMessage(content="My order number is ORD-123")]},
    config=config
)

# ============================================================
# Time Travel: replay from any past state
# ============================================================
# Get all checkpoints for a thread
checkpoints = list(checkpointer.list(config))

# Replay from a specific checkpoint
past_config = {"configurable": {
    "thread_id": "user_session_abc123",
    "checkpoint_id": checkpoints[2].checkpoint_id  # Go back to checkpoint 2
}}

# Continue from that point with different input
alternate_result = graph.invoke(
    {"messages": [HumanMessage(content": "Actually, cancel my order")]},
    config=past_config
)
```

---

## 7. Multi-Agent Workflows

![LangGraph Multi-Agent Workflow](diagrams/langgraph_multi_agent.svg)

### Supervisor Pattern

One supervisor agent coordinates specialist agents.

```python
from langgraph.graph import StateGraph, END, START
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

class MultiAgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    next_agent: str
    research_notes: str
    draft: str
    final_output: str

# ============================================================
# Specialist Agents
# ============================================================
researcher_model = ChatOpenAI(model="gpt-4o-mini", temperature=0)
writer_model = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
editor_model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def researcher(state: MultiAgentState) -> dict:
    """Research agent: gathers information."""
    query = state["messages"][-1].content
    
    response = researcher_model.invoke([
        SystemMessage(content="You are a research assistant. Gather key facts and data points about the topic. Be thorough and cite sources."),
        HumanMessage(content=f"Research this topic: {query}")
    ])
    
    return {
        "research_notes": response.content,
        "messages": [AIMessage(content=f"[Researcher] {response.content[:200]}...")]
    }

def writer(state: MultiAgentState) -> dict:
    """Writer agent: creates content from research."""
    response = writer_model.invoke([
        SystemMessage(content="You are a skilled writer. Create engaging, well-structured content based on the research notes provided."),
        HumanMessage(content=f"Write an article based on these research notes:\n\n{state['research_notes']}")
    ])
    
    return {
        "draft": response.content,
        "messages": [AIMessage(content=f"[Writer] Draft complete ({len(response.content)} chars)")]
    }

def editor(state: MultiAgentState) -> dict:
    """Editor agent: reviews and improves the draft."""
    response = editor_model.invoke([
        SystemMessage(content="You are an editor. Review the draft for clarity, accuracy, grammar, and structure. Provide the final polished version."),
        HumanMessage(content=f"Edit and polish this draft:\n\n{state['draft']}")
    ])
    
    return {
        "final_output": response.content,
        "messages": [AIMessage(content=f"[Editor] Final version ready.")]
    }

# ============================================================
# Supervisor: decides which agent goes next
# ============================================================
supervisor_model = ChatOpenAI(model="gpt-4o", temperature=0)

def supervisor(state: MultiAgentState) -> dict:
    """Supervisor decides the next step."""
    
    # Determine what's been done
    has_research = bool(state.get("research_notes"))
    has_draft = bool(state.get("draft"))
    has_final = bool(state.get("final_output"))
    
    if not has_research:
        return {"next_agent": "researcher"}
    elif not has_draft:
        return {"next_agent": "writer"}
    elif not has_final:
        return {"next_agent": "editor"}
    else:
        return {"next_agent": "FINISH"}

def route_supervisor(state: MultiAgentState) -> str:
    return state["next_agent"]

# ============================================================
# Build the graph
# ============================================================
builder = StateGraph(MultiAgentState)

builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("editor", editor)

builder.add_edge(START, "supervisor")

builder.add_conditional_edges(
    "supervisor",
    route_supervisor,
    {
        "researcher": "researcher",
        "writer": "writer",
        "editor": "editor",
        "FINISH": END
    }
)

# After each agent, go back to supervisor
builder.add_edge("researcher", "supervisor")
builder.add_edge("writer", "supervisor")
builder.add_edge("editor", "supervisor")

multi_agent_graph = builder.compile()

# Run
result = multi_agent_graph.invoke({
    "messages": [HumanMessage(content="Write an article about the future of quantum computing")]
})

print(result["final_output"])
```

### Collaboration Pattern (Agents Pass Work to Each Other)

```python
from langgraph.graph import StateGraph, END, START

class CollabState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    code: str
    review_feedback: str
    tests: str
    iteration: int

def coder(state: CollabState) -> dict:
    """Write or revise code based on feedback."""
    feedback = state.get("review_feedback", "")
    task = state["messages"][0].content
    
    prompt = f"Task: {task}"
    if feedback:
        prompt += f"\n\nPrevious code:\n{state['code']}\n\nReview feedback:\n{feedback}\n\nRevise the code to address the feedback."
    
    response = coder_llm.invoke(prompt)
    return {"code": response.content, "iteration": state.get("iteration", 0) + 1}

def reviewer(state: CollabState) -> dict:
    """Review code and provide feedback."""
    response = reviewer_llm.invoke(
        f"Review this code for bugs, style, and performance:\n\n{state['code']}\n\n"
        "If it's good, respond with 'APPROVED'. Otherwise, list specific issues."
    )
    return {"review_feedback": response.content}

def tester(state: CollabState) -> dict:
    """Write tests for the code."""
    response = tester_llm.invoke(
        f"Write unit tests for this code:\n\n{state['code']}"
    )
    return {"tests": response.content}

def should_continue(state: CollabState) -> str:
    """Check if code is approved or needs revision."""
    if "APPROVED" in state.get("review_feedback", ""):
        return "approved"
    elif state.get("iteration", 0) >= 3:
        return "max_iterations"  # Safety limit
    else:
        return "needs_revision"

# Build
builder = StateGraph(CollabState)
builder.add_node("coder", coder)
builder.add_node("reviewer", reviewer)
builder.add_node("tester", tester)

builder.add_edge(START, "coder")
builder.add_edge("coder", "reviewer")

builder.add_conditional_edges(
    "reviewer",
    should_continue,
    {
        "approved": "tester",      # Good code → write tests
        "needs_revision": "coder", # Bad code → revise (CYCLE!)
        "max_iterations": "tester" # Give up after 3 tries
    }
)

builder.add_edge("tester", END)

collab_graph = builder.compile()
```

---

## 8. Streaming

LangGraph provides rich streaming for real-time UIs.

```python
# ============================================================
# Stream modes
# ============================================================

# Mode 1: "values" — full state after each node
for state in graph.stream(input, stream_mode="values"):
    print(f"Messages so far: {len(state['messages'])}")

# Mode 2: "updates" — only the delta from each node
for update in graph.stream(input, stream_mode="updates"):
    node_name = list(update.keys())[0]
    node_output = update[node_name]
    print(f"Node '{node_name}' produced: {node_output}")

# Mode 3: "messages" — stream individual LLM tokens
async for event in graph.astream_events(input, version="v2"):
    if event["event"] == "on_chat_model_stream":
        token = event["data"]["chunk"].content
        print(token, end="", flush=True)

# ============================================================
# Production streaming with FastAPI
# ============================================================
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

@app.post("/chat/stream")
async def stream_chat(request: ChatRequest):
    async def generate():
        async for event in graph.astream_events(
            {"messages": [HumanMessage(content=request.message)]},
            config={"configurable": {"thread_id": request.session_id}},
            version="v2"
        ):
            if event["event"] == "on_chat_model_stream":
                token = event["data"]["chunk"].content
                if token:
                    yield f"data: {json.dumps({'token': token})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## 9. Tool Orchestration & Subgraphs

### Subgraphs for Modularity

```python
from langgraph.graph import StateGraph, END, START

# ============================================================
# Define a subgraph (reusable component)
# ============================================================
class RAGState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    query: str
    documents: list[str]
    answer: str

def retrieve(state: RAGState) -> dict:
    docs = retriever.invoke(state["query"])
    return {"documents": [d.page_content for d in docs]}

def generate(state: RAGState) -> dict:
    context = "\n".join(state["documents"])
    answer = rag_llm.invoke(f"Context: {context}\n\nQuestion: {state['query']}")
    return {"answer": answer.content}

# Build subgraph
rag_builder = StateGraph(RAGState)
rag_builder.add_node("retrieve", retrieve)
rag_builder.add_node("generate", generate)
rag_builder.add_edge(START, "retrieve")
rag_builder.add_edge("retrieve", "generate")
rag_builder.add_edge("generate", END)

rag_subgraph = rag_builder.compile()

# ============================================================
# Use subgraph in parent graph
# ============================================================
class ParentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    query: str
    final_answer: str

def run_rag(state: ParentState) -> dict:
    """Call the RAG subgraph."""
    result = rag_subgraph.invoke({
        "messages": state["messages"],
        "query": state["query"],
        "documents": [],
        "answer": ""
    })
    return {"final_answer": result["answer"]}

parent_builder = StateGraph(ParentState)
parent_builder.add_node("rag", run_rag)
parent_builder.add_edge(START, "rag")
parent_builder.add_edge("rag", END)

parent_graph = parent_builder.compile()
```

---

## 10. Production Patterns

### Self-Correcting Agent

```python
class SelfCorrectState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    query: str
    answer: str
    is_valid: bool
    attempts: int

def generate_answer(state: SelfCorrectState) -> dict:
    answer = model.invoke(state["messages"]).content
    return {"answer": answer, "attempts": state.get("attempts", 0) + 1}

def validate_answer(state: SelfCorrectState) -> dict:
    """Check answer quality with a separate LLM call."""
    check = validator_llm.invoke(
        f"Question: {state['query']}\nAnswer: {state['answer']}\n\n"
        "Is this answer correct, complete, and well-formatted? YES or NO with reason."
    ).content
    
    return {"is_valid": "YES" in check.upper()}

def should_retry(state: SelfCorrectState) -> str:
    if state["is_valid"]:
        return "done"
    elif state["attempts"] >= 3:
        return "done"  # Give up
    else:
        return "retry"

# Build self-correcting graph
builder = StateGraph(SelfCorrectState)
builder.add_node("generate", generate_answer)
builder.add_node("validate", validate_answer)

builder.add_edge(START, "generate")
builder.add_edge("generate", "validate")

builder.add_conditional_edges(
    "validate",
    should_retry,
    {"retry": "generate", "done": END}  # CYCLE for self-correction
)

self_correct_graph = builder.compile()
```

### Map-Reduce Pattern

```python
from langgraph.graph import StateGraph, END, START
from langgraph.constants import Send

class MapReduceState(TypedDict):
    topics: list[str]
    summaries: Annotated[list[str], add]
    final_report: str

class TopicState(TypedDict):
    topic: str
    summary: str

def fan_out(state: MapReduceState):
    """Send each topic to be researched in parallel."""
    return [Send("research_topic", {"topic": t}) for t in state["topics"]]

def research_topic(state: TopicState) -> dict:
    """Research a single topic (runs in parallel for each topic)."""
    summary = model.invoke(f"Write a 2-sentence summary of: {state['topic']}").content
    return {"summaries": [summary]}

def combine_results(state: MapReduceState) -> dict:
    """Reduce: combine all summaries into final report."""
    all_summaries = "\n\n".join(state["summaries"])
    report = model.invoke(f"Combine these summaries into a cohesive report:\n\n{all_summaries}").content
    return {"final_report": report}

builder = StateGraph(MapReduceState)
builder.add_node("research_topic", research_topic)
builder.add_node("combine", combine_results)

builder.add_conditional_edges(START, fan_out, ["research_topic"])
builder.add_edge("research_topic", "combine")
builder.add_edge("combine", END)

map_reduce_graph = builder.compile()

result = map_reduce_graph.invoke({
    "topics": ["quantum computing", "gene therapy", "fusion energy"],
    "summaries": []
})
print(result["final_report"])
```

---

## 11. LangGraph vs LangChain Agents

| Aspect | LangChain AgentExecutor | LangGraph Agent |
|---|---|---|
| **Control flow** | Hidden in executor loop | Explicit in graph |
| **Customization** | Limited (callbacks) | Full (add any node/edge) |
| **Human-in-the-loop** | Not built-in | Native (`interrupt_before`) |
| **Persistence** | Not built-in | Native (checkpointer) |
| **Multi-agent** | Manual orchestration | Native patterns |
| **Streaming** | Limited | Full (events, tokens, states) |
| **Error handling** | Generic retry | Custom recovery nodes |
| **Debugging** | Hard (black box loop) | Easy (visualize graph, inspect state) |
| **Testing** | Integration tests only | Unit test each node |

**Recommendation**: Use LangGraph for anything beyond a simple single-agent tool-calling loop. The explicit graph gives you control, debuggability, and production-readiness that AgentExecutor cannot match.

---

## 12. Complete Project: Research Assistant with Human Review

```python
"""
Production research assistant:
- Plans research approach
- Searches multiple sources
- Synthesizes findings
- Human reviews before final output
- Self-corrects based on feedback
"""

from langgraph.graph import StateGraph, END, START
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langchain_core.tools import tool
from typing import Annotated, TypedDict, Sequence
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages
from operator import add


class ResearchAssistantState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    research_query: str
    plan: str
    findings: Annotated[list[str], add]
    draft_report: str
    human_feedback: str
    final_report: str
    step: str
    iteration: int


model = ChatOpenAI(model="gpt-4o", temperature=0)


def planner(state: ResearchAssistantState) -> dict:
    """Create a research plan."""
    response = model.invoke([
        SystemMessage(content="You are a research planner. Create a 3-step research plan with specific questions to investigate."),
        HumanMessage(content=f"Research topic: {state['research_query']}")
    ])
    return {
        "plan": response.content,
        "step": "planned",
        "messages": [AIMessage(content=f"[Planner] Created research plan.")]
    }


def researcher(state: ResearchAssistantState) -> dict:
    """Execute research based on plan."""
    response = model.invoke([
        SystemMessage(content="You are a researcher. Based on the plan below, provide detailed findings with facts and data."),
        HumanMessage(content=f"Plan:\n{state['plan']}\n\nProvide comprehensive findings.")
    ])
    return {
        "findings": [response.content],
        "step": "researched",
        "messages": [AIMessage(content="[Researcher] Research complete.")]
    }


def synthesizer(state: ResearchAssistantState) -> dict:
    """Synthesize findings into a draft report."""
    all_findings = "\n\n".join(state["findings"])
    
    feedback_instruction = ""
    if state.get("human_feedback"):
        feedback_instruction = f"\n\nPrevious feedback to address:\n{state['human_feedback']}"
    
    response = model.invoke([
        SystemMessage(content="You are a report writer. Create a clear, structured report from the research findings."),
        HumanMessage(content=f"Findings:\n{all_findings}{feedback_instruction}\n\nWrite the report.")
    ])
    return {
        "draft_report": response.content,
        "step": "drafted",
        "iteration": state.get("iteration", 0) + 1,
        "messages": [AIMessage(content=f"[Synthesizer] Draft report ready for review.")]
    }


def should_revise(state: ResearchAssistantState) -> str:
    """Check if human approved or wants revision."""
    feedback = state.get("human_feedback", "")
    if "APPROVED" in feedback.upper():
        return "finalize"
    elif state.get("iteration", 0) >= 3:
        return "finalize"  # Max revisions reached
    else:
        return "revise"


def finalize(state: ResearchAssistantState) -> dict:
    """Produce final report."""
    return {
        "final_report": state["draft_report"],
        "step": "complete",
        "messages": [AIMessage(content="[System] Research complete. Final report ready.")]
    }


# Build graph
builder = StateGraph(ResearchAssistantState)

builder.add_node("planner", planner)
builder.add_node("researcher", researcher)
builder.add_node("synthesizer", synthesizer)
builder.add_node("finalize", finalize)

builder.add_edge(START, "planner")
builder.add_edge("planner", "researcher")
builder.add_edge("researcher", "synthesizer")

# Human review happens here (interrupt_after synthesizer)
builder.add_conditional_edges(
    "synthesizer",
    should_revise,
    {"revise": "synthesizer", "finalize": "finalize"}
)
builder.add_edge("finalize", END)

# Compile with checkpointing + human interrupt
checkpointer = MemorySaver()
research_graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_after=["synthesizer"]  # Pause after draft for human review
)


# ============================================================
# Usage flow
# ============================================================
config = {"configurable": {"thread_id": "research_001"}}

# Step 1: Start research (runs until synthesizer completes, then pauses)
result = research_graph.invoke(
    {
        "messages": [HumanMessage(content="Research the impact of AI on healthcare")],
        "research_query": "Impact of AI on healthcare in 2024",
        "findings": [],
        "iteration": 0
    },
    config=config
)

print("Draft Report:")
print(result["draft_report"][:500])
print("\n--- Waiting for human review ---")

# Step 2: Human provides feedback
research_graph.update_state(
    config,
    {"human_feedback": "Good start, but add more about regulatory challenges and include specific statistics about diagnostic AI accuracy."}
)

# Step 3: Resume (synthesizer will revise based on feedback)
result = research_graph.invoke(None, config=config)

print("\nRevised Draft:")
print(result["draft_report"][:500])

# Step 4: Human approves
research_graph.update_state(config, {"human_feedback": "APPROVED"})
final = research_graph.invoke(None, config=config)

print("\nFinal Report:")
print(final["final_report"])
```

---

## Interview Mastery

### Beginner Questions

**Q1: What is LangGraph? How is it different from LangChain?**

**A:** LangGraph is a framework for building stateful, multi-step AI workflows as directed graphs. While LangChain LCEL creates linear pipelines (A → B → C), LangGraph enables cycles (agent retries), conditional branching (routing), persistence (save/resume state), and human-in-the-loop (pause for approval). Think of LCEL as a conveyor belt and LangGraph as a state machine. Use LangChain for simple chains; use LangGraph when you need loops, memory across sessions, or human intervention.

---

**Q2: What are nodes, edges, and state in LangGraph?**

**A:** **Nodes** are functions that process and modify state (LLM calls, tool execution, business logic). **Edges** connect nodes and define the flow — either unconditional (always go A→B) or conditional (go to B or C based on state). **State** is a typed dictionary (TypedDict) that flows through the graph, accumulated by each node. Reducers (like `add_messages`) define how values are combined when multiple nodes write to the same field — either overwrite or append.

---

### Intermediate Questions

**Q3: Explain the human-in-the-loop pattern. How does checkpointing enable it?**

**A:** Human-in-the-loop pauses graph execution at a designated point, presents the current state to a human for review, and resumes based on their input. Implementation: (1) Compile the graph with a `checkpointer` (MemorySaver, PostgresSaver) and `interrupt_before` or `interrupt_after` a specific node. (2) When execution hits the interrupt point, state is saved and the graph returns. (3) Present the state to the human (e.g., "Should we execute this action?"). (4) Call `graph.update_state()` with their feedback, then `graph.invoke(None)` to resume from the checkpoint. Without checkpointing, the state would be lost at the pause point.

---

**Q4: Compare the Supervisor and Collaboration multi-agent patterns.**

**A:** **Supervisor**: One central agent (supervisor) decides which specialist to invoke next. Specialists report back to the supervisor who routes to the next one. Pros: centralized control, easy to reason about. Cons: supervisor is a bottleneck, can make poor routing decisions.

**Collaboration**: Agents pass work directly to each other in a defined workflow (coder → reviewer → tester). No central coordinator. Pros: more natural workflows, less overhead. Cons: harder to modify flow dynamically, less adaptive.

Choose supervisor when: tasks are heterogeneous and routing logic is complex. Choose collaboration when: the workflow is well-defined and sequential (like a pipeline).

---

### Advanced Questions

**Q5: Design a self-correcting code generation agent with LangGraph.**

**A:**
```
START → generate_code → run_tests
                              ↓
                    tests pass? ─── YES → END
                              ↓
                             NO
                              ↓
                    attempts < 3? ─── NO → END (return best attempt)
                              ↓
                            YES
                              ↓
                    analyze_errors → generate_code (CYCLE)
```

State: `{code, test_results, errors, attempt_count}`. The `analyze_errors` node feeds error messages back so the code generator can fix specific issues. Guardrail: max 3 iterations. This pattern consistently produces better code than single-shot generation because it mimics the human debug cycle.

---

**Q6: How would you deploy a LangGraph agent for production with 10K concurrent users?**

**A:**
1. **Stateless compute**: Deploy the graph behind a load balancer. Each request is routed to any server (state lives in the checkpointer, not in memory)
2. **PostgreSQL checkpointer**: All state persisted in a shared Postgres cluster. Enables resume from any server
3. **Redis for short-term state**: Session cache for active conversations (faster than Postgres for hot state)
4. **Async execution**: Use `graph.ainvoke()` for non-blocking processing
5. **Streaming**: SSE endpoints for real-time token streaming to frontend
6. **Thread management**: Each user conversation = one `thread_id`. Clean up old threads periodically
7. **Monitoring**: LangSmith for trace visualization; custom metrics for graph execution time per node
8. **Error handling**: Retry nodes with exponential backoff; dead-letter queue for failed executions

---

[Download This File](#)
