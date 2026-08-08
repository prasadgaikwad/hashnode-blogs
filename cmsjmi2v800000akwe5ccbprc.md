---
title: "Chains and Agents with LangChain4j"
datePublished: 2026-08-08T00:16:07.969Z
cuid: cmsjmi2v800000akwe5ccbprc
slug: chains-and-agents-with-langchain4j

---

So far the demo's LLM calls have been single-shot: the chat model answers a question in one step. But many real tasks need more than that — a task may require arithmetic, a lookup in a knowledge base, or a sequence of steps. This post brings in **chains** (deterministic pipelines we compose in Java) and **agents** (LLM-driven loops that decide to call **tools**).

## Two Kinds of "Chains"

The word "chain" means two different things in the LLM world, and LangChain4j maps them to two different mechanisms:

1.  **Processing chains** — *we* decide the steps, in code. Input flows through fixed stages: preprocess, route, transform, postprocess. Deterministic and testable.
    
2.  **Agentic loops** — *the model* decides the steps, calling functions in a loop ("reason, call tool, observe result") until the task is done. Non-deterministic; correctness is delegated to the model.
    

LangChain4j implements the second one as **function calling** under the hood (OpenAI-style tool calls), and wraps it in `AiServices`. We implement the first one ourselves as plain Java composition.

## Tools: The Agent's Hands

A *tool* is an ordinary method annotated with `@Tool`. The chat model sees a description of each tool and its parameters, and decides when to call it. `AiServices` executes the call and feeds the result back to the model — automatically, in a loop, with no glue code from us.

Our agent gets three tools. First, an arithmetic calculator. The trickiest part is *safety*: we never want to evaluate arbitrary model-produced strings as code, so we wrote a small recursive-descent parser that only accepts digits, `+ - * /`, parentheses, and dots:

```java
@Component
public class CalculatorTool {

    @Tool("Calculates the result of an arithmetic expression using +, -, *, / and parentheses")
    public double calculate(@P("The arithmetic expression to evaluate, e.g. \"(1 + 2) * 3\"") String expression) {
        if (expression == null || expression.isBlank()) {
            throw new IllegalArgumentException("Expression must not be empty");
        }
        return new Evaluator(expression).evaluate();
    }
}
```

`@Tool` marks the callable function; `@P` documents a single parameter (LangChain4j turns these into the model's function schema). Second, a document-search tool that gives the agent access to the same embedded knowledge base the RAG pipeline uses:

```java
@Tool("Searches the indexed documents and returns the most relevant passages with their relevance scores")
public String searchDocuments(@P("The search query, e.g. \"what does the document say about RAG\"") String query) {
    List<EmbeddingMatch<TextSegment>> matches = searchService.search(query);
    if (matches.isEmpty()) {
        return "No matching documents found. Documents may need to be indexed first with /index.";
    }
    // ... format matches with scores
}
```

And third, a stats tool that reports the embedding store's model and size.

## The Agent: AiServices with Tools

An agent is just an AI Service interface whose proxy gets built with `.tools(...)`:

```java
public interface Agent {

    @SystemMessage("""
            You are an agent that accomplishes the user's task using the available tools.
            Use the "searchDocuments" tool when the task asks about the indexed documents or your own data.
            Use the "calculate" tool for arithmetic computations.
            If the task does not need a tool, answer directly. Be concise.
            """)
    String execute(@MemoryId String memoryId, @UserMessage String task);
}
```

Wired in `AiConfig` alongside the other AI services, reusing the same per-conversation memory provider:

```java
@Bean
public Agent agent(ChatModel chatModel,
                   CalculatorTool calculatorTool,
                   DocumentSearchTool documentSearchTool,
                   EmbeddingStoreStatsTool storeStatsTool,
                   ChatMemoryRegistry chatMemoryRegistry,
                   ...) {
    return AiServices.builder(Agent.class)
            .chatModel(chatModel)
            .chatMemoryProvider(createChatMemoryProvider(chatMemoryRegistry, modelName, maxMessages, maxTokens))
            .tools(calculatorTool, documentSearchTool, storeStatsTool)
            .build();
}
```

The system message is important: it tells the model *which* tool fits *which* kind of task. Without it the model still sees the tool schemas, but good guidance reduces wasted calls.

## The Chain: Deterministic Routing

`ChainService` is our processing chain — a pipeline whose stages we control:

```java
@Service
public class ChainService {

    private final Agent agent;
    private final CalculatorTool calculatorTool;

    public String ask(String memoryId, String task) {
        String normalized = task.trim();
        if (CalculatorTool.isArithmetic(normalized)) {
            return "Result: " + calculatorTool.calculate(normalized);
        }
        return agent.execute(memoryId, normalized);
    }
}
```

Stage 1 (preprocess + route): if the task is a *pure* numeric expression, resolve it locally — no model call, no tokens burned, and no chance of the LLM hallucinating a sum. Stage 2 (execute): everything else — including worded questions like "what is two plus two?" — goes to the agent, which can still decide to call the calculator itself.

This hybrid is a nice demonstration of why chains and agents are complements, not competitors: cheap, deterministic stages handle the obvious cases; the flexible, model-driven loop handles the rest.

## Trying It Out

```plaintext
/agent compute (20 * 5) - 8
Agent > Result: 92.0
```

Pure arithmetic never touches the model — the chain resolves it. Now a worded version that needs the agent *and* its document-search tool (after `/index sample-data`):

```plaintext
/agent what do the documents say about agents?
Agent > The documents describe agents as programs that use tools to complete tasks:
        "Agents use tools to complete tasks to achieve a goal." (from langchain4j.txt)
```

Behind the scenes the model called `searchDocuments`, saw the retrieved passages, and answered from them — the RAG loop, but driven by the model's own decision instead of a fixed augmentor.

## Testing Without a Live Model

The calculator and the chain are deterministic, so they get real unit tests — no mocks, no network:

```java
@Test
void respectsOperatorPrecedence() {
    assertThat(calculatorTool.calculate("2 + 3 * 4")).isEqualTo(14.0);
}

@Test
void routesArithmeticToTheCalculatorWithoutTheAgent() {
    ChainService chain = new ChainService((memoryId, task) -> "agent called: " + task, calculatorTool);
    assertThat(chain.ask("main", " 2 + 3 * 4 ")).isEqualTo("Result: 14.0");
}

@Test
void delegatesWordedTasksToTheAgent() {
    ChainService chain = new ChainService((memoryId, task) -> "agent handled \"" + task + "\"", calculatorTool);
    assertThat(chain.ask("main", "What can you do?")).isEqualTo("agent handled \"What can you do?\"");
}
```

Because `Agent` is an interface, the chain test substitutes a lambda — asserting both the routing *and* that the memory id is passed through, entirely offline. The document-search tool is tested with the same `FakeEmbeddingModel` we use elsewhere: index real text, call the tool, assert it returns ranked passages.

## Design Notes

*   `@Tool` **on methods,** `@P` **on parameters.** That's the whole declarative surface for function calling in LangChain4j. Parameters map to the OpenAI-style schema the model uses to format its calls.
    
*   **Tools are Spring beans.** The three tool classes are `@Component`s with their own dependencies (the search tool reuses `SemanticSearchService`), so they compose cleanly with the rest of the context.
    
*   **Memory composes for free.** The agent uses the same `ChatMemoryProvider` as the assistant and QA services, so it can hold a multi-turn, tool-using conversation with `@MemoryId` scoping.
    
*   **Safety matters.** Never evaluate strings the model returns as code. Our evaluator parses a grammar, rejects unknown characters, throws on malformed input and division by zero, and is covered by tests for exactly those failure modes.
    

## Next Steps

The natural follow-ups from our checklist: **prompting techniques** (few-shot examples, templates) and **evaluation** (benchmarking whether the agent's tool choices are good ones). There is also a new, explicitly experimental `langchain4j-agentic` module in LangChain4j with structured "agent definitions" — worth a look once the stable `AiServices` + `@Tool` path has proven itself here.

## Resources

*   [Official LangChain4j AI Services Documentation](https://docs.langchain4j.dev/tutorials/ai-services)
    
*   [LangChain4j Tools / Function Calling](https://docs.langchain4j.dev/tutorials/tools)
    
*   [Chains and Agents Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/198)
    

## Conclusion

Chains and agents are two complementary ways to make an LLM application *do things* instead of just *say things*. Deterministic processing chains give us fast, testable, token-cheap routing for the cases we can predict; `AiServices` with `@Tool` gives the model the ability to call our own code when it decides that's needed — whether that's arithmetic, searching the knowledge base, or anything else we want to expose as a function. Both compose with the memory and RAG infrastructure we already built, which is exactly the point of a demo that grows feature by feature.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).