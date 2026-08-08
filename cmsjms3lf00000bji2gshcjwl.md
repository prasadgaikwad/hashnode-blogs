---
title: "Prompting Techniques with LangChain4j"
datePublished: 2026-08-08T00:23:55.472Z
cuid: cmsjms3lf00000bji2gshcjwl
slug: prompting-techniques-with-langchain4j

---

Everything so far in this series has relied on a single well-written prompt. But prompts are code: they deserve templates, examples, and reliable output contracts. This post covers the three prompting techniques from our checklist — **prompt templates**, **few-shot learning examples**, and **output parsers** — and how LangChain4j makes each one a first-class, testable concept.

## Prompt Templates

A prompt template is a prompt skeleton with `{{placeholders}}`. Instead of string-concatenating prompts in handlers all over the codebase, you define the template once and render it with variables.

LangChain4j's `PromptTemplate` is a pure, offline component — no model involved. It renders a template into a `Prompt`, which converts to a system or user message:

```java
@Service
public class PromptService {

    private static final String SYSTEM_TEMPLATE = """
            You are a professional movie critic.
            Always write your reviews in a {{tone}} tone.
            """;

    private static final String USER_TEMPLATE = """
            Write a short review for the movie "{{movie}}" ({{year}}).
            Include a rating out of 10 in your review.
            """;

    public List<ChatMessage> buildMovieReviewMessages(String movie, int year, String tone) {
        Prompt systemPrompt = PromptTemplate.from(SYSTEM_TEMPLATE).apply(Map.of("tone", tone));
        Prompt userPrompt = PromptTemplate.from(USER_TEMPLATE).apply(Map.of("movie", movie, "year", year));
        return List.of(systemPrompt.toSystemMessage(), userPrompt.toUserMessage());
    }
}
```

The CLI command `/template` prints the fully rendered messages *without calling any API*:

```plaintext
/template Inception
Rendered prompt template for "Inception" (year 2010, enthusiastic tone):

SYSTEM:
You are a professional movie critic.
Always write your reviews in an enthusiastic tone.

USER:
Write a short review for the movie "Inception" (2010).
Include a rating out of 10 in your review.
```

Because rendering is deterministic, it is fully unit-testable: same template + same variables → same messages. We assert exactly that in `PromptServiceTest`.

AI Services take templates one step further: `@SystemMessage` and `@UserMessage` accept templates whose variables are resolved from method parameters (via `@V`, or plain parameter names with Spring's `-parameters`):

```java
@UserMessage("Write a review of {{movie}} from {{year}}")
String review(@V("movie") String movie, @V("year") int year);
```

## Few-Shot Learning Examples

Sometimes a short instruction isn't enough — the model benefits from *seeing* correct input/output pairs. Few-shot prompting embeds a handful of labeled examples into the prompt. Our `FewShotAssistant` classifies sentiment into `POSITIVE` / `NEGATIVE` / `NEUTRAL` using a mini dataset in the system message:

```java
public interface FewShotAssistant {

    @SystemMessage("""
            You are a sentiment classifier. Classify the sentiment of the given text
            as one of: POSITIVE, NEGATIVE, NEUTRAL. Reply with exactly one of these words.

            Examples:
            Text: "I absolutely loved this movie, best film of the year!"
            Sentiment: POSITIVE

            Text: "This restaurant is terrible, the food was cold and the service rude."
            Sentiment: NEGATIVE

            Text: "The package arrived on time. Nothing more to say."
            Sentiment: NEUTRAL
            """)
    Sentiment classify(@UserMessage String text);
}
```

Three well-chosen examples do double duty: they pin down the *format* (exactly one word) and illustrate the *decision boundary* (contrasting strong opinions with a neutral, factual statement). `AiServices` renders the system message with the examples on every call.

## Output Parsers (Structured Output)

The most robust prompting technique is making the model return a *typed value* instead of free text. In LangChain4j this is done by choosing the AI Service's return type — the framework derives a schema, requests JSON matching it, and parses the reply into an object. The internal machinery is `OutputParser` (`EnumOutputParser`, `PojoOutputParser`, `StringListOutputParser`, and friends); we never touch it directly, we just pick the return type.

Returning an enum — the sentiment above returns `Sentiment`, parsed into the matching constant.

Returning a POJO — `MovieExtractor` returns a `MovieReview` record:

```java
public record MovieReview(String title, int year, String director, double rating, String summary) {}

public interface MovieExtractor {
    @SystemMessage("""
            Extract information about the movie from the given text.
            Return the data as a JSON object with exactly these fields:
            title (string), year (integer), director (string), rating (number from 1 to 10), summary (string).
            """)
    MovieReview extract(@UserMessage String text);
}
```

The handler gets an object, not a string to hand-parse:

```plaintext
/movie Inception is a 2010 film directed by Christopher Nolan about planting ideas in dreams.
Movie > MovieReview[title=Inception, year=2010, director=Christopher Nolan, rating=9.0, summary=...]
```

Returning a collection — `TopicExtractor` returns `List<String>`. (Collection-of-strings output is parsed as one item per line, so the system message tells the model that format.)

## Testing Without a Live Model

Prompt engineering is all about iteration, so offline testing is a big win. We added a second shared test helper, `FakeChatModel`, which mirrors `FakeEmbeddingModel`: it implements the `ChatModel` interface, captures the exact `ChatRequest` LangChain4j builds, and returns a canned reply. That lets tests verify two things at once:

1.  **What goes in** — the generated system/user messages contain the few-shot examples and the interpolated template variables.
    
2.  **What comes out** — the canned reply is parsed into the correct enum constant, record, or list.
    

```java
@Test
void embedsFewShotExamplesInTheSystemMessage() {
    FakeChatModel chatModel = new FakeChatModel("POSITIVE");
    FewShotAssistant assistant = AiServices.builder(FewShotAssistant.class)
            .chatModel(chatModel)
            .build();

    Sentiment sentiment = assistant.classify("This movie is amazing!");

    assertThat(chatModel.lastSystemMessage())
            .contains("Examples:")
            .contains("Sentiment: NEGATIVE");
    assertThat(sentiment).isEqualTo(Sentiment.POSITIVE);
}

@Test
void parsesTheModelReplyIntoTheMovieReviewRecord() {
    FakeChatModel chatModel = new FakeChatModel("""
            {"title": "Inception", "year": 2010, "director": "Christopher Nolan",
             "rating": 9.0, "summary": "A thief enters dreams to plant an idea."}
            """);
    MovieReview review = AiServices.builder(MovieExtractor.class)
            .chatModel(chatModel)
            .build()
            .extract("Tell me about Inception.");

    assertThat(review.title()).isEqualTo("Inception");
    assertThat(review.rating()).isEqualTo(9.0);
}
```

These tests exercise the real `AiServices` proxy generation and the real output parsers — the only substitute is the model itself. Combined with `PromptServiceTest`, the whole prompting layer is verified without a single network call.

## Design Notes

*   **Templates are the API, not an afterthought.** `PromptTemplate` and `@UserMessage`/`@SystemMessage` templates keep prompts next to their consumers and make variables explicit.
    
*   **Examples live in the system message.** They ship with every call and double as both format and boundary guidance.
    
*   **Output contracts beat prompt begging.** Telling the model "return JSON" is unreliable; returning `MovieReview` makes the schema explicit and the parsing automatic and typed.
    
*   **FakeChatModel closes the loop.** It lets us assert on both the rendered prompt and the parsed result, which is where most prompt-engineering bugs actually live.
    

## Next Steps

From our checklist: **LLM integration** (pluggable providers/models), **integration features** (REST endpoints, streaming), and **evaluation** — now that outputs are typed (`Sentiment`, `MovieReview`, `List<String>`), they become much easier to evaluate automatically.

## Resources

*   [Official LangChain4j Prompting Documentation](https://docs.langchain4j.dev/tutorials/ai-services)
    
*   [LangChain4j Structured Output](https://docs.langchain4j.dev/tutorials/structured-output)
    
*   [Prompting Techniques Issue](https://github.com/prasadgaikwad/langchain4j-demo/issues/200)
    

## Conclusion

Prompting is the part of an LLM application you will iterate on most, so it pays to make it structured. Templates keep prompts DRY and inspectable; few-shot examples teach format and boundaries with zero extra code; typed return values turn model output from a string to parse into a contract to rely on. And because all three are pure or deterministic, they test cleanly offline — which is exactly what you want when the thing you're tuning changes every day.

The full implementation lives in our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo).