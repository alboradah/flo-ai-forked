# How Routers Are Used in `flo_ai`

## Introduction

This document explains the practical application, creation, and management of Routers within the `flo_ai` framework. It builds upon the general understanding of what Routers are by detailing how they are instantiated and utilized in code, based on analysis of the `flo_ai` codebase and examples.

## Core Concept: The `Flo` Class

The primary way users interact with routed agent teams in `flo_ai` is through the `Flo` class, found in `flo_ai.core`. An instance of the `Flo` class encapsulates:
1.  A `FloSession`: Provides shared resources like LLMs, tools, and configurations.
2.  An `ExecutableFlo`: This is typically a `FloRoutedTeam` object, which is the compiled execution graph built by a router.

The `Flo` class provides convenient methods like `invoke()` and `stream()` to run the underlying routed team.

## Router Creation Strategies

There are two main strategies for creating and configuring routers, both usually culminating in a `Flo` object:

### 1. YAML-Based Configuration (Recommended & Common)

This is the most common approach highlighted in the examples. It involves defining the team structure, agents, and router configuration in a YAML file or string.

-   **Instantiation**:
    ```python
    from flo_ai import Flo, FloSession
    from langchain_openai import ChatOpenAI

    llm = ChatOpenAI() # Or your specific LLM
    session = FloSession(llm)
    # Register tools if needed: session.register_tool(...)

    yaml_definition = """
    apiVersion: flo/alpha-v1
    kind: FloRoutedTeam
    name: my-processing-team
    team:
        name: MyTeam
        router:
            name: my-router
            kind: linear # or 'llm', 'supervisor'
            # ... other router-specific configs (e.g., 'job' for llm/supervisor)
        agents:
            # ... agent definitions ...
    """
    flo_instance = Flo.build(session, yaml=yaml_definition)
    # Or from a file:
    # flo_instance = Flo.build(session, yaml_path="path/to/your/config.yaml")
    ```
-   **Behind the Scenes**:
    -   `Flo.build()` uses `flo_ai.builders.yaml_builder.build_supervised_team()`.
    -   The `yaml_builder` parses the YAML and uses `flo_ai.router.FloRouterFactory.create()` to instantiate the correct router (e.g., `FloLinear`, `FloLLMRouter`) based on the `kind` specified in the YAML's `team.router.kind` field.
    -   The factory then calls the router's `build_routed_team()` method to get the `FloRoutedTeam` (an `ExecutableFlo`).
-   **Examples**: `flo_ai/examples/python/linear_router_team.py`, `flo_ai/examples/linear_router_example.ipynb`, and `flo_ai/examples/llm_router_example.ipynb` all demonstrate this pattern.

### 2. Programmatic Creation with `Flo` Wrappers

This approach is used when you want to define the team structure, agents, and routers directly in Python code.

-   **Instantiation**:
    ```python
    from flo_ai import Flo, FloSession, FloAgent
    from flo_ai.router import FloLinear # Or FloLLMRouter, FloSupervisor
    from flo_ai.models import FloTeam
    from langchain_openai import ChatOpenAI

    llm = ChatOpenAI()
    session = FloSession(llm)

    # 1. Create agents (e.g., using FloAgent.Builder or loading from YAML snippet)
    #    For simplicity, let's assume agent1, agent2 are FloAgent instances
    # agent1 = FloAgent.Builder(session, name="MyAgent1", ...).build()
    # agent2 = FloAgent.Builder(session, name="MyAgent2", ...).build()

    # 2. Create a FloTeam
    # my_team = FloTeam.Builder(session, "MyProgrammaticTeam", members=[agent1, agent2]).build()

    # 3. Create a FloRouter instance programmatically
    # linear_router = FloLinear.create(session, name="MyLinearRouter", team=my_team)
    # supervisor_router = FloSupervisor.create(session, name="MySupervisor", team=my_team, llm=session.llm)

    # Let's assume 'my_router_instance' is a configured FloRouter object (FloLinear, FloLLMRouter etc.)
    # flo_instance = Flo.create(session, routed_team=my_router_instance)

    # Alternatively, using Flo.build for a programmatically created router:
    # flo_instance = Flo.build(session, routed_team=my_router_instance)

    # If you have a single agent and want to run it via Flo:
    # single_agent_instance = FloAgent.Builder(session, ...).build()
    # flo_instance_for_agent = Flo.create(session, routed_team=single_agent_instance)
    # This will wrap single_agent_instance in a FloAgentRouter.
    ```
-   **How it works**:
    -   `Flo.create(session, routed_team=...)` or `Flo.build(session, routed_team=...)` takes your programmatically created `FloRouter` instance.
    -   It then calls `your_router_instance.build_routed_team()` to get the `FloRoutedTeam` executable.
    -   If you pass a `FloAgent` directly to `Flo.create()`, it automatically wraps it in a `FloAgentRouter`.
-   **Note**: While individual router classes (`FloLinear`, `FloLLMRouter`, etc.) have public `Builder()` and static `create()` methods, using them in conjunction with `Flo.create()` or `Flo.build()` is the standard way to get a runnable `Flo` object.

## Role of `FloSession`

The `FloSession` object is crucial and serves as a central hub for shared resources and configurations:

-   **LLMs**: It holds the default LLM and can store other named LLMs (`session.register_model()`). Routers (especially `FloLLMRouter` and `FloSupervisor`) and agents access LLMs through the session.
-   **Tools**: Tools used by agents are registered with the session (`session.register_tool(name="MyTool", tool=MyToolInstance)`). Agents defined in YAML can then refer to these tools by name.
-   **Configuration**: The session can prepare run configurations (e.g., callbacks) via `session.prepare_config()`.
-   **Logging & Callbacks**: It manages logging and callbacks for tracing execution.

The session is passed when creating a `Flo` instance and is utilized by various components during both the building phase (e.g., `yaml_builder`, `FloRouterFactory`) and the execution phase.

## Execution of Routed Teams

Once you have a `Flo` instance (`flo_instance`), you execute the routed team using one of its methods:

-   `flo_instance.invoke(input_data, config=None)`: Runs the graph and returns the final output.
-   `flo_instance.stream(input_data, config=None)`: Returns an iterator for streaming intermediate steps and the final output.
-   `ainvoke()` and `astream()`: Asynchronous versions of the above.

**Execution Flow**:
1.  User calls `flo_instance.invoke("User query")`.
2.  The `Flo` object's `invoke` method calls the `invoke` method of its internal `self.runnable` (which is the `FloRoutedTeam` instance).
3.  The `FloRoutedTeam` (being an `ExecutableFlo`) wraps the input `work` into the format `{"messages": [HumanMessage(content=work)]}`.
4.  It then calls the `invoke` method of the actual compiled `langgraph.CompiledGraph` it holds.
5.  The LangGraph engine executes the graph according to the router's logic and agent definitions.

## Visualization

The `Flo` class provides methods to visualize the compiled agent graph:
-   `flo_instance.draw()`: Displays the graph image (e.g., in a Jupyter Notebook).
-   `flo_instance.draw_to_file("graph.png")`: Saves the graph image to a file.
This is useful for understanding the structure defined by your router and agents.

## Overall Flow Summary (Conceptual)

1.  **Setup**:
    -   **YAML**: Define team, agents, router (`kind`, etc.) in YAML.
        -> `Flo.build(session, yaml=...)`
    -   **Programmatic**: Create `FloAgent`s, `FloTeam`, `FloRouter` (e.g., `FloLinear.create(...)`) in Python.
        -> `Flo.create(session, routed_team=your_router)` or `Flo.build(session, routed_team=your_router)`
2.  **Instantiation (inside `Flo.build`/`create`)**:
    -   `FloSession` provides resources (LLMs, tools).
    -   If YAML: `yaml_builder` -> `FloRouterFactory` -> Specific Router (`FloLinear`, `FloLLMRouter`, etc.).
    -   The chosen router's `build_routed_team()` is called -> `FloRoutedTeam` (an `ExecutableFlo`) is created.
3.  **`Flo` Object Ready**: A `Flo` instance now holds the `FloSession` and the executable `FloRoutedTeam`.
4.  **Execution**:
    -   User calls `flo_instance.invoke("query")` or `flo_instance.stream("query")`.
    -   `Flo` obj. -> `ExecutableFlo.invoke/stream` -> `langgraph.CompiledGraph.invoke/stream`.
    -   Router logic dictates agent execution sequence.

This flow shows how `flo_ai` provides both declarative (YAML) and programmatic ways to define complex agent behaviors, with routers managing the interactions, all orchestrated through the `Flo` class and `FloSession`.
