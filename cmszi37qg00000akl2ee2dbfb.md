---
title: "LLM Integration: Multiple Providers, Runtime Switching, and Model Comparison"
datePublished: 2026-08-19T02:56:54.792Z
cuid: cmszi37qg00000akl2ee2dbfb
slug: llm-integration-multiple-providers-runtime-switching-and-model-comparison

---

The demo's checklist has one item left: **LLM integration** — connect with different providers (Anthropic, Google, local models via Ollama), experiment with different models, and compare their performance. Previous rounds built a dependable yardstick (`/eval` with golden datasets) and a single, interface-based chat model wired into every AI service. This round exploits both: put a *registry* behind that one interface, and reuse the evaluation harness to compare models head-to-head.

## The Problem: One Bean, Many Providers

Until now every AI service (assistant, RAG, agent, few-shot, judge) injected a `ChatModel` bean built from OpenAI only. Adding providers the obvious way — more `@Bean` methods, more constructor parameters — would ripple through the whole app. Instead, LangChain4j models all share one interface, `ChatModel`. So we can keep exactly one `ChatModel` bean and make it a *delegating registry*: whichever `provider:model` is selected is the one that actually answers every service's calls.

### LlmProvider

A small enum encodes the four providers and each one's default model:

```java
public enum LlmProvider {
    OPENAI("openai", "gpt-4o-mini"),
    ANTHROPIC("anthropic", "claude-haiku-4-5-20251001"),
    GEMINI("gemini", "gemini-2.5-flash"),
    OLLAMA("ollama", "llama3.2");
    // label() + defaultModelName() + fromLabel(String)
}
```

### ModelRegistry

The registry implements `ChatModel`, holds the current selection, and lazily builds the real model on first use — cached per `provider:model` so switching back is free and starting the app never requires a key for a provider you aren't using:

```java
public ChatModel currentChatModel() {
    return models.computeIfAbsent(key(currentProvider, currentModelName),
            ignored -> buildChatModel(currentProvider, currentModelName));
}

private ChatModel buildChatModel(LlmProvider provider, String modelName) {
    return switch (provider) {
        case OPENAI    -> OpenAiChatModel.builder().apiKey(System.getenv("OPENAI_API_KEY")).modelName(modelName).build();
        case ANTHROPIC -> AnthropicChatModel.builder().apiKey(System.getenv("ANTHROPIC_API_KEY")).modelName(modelName).build();
        case GEMINI    -> GoogleAiGeminiChatModel.builder().apiKey(System.getenv("GOOGLE_AI_GEMINI_API_KEY")).modelName(modelName).build();
        case OLLAMA    -> OllamaChatModel.builder().baseUrl(ollamaBaseUrl).modelName(modelName).build();
    };
}
```

Each provider is one case. The registry also reports which models are *available* — every provider whose key is present in the environment, plus Ollama, which is local. That list drives the comparison.

Because the registry is a `ChatModel` and the *only* `ChatModel` bean, `AiConfig` shrinks: the old OpenAI-specific bean disappears and services keep injecting `ChatModel` unchanged. Switching the registry switches the entire pipeline at once — including the LLM-as-a-judge metric used by `/eval`.

## Runtime Switching in the CLI

`/model` now covers both kinds of model. Chat switching accepts a provider with or without a model name:

```plaintext
/model
  Chat model     : openai:gpt-4o-mini
  Embedding model: text-embedding-3-small

/model chat anthropic
  Switched chat model to 'anthropic:claude-haiku-4-5-20251001'. Every AI service now uses this model.

/model chat gemini:gemini-2.5-flash
```

A typo prints the available models with their status (`ready`, `no api key`, `local`), so the CLI doubles as a discovery tool.

## Comparing Models: `/eval compare`

The previous round's `EvaluationService` evaluates a golden dataset against an answer provider. Cross-model comparison is just running that evaluation once per available model and collecting the averages:

```java
for (String model : modelRegistry.availableModels()) {
    modelRegistry.setModel(model);
    EvaluationReport report = evaluationService.evaluate(dataset, provider);
    rows.add(new ModelScore(model, report.averageScores()));
}
```

The original selection is restored in a `finally` block — switching models is a side effect the caller shouldn't be left with. The CLI prints a table:

```plaintext
=== Model comparison: chat ===
Model                                exact  contains  f1  rougeL  embed  judge
openai:gpt-4o-mini                    0.33      0.67 ...
anthropic:claude-haiku-4-5-20251001   0.67      1.00 ...
ollama:llama3.2                       0.33      0.67 ...
```

This makes the third checklist item ("compare performance and capabilities") concrete: the same golden questions, the same metrics, one row per model.

## Testing It Offline

Eight new tests, all offline and deterministic:

*   **Registry routing** — the registry has a test-only constructor pre-populated with `FakeChatModel`s keyed by `provider:model`. Tests assert the initial selection, `provider` and `provider:model` specs both switch correctly, unknown providers are rejected, and `doChat` delegates to the selected model.
    
*   **Comparison end-to-end** — a real `EvaluationService` (with `FakeEmbeddingModel` for the embed metric and a fake judge) runs against a two-model registry. Tests verify one row per model, all six metric keys in range, that the answer provider observed each model being selected, and that the original selection is restored afterwards.
    
*   **Key-free startup** — a real constructor test switches to Ollama and builds a model without any API key, proving startup never hard-requires keys.
    

## Design Notes

*   **One interface, many implementations.** Because LangChain4j models are interface-based, "support multiple providers" collapsed into a registry over a single bean type.
    
*   **Lazy builds, cached models.** No provider is instantiated until used; switching back after a compare costs nothing.
    
*   **The judge is a model too.** Cross-model comparison keeps its own metric, the LLM-as-a-judge, honest: the judge is the current model, so every row was scored by that model's own judgment.
    
*   **Deterministic test constructor.** A package-private constructor accepting a pre-populated model map makes the registry testable without network or environment variables; `availableModels()` is pinned to that map in tests.
    

## Next Steps

The checklist is complete. The demo now spans memory, embeddings, RAG, agents, prompting, integration, tooling, evaluation, advanced features, and multi-provider LLM integration. The natural next experiment is advanced agent orchestration (LangChain4j's `agentic` module) — with a `ChatModel` registry that can point it at any provider.

## Resources

*   [LangChain4j Chat Models](https://docs.langchain4j.dev/tutorials/chat)
    
*   [LangChain4j Model Providers](https://docs.langchain4j.dev/integration/language-models)
    
*   [LLM Integration Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/194)
    

## Conclusion

The final checklist item turned out to be mostly plumbing: LangChain4j's shared `ChatModel` interface meant supporting OpenAI, Anthropic, Gemini, and Ollama was a registry of four builder cases behind one bean. Runtime switching gives the demo a way to *experiment* with models interactively, and `/eval compare` turns that experimentation into numbers — the same golden datasets and metrics from the evaluation round, now applied per model. The demo began as a single chat command and ends with a switchboard for every major LLM provider plus a local one, all measured by the same yardstick.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).