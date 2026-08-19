---
title: "Advanced Orchestration: An Agentic Crew and Streaming Function Calling"
datePublished: 2026-08-19T03:05:45.241Z
cuid: cmsziel1d00000akyb856ecv7
slug: advanced-orchestration-an-agentic-crew-and-streaming-function-calling

---

The previous round left the demo with one open idea in its "Future Experiments" list: **advanced agent orchestration** with LangChain4j's `agentic` module, plus **streaming function calling**. Everything before this ran agents through plain `AiServices` with `@Tool` methods — a single agent, one tool-calling round, one final answer. This round adds a real multi-agent system: a *supervisor* that decides which specialized sub-agent should handle a task, and a streaming AI service that can call tools while tokens are still flowing.

## The Two Experiments

1.  **The crew.** The `agentic` module (currently `1.18.0-beta28`) provides `AgenticServices.agentBuilder()` for standalone agents and `AgenticServices.supervisorBuilder()` for a supervisor that delegates to sub-agents. It isn't in the BOM, so `pom.xml` pins its own `-betaNN` version explicitly; it depends on `langchain4j` `1.18.0`, which the BOM's `1.18.1` overrides.
    
2.  **Streaming function calling.** `AiServices` already supports streaming services — a `TokenStream` returned from the method, tokens arriving via `onPartialResponse`. Binding `@Tool`s to a streaming service is no different from binding them to a blocking one. The two compose: `StreamingAgent.chat(...)` returns a `TokenStream`, and the tool is still available to the model mid-generation.
    

## CrewService: A Supervisor With Three Workers

`CrewService` wires a supervisor to three sub-agents, each bound to one of the demo's existing `@Tool`s — the calculator, the weather lookup, and the document research tool:

```java
CrewTaskAgent calculatorAgent = buildAgent(
        "Calculator", "Useful for arithmetic and any kind of math. Delegate calculations here.",
        chatModel, calculatorTool);
CrewTaskAgent weatherAgent = buildAgent(
        "Weather", "Useful for current weather in known cities. Delegate weather questions here.",
        chatModel, weatherTool);
CrewTaskAgent researchAgent = buildAgent(
        "Research", "Useful for questions about the indexed documents. Delegate document questions here.",
        chatModel, documentSearchTool);

this.supervisor = AgenticServices.supervisorBuilder()
        .name("Crew")
        .supervisorContext("You are the crew supervisor. Decide which agent is best suited for the task ...")
        .chatModel(chatModel)
        .chatMemoryProvider(memoryProvider)
        .subAgents(calculatorAgent, weatherAgent, researchAgent)
        .responseStrategy(SupervisorResponseStrategy.LAST)
        .maxAgentsInvocations(10)
        .build();
```

Two details worth highlighting:

*   **Every agent shares one** `ChatModel` — the `ModelRegistry` from the LLM-integration round. A sub-agent is just a `ChatModel` consumer, so switching providers with `/model chat` switches the whole crew in one move. The registry abstraction keeps paying for itself.
    
*   **Sub-agents must be typed.** The supervisor hands a sub-agent an `arguments` map whose keys are matched to the sub-agent's parameters *by name*. That only works when the sub-agent declares its input as a `@V` parameter on a typed method with a `@UserMessage` template:
    

```java
public interface CrewTaskAgent {

    @SystemMessage("You are a specialist agent. Complete the task delegated to you using the tools "
            + "available. Return the final answer and nothing else.")
    @UserMessage("You have been delegated the following task. Use the tools available to complete it.\n"
            + "Delegated task: {{task}}")
    @Agent
    String run(@V("task") String task);
}
```

It was tempting to build sub-agents as untyped `AgenticServices.agentBuilder()` with a `userMessageProvider` extracting the `task` key. That fails silently: the provider never receives the delegated map (it gets the default memory id instead), so the worker ends up answering the literal string "default". The typed contract makes the hand-off explicit — the supervisor's `{"task": ...}` argument becomes the `run(...)` argument, and the template builds the worker's prompt from it.

### How the supervisor actually works

The supervisor does **not** hand the model a JSON-tools list of its sub-agents. Its `SupervisorPlanner` asks the model for a decision in a specific shape — an `AgentInvocation` POJO with `agentName` and `arguments` — and the runtime invokes the matching sub-agent:

*   Model returns `{"agentName": "Weather", "arguments": {...}}` → the Weather sub-agent runs, calls the model itself (and, if needed, its `getWeather` tool).
    
*   Model returns `{"agentName": "done"}` → the planner stops; with `SupervisorResponseStrategy.LAST`, the final answer is the last sub-agent's output.
    

That protocol is what made the tests deterministic (more below).

## Streaming Function Calling

`StreamingAgent` is a one-method AI service:

```java
public interface StreamingAgent {
    TokenStream chat(@MemoryId String memoryId, @UserMessage String message);
}
```

Its bean binds the existing `OpenAiStreamingChatModel` and the `CalculatorTool`:

```java
AiServices.builder(StreamingAgent.class)
        .streamingChatModel(streamingChatModel)
        .chatMemoryProvider(createChatMemoryProvider(...))
        .tools(calculatorTool)
        .build();
```

The `/stream` command prints tokens as they arrive; the model can still decide to call `calculate` before producing its final text.

## The CLI Commands

Two new commands complete the surface:

```plaintext
/crew <task>     Execute a task with the agentic supervisor crew
/stream <task>   Stream a task with streaming function calling
```

```java
TokenStream tokenStream = streamingAgent.chat(memoryId, task);
tokenStream
        .onPartialResponse(System.out::print)
        .onCompleteResponse(response -> System.out.println())
        .onError(error -> System.out.println("Stream error: " + error.getMessage()))
        .start();
```

## Testing It Offline

Three new tests, all offline and deterministic:

*   **Crew delegation protocol.** A `ScriptedSupervisorChatModel` returns the three model replies in order: `{"agentName":"Weather",...}` (supervisor delegates), then a plain sub-agent answer, then `{"agentName":"done"}` (supervisor wraps up). The test asserts the returned answer is the sub-agent's text and that the supervisor made at least three calls.
    
*   **Delegated task delivery.** The recorded second request — the Weather sub-agent's call — must contain the delegated task ("weather report") in its user message, proving the supervisor's `arguments` really reach the worker's prompt.
    
*   **Sub-agent tool exposure.** The same recorded second request must carry the `getWeather` tool specification, proving the `@Tool` really is bound inside the crew.
    
*   **Streaming tokens + tools.** A `FakeStreamingChatModel` (extended with `lastRequest()`) emits three tokens; the test collects them through `onPartialResponse`, asserts the full text on `onCompleteResponse`, and verifies the request's tool specifications contain `calculate`.
    

Full suite: **106 tests**, all green.

## Design Notes

*   **The supervisor is a small protocol, not a tool call.** Understanding that the model's output is parsed into an `AgentInvocation` was the key to both the implementation and the tests. Get the shape wrong and you get `OutputParsingException: Failed to parse ... into AgentInvocation`.
    
*   **Arguments are matched to sub-agent parameters by name.** That is why the worker must be typed with `@V("task")`; a generic untyped `userMessageProvider` never receives the delegated arguments. The regression test asserting the delegated text in the worker's prompt would have caught this earlier.
    
*   `LAST` **keeps the loop short.** `SupervisorResponseStrategy.LAST` returns the delegated result directly instead of scoring or summarizing it, which also makes the test's call sequence stable.
    
*   **Shared model = coherent crew.** Because sub-agents take a `ChatModel`, the whole crew inherits runtime provider switching for free.
    

## Next Steps

The checklist and the "Future Experiments" list are now empty: memory, embeddings, RAG, agents, prompting, integration, tooling, evaluation, advanced features, multi-provider LLM integration, and now agentic orchestration plus streaming function calling. A natural follow-up is giving the crew more sub-agents (e.g., an image or audio worker) and a plan-then-execute workflow — the `agentic` module keeps growing.

## Resources

*   [LangChain4j agentic module](https://github.com/langchain4j/langchain4j/tree/main/langchain4j-agentic)
    
*   [LangChain4j Streaming](https://docs.langchain4j.dev/tutorials/ai-services)
    
*   [Advanced Orchestration Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/220)
    

## Conclusion

The last item on the board turned out to be the most mechanically involved. The `agentic` module moves orchestration out of your prompt and into typed primitives: agents as builders, delegation as a parsed decision, and a supervisor loop that knows when to stop. Combined with streaming function calling, the demo now shows both ends of the spectrum — a single streaming tool-calling round and a multi-agent crew that plans its own hand-off. And because everything shares the `ModelRegistry`, one command switches the entire crew to a different provider or model.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).