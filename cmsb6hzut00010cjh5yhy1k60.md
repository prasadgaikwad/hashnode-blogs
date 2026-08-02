---
title: "Conversation Memory with LangChain4j: Keeping Context Across Turns"
datePublished: 2026-08-02T02:26:00.793Z
cuid: cmsb6hzut00010cjh5yhy1k60
slug: conversation-memory-with-langchain4j-keeping-context-across-turns
tags: java, memory, agent, langchain4j

---

In our [previous post](https://blog.prasadgaikwad.dev/getting-started-with-langchain4j-building-your-first-ai-powered-java-application), we built a simple command-line chat with LangChain4j. It worked, but there was one big limitation: the model had no memory. Every question was answered in isolation, as if the conversation had never happened. In this post, we'll fix that by adding conversation memory — and we'll explore two different memory strategies you can switch between at runtime.

## Why Does a Chatbot Need Memory?

Large language models are stateless. Each request is processed independently, and the model has no idea what you asked it one minute ago. To hold a real conversation, the application must keep the history and send it along with every new question.

This is exactly what LangChain4j's memory module does. It stores the conversation, decides what to include on each call, and automatically manages the size of what's sent so you stay within the model's context window.

## Two Memory Strategies

LangChain4j 1.18.1 ships two built-in `ChatMemory` implementations, each controlling the sliding window differently:

1.  **MessageWindowChatMemory** — a buffer limited by the *number of messages*. When the window is full, the oldest messages are evicted.
    
2.  **TokenWindowChatMemory** — a window limited by the *token budget*. Messages are evicted until the history fits within a configured token count, using an actual tokenizer (in our case, OpenAI's).
    

> **Note:** Older LangChain4j versions listed "Summary" and "Vector" memory types. These are no longer part of the current API — the two sliding-window strategies above are what's available today, and they map nicely to the classic "buffer" and "context-window" ideas.

## Setup

We start from the getting-started project and add the core `langchain4j` module, which contains `AiServices` and the memory implementations:

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
</dependency>

<!-- Adds AiServices, MessageWindowChatMemory, TokenWindowChatMemory, ... -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
</dependency>
```

## Defining an AI Service

The idiomatic way to add memory in LangChain4j is through an **AI Service** — a plain Java interface that `AiServices` turns into a working implementation.

```java
public interface Assistant {

    @SystemMessage("You are a helpful assistant. Answer the question in a very concise way, only in 2 sentences maximum.")
    String chat(@MemoryId String memoryId, @UserMessage String message);
}
```

The annotations tell LangChain4j everything it needs:

*   `@SystemMessage` — the system prompt injected on every call.
    
*   `@MemoryId` — selects which conversation's memory to use, so you can have one memory per user or per chat.
    
*   `@UserMessage` — marks the parameter holding the user's input.
    

## Wiring It Up with Spring

Next we define the beans: a `ChatModel`, and the `Assistant` built with a `ChatMemoryProvider`. The provider is called by `AiServices` the first time a new memory ID is seen, so we can decide which memory type to create based on the ID itself. This is the trick that lets us switch memory types at runtime — the memory type is embedded in the memory ID.

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatModel chatModel(@Value("${app.chat.model-name:gpt-4o-mini}") String modelName) {
        return OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName(modelName)
                .build();
    }

    @Bean
    public Assistant assistant(ChatModel chatModel,
                               ChatMemoryRegistry chatMemoryRegistry,
                               @Value("${app.chat.model-name:gpt-4o-mini}") String modelName,
                               @Value("${app.memory.max-messages:10}") int maxMessages,
                               @Value("${app.memory.max-tokens:2000}") int maxTokens) {
        ChatMemoryProvider chatMemoryProvider = memoryId -> {
            ChatMemory chatMemory = createMemory((String) memoryId, modelName, maxMessages, maxTokens);
            chatMemoryRegistry.register((String) memoryId, chatMemory);
            return chatMemory;
        };

        return AiServices.builder(Assistant.class)
                .chatModel(chatModel)
                .chatMemoryProvider(chatMemoryProvider)
                .build();
    }

    private ChatMemory createMemory(String memoryId, String modelName, int maxMessages, int maxTokens) {
        if (memoryId.startsWith(MemoryType.MESSAGE_WINDOW.label())) {
            return MessageWindowChatMemory.builder()
                    .id(memoryId)
                    .maxMessages(maxMessages)
                    .build();
        }
        return TokenWindowChatMemory.builder()
                .id(memoryId)
                .maxTokens(maxTokens, new OpenAiTokenCountEstimator(modelName))
                .build();
    }
}
```

The `MemoryType` enum encodes each strategy with a label and produces the memory ID:

```java
public enum MemoryType {

    MESSAGE_WINDOW("message-window"),
    TOKEN_WINDOW("token-window");

    // ...

    public String memoryId(String conversationId) {
        return label + ":" + conversationId;
    }
}
```

So a conversation id of `main` becomes either `message-window:main` or `token-window:main`. When you switch strategies, the ID changes, `AiServices` sees a new memory ID, and the provider hands back a fresh memory of the new type. Switching types effectively starts a new conversation — which is exactly the behavior you'd want.

`AiServices` retains the created memories itself, but it doesn't expose them. To let the CLI inspect and clear memory, we keep a small registry:

```java
@Component
public class ChatMemoryRegistry {

    private final ConcurrentMap<String, ChatMemory> memories = new ConcurrentHashMap<>();

    public void register(String memoryId, ChatMemory memory) {
        memories.put(memoryId, memory);
    }

    public ChatMemory get(String memoryId) {
        return memories.get(memoryId);
    }
}
```

## The Command-Line Interface

We moved the interactive loop out of the main application class into a dedicated `ChatCli` component, gated by a property so tests can load the Spring context without blocking on stdin.

```properties
app.cli.enabled=true
app.chat.model-name=gpt-4o-mini
app.memory.max-messages=10
app.memory.max-tokens=2000
```

The CLI now understands a few commands alongside plain chat:

```plaintext
/help                 Show this help
/memory               Show current memory type and state
/memory <type>        Switch memory type (message-window | token-window)
/clear                Clear the current conversation memory
quit                  Exit the application
```

Try it: ask *"My name is Alice"*, then follow up with *"What is my name?"*. With memory enabled, the model remembers. Then run `/memory token-window` and notice the conversation history was reset — a new memory of the new type starts fresh.

## Running the Application

1.  Set your API key: `export OPENAI_API_KEY=...`
    
2.  Run it: `./mvnw spring-boot:run`
    
3.  Chat, and use `/memory` and `/clear` to explore how each memory type behaves.
    

## Next Steps

Conversation memory is the foundation for much more advanced features:

1.  **RAG (Retrieval Augmented Generation)** — combine memory with document retrieval
    
2.  **Chains and Agents** — let the model use tools, with full conversation context
    
3.  **Embeddings** — semantic search over past conversations
    
4.  **Streaming responses** — stream tokens back to the user
    

## Resources

*   [Official LangChain4j Documentation](https://docs.langchain4j.dev)
    
*   [GitHub Examples Repository](https://github.com/langchain4j/langchain4j-examples)
    
*   [Conversation Memory Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/195)
    

## Conclusion

Adding memory transforms a stateless question-answer loop into a real conversation. With LangChain4j's AI Services and `@MemoryId`, it takes only a few lines of code, and the built-in `MessageWindowChatMemory` and `TokenWindowChatMemory` give you two ways to manage the context window — switchable at runtime.

Check out the full implementation in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo). Stay tuned for more explorations — RAG, agents, and tool use are next on the list!