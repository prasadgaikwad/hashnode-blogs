---
title: "Advanced Features: Multi-Modal, Function Calling, and Structured Output"
datePublished: 2026-08-15T03:33:48.314Z
cuid: cmsttn91300000ai24cbw4dr4
slug: advanced-features-multi-modal-function-calling-and-structured-output

---

The demo has covered memory, RAG, agents, prompting, integration, tooling, and evaluation. This round attacks the checklist's **advanced features** item: multi-modal capabilities, function calling, structured output, and custom tools. The pattern that worked in earlier rounds holds: every capability is exercised offline in tests using fake models, while the real OpenAI models are wired for runtime use.

## Multi-Modal

"Multi-modal" here means three separate capabilities, each backed by a different model interface in LangChain4j.

### Vision: images into the chat model

A multimodal chat model (GPT-4o-mini) accepts image content alongside text. `VisionService` builds a `UserMessage` that mixes a `TextContent` (the question) with an `ImageContent`:

```java
UserMessage.from(TextContent.from(question), ImageContent.from(URI.create(imageUrl)))
```

The image can come from a public URL or, more usefully, as raw bytes — the service base64-encodes them and passes a mime type, which is how an uploaded file reaches the model:

```java
ImageContent.from(Base64.getEncoder().encodeToString(imageData), mimeType)
```

`POST /api/describe` exposes this over HTTP (`{imageUrl | imageData, mimeType, question}`), and `/describe <url> [question]` works from the CLI.

### Image generation

`ImageGenerationService` wraps an `ImageModel` — OpenAI's `gpt-image-1` by default, configurable via `app.image.model-name`. The returned `Image` carries a URL, or base64 data plus a mime type, or a `revisedPrompt`. The CLI `/generate <prompt>` prints whichever form came back.

### Speech-to-text

`SpeechToTextService` wraps an `AudioTranscriptionModel` (whisper-1). It wraps raw audio bytes into an `Audio` value and calls `transcribeToText`:

```java
Audio audio = Audio.builder()
        .base64Data(Base64.getEncoder().encodeToString(audioData))
        .mimeType(mimeType)
        .build();
return transcriptionModel.transcribeToText(audio);
```

CLI: `/transcribe <file>`. The wav/mp3/ogg/m4a extension is mapped to a mime type so the model knows what it's parsing.

## Function Calling and Custom Tools

The agent from an earlier round already calls `@Tool` methods. This round pushes further on **what a tool can be**.

### Structured tool parameters

`WeatherTool.getWeather` takes a `WeatherRequest` record instead of a handful of scalars:

```java
@Tool("Gets the current temperature for a city, returning it in the requested unit")
public String getWeather(WeatherRequest request) { ... }

@Description("Weather request parameters")
public record WeatherRequest(String city, TemperatureUnit unit) { }

public enum TemperatureUnit { CELSIUS, FAHRENHEIT }
```

LangChain4j derives a nested object schema from the record — `city` as a string, `unit` as an enum — so the model can fill in both fields in one call. The weather data is fake and deterministic, so the tool works offline. One subtlety surfaced in testing: for a POJO parameter named `request`, the model's arguments must be `{"request": {"city": "...", "unit": "..."}}`, keyed by the parameter name, not the bare object.

### Conversation-scoped tool state

`NoteTool` demonstrates a tool with state that is aware of *which conversation* it is running in, via `@ToolMemoryId`:

```java
@Tool("Saves a note for the current conversation")
public String saveNote(@ToolMemoryId String memoryId, @P("The note text to save") String note) { ... }
```

LangChain4j injects the current conversation's memory id into the call, so notes saved in one conversation never leak into another — a real pattern for memory-adjacent tools.

### Dynamic tool selection with a ToolProvider

The static agent registers its tools at build time with `.tools(...)`. The new `DynamicToolProvider` instead decides **per request** which tools are available, using `ToolProviderRequest` to inspect the user message:

```java
@Override
public ToolProviderResult provideTools(ToolProviderRequest request) {
    List<AiServiceTool> tools = new ArrayList<>(ToolService.findTools(calculatorTool));
    tools.addAll(ToolService.findTools(noteTool));
    if (request.userMessage().singleText().toLowerCase().contains("weather")) {
        tools.addAll(ToolService.findTools(weatherTool));
    }
    return ToolProviderResult.builder().addAll(tools).build();
}
```

The calculator and note tools are always exposed; the weather tool only appears when the task is actually about the weather. That keeps the model's function-call surface — and the tokens describing it — as small as possible. `AiServices.builder(DynamicAgent.class).toolProvider(provider)` wires it up, and `/dynamic <task>` exercises it from the CLI.

## Structured Output at the Model Level

The demo already had structured output through AI Service return types (`MovieReview`, enums, `List<String>`), where LangChain4j asks for JSON in the prompt and parses the reply. This round adds the **model-level** approach: attach the JSON schema *derived from the record* to the request as a response format, constraining the model to emit exactly that shape:

```java
JsonSchema schema = JsonSchemas.jsonSchemaFrom(MovieReview.class)
        .orElseThrow(...);

ChatRequest request = ChatRequest.builder()
        .messages(List.of(SystemMessage.from(SYSTEM_PROMPT), UserMessage.from(text)))
        .parameters(ChatRequestParameters.builder()
                .responseFormat(schema)   // model is constrained to this schema
                .build())
        .build();
```

`/schema <text>` runs this from the CLI. The guarantee is stronger than prompt-based extraction: with JSON mode the model's output is structurally enforced rather than merely requested.

## Testing It All Offline

The suite grew to 92 tests, all offline:

*   **Vision** — a `FakeChatModel` captures the built `ChatRequest`; tests assert the user message contains exactly one `TextContent` and one `ImageContent`, and that the base64 variant carries the right mime type.
    
*   **Image generation / STT** — dedicated fakes (`FakeImageModel`, `FakeAudioTranscriptionModel`) capture the prompt and the `Audio` payload respectively.
    
*   **Custom tools** — direct unit tests for `WeatherTool` (unit conversion, unknown city) and `NoteTool` (notes scoped per conversation).
    
*   **Function calling end-to-end** — `DynamicToolProviderTest` builds a real `DynamicAgent` over `AiServices` with a fake chat model that returns a `ToolExecutionRequest` on its first call and a plain answer on its second. The test watches the tool-specification list on each round: the weather tool is present only when the task mentions weather, the tool actually executes, and its result flows back into the second model call.
    
*   **Structured output** — asserts the request's `responseFormat().jsonSchema()` is named `MovieReview` and that the model's JSON reply parses into the record.
    

The agent-loop test deserves emphasis: it exercises the real `AiServices` machinery — schema generation, tool dispatch, the round-trip of results back into the model — with zero network calls, purely by scripting what the model would say.

## Design Notes

*   **Fake models everywhere.** No capability ships without an offline test proving the request wiring is correct.
    
*   **Runtime configurability.** Image and STT models are beans you can swap by property, matching the existing chat/embedding model pattern.
    
*   **Dynamic > static where it matters.** Exposing a tool per request beats registering every tool for every task — smaller prompts, fewer wrong calls.
    
*   **Nested tool schemas.** A record parameter is the clean way to give a tool rich, typed inputs; just remember the arguments are keyed by the parameter name.
    

## Next Steps

The last checklist item is **LLM integration**: wiring alternative providers (Anthropic, Google, Ollama) and comparing models. The `/eval` harness from the previous round is exactly the yardstick that comparison needs.

## Resources

*   [LangChain4j Tools Documentation](https://docs.langchain4j.dev/tutorials/tools)
    
*   [LangChain4j Multi-Modality](https://docs.langchain4j.dev/tutorials/chat)
    
*   [Advanced Features Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/203)
    

## Conclusion

The advanced-features round turned the demo from a text-only assistant into one that sees images, paints them, listens, calls tools with structured parameters, and emits schema-constrained JSON. Two ideas carried the round: fakes make even a tool-calling agent loop testable offline, and the interface-based model abstraction in LangChain4j (chat, image, audio) means each new modality was a small service rather than a rewrite.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).