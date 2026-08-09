---
title: "Integration Features: REST, Streaming, WebSocket, and a Database"
datePublished: 2026-08-09T23:34:51.549Z
cuid: cmsmfwpdx00000akmcerf76x3
slug: integration-features-rest-streaming-websocket-and-a-database

---

So far everything in this series has lived behind the command line. The model, the memory, the RAG pipeline, the agents — all real, but all reachable only through a REPL. Our checklist's next milestone is **integration features**: exposing all of that as a web application. This post covers the four pieces — **REST API**, **streaming**, **WebSocket**, and **database integration** — and the one trick that lets all of them stay testable without a network connection.

## The Shape of It

Spring Boot 3 already has the web layer; LangChain4j has the AI layer. The integration milestone is mostly about wiring the two together. The application now ships:

*   a REST API under `/api` (chat, RAG, agent, prompt helpers, search, history)
    
*   a Server-Sent Events endpoint for token-by-token streaming
    
*   a WebSocket endpoint for the same, frame-by-frame
    
*   an H2 database, via Spring Data JPA, for conversation history
    

The `pom.xml` gains four starters:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

## REST API

Controllers are thin: take a request record, call the service that already exists from the earlier milestones, return a response record. The chat endpoint reuses the memory-backed `Assistant` from the memory milestone, with `conversationId` defaulting to `"api"` for curl users:

```java
@PostMapping("/chat")
public ChatResponse chat(@RequestBody ChatRequest request) {
    String answer = assistant.chat(request.conversationId(), request.message());
    return new ChatResponse(answer);
}
```

Because RAG (`/api/ask`) and the agent (`/api/agent`) were built as services in their own milestones, exposing them is a three-liner each. The prompt helpers from the last milestone (`/api/prompt/sentiment`, `/api/prompt/movie`, `/api/prompt/topics`) and semantic search (`/api/search`) get the same treatment. A static `index.html` lists every endpoint, and `chat.html` is a working browser client.

## Streaming with Server-Sent Events

Streaming needed something we didn't have yet: a `StreamingChatModel`. That's a second bean in `AiConfig`, alongside the existing chat model:

```java
@Bean
public StreamingChatModel streamingChatModel() {
    return OpenAiStreamingChatModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .modelName("gpt-4o-mini")
            .build();
}
```

The streaming logic itself lives in a small service with no web-layer dependencies. That separation is deliberate — it means the same code serves both the SSE endpoint and the WebSocket handler, and it means we can unit-test it with a fake model:

```java
@Service
public class ChatStreamingService {

    public void stream(String message, StreamConsumer consumer) {
        streamingChatModel.chat(ChatRequest.builder()
                .messages(List.of(
                        SystemMessage.from("You are a helpful assistant. Answer concisely."),
                        UserMessage.from(message)))
                .build(), new StreamingChatResponseHandler() {
                    @Override
                    public void onPartialResponse(String partialResponse) {
                        consumer.onToken(partialResponse);
                    }
                    @Override
                    public void onCompleteResponse(ChatResponse completeResponse) {
                        consumer.onComplete(completeResponse.aiMessage().text());
                    }
                    @Override
                    public void onError(Throwable error) {
                        consumer.onError(error);
                    }
                });
    }
}
```

The consumer is a plain interface: `onToken`, `onComplete`, `onError`. The SSE endpoint adapts it to an `SseEmitter`:

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream(@RequestParam String message) {
    SseEmitter emitter = new SseEmitter(60_000L);
    streamingService.stream(message, new ChatStreamingService.StreamConsumer() {
        @Override public void onToken(String token) {
            try {
                emitter.send(SseEmitter.event().data(token));
            } catch (IOException e) {
                emitter.completeWithError(e);
            }
        }
        @Override public void onComplete(String fullText) { emitter.complete(); }
        @Override public void onError(Throwable error) { emitter.completeWithError(error); }
    });
    return emitter;
}
```

The browser client reads the stream with a plain `EventSource` — no library needed.

## WebSocket

The WebSocket handler is the same streaming service wearing a different transport. The client sends a JSON message; the server replies with one text frame per token and a final `[DONE]` frame:

```java
@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        ChatMessagePayload payload;
        try {
            payload = objectMapper.readValue(message.getPayload(), ChatMessagePayload.class);
        } catch (IOException e) {
            sendQuietly(session, new TextMessage("[ERROR] Invalid message payload: " + e.getMessage()));
            return;
        }
        streamingService.stream(payload.message(), new ChatStreamingService.StreamConsumer() {
            @Override public void onToken(String token) {
                sendQuietly(session, new TextMessage(token));
            }
            @Override public void onComplete(String fullText) {
                sendQuietly(session, new TextMessage("[DONE]"));
            }
            @Override public void onError(Throwable error) {
                sendQuietly(session, new TextMessage("[ERROR] " + error.getMessage()));
            }
        });
    }
}
```

`WebSocketConfig` registers the handler at `/ws/chat` and opens CORS for a browser client:

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatWebSocketHandler, "/ws/chat").setAllowedOrigins("*");
    }
}
```

## Database Integration

The last piece: conversation history moved out of memory and into a real store. The entity is a plain JPA `@Entity`; the repository is one interface with derived queries:

```java
public interface ConversationEntryRepository extends JpaRepository<ConversationEntry, Long> {
    List<ConversationEntry> findByConversationIdOrderByTimestampAsc(String conversationId);
    List<String> findDistinctConversationIds();
    void deleteByConversationId(String conversationId);
}
```

The service wraps it with `@Transactional` — the derived `deleteByConversationId` needs a real transaction, something the test suite caught on the first run:

```java
@Service
@Transactional
public class ConversationHistoryService {
    public ConversationEntry record(String conversationId, String role, String text) { ... }
    public List<ConversationEntry> history(String conversationId) { ... }
    public List<String> conversationIds() { ... }
    public void clear(String conversationId) { ... }
}
```

The REST layer exposes it as `GET /api/history`, `GET /api/history/{id}`, and `DELETE /api/history/{id}`. The database is in-memory H2 (`jdbc:h2:mem:demo`), so it resets on restart — but it's a real database with real JPA semantics, and swapping in Postgres would just be a JDBC URL and driver change.

## Keeping It Testable Offline

The web layer added a second AI surface — streaming — which needed a fake just like `FakeChatModel`. `FakeStreamingChatModel` implements `StreamingChatModel` and plays back a fixed list of tokens:

```java
class FakeStreamingChatModel implements StreamingChatModel {

    private final List<String> tokens;

    FakeStreamingChatModel(String... tokens) {
        this.tokens = List.of(tokens);
    }

    @Override
    public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
        StringBuilder full = new StringBuilder();
        for (String token : tokens) {
            handler.onPartialResponse(token);
            full.append(token);
        }
        handler.onCompleteResponse(ChatResponse.builder()
                .aiMessage(AiMessage.from(full.toString()))
                .build());
    }
}
```

With that in place the tests split cleanly by layer:

*   `ChatStreamingServiceTest` — token forwarding and completion, unit-level
    
*   `ChatWebSocketHandlerTest` — frames arrive in order followed by `[DONE]`, and a bad payload yields `[ERROR]`
    
*   `ConversationHistoryServiceTest` — `@DataJpaTest`: ordering, empty history, distinct conversation ids, scoped deletes
    
*   `ChatApiControllerTest` — `@SpringBootTest` + `@AutoConfigureMockMvc` with `@MockitoBean` stand-ins for the services; the SSE test uses `asyncDispatch` on the returned `MvcResult`
    
*   `HistoryApiControllerTest` — end-to-end through the real H2-backed service
    

All 56 tests run offline.

## Design Notes

*   **Streaming is transport-agnostic.** `ChatStreamingService` knows nothing about `SseEmitter` or `WebSocketSession`; both transports are thin adapters over the same `StreamConsumer`. That is what makes it unit-testable and gives SSE and WebSocket identical behavior for free.
    
*   **Reuse beats new abstractions.** The controller didn't reinvent RAG or agents — it calls the existing `QaService` and `ChainService`.
    
*   **Transactions are not optional.** Derived delete queries in Spring Data JPA require a transaction; the failing test on the first run was the cheapest possible proof.
    
*   **Fakes scale with the surface.** One fake per AI boundary (`FakeChatModel`, `FakeEmbeddingModel`, now `FakeStreamingChatModel`) keeps every layer of the app testable without API keys.
    

## Next Steps

From our checklist: **evaluation** (now that prompts and outputs are typed, they're ready to be scored automatically) and **LLM integration** (swapping in other providers and models). Also worth exploring: multi-modal input and the new `agentic` module for advanced orchestration.

## Resources

*   [Official LangChain4j Streaming Documentation](https://docs.langchain4j.dev/tutorials/ai-services)
    
*   [Spring Boot SSE and WebSocket](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
    
*   [Integration Features Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/201)
    

## Conclusion

Integration was the least "AI" milestone in the whole demo — no new prompting tricks, no model gymnastics. But it's the one that turns a library into an application: a browser can now chat, stream, and ask questions against every feature built so far, and conversation state lives in a real database. The design payoff was architectural: keeping streaming logic transport-free meant SSE and WebSocket both shipped as thin adapters, and keeping every fake at the model boundary meant all of it stays offline-testable.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).