# Development Guide

This guide covers how to extend Lingxi by adding new agents, tools, and graphs. For project setup and usage, see the main [README](../README.md).

---

## Adding Agents

Agents are generally defined via:
1. System prompt (optional)
2. User prompt or past conversation history
3. Tools (optional)

In LangGraph, there are multiple ways to create and define agents depending on their purpose. See [LangGraph Documentation: Build an Agent](https://python.langchain.com/docs/tutorials/agents/) for more information, or view the related graph files in our project.

### Tool-less Agents

These agents are defined to be used without tools. In this case, the LLM can be used directly:

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-3-5-haiku-latest", temperature=0.0)
response = llm.invoke("Tell me a joke")  # Single string input
response = llm.invoke(["What is 2+2", "What is the previous result times 5"])  # List of strings
```

### ReACT Agents

These agents are defined with tools which they will decide to use on their own. The agent should be created using [`create_react_agent`](https://langchain-ai.github.io/langgraph/reference/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent):

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage

llm = ChatAnthropic(model="claude-3-5-haiku-latest", temperature=0.0)
tools = [math_tool]
agent = create_react_agent(
    llm,
    tools=tools,
    prompt="You must solve the given problems!"  # Optional system prompt
)
agent.invoke({"messages": [HumanMessage(content="What is 2+2")]})
```

ReACT agents create a subgraph with the following structure:

![ReACT Agent Subgraph](../imgs/graph3.png)

### Agent Expected Input and LangGraph State

Different agent creation methods expect different inputs:

- **Tool-less agents** (e.g., `ChatAnthropic` directly) accept flexible inputs:
  - A string
  - A list of strings
  - A list of `BaseMessage` subclasses (`SystemMessage`, `HumanMessage`, `AIMessage`, etc.)

- **ReACT agents** expect structured input via a `MessagesState` object:
  - `MessagesState` holds the conversation history and any additional custom variables (e.g., see `src/agent/state.py:CustomState`)
  - It must be a dictionary with a `messages` key containing a list of `BaseMessage` subclasses
  - Messages are updated automatically by the LangGraph graph within defined nodes
  - Custom variables can be updated when returning from each node using `Command(update={"custom_var": 2})`

> **Important**: The `messages` list follows a strict structure in ReACT agents. When an agent uses a tool, the `AIMessage` is followed by a `ToolMessage` containing the tool's result. Removing either message from the state will break the workflow.

### System Prompts

A system prompt can take different forms in LangGraph:
- `prompt` kwarg of `create_react_agent` (str): Converted to a `SystemMessage` and prepended to `state["messages"]`
- `SystemMessage`: Prepended to `state["messages"]`
- `Callable`: Takes full graph state as input, output is passed to the language model
- `Runnable`: Takes full graph state as input, output is passed to the language model

---

## Adding Tools

Tools are Python functions that can be used by agents/LLM (defined via `create_react_agent`). They are defined using the `@tool` decorator, and require a docstring explaining the function, args, and expected return info. This information is passed to the agent as context.

```python
from langchain_core.tools import tool
from typing import Annotated, List

# Via type hints
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

# Via annotated type hints
@tool
def multiply_by_max(
    a: Annotated[int, "scale factor"],
    b: Annotated[List[int], "list of ints over which to take maximum"],
) -> int:
    """Multiply a by the maximum of b."""
    return a * max(b)
```

Once tools are created, link them to agents via the `tools` keyword in `create_react_agent`:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage
from tools import multiply, multiply_by_max

llm = ChatAnthropic(model="claude-3-5-haiku-latest", temperature=0.0)
agent = create_react_agent(
    llm,
    tools=[multiply, multiply_by_max],
    prompt="You must solve the given problems!"
)

agent.invoke({"messages": [HumanMessage(content="What is 2*5")]})
agent.invoke({"messages": [
    HumanMessage(content="What is 2 multiplied by the max of this list: [5, 10, 15]")
]})
```

Additional info: [LangChain Documentation: How to create tools](https://python.langchain.com/docs/how_to/custom_tools/)

---

## Creating Graphs

A graph defines a multi-agent workflow. Follow these steps:

1. Create a Python file for the graph
2. Define agents
3. Define nodes for each agent (containing pre/post-invocation logic)
4. Build the graph using `StateGraph`, constructing edges between nodes
5. Compile the graph using `graph.compile()`
6. Register in `langgraph.json`

```python
# graph.py
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage
from langgraph.prebuilt import create_react_agent
from langgraph.graph import MessagesState, StateGraph, START, END

# Define agents
from .tools import multiply, multiply_by_max

llm = ChatAnthropic(model="claude-3-5-haiku-latest", temperature=0.0)
problem_solver_agent = create_react_agent(
    llm,
    tools=[multiply, multiply_by_max],
    prompt="You must solve the given problems!"
)

# Define nodes
def problem_solver_node(state: MessagesState):
    response = problem_solver_agent.invoke(state)
    new_messages = response["messages"][len(state["messages"]):]
    return new_messages

# Build the graph
workflow = StateGraph(MessagesState)
workflow.add_edge(START, "problem_solver")
workflow.add_node("problem_solver", problem_solver_node)
workflow.add_edge("problem_solver", END)

# Compile
graph = workflow.compile()
```

Register the graph in `langgraph.json`:

```json
{
  "dependencies": ["."],
  "graphs": {
    "demo_graph": "graph.py:graph",
    "hierarchy_graph": "./src/agent/hierarchy_graph_demo.py:hierarchy_graph"
  },
  "env": ".env"
}
```

---

## Creating Subgraphs

Subgraphs (graphs within a graph) allow composing complex workflows. The Hierarchy Demo graph uses the Issue Resolve graph as a subgraph:

```python
# hierarchy_graph_demo.py
from langgraph.graph import END, START, StateGraph
from agent.supervisor_graph_demo import issue_resolve_graph

builder = StateGraph(CustomState)
builder.add_node(
    "issue_resolve_graph",
    issue_resolve_graph,
    destinations=({"mam_node": "mam_node-issue_resolve_graph"}),
)
builder.add_node(
    "mam_node",
    mam_node,
    destinations=(
        {
            "reviewer_node": "mam_node-reviewer_node",
            "issue_resolve_graph": "mam_node-issue_resolve_graph",
        }
    ),
)
builder.add_node(
    "reviewer_node",
    reviewer_node,
    destinations=({"mam_node": "mam_node-reviewer_node"}),
)
builder.add_edge(START, "mam_node")
builder.add_edge("issue_resolve_graph", "mam_node")
hierarchy_graph = builder.compile()
```

A ReACT agent subgraph consists of:
- `__start__`: Start of the subgraph
- `agent`: The agent node
- `tools`: The tools bound to the agent
- `__end__`: End of the subgraph

Conditional edges (dashed lines) mean the agent decides on its own whether to transition based on criteria (e.g., whether it needs tool use).

More info: [LangGraph Documentation: How to use subgraphs](https://langchain-ai.github.io/langgraph/how-tos/subgraph/)

---

## File-Level Documentation

For detailed descriptions of each file's role and responsibilities, see [CLAUDE.md](../CLAUDE.md).

### Key Files

| File | Description |
|------|-------------|
| `src/agent/constant.py` | Constants: `RUNTIME_DIR`, `PATCH_RESULT_DIR`, tree-sitter parsers |
| `src/agent/github_utils.py` | GitHub API utilities (`get_issue_description`, `get_issue_close_commit`) |
| `src/agent/llm.py` | LLM provider factory based on env vars |
| `src/agent/runtime_config.py` | Singleton runtime config, clones repos from issue URLs |
| `src/agent/state.py` | Custom LangGraph state definitions |
| `src/agent/tool_set/context_tools.py` | Vector DB creation, embedding search, summarizer |
| `src/agent/tool_set/edit_tool.py` | File editing via OHEditor (adapted from [OpenHands](https://github.com/All-Hands-AI/openhands-aci)) |
| `src/agent/tool_set/sepl_tools.py` | Core tools: `view_directory`, `view_file_content`, `run_shell_cmd`, `git diff` |
