---
title: "Evaluation and Testing: Scoring LLM Output with Golden Datasets"
datePublished: 2026-08-14T02:18:12.001Z
cuid: cmssbi64m00000ai153wl8xyx
slug: evaluation-and-testing-scoring-llm-output-with-golden-datasets

---

A chat application that you can't measure is a wish — "it seems to work" scales poorly once there are dozens of features and you start swapping models. This round adds an **evaluation harness** to the demo: golden datasets for each capability, a set of scoring metrics, and a `/eval` command that grades a live system question by question. The design constraint was that everything must run **fully offline in tests** — no API keys, no network, deterministic results — while still using the *real* retrieval and AI-service machinery.

## The Setup: Golden Datasets

Evaluation needs a ground truth. Each `GoldenDataset` is a named list of `(question, expectedAnswer)` pairs, and we bundle three, one per capability:

```java
public record GoldenDataset(String name, List<GoldenQuestion> goldenQuestions) {
    public static GoldenDataset rag() {
        return new GoldenDataset("rag", List.of(
                new GoldenQuestion(
                        "How does LangChain4j keep conversation memory?",
                        "LangChain4j offers MessageWindowChatMemory and TokenWindowChatMemory."),
                new GoldenQuestion(
                        "How does semantic search find similar texts?",
                        "Semantic search embeds the query and finds stored vectors with the highest cosine similarity."),
                new GoldenQuestion(
                        "What is LangChain4j?",
                        "LangChain4j is a Java framework that simplifies building applications with LLMs.")
        ));
    }
    // sentiment() and chat() follow the same shape
}
```

The RAG dataset's expected answers are written against the bundled `sample-data/` documents, so the questions are genuinely answerable by retrieval alone. Each dataset is deliberately small — three samples — because small and fast means we can run it on every test run and in CI.

## The Metrics

`Metric` is a one-method interface returning a score in `[0, 1]`:

```java
public interface Metric {
    String name();
    double evaluate(String question, String expected, String actual);
}
```

The `Metrics` factory builds six of them:

| Metric | What it measures |
| --- | --- |
| `exact` | Normalized string equality (case- and punctuation-insensitive) |
| `contains` | Whether the expected answer appears inside the produced answer |
| `f1` | Token F1 over the shared words — punishes missing facts (recall) and hallucinated extras (precision) |
| `rougeL` | F-measure of the longest common word *subsequence* — order-aware, unlike `f1` |
| `embed` | Cosine similarity between the embeddings of expected and actual answers |
| `judge` | LLM-as-a-judge: the chat model rates faithfulness on 0–5, normalized to `[0,1]` |

The first four are pure string math. `embed` reuses the app's embedding service, so *"v1.0 vectors with highest cosine similarity"* scores high against *"embeds the query and finds stored vectors with the highest cosine similarity"* even though only some words match — that's exactly the semantic tolerance you want from a search system.

### LLM-as-a-judge

The judge metric is the only one that calls a model. It sends a strict system prompt plus the question, expected answer, and produced answer, then asks for a single integer 0–5:

```java
ChatResponse response = judge.chat(ChatRequest.builder()
        .messages(List.of(
                SystemMessage.from(JUDGE_SYSTEM_PROMPT),
                UserMessage.from("Question: " + question + "\n"
                        + "Expected answer: " + expected + "\n"
                        + "Produced answer: " + actual)))
        .build());
return parseJudgeScore(response.aiMessage().text()) / 5.0;
```

LLMs are notoriously bad at formatting, so `parseJudgeScore` doesn't trust the reply: it extracts the first run of digits with a regex and clamps into 0–5. "Score: 5/5", "5", and "I'd say 5!" all parse to `1.0`. A reply with no digits — say the model went on a tangent — scores `0.0` rather than crashing the run.

## EvaluationService: Score a Provider

`EvaluationService` ties it together. It takes an `AnswerProvider` — a plain `String answer(String question)` function — runs the dataset through it, and scores every answer with every metric:

```java
public EvaluationReport evaluate(GoldenDataset dataset, AnswerProvider provider, List<Metric> metrics) {
    // for each golden question:
    //   actual = provider.answer(question)
    //   scores = each metric.evaluate(question, expected, actual)
    // averages = per-metric mean, rounded to 2 decimals
}
```

The abstraction is what makes evaluation reusable. The `/eval` command binds each dataset to its real system with a one-line lambda:

```java
case "rag" -> {
    if (searchService.storeSize() == 0) { /* nudge user to /index first */ }
    provider = question -> qaService.ask(memoryId, question);   // full RAG pipeline
}
case "chat" -> provider = question -> assistant.chat(memoryId, question);
case "sentiment" -> provider = question -> fewShotAssistant.classify(question).name();
```

The default metric set — `exact, contains, f1, rougeL, embed, judge` — is wired in `EvaluationService.defaultMetrics()`.

## The Report

`/eval` prints a per-question breakdown and the averages:

```plaintext
=== Evaluation: sentiment ===
Metrics: exact, contains, f1, rougeL, embed, judge

[1] I absolutely loved this movie!
    Expected: POSITIVE
    Actual  : POSITIVE
    Scores  : exact=1.00  contains=1.00  f1=1.00  rougeL=1.00  embed=1.00  judge=1.00

[2] The service was okay, nothing special.
    Expected: NEUTRAL
    Actual  : NEUTRAL
    Scores  : exact=1.00  contains=1.00  f1=1.00  rougeL=1.00  embed=1.00  judge=1.00

Average: exact=1.00  contains=1.00  f1=1.00  rougeL=1.00  embed=1.00  judge=1.00
```

The multi-metric view is the point: `exact` and `contains` are brittle, `f1`/`rougeL` reward partial matches, `embed` tolerates rephrasing, and `judge` captures quality the string metrics can't. When one model replaces another, the averages shift in ways that tell you *which* failure mode changed — verbatim accuracy versus factual recall versus phrasing.

## Offline Testing: The Real Pipeline, Fake Models

The golden datasets double as tests. `EvaluationServiceTest` runs the **full RAG stack** with real components — `AiServices.builder` creating the `QaAssistant` proxy, `MessageWindowChatMemory`, a `DefaultRetrievalAugmentor` with the real `SemanticSearchContentRetriever`, a temp document indexed into a real `InMemoryEmbeddingStore` — and only the two models are fakes:

```java
QaAssistant qaAssistant = AiServices.builder(QaAssistant.class)
        .chatModel(fakeChatModel)
        .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
        .retrievalAugmentor(DefaultRetrievalAugmentor.builder()
                .contentRetriever(new SemanticSearchContentRetriever(searchService))
                .build())
        .build();

EvaluationReport report = evaluationService.evaluate(
        GoldenDataset.rag(), question -> qaService.ask("rag", question));
```

The fake chat model returns a canned answer for every prompt, so every RAG question scores `exact=1.0` — the test asserts the harness behaves end-to-end rather than testing the LLM (which you can't, offline). The metric math itself gets direct unit tests: ROUGE-L is order-aware (reversed word order scores lower than exact, but above zero), F1 handles empty-vs-empty as `1.0`, embedding similarity ranks identical > related > unrelated, and the judge parser clamps `"10"` down to `1.0` and `"no score"` down to `0.0`.

Two expectations in that first draft were flat-out wrong, which is the test suite working: cosine similarity of a vector with itself is `1.0000000000000002` (not exactly `1.0`), so the assertion became "close to 1.0 within 1e-6"; and my mental ROUGE-L on reversed word order was miscalculated — the LCS of `"a b c d"` and `"d c b a"` is 1, an F of 0.25, not the >0.4 I'd guessed. The tests caught both before the PR shipped.

## Design Notes

*   **Offline by default.** Every metric except the judge is deterministic string/vector math; tests swap in fake chat and embedding models, so the whole suite (now 73 tests) runs with no network and no key.
    
*   **Small golden sets.** Three questions per dataset is enough to catch regressions and keep runs sub-second. Growing to dozens makes the suite flaky and slow.
    
*   **Metrics as interchangeable scores.** A `List<Metric>` is passed to `evaluate`, so a future report could weight `judge` differently or drop `exact` entirely without touching the service.
    
*   **Defensive parsing.** The judge prompt says "reply with only a single integer" because models don't; the regex+clamp makes the metric robust to anything it actually returns.
    

## Next Steps

Evaluation gives us a yardstick, which is the precondition for the remaining checklist items: **advanced features** (function calling, multi-modal, structured output) and **LLM integration** (swapping in Anthropic/Google/Ollama and comparing models). With `/eval` in place, those comparisons will have numbers instead of vibes.

## Resources

*   [LangChain4j documentation](https://docs.langchain4j.dev)
    
*   [Evaluation and Testing Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/202)
    

## Conclusion

A golden-dataset harness turns "the demo seems fine" into six numbers per capability. The most instructive part of building it was deciding what *not* to trust: an LLM judge needs regex parsing, string metrics need normalization, and your own intuitions about ROUGE-L need tests. With `/eval` reporting exact, contains, F1, ROUGE-L, embedding similarity, and judge scores, the next model swap will be measured, not guessed.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).