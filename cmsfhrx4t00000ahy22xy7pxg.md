---
title: "Retrieval Augmented Generation (RAG) with LangChain4j"
datePublished: 2026-08-05T02:52:44.316Z
cuid: cmsfhrx4t00000ahy22xy7pxg
slug: retrieval-augmented-generation-rag-with-langchain4j

---

In the last few posts we gave our chatbot a memory and the ability to search documents by meaning. Now it's time to combine the two. A large language model knows a lot, but it doesn't know *your* documents. **Retrieval Augmented Generation (RAG)** fixes that: retrieve the most relevant chunks for a question, stuff them into the prompt, and let the model answer from your own data. In this post we'll wire that pipeline together with LangChain4j's `RetrievalAugmentor`.

## The RAG Pipeline

RAG adds three stages in front of a normal chat call:

1.  **Query transformation** — optionally rewrite the user's question (e.g. to compress a follow-up into a standalone query).
    
2.  **Retrieval** — embed the question and pull the most similar chunks out of the vector store.
    
3.  **Aggregation + injection** — combine the hits and append them to the user message before it reaches the model.
    

LangChain4j models this as an interface: `RetrievalAugmentor`. The default implementation, `DefaultRetrievalAugmentor`, composes pluggable pieces — a `QueryTransformer`, a `ContentRetriever`, a `ContentAggregator`, and a `ContentInjector`. We only need to supply the retriever; sensible defaults handle the rest.

## A ContentRetriever That Uses Our Search Service

The natural place to plug RAG in is our `SemanticSearchService` from the embeddings post. It owns both the embedding model and the store, so the retriever just delegates to it:

```java
public class SemanticSearchContentRetriever implements ContentRetriever {

    private final SemanticSearchService searchService;
    private final int maxResults;

    public SemanticSearchContentRetriever(SemanticSearchService searchService, int maxResults) {
        this.searchService = searchService;
        this.maxResults = maxResults;
    }

    @Override
    public List<Content> retrieve(Query query) {
        return searchService.search(query.text(), maxResults).stream()
                .map(this::toContent)
                .toList();
    }

    private Content toContent(EmbeddingMatch<TextSegment> match) {
        return Content.from(match.embedded(), Map.of(ContentMetadata.SCORE, match.score()));
    }
}
```

Delegating keeps a single source of truth: the store *and* the current embedding model live in the service, so a runtime `/model` switch is picked up by the RAG flow automatically. LangChain4j also ships a ready-made `EmbeddingStoreContentRetriever` if you'd rather wire the store directly.

## Assembling the Augmentor

In `AiConfig` we expose the retriever as a bean, wrap it in a `RetrievalAugmentor`, and attach the augmentor to a dedicated question-answering AI service:

```java
@Bean
public ContentRetriever contentRetriever(SemanticSearchService searchService,
                                         @Value("${app.rag.max-results:5}") int maxResults) {
    return new SemanticSearchContentRetriever(searchService, maxResults);
}

@Bean
public RetrievalAugmentor retrievalAugmentor(ContentRetriever contentRetriever) {
    return DefaultRetrievalAugmentor.builder()
            .contentRetriever(contentRetriever)
            .build();
}

@Bean
public QaAssistant qaAssistant(ChatModel chatModel,
                               RetrievalAugmentor retrievalAugmentor,
                               ...) {
    return AiServices.builder(QaAssistant.class)
            .chatModel(chatModel)
            .chatMemoryProvider(createChatMemoryProvider(...))
            .retrievalAugmentor(retrievalAugmentor)
            .build();
}
```

The QA assistant is a plain AI service interface, just like the chat assistant — the only difference is the augmentor:

```java
public interface QaAssistant {

    @SystemMessage("""
            You are a question-answering assistant that answers only from the provided context.
            Answer the question using the information supplied in the user message. If the context
            does not contain the answer, respond with "I don't know". Keep the answer concise.
            """)
    String ask(@MemoryId String memoryId, @UserMessage String question);
}
```

The `@MemoryId` gives each conversation its own memory, so follow-up questions work across turns.

## How the Pieces Fit Together

When `ask()` is called, `AiServices` runs the augmentor before hitting the model. The `DefaultRetrievalAugmentor`:

*   passes the question through the default `QueryTransformer` unchanged,
    
*   hands it to our `SemanticSearchContentRetriever`, which embeds it and searches the store,
    
*   aggregates the results with the default `ContentAggregator`,
    
*   and injects them into the user message via the default `ContentInjector`, producing something like:
    

```plaintext
How does RAG work?

Answer using the following information:
Retrieval Augmented Generation combines a vector database with a chat model ...
```

Then the chat model answers using only that context — grounded in your documents, not its own guesses.

## Trying It Out

The CLI gained an `/ask` command that chains retrieval and generation. With the bundled sample documents indexed:

```plaintext
/index sample-data
Indexed 6 segment(s). Store now holds 6 embedding(s).

/ask what is a vector database?
RAG > A vector database stores embeddings (numerical representations of text)
      so that similar meanings can be found quickly by vector similarity.
```

Ask a follow-up and the memory kicks in:

```plaintext
/ask why does that matter for answering questions?
RAG > It lets the model find relevant information from your own documents
      instead of relying only on what it learned during training.
```

Because retrieval happens per question, the answers are grounded in the indexed material — and with an empty store the augmentor returns nothing, so the assistant honestly says it doesn't know rather than hallucinating.

## Testing the Pipeline Without a Live Model

The best part of separating retrieval from generation is that retrieval is fully testable offline. With the deterministic `FakeEmbeddingModel` from the embeddings post, we index a document, then assert that the augmentor retrieves the right chunk and injects it into the user message:

```java
SemanticSearchService searchService = new SemanticSearchService(
        modelName -> new FakeEmbeddingModel(), "test", null, 5);
searchService.indexDirectory(docs);

ContentRetriever retriever = new SemanticSearchContentRetriever(searchService, 5);
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(retriever)
        .build();

AugmentationResult result = augmentor.augment(new AugmentationRequest(
        UserMessage.from("How does RAG work?"),
        Metadata.from(question, "qa", List.of())));

assertThat(result.contents()).isNotEmpty();
assertThat(((UserMessage) result.chatMessage()).singleText())
        .contains("Retrieval Augmented Generation");
```

This exercises the whole RAG flow — transform, retrieve, aggregate, inject — with zero network calls, which is exactly the kind of test you want before paying for model invocations.

## Configuration

| Property | Default | Description |
| --- | --- | --- |
| `app.rag.max-results` | `5` | Max document chunks retrieved for each question |

## Next Steps

Our RAG is grounded, but it's the simplest flavor. LangChain4j has more pieces to explore: `CompressingQueryTransformer` (rewrites follow-up questions using chat memory), re-ranking aggregators, and dedicated vector databases like `pgvector` or Elasticsearch that replace the in-memory store at scale. Document processing (parsing PDFs, splitting smarter) is the other natural next step.

## Resources

*   [Official LangChain4j RAG Documentation](https://docs.langchain4j.dev/tutorials/rag)
    
*   [GitHub Examples Repository](https://github.com/langchain4j/langchain4j-examples)
    
*   [RAG Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/199)
    

## Conclusion

RAG turns a general-purpose chatbot into one that answers from your own knowledge base. With LangChain4j, that's a `ContentRetriever`, a `RetrievalAugmentor`, and one line on the `AiServices` builder — and because the pipeline is composed of small interfaces, the retrieval half is testable without any API calls. Combined with conversation memory and semantic search from earlier posts, the demo is now a complete grounded question-answering system.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).