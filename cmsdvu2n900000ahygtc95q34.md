---
title: "Embeddings and Semantic Search with LangChain4j"
datePublished: 2026-08-03T23:50:47.042Z
cuid: cmsdvu2n900000ahygtc95q34
slug: embeddings-and-semantic-search-with-langchain4j

---

In previous posts we got LangChain4j up and running and gave our chatbot memory. But chatting only goes so far — what if you want to search through a pile of documents using *meaning* rather than keywords? That's where embeddings come in. In this post we'll build a semantic search tool that indexes text files, stores their embeddings, and answers queries by finding the most *conceptually similar* content.

## What Is an Embedding?

An embedding is a vector (a list of numbers) that represents the meaning of a piece of text. Texts with similar meanings end up with similar vectors — they sit close together in a high-dimensional space. "How do I fix my computer" and "my laptop is broken" will have nearby vectors, even though they share almost no words.

Semantic search works in two phases:

1.  **Indexing** — split documents into chunks, embed each chunk, and store the vectors in a vector store.
    
2.  **Querying** — embed the user's question and find stored vectors that are closest to it (using cosine similarity).
    

LangChain4j gives us the building blocks: `EmbeddingModel` (embeds text), `EmbeddingStore` (stores vectors), `EmbeddingStoreIngestor` (runs the indexing pipeline), and `DocumentSplitters` (chunks documents).

## The Semantic Search Service

We added a `SemanticSearchService` that ties these pieces together. It's a Spring component that holds an `EmbeddingModel` and an `InMemoryEmbeddingStore`, which we persist to a JSON file.

```java
@Service
public class SemanticSearchService {

    @Autowired
    public SemanticSearchService(Function<String, EmbeddingModel> modelFactory,
                                 @Value("${app.embedding.model-name:text-embedding-3-small}") String modelName,
                                 @Value("${app.embedding.store-path:}") String storePath,
                                 @Value("${app.embedding.max-results:5}") int defaultMaxResults) {
        this.modelFactory = modelFactory;
        this.modelName = modelName;
        this.embeddingModel = modelFactory.apply(modelName);
        this.storePath = storePath == null || storePath.isBlank() ? null : Path.of(storePath);
        this.defaultMaxResults = defaultMaxResults;
        this.embeddingStore = loadStore();
    }

    public int indexDirectory(Path directoryPath) {
        return ingest(FileSystemDocumentLoader.loadDocuments(directoryPath));
    }

    private int ingest(List<Document> documents) {
        int sizeBefore = embeddingStore.size();
        EmbeddingStoreIngestor.builder()
                .documentSplitter(DocumentSplitters.recursive(200, 20))
                .embeddingModel(embeddingModel)
                .embeddingStore(embeddingStore)
                .build()
                .ingest(documents);
        save();
        return embeddingStore.size() - sizeBefore;
    }

    public List<EmbeddingMatch<TextSegment>> search(String query) {
        Response<Embedding> response = embeddingModel.embed(query);
        return embeddingStore.search(EmbeddingSearchRequest.builder()
                        .queryEmbedding(response.content())
                        .maxResults(defaultMaxResults)
                        .build())
                .matches();
    }
}
```

A few highlights:

*   `FileSystemDocumentLoader` reads every file in a directory as a `Document`.
    
*   `DocumentSplitters.recursive(200, 20)` chunks documents into segments of at most 200 characters with 20 characters of overlap, so meaning isn't cut off at chunk boundaries.
    
*   `EmbeddingStoreIngestor` is the pipeline: split → embed → store. It returns token usage so you can see how much a document cost to embed.
    
*   `InMemoryEmbeddingStore` supports `serializeToFile` / `fromFile`, so the store survives restarts. We load it in the constructor if the file already exists.
    

## Switching Embedding Models

The issue's checklist asked us to try different embedding models, so we made the model switchable at runtime. In `AiConfig` we expose a small factory bean:

```java
@Bean
public Function<String, EmbeddingModel> embeddingModelFactory() {
    return modelName -> OpenAiEmbeddingModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .modelName(modelName)
            .build();
}
```

`SemanticSearchService` uses this factory to build its model, so `setEmbeddingModel("text-embedding-3-large")` swaps it on the fly. We support the three OpenAI embedding models:

```plaintext
text-embedding-3-small | text-embedding-3-large | text-embedding-ada-002
```

Note that switching models changes the vector dimensions, so re-index your documents after a switch — that's why the CLI reminds you.

## Trying It Out

The CLI (our `ChatCli` from the memory post) gained embedding commands alongside chat. Index the bundled sample documents and search:

```plaintext
/index sample-data
Indexed 4 segment(s). Store now holds 4 embedding(s).

/search what is a vector database?
Top 5 results for: "what is a vector database?"
1. [score 0.5678] Retrieval Augmented Generation (RAG) combines a vector database with a chat model. ...
2. [score 0.4321] Embeddings are dense vector representations of text that capture semantic meaning. ...
```

The query never contains the exact phrase "vector database" from the top hit's first words, yet the model ranked it first because the *meaning* matches.

You can also inspect a raw vector:

```plaintext
/embed hello world
Embedding (1536 dimensions) of "hello world":
[0.0051, -0.0123, 0.0098, ...]
```

## Configuration

| Property | Default | Description |
| --- | --- | --- |
| `app.embedding.model-name` | `text-embedding-3-small` | Embedding model for indexing and search |
| `app.embedding.store-path` | `embedding-store.json` | JSON file where the store is persisted |
| `app.embedding.max-results` | `5` | Default number of search results |

## Next Steps

Embeddings are the foundation of **Retrieval Augmented Generation (RAG)**: retrieve the most relevant chunks for a question, then feed them to the chat model so it can answer from your own documents. That's a natural next exploration — and it builds directly on the memory, chat, and embedding pieces we now have.

## Resources

*   [Official LangChain4j Documentation](https://docs.langchain4j.dev)
    
*   [GitHub Examples Repository](https://github.com/langchain4j/langchain4j-examples)
    
*   [Embeddings Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/196)
    

## Conclusion

Embeddings turn unstructured text into geometry, and semantic search is just a nearest-neighbor query over that geometry. With LangChain4j's `EmbeddingStoreIngestor` and `InMemoryEmbeddingStore`, the whole indexing pipeline is a few lines of code — and the store even persists to a plain JSON file. Combined with conversation memory, we're one step away from a full RAG application.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo). Stay tuned for RAG!