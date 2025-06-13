# Flo AI Router Functionality Explanation

## Introduction

In the `flo_ai` framework, Routers play a crucial role in orchestrating and managing the flow of execution among different AI agents or teams of agents. They act as controllers that determine which agent should perform a task, in what order, and how information is passed between them. This allows for the creation of complex, multi-agent systems that can collaborate to achieve a common goal.

Under the hood, `flo_ai` Routers leverage the `langgraph` library to define and compile these execution flows as graphs. Each agent or sub-team can be represented as a node in the graph, and the router defines the edges and logic that dictate the path of execution through these nodes.

## Router Types

`flo_ai` provides several types of routers, each with a different strategy for managing execution flow.

### 1. `FloLinear` Router

The `FloLinear` router executes a sequence of agents or teams in a strictly predefined, linear order. You define the order of members (agents/teams), and the `FloLinear` router ensures they are called one after another.

**Key Features:**
- **Sequential Execution:** Agents are executed in the exact order they are listed.
- **Handles Special Nodes:**
    - `Delegator Nodes`: If a member is a delegator node (designed to pass control to a specific subset of further agents), the `FloLinear` router correctly wires it to the appropriate next agent(s) and then continues the main sequence.
    - `Reflection Nodes`: If a member is a reflection node (designed to potentially loop back to itself or a previous agent for refinement), the `FloLinear` router incorporates this looping logic.
- **No LLM Required:** This router does not use a Language Model for decision-making, as the path is fixed.

**Use Case:**
Ideal for workflows where the sequence of tasks is static and known in advance. For example, a data processing pipeline where data ingestion, transformation, and then storage always happen in that specific order.

### 2. `FloLLMRouter`

The `FloLLMRouter` uses a Language Model (LLM) to dynamically decide which agent (or a special `FLO_FINISH` signal) should be executed next. This allows for more flexible and intelligent routing based on the ongoing conversation or state.

**Key Features:**
- **LLM-Powered Decisions:** At each routing step, an LLM is consulted to determine the next action.
- **Prompt-Driven:** The LLM's decision-making is guided by a `ChatPromptTemplate`. This template typically includes:
    - The overall goal or context.
    - The list of available agents/members it can choose from.
    - The option to `FLO_FINISH` if the task is complete.
    - The history of the conversation or current state.
- **JSON Output Parsing:** The router expects the LLM to return its decision in a JSON format, typically specifying the name of the `next` agent to call.
- **Central Router Node:** In the execution graph, members are nodes, and control typically returns to the `FloLLMRouter` node after a member executes. The router then queries the LLM to decide the subsequent step.

**Use Case:**
Suitable for scenarios where the workflow is not fixed and requires adaptive routing. For instance, in a customer support system where the next step depends on the nature of the user's query, or in complex problem-solving where the system needs to decide which specialist agent to consult based on intermediate results.

### 3. `FloSupervisor` Router

The `FloSupervisor` router is a specialized version of the `FloLLMRouter`. It also uses an LLM to make routing decisions but is specifically designed for scenarios where a "supervisor" AI manages a team of "worker" agents.

**Key Features:**
- **LLM-Based Supervision:** Like `FloLLMRouter`, it relies on an LLM to choose the next agent or decide to `FLO_FINISH`.
- **Tailored System Prompt:** The key difference lies in its system prompt, which is crafted to instruct the LLM to act as a supervisor. This prompt typically frames the task as managing a conversation or workflow between a set of workers to fulfill a user request.
- **Collaborative Task Management:** It's designed to oversee the collaborative efforts of multiple agents, directing their work towards a common objective.

**Use Case:**
Best suited for building hierarchical agent teams where one LLM-powered agent (the supervisor) orchestrates the work of other agents. For example, a research assistant team where a supervisor agent breaks down a complex research question and assigns sub-tasks to specialized research agents, then synthesizes their findings.

### 4. `FloAgentRouter`

The `FloAgentRouter` is a simpler router type, primarily designed to wrap a single `FloBaseAgent` instance. It essentially allows an individual agent to be treated as a "team" with a very straightforward routing logic.

**Key Features:**
- **Single Agent Focus:** It manages a graph that typically contains just one operational agent.
- **Basic Graph:** The execution graph built by `FloAgentRouter` is simple: `START` -> `AGENT_NODE` -> `END`.
- **Consistency:** It provides a way to ensure that even single agents can be invoked using the same routed team mechanics as more complex multi-agent teams.

**Use Case:**
Useful when you need to incorporate a single agent into a system that expects a `FloRoutedTeam` interface. It ensures consistency in how components are structured and executed, even if the "team" consists of only one member. This can also be helpful for future scalability, where a single agent might later be expanded into a team.

## Key Concepts Summary

Regardless of the specific type, `flo_ai` Routers are built upon several core concepts:

- **Graph-Based Execution:** Routers define execution flows as directed graphs (using `langgraph`), where nodes are agents or sub-teams, and edges represent the transitions between them.
- **State Management:** The state of the ongoing process (e.g., conversation history, intermediate results) is typically managed within a `TeamFloAgentState` object, which is passed between nodes in the graph.
- **`FloRouterFactory`:** This factory class is responsible for instantiating the appropriate router type (`linear`, `llm`, `supervisor`, `agent`) based on configuration. This promotes a clean separation of concerns and makes it easy to switch between different routing strategies.
- **Flexibility and Extensibility:** The router system is designed to be flexible, allowing developers to choose the routing logic that best fits their application's needs, from simple linear sequences to complex LLM-driven decisions.
