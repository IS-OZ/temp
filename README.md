# ReAct Agent Flow Documentation

This document explains the functionality and working of the ReAct agent defined in `react.py`.

## Overview

The `AgentExecutor` class orchestrates a LangChain-based ReAct agent designed to process user queries, interact with various tools (SQL generation, fuzzy search, etc.), and return structured results. It acts as a coordinator (Swarm/Coordinator pattern) that manages the lifecycle of a user request.

## Components

- **AgentExecutor**: The main class that manages the agent execution lifecycle, memory, and result processing.
- **Main Agent**: A LangChain agent initialized with a specific LLM (`get_llm`), a set of tools, and a system prompt (`main_react_prompt`).
- **Tools**: A collection of tools available to the agent for performing specific tasks:
    - `relevancy_check`: Checks if the query is relevant to the domain.
    - `rewritten_query`: Rewrites the query for better understanding.
    - `entity_extraction`: Extracts entities from the query.
    - `fuzzy_search`: Performs fuzzy search on data.
    - `generate_sql`: Generates SQL queries from natural language.
    - `fuzzy_search_sql_generator`: Helper for SQL generation with fuzzy search.
    - `chart_and_blurb`: Generates charts and summaries.
    - `execute_sql`: Executes the generated SQL.

## Execution Flow

The following diagram illustrates the control flow within the `AgentExecutor.run` method:

```mermaid
flowchart TD
    Start([Start: run()]) --> UpdateMem[Update Memory\n(User Query, Context)]
    UpdateMem --> ConstructInput[Construct Agent Input String]
    ConstructInput --> InitCallbacks[Initialize Callbacks\n(ToolTracking, Langfuse)]
    
    subgraph AgentExecution [Agent Execution (astream)]
        InitCallbacks --> StreamStart[Start Agent Stream]
        StreamStart --> ReceiveChunk{Receive Chunk}
        
        ReceiveChunk -- New Chunk --> ProcessStep[Process Step/Message]
        ProcessStep --> Log[Log Step Details]
        
        Log --> CheckType{Content Type?}
        CheckType -- "text" --> ParseOutput[Parse Output JSON]
        ParseOutput --> AppendResult[Append to main_result]
        CheckType -- "other" --> ReceiveChunk
        AppendResult --> ReceiveChunk
        
        ReceiveChunk -- End of Stream --> CheckEmpty{main_result Empty?}
    end
    
    CheckEmpty -- Yes --> Error[Raise RuntimeError]
    CheckEmpty -- No --> PostProcess[Post-Processing]
    
    subgraph PostProcessGroup [Post-Processing]
        PostProcess --> CheckRel[Check Relevancy\n(Scan for 'irrelevant')]
        CheckRel --> GetReasoning[Extract Reasoning\n(Get 'reasoning' field)]
    end
    
    GetReasoning --> Return([Return Result Dictionary])
```

## Detailed Process

1.  **Initialization**:
    *   The `AgentExecutor` initializes the underlying LangChain agent using `create_main_agent_executor`.
    *   It sets up the LLM and the list of `main_tools`.

2.  **Execution (`run` method)**:
    *   **Context Setup**: The method accepts `user_query`, `previous_query`, `workspace_id`, etc. It updates the internal `self.memory` with these values.
    *   **Input Construction**: It constructs a formatted input string that includes the user query and context metadata.
    *   **Streaming Execution**: It calls `self.main_agent.astream` to execute the agent. This uses `stream_mode="updates"` to receive real-time updates from the agent's steps.

3.  **Step Processing Loop**:
    *   The executor iterates asynchronously over the chunks yielded by the agent.
    *   For each step, it logs the activity (tool calls, model thoughts).
    *   **Output Parsing**: If the step content is text, it attempts to parse it as JSON using `_tool_output_parser`. If JSON parsing fails, it treats the text as raw reasoning.
    *   **Result Collection**: Valid outputs are appended to `main_result` as tuples of `(tool_name, output)`.

4.  **Post-Processing**:
    *   **Validation**: It checks if `main_result` is empty and raises a `RuntimeError` if so.
    *   **Relevancy Check**: The `_check_relevancy` method scans the results. If any step indicates the query is "not relevant", the overall result is marked as irrelevant.
    *   **Reasoning Extraction**: The `_get_reasoning` method extracts the last valid reasoning string from the steps.

5.  **Return Value**:
    *   The method returns a dictionary containing:
        *   `main_result`: The list of tool outputs and thoughts.
        *   `memory`: The context memory.
        *   `relevant`: Boolean indicating if the query was relevant.
        *   `reasoning`: The final reasoning text from the agent.
