# Agentic Context Management and Trajectory Retrieval Architecture

Yes, a subagent can return its entire trajectory to the lead agent, although the default behavior is designed to prevent this to maintain **context quarantine** 

### Default vs. Customized Output

* **Default (Summarization):** By default, subagents are designed to perform their work (multiple tool calls, internal reasoning, and raw data processing) in an isolated context window 1, 3. Once finished, they return only a **single, comprehensive summary** or final message to the lead agent 1, 4. This ensures the lead agent's context remains focused on high-level planning rather than being "polluted" by intermediate "fluff"
* **Custom Trajectory:** You can customize subagent outputs to **return the entire message history** to the main agent 2. This is useful if the lead agent requires a deep understanding of the subagent's specific steps or trajectory to effectively handle the broader objective 2.

### Alternative: Offloading to the Virtual Filesystem

If you want the lead agent to have access to the full trajectory without immediately bloating its active context window, you can use the **virtual filesystem** 

* **Scratchpad Storage:** The subagent can write its detailed findings, notes, or full execution logs to a file in the virtual filesystem (the agent's internal state)  
* **On-Demand Retrieval:** The subagent then returns a **short summary** (e.g., 200 tokens) containing a reference to that file 1. The lead agent can then choose to use read\_file to pull in the full trajectory only if it determines that information is necessary for the next step of the plan

This flexible architecture allows you to balance the need for **detailed observability** with the necessity of **context management** in long-running agentic tasks   


# Architectural Framework for Concurrent Subagent Orchestration

The lead agent launches subagents concurrently by leveraging a specific tool and an asynchronous architecture designed to handle multi-step planning and parallel task execution.

### The task Tool and Orchestration

The primary mechanism for launching subagents is the built-in **task tool** (also referred to as a single dispatch tool) 1-3. The system instructions for this tool explicitly direct the lead agent to **"launch multiple agents concurrently whenever possible"** to improve efficiency 4-6.  
The process typically follows these steps:

* **Planning:** The lead agent first uses the write\_todos tool to decompose a complex query into discrete tasks 6-8.  
* **Delegation:** Once tasks are identified, the lead agent emits multiple calls to the task tool, specifying a subagent\_type and a description for each independent task 9-11.  
* **Context Isolation:** Each subagent is launched with an isolated context window—a process called **"context quarantine"**—receiving only the specific instructions for its task to prevent raw data from polluting the main agent's memory 6, 12-14.

### Asynchronous Execution and "Wait" Behavior

Subagents can be invoked in either a synchronous or asynchronous manner:

* **Synchronous:** The lead agent is blocked and must wait for all subagent results before continuing
* **Asynchronous:** Subagents run as **background tasks**, allowing the lead agent to remain responsive and continue its own reasoning or task management while they work independently 6, 15-17. This architecture is generally preferred for strict latency requirements.

### State Management: The File Reducer

To handle multiple subagents writing results at the same time, the system uses a specialized **"file reducer"** within the agent's state.

* **Merging Data:** If two agents (Agent A and Agent B) finish their tasks simultaneously and write to the virtual filesystem, the reducer **merges their dictionaries**.
* **Preventing Loss:** This ensures that files created by different agents are combined rather than overwritten, allowing the lead agent to access all findings once the sub-runs are complete.

### Real-Time Monitoring

From a UI perspective, the **useDeepAgentStream** hook allows developers to render these parallel activities in separate components or "cards". This enables users to see the status, ool calls, and individual streams of multiple subagents (e.g., a "weather scout" and a "budget optimizer") as they work toward a final combined response.  

