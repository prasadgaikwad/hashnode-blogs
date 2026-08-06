---
title: "Document Processing with LangChain4j"
datePublished: 2026-08-06T02:01:15.256Z
cuid: cmsgvdk9r00000ahyc8znhfg5
slug: document-processing-with-langchain4j

---

In the last post we built a RAG pipeline that answers questions from indexed documents. But the pipeline is only as good as its input: real-world data lives in PDFs, mixed file types, and messy formats — not clean `.txt` files. In this post we'll teach the demo to load PDFs, pick the right parser per file type, and split documents with configurable strategies.

## The Document Pipeline

LangChain4j models the journey from raw file to searchable chunk as a chain of small steps:

1.  **Load** — read bytes from a file or URL (`FileSystemDocumentLoader`).
    
2.  **Parse** — turn bytes into text using a `DocumentParser` chosen for the format (PDFBox for PDFs, plain text otherwise).
    
3.  **Split** — break the text into `TextSegment`s with a `DocumentSplitter`.
    
4.  **Transform** — enrich each segment (we prepend the source file name).
    

Every step is swappable. We used to do this inside `SemanticSearchService` with hardcoded settings; now it lives in a dedicated `DocumentService` so the processing concerns are separate from the embedding concerns.

## Parsing PDFs

PDF parsing needs a parser. LangChain4j has several; we added the Apache PDFBox one:

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-document-parser-apache-pdfbox</artifactId>
</dependency>
```

(The BOM manages its version, so no version tag is needed.)

`ApachePdfBoxDocumentParser` extracts text from a PDF using PDFBox. `DocumentService` picks the right parser per extension — PDFs get PDFBox, everything else gets the plain-text parser:

```java
private DocumentParser parserFor(Path path) {
    String name = path.getFileName().toString().toLowerCase(Locale.ROOT);
    if (name.endsWith(".pdf")) {
        return new ApachePdfBoxDocumentParser();
    }
    return new TextDocumentParser();
}
```

And loads a single file or a whole directory:

```java
public List<TextSegment> loadAndSplit(Path filePath) {
    return split(FileSystemDocumentLoader.loadDocument(filePath, parserFor(filePath)));
}

public List<TextSegment> loadAndSplitDirectory(Path directoryPath) {
    try (Stream<Path> files = Files.list(directoryPath)) {
        return files.filter(Files::isRegularFile)
                .filter(file -> !file.getFileName().toString().startsWith("."))
                .flatMap(file -> loadAndSplit(file).stream())
                .toList();
    } catch (IOException e) {
        throw new RuntimeException("Failed to list documents in " + directoryPath, e);
    }
}
```

Note that we skip hidden files (like macOS `.DS_Store`) — a small but necessary bit of processing hygiene. The loader attaches useful `Metadata` automatically, including `file_name` and the absolute directory path.

## Splitting Strategies

LangChain4j ships several `DocumentSplitter`s out of the box:

*   `DocumentByParagraphSplitter` — chunks by paragraphs (blank-line separated blocks)
    
*   `DocumentByLineSplitter` — chunks by lines
    
*   `DocumentBySentenceSplitter` — chunks by sentences (OpenNLP sentence detection)
    
*   `DocumentByWordSplitter` — chunks by words
    
*   `DocumentByCharacterSplitter` — fixed-size character chunks
    
*   `DocumentSplitters.recursive(...)` — paragraphs first, falling back to lines, then sentences, then words for anything still too big
    

We wrapped these in a small enum so the strategy is configurable and switchable at runtime:

```java
public enum DocumentSplitterType {
    RECURSIVE("recursive"),
    PARAGRAPH("paragraph"),
    LINE("line"),
    SENTENCE("sentence"),
    WORD("word"),
    CHARACTER("character");

    public DocumentSplitter create(int maxChunkSize, int maxOverlap) {
        return switch (this) {
            case RECURSIVE -> DocumentSplitters.recursive(maxChunkSize, maxOverlap);
            case PARAGRAPH -> new DocumentByParagraphSplitter(maxChunkSize, maxOverlap);
            case LINE -> new DocumentByLineSplitter(maxChunkSize, maxOverlap);
            case SENTENCE -> new DocumentBySentenceSplitter(maxChunkSize, maxOverlap);
            case WORD -> new DocumentByWordSplitter(maxChunkSize, maxOverlap);
            case CHARACTER -> new DocumentByCharacterSplitter(maxChunkSize, maxOverlap);
        };
    }
}
```

`DocumentService` uses the selected strategy together with a configurable chunk size and overlap:

```java
private List<TextSegment> split(Document document) {
    String fileName = document.metadata().getString(FILE_NAME);
    return splitterType.create(maxChunkSize, maxOverlap).split(document).stream()
            .map(segment -> withFileNamePrefix(segment, fileName))
            .toList();
}
```

### Why the file-name prefix?

Each splitter copies the document's metadata into every `TextSegment`. On top of that we prepend the file name to the segment *text*. This is a documented LangChain4j trick: retrieval improves when every chunk knows where it came from, because the embedding of e.g. `rag.pdf\nRetrieval Augmented Generation combines...` is closer to a query about RAG than the bare sentence would be.

## Wiring It In

`SemanticSearchService` now delegates the load-parse-split stage and keeps only the embed-and-store stage:

```java
public int indexDocument(Path filePath) {
    return indexSegments(documentService.loadAndSplit(filePath));
}

public int indexDirectory(Path directoryPath) {
    return indexSegments(documentService.loadAndSplitDirectory(directoryPath));
}

private int indexSegments(List<TextSegment> segments) {
    if (segments.isEmpty()) {
        return 0;
    }
    int sizeBefore = embeddingStore.size();
    List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
    embeddingStore.addAll(embeddings, segments);
    save();
    return embeddingStore.size() - sizeBefore;
}
```

A nice side effect: without an in-place splitter anymore, `SemanticSearchService` just embeds and stores whatever segments it is given — the splitter lives where it belongs, in `DocumentService`.

## Trying It Out

We added a sample PDF (`sample-data/rag.pdf`) and a `/splitter` command to the CLI. Index the directory — PDFs and text files alike:

```plaintext
/index sample-data
Indexed 7 segment(s). Store now holds 7 embedding(s).
```

Then ask a question grounded in the PDF:

```plaintext
/ask how does RAG combine a vector database with a chat model?
RAG > RAG combines a vector database with a chat model so the model can answer
      questions using your own documents instead of relying only on training data.
```

Switch splitting strategies and see how the chunking changes:

```plaintext
/splitter
Splitter type: recursive
Max chunk size: 200 chars
Max overlap   : 20 chars

/splitter paragraph
Switched splitter to 'paragraph'. Re-index documents to re-chunk them with the new strategy.
```

## Testing Without a Live Model

Processing is fully testable offline. The test suite generates a real PDF at runtime with PDFBox, then asserts the parser extracts the expected text:

```java
@Test
void parsesPdfDocuments() throws Exception {
    Path pdf = tempDir.resolve("guide.pdf");
    createPdf(pdf, "Retrieval augmented generation lets a chat model answer from your own documents.");

    DocumentService service = new DocumentService("recursive", 200, 20);

    List<TextSegment> segments = service.loadAndSplit(pdf);

    assertThat(segments).isNotEmpty();
    assertThat(segments.get(0).text()).contains("Retrieval augmented generation");
}
```

Another test asserts the file-name prefix lands on every segment, and a third proves smaller chunk sizes yield more segments — all without a single network call.

## Configuration

| Property | Default | Description |
| --- | --- | --- |
| `app.document.splitter` | `recursive` | Splitting strategy (`recursive`, `paragraph`, `line`, `sentence`, `word`, `character`) |
| `app.document.max-chunk-size` | `200` | Max segment size in characters |
| `app.document.max-overlap` | `20` | Overlap between segments in characters |

## Next Steps

Now that documents are first-class, the RAG question-answering from the previous post gets much better input. Natural next explorations: `DocumentBySentenceSplitter` tuning, `EmbeddingStoreIngestor`'s `textSegmentTransformer` for richer metadata, and — later in our checklist — chains and agents that operate on the processed documents.

## Resources

*   [Official LangChain4j RAG Documentation](https://docs.langchain4j.dev/tutorials/rag)
    
*   [Apache PDFBox](https://pdfbox.apache.org)
    
*   [Document Processing Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/197)
    

## Conclusion

Document processing is where RAG becomes real: PDF parsing, per-format parsers, and configurable splitting turn arbitrary files into clean, retrievable chunks. With LangChain4j's `DocumentParser` and `DocumentSplitter` interfaces, each stage is a small pluggable component — and with a dedicated `DocumentService`, the demo now separates *how documents become segments* from *how segments become searchable embeddings*.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).