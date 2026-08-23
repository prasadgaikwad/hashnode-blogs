---
title: "LangGraph4j ReACT Agent: Explicit State Graphs for Tool Orchestration"
datePublished: 2026-08-23T18:32:04.988Z
cuid: cmt6599b200010ajfdcop3fdm
slug: langgraph4j-react-agent-explicit-state-graphs-for-tool-orchestration

---

When building LLM applications, there's a spectrum between "just call tools automatically" and "I need to see and control every step of the reasoning loop." LangChain4j's built-in tool calling does the former. LangGraph4j does the latter — and the difference is more useful than you'd think.

## The ReACT Pattern

ReACT (Reason + Act) is a simple loop: the LLM reasons about what to do, picks a tool, observes the result, and decides whether to continue or answer. Most frameworks hide this loop behind a single `chat()` call. LangGraph4j exposes it as a first-class state graph.

The graph structure:

```plaintext
__START__ → agent (LLM reasons) → action (tool executes) → agent → ... → __END__
```

Each node is named, traceable, and inspectable. You can see exactly when the LLM decided to call a tool, what tool it called, what result it got, and how it decided to stop.

## What We Built

We integrated LangGraph4j's `AgentExecutor` into the demo project as a new orchestration endpoint alongside the existing agentic-services patterns (supervisor, chain, parallel, loop, conditional).

```java
@Service
public class ReactAgentService {

    private final CompiledGraph<AgentExecutor.State> compiledGraph;

    public ReactAgentService(ChatModel chatModel, 
                             CalculatorTool calculatorTool,
                             DocumentSearchTool documentSearchTool,
                             WeatherTool weatherTool,
                             EmbeddingStoreStatsTool storeStatsTool) throws GraphStateException {
        StateGraph<AgentExecutor.State> graph = AgentExecutor.builder()
                .chatModel(chatModel)
                .toolsFromObject(calculatorTool, documentSearchTool, weatherTool, storeStatsTool)
                .build();
        this.compiledGraph = graph.compile();
    }

    public ReactResult run(String task) {
        // Stream to capture each graph node transition
        var generator = compiledGraph.stream(Map.of("messages", UserMessage.from(task)));
        for (var item : generator) {
            steps.add(item.node());  // "agent", "action", "agent", ...
        }
        
        // Get final state
        var finalState = compiledGraph.invoke(Map.of("messages", UserMessage.from(task)));
        String answer = finalState.get().finalResponse().orElse("No response");
        
        return new ReactResult(task, answer, steps, allMessages);
    }
}
```

## How It Differs from Built-in Tool Calling

|  | LangChain4j Tool Calling | LangGraph4j AgentExecutor |
| --- | --- | --- |
| Control | Implicit loop | Explicit state graph |
| Traceability | Final result only | Every node transition visible |
| Extension point | Limited | Full graph modification |
| Mental model | "Call this function" | "Build a state machine" |

The explicit graph approach means you can:

*   **Debug reasoning** — see exactly why the LLM chose tool A over tool B
    
*   **Add guardrails** — insert nodes between agent and action
    
*   **Build complex patterns** — conditional branching, human-in-the-loop (issues #236, #237)
    
*   **Monitor performance** — track which tools are called, how many loops, where bottlenecks occur
    

## The Pitfalls

LangGraph4j's API has some rough edges:

1.  `toolsFromObject()` **takes** `Object...` — not `List<Object>`. Easy to get wrong.
    
2.  `AsyncGenerator` **doesn't have** `forEachRemaining()` — use a for-each loop instead.
    
3.  `AgentExecutor.State` **messages include ALL intermediate steps** — the final state has the complete conversation history including every reasoning step, not just the initial prompt and final answer.
    

## Running It

```bash
# CLI
./mvnw spring-boot:run
/react compute 2+2

# REST
curl -X POST http://localhost:8080/api/react \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the weather in Tokyo and what is 15% of 340?"}'
```

The response includes the full trace:

```json
{
  "task": "What is the weather in Tokyo and what is 15% of 340?",
  "answer": "The weather in Tokyo is 22°C and partly cloudy. 15% of 340 is 51.",
  "steps": ["agent", "action", "agent", "action", "agent"],
  "agentMessages": ["I need to check the weather and calculate 15% of 340.", ...]
}
```

## What's Next

This is the foundation for two more patterns:

*   **Stateful Pipeline** — persist graph state across invocations, resume from checkpoint
    
*   **Human-in-the-Loop** — pause the graph for human approval before executing sensitive tools
    

The LangGraph4j integration is the most powerful orchestration pattern we've added — but also the most opinionated. For simple tool use, stick with `@AiService`. When you need visibility, control, and extensibility, reach for the graph.