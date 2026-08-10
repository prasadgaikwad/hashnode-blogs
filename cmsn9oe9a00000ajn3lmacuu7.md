---
title: "Developer Tooling: DevTools, Actuator, and Swagger UI"
datePublished: 2026-08-10T13:28:12.359Z
cuid: cmsn9oe9a00000ajn3lmacuu7
slug: developer-tooling-devtools-actuator-and-swagger-ui

---

The demo is now a real web application with a dozen REST endpoints, streaming, and a database — which means it's time to make it comfortable to develop and easy to inspect. This short post covers the three pieces from our checklist's tooling round: **Spring Boot DevTools**, **Actuator**, and **Swagger UI**.

## The Dependencies

Three additions to `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.9.0</version>
</dependency>
```

DevTools is scoped `runtime` and `optional` so it never ships in a production artifact. Springdoc 2.9.0 is the latest 2.x line — 3.x targets Spring Boot 4, and we're on 3.5.

## DevTools: Auto-Restart

DevTools watches the classpath and restarts the app whenever a source file changes — the modern descendant of `spring-boot-devtools`' classic auto-restart. While editing the demo, saving a Java file triggers a restart automatically, so the old manual stop/`spring-boot:run` loop disappears. Nothing else to configure.

## Actuator: Monitoring

Actuator exposes health and metrics without any code. The demo opts into a focused set — `health`, `info`, and `metrics` — rather than the kitchen sink:

```properties
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
management.info.env.enabled=true
info.app.name=langchain4j-demo
info.app.description=LangChain4j feature demo: memory, RAG, agents, prompting, streaming, WebSocket, DB
```

`/actuator/health` now reports the app plus its H2 database component:

```json
{"status":"UP","components":{"db":{"status":"UP","details":{"database":"H2"}},"diskSpace":{"status":"UP"},"ping":{"status":"UP"},"ssl":{"status":"UP"}}}
```

One configuration gotcha surfaced during this work: `management.info.env.enabled` is required to expose `info.*` properties — without it, `/actuator/info` returns `{}` no matter how many `info.app.*` keys you define. That's the kind of thing a one-line test (see below) would have caught on the first run.

## Swagger UI: Testing from the Browser

springdoc scans the `@RestController`s and generates an OpenAPI document automatically — every endpoint, request record, and response record shows up with schemas. Open the UI at `/swagger-ui.html` and each of the thirteen `/api/**` paths becomes a clickable "Try it out" form:

```plaintext
GET  /api/chat/stream  (SSE)
POST /api/chat  {"message": "..."}
POST /api/ask   {"question": "..."}
...and the prompt, search, index, store, and history endpoints
```

`/v3/api-docs` serves the raw JSON if you want to feed it to other tools.

### Making the docs informative

Auto-generated docs are accurate, but they start out terse — just parameter names and types. So each endpoint got explicit OpenAPI annotations: a `@Tag` per controller, an `@Operation` summary and description per method, `@Parameter` descriptions and examples for query/path params, `@Schema` descriptions on the request/response records, and `@ApiResponse` status codes. A tiny `OpenApiConfig` bean sets the title and description in the UI header.

The result reads like a hand-written API reference that can't drift:

```java
@PostMapping("/chat")
@Operation(summary = "Chat with the memory-backed assistant",
        description = "Sends the message to the assistant with conversation memory. "
                + "Use a conversationId to keep a multi-turn conversation; a new id starts fresh.")
@ApiResponse(responseCode = "200", description = "The assistant's answer",
        content = @Content(schema = @Schema(implementation = ChatResponse.class)))
public ChatResponse chat(@RequestBody ChatRequest request) { ... }
```

with the request schema annotated on the record:

```java
public record ChatRequest(
        @Schema(description = "Conversation (memory) id; a fresh id starts a new conversation",
                example = "web", defaultValue = "api")
        String conversationId,
        @Schema(description = "The user message or task to run", example = "Hello!")
        String message) { ... }
```

Every query parameter also carries a description and example, so "Try it out" comes pre-filled with sensible values.

## Testing the Tooling

`DevToolingTest` treats the tooling itself as a feature to verify:

```java
@SpringBootTest(properties = "app.cli.enabled=false")
@AutoConfigureMockMvc
class DevToolingTest {

    @Test
    void actuatorHealthReportsUp() throws Exception {
        mockMvc.perform(get("/actuator/health"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("UP"));
    }

    @Test
    void openApiDocsDescribeTheRestApi() throws Exception {
        mockMvc.perform(get("/v3/api-docs"))
                .andExpect(content().string(containsString("/api/chat")))
                .andExpect(content().string(containsString("/api/chat/stream")));
    }
}
```

The OpenAPI test is genuinely useful: if someone renames a route, the docs test catches it. All 60 tests pass offline.

## Design Notes

*   **Expose the minimum.** `health`, `info`, `metrics` cover monitoring without opening `env` or `beans` to the world.
    
*   **Let the docs come from the code.** Every endpoint we documented by hand in the README previously had drifted (`/api/prompt/sentiment` was actually `/api/sentiment`). springdoc derives docs from the running code, so they can't drift — and it flagged the README table to fix.
    
*   **Tooling gets tests too.** A 4-test class proves health, info, docs, and the UI redirect without a browser.
    

## Next Steps

Back to the feature checklist: **evaluation** (automated scoring of the typed outputs) and **LLM integration** (alternative providers and models). Both become more tractable now that every endpoint is testable from Swagger UI and observable through Actuator.

## Resources

*   [Spring Boot Actuator Documentation](https://docs.spring.io/spring-boot/reference/actuator/index.html)
    
*   [springdoc-openapi](https://springdoc.org)
    
*   [Developer Tooling Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/211)
    

## Conclusion

Tooling work is the least glamorous milestone in the demo, but it pays off immediately: DevTools removed the restart loop, Actuator made the app observable with four config lines, and Swagger UI turned "let me check the API" into a browser tab instead of a curl session. The best return, though, was the discovery that hand-maintained docs had already drifted — a good argument for generating documentation from the code that implements it.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).