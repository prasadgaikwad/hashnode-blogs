---
title: "Getting Started with LangChain4j: Building Your First AI-Powered Java Application"
seoTitle: "LangChain4j: Build Your First AI Java App"
seoDescription: "Learn how to build an AI-powered Java application using LangChain4j and Spring Boot for interactive, language-driven solutions"
datePublished: 2025-11-04T04:14:03.898Z
cuid: cmhk213lm000702lefvz241nd
slug: getting-started-with-langchain4j-building-your-first-ai-powered-java-application
tags: ai, java, springboot, langchain4j

---

In this blog post, we'll walk through creating your first LangChain4j application using Spring Boot. We'll build a simple interactive command-line application that demonstrates the basic setup and usage of LangChain4j in a Java environment.

## **What is LangChain4j?**

LangChain4j is a powerful Java framework designed to simplify the development of applications powered by Large Language Models (LLMs). It provides a comprehensive set of tools and abstractions that make it easier to build sophisticated AI-powered applications.

## **Project Setup**

### **Prerequisites**

* Java 17 or higher
    
* Maven
    
* Your favorite IDE (IntelliJ IDEA, Eclipse, or VS Code)
    
* An API key from your chosen LLM provider (e.g., OpenAI)
    

### **Step 1: Create a Spring Boot Project**

Start by creating a new Spring Boot project using Spring Initializr or your IDE. We'll need the following dependencies:

* Spring Boot Starter
    
* LangChain4j BOM (Bill of Materials)
    

Here's our `pom.xml` configuration:

```xml
<!-- Parent Spring Boot -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.7</version>
</parent>

<!-- LangChain4j BOM for version management -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-bom</artifactId>
            <version>1.8.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    
    <!-- LangChain4j OpenAI Integration -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-open-ai</artifactId>
    </dependency>
</dependencies>
```

### **Step 2: Configuration**

Create or update your `.env` file: (make sure to add this .env file to your .gitignore to keep your API key secure)

```plaintext
# OpenAI API Configuration
OPENAI_API_KEY=your-api-key-here
```

### **Step 3: Creating the Application**

We'll create a simple command-line application that:

1. Asks for user's question
    
2. Answers concisely using the OpenAI GPT-4o-mini model
    
3. Continues interaction until they type 'quit'
    

The implementation uses Spring's CommandLineRunner for the interactive loop.

## **Understanding the Code**

Let's break down the key components:

1. **The Main Application Class**
    

```java

@SpringBootApplication
public class LangChain4jDemoApplication implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(LangChain4jDemoApplication.class, args);
    }

    /**
     * A simple command-line application that interacts with the user,
     * takes a question as input, and provides a concise answer using
     * the OpenAI GPT-4o-mini model.
     */
    @Override
    public void run(String... args) {
        ChatModel model = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o-mini")
                .build();

        try (Scanner scanner = new Scanner(System.in)) {
            while (true) {
                System.out.print("Please enter your question (type 'quit' to exit): ");
                String question = scanner.nextLine();

                if ("quit".equalsIgnoreCase(question)) {
                    System.out.println("Goodbye!");
                    break;
                }

                String response = model.chat("You are an helpful assistant. " +
                        "Answer this question in very concise way, only in 2 sentences maximum. Question: " + question);
                System.out.println("Answer: " + response);
                System.out.println(); // Add a blank line for better readability
            }
        }
    }
}
```

## **Running the Application**

To run the application:

1. Ensure you have set your API key in `.env` file
    
2. Run the following command:
    
    ```plaintext
    ./mvnw spring-boot:run
    ```
    

## **Next Steps**

This basic setup provides a foundation for exploring more advanced LangChain4j features:

1. **Adding LLM Integration**
    
    * Implement chat completions
        
    * Add streaming responses
        
    * Experiment with different models
        
2. **Implementing Memory**
    
    * Add conversation history
        
    * Implement different memory types
        
3. **Creating Chains**
    
    * Build processing pipelines
        
    * Add prompt templates
        

## **Resources**

For more information and advanced features, check out:

* [Official LangChain4j Documentation](https://docs.langchain4j.dev/)
    
* [GitHub Examples Repository](https://github.com/langchain4j/langchain4j-examples)
    

## **Conclusion**

This simple example demonstrates how easy it is to get started with LangChain4j. The framework's integration with Spring Boot makes it particularly attractive for Java developers looking to build AI-powered applications.

Stay tuned for more blog posts where we'll explore advanced features like:

* Working with different LLM providers
    
* Implementing RAG (Retrieval Augmented Generation)
    
* Building custom agents and tools
    
* Creating sophisticated conversation chains
    

Remember to check out our [GitHub repository](https://github.com/prasadgaikwad/langchain4j-demo) for the complete source code and future updates!